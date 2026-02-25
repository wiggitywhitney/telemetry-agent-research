# Evaluation Patterns: What Worked, What Didn't, What Was Missing

**Date**: 2026-02-25
**Source**: PRD #2 evaluation (25 findings, 24 rubric rules, 8 runs)
**Purpose**: Categorize evaluation findings into themes to feed PRD #3 milestones (spec updates, architectural recommendations, design document)

## How to Read This Document

Each category contains three sections:

- **What worked** — strengths to preserve in the next iteration
- **What didn't work** — weaknesses to address
- **What was missing** — capabilities the spec describes but the build never exercised

Every claim references specific findings (F1–F25) from `run-1/README.md` and rubric rules from `run-1/rubric-scores.md`. The [cross-cutting themes](#cross-cutting-themes) section captures patterns that span multiple categories.

---

## Architecture

How the system is structured: coordinator pattern, interface layers, validation pipeline, git workflow.

### What Worked

**The coordinator + per-file agent pattern is sound.** The coordinator discovers files, dispatches them to fresh LLM instances, and collects results. This separation means each file gets a clean context window — no cross-file confusion, no prompt bloat from accumulated state. When the agent finally produced output (run 8), all 7 files were processed correctly: 4 instrumented, 3 skipped. The pattern works. (F24)

**The validation chain architecture is well-designed.** The chain runs checkers in sequence (elision → syntax → shadowing → lint → weaver), short-circuits on first failure, and produces clear diagnostic messages with step name, error count, line numbers, and fix suggestions. The orchestration logic is correct — the problems are in the individual checkers, not the chain itself. (F18)

**Callback hooks exist for progress reporting.** The coordinator defines `onFileStart` and `onFileComplete` hooks, which is the right abstraction — interface layers subscribe to events rather than polling. The architecture supports progress feedback even though no interface wires the hooks. (F7)

**Non-destructive rollback on failure.** When validation fails, the agent reverts its modifications. After 7 failed runs, `git diff` showed no changes to source files. The target codebase was never left in a corrupted state. (F12)

### What Didn't Work

**CLI commands are argument parsers with no handlers.** Both `init` and `instrument` are registered in yargs with full argument definitions but no `.handler()` callbacks. They parse arguments, exit 0, and produce no output or side effects. The CLI is a stub. (F1, F2)

**The MCP server is the only wired interface — and it was missing an entry point.** `runCoordinator()` is imported and called only from the MCP server handler. `startMcpServer()` is exported but never called from any bin script. We had to create `bin/telemetry-agent-mcp.js` to launch it. (F6, Patch 1)

**Progress hooks are never wired.** The MCP handler calls the coordinator but doesn't subscribe to `onFileStart`/`onFileComplete`. During the 84-second run, the user sees only "Running..." with no indication of which file is being processed or how many remain. (F7)

**The PR description is assembled and thrown away.** The coordinator builds a detailed markdown document at step 7 (`assemblePrDescription()`) with summary stats, per-file results, schema diff, agent notes, and token usage. The MCP handler calls `formatJsonOutput()` instead — the primary review artifact is computed and discarded. (F22)

**The git workflow is entirely unwired.** `branch-manager.ts` has git operations (checkout, commit, branch management), but no interface calls them. The coordinator processes files and returns results; creating branches, per-file commits, and PRs is deferred to the interface layer, and neither interface does it. Files are modified in-place on whatever branch the user is on. (F23)

### What Was Missing

**No entry point for the MCP server.** The implementation exports `startMcpServer()` but has no bin script that calls it. This is a wiring gap — the function exists, the executable doesn't. (F6, Patch 1)

**No handler wiring for CLI commands.** The gap is specific: yargs command definitions exist, handler callbacks don't. The library code that should back the handlers (prerequisite checker, project detector, config writer, coordinator) all exist and pass unit tests. The last mile — connecting CLI arguments to library calls — was never built. (F1, F2)

---

## Agent Behavior

How the LLM agent makes decisions: what to instrument, what to skip, how to handle edge cases.

### What Worked

**Span placement follows the spec's priority hierarchy.** External calls (kubectl subprocesses, LLM API calls) got spans first. Service entry points got spans next. 11 internal helper functions were all correctly left uninstrumented. The agent demonstrates real understanding of the instrumentation priority order. (F24)

**Skip decisions show genuine analysis.** The agent didn't skip files mechanically — it analyzed each one:
- `types.ts`: identified as type-only with no runtime behavior
- `tracing/index.ts`: identified as telemetry infrastructure, not application code
- `utils/kubectl.ts`: analyzed existing spans, error handling, and attributes before concluding no changes needed

Each skip includes detailed notes explaining the reasoning. (F24)

**Nested spans show hierarchy awareness.** In `inference.ts`, the agent created a child `llm.invoke` span inside the parent `inference.infer_capability` span, correctly modeling the LLM call as a sub-operation of the inference pipeline. (F24)

**Conditional attributes show attention to data quality.** In `kubectl-get.ts`, the agent only sets `kubectl.namespace` and `kubectl.name` when values are defined, avoiding spurious `undefined` attribute values. This is a subtle quality signal — the rubric doesn't have a rule for it, but it matters for observability backend data quality. (F24)

**Library recommendations show awareness beyond the current task.** The agent recommended `@traceloop/instrumentation-langchain` for LangChain calls, with a nuanced note about auto-instrumentation vs. manual span tradeoffs. The agent understood the ecosystem context, not just the immediate file. (F24)

### What Didn't Work

**Already-instrumented detection is pattern-dependent.** `utils/kubectl.ts` (explicit `tracer.startActiveSpan()` calls) was correctly detected and skipped. `kubectl-get.ts` (OTel imported from a shared utility module) was not detected — the agent added a second layer of tracing on top of existing instrumentation. Detection catches inline tracer patterns but misses imported tracer factories. (F15, F25)

**No early abort on repeated identical failures.** When all 4 files failed with the same error (missing tsconfig.json), the agent processed each one sequentially with full LLM calls. A heuristic like "abort after N consecutive files fail with the same error" would save tokens and surface systemic issues faster. The coordinator has abort mechanics (exit code 2) but doesn't detect failure patterns. (F10)

**COV-006: awareness without compliance.** The agent added a manual span around a LangChain `ChatAnthropic.invoke()` call, which is coverable by auto-instrumentation. The agent's notes explicitly recommended the auto-instrumentation library — it knew the right answer and explained it, then did the opposite. This is arguably better than being unaware, but the manual span still violates the auto-instrumentation preference principle. (COV-006, F24)

### What Was Missing

**The fix loop was never implemented.** The spec describes a 3-attempt hybrid strategy: initial generation → multi-turn fixing with validator feedback → fresh regeneration. The agent-bridge explicitly states this is a "single-attempt processor" with the fix loop deferred to a later phase. When the shadowing checker rejected `const tracer = ...` and suggested `otelTracer`, the agent had no mechanism to retry with that feedback. (F16)

**Schema extensions were never created.** Every file reported `schema_extensions: []`. The agent created sensible custom attributes (`discovery.resource_count`, `kubectl.resource`, etc.) but never registered them in the Weaver schema registry. The mechanism to create extensions didn't exist — Weaver validation was disabled. (F21)

---

## Schema Handling

How the agent interacts with the Weaver telemetry schema: attribute naming, schema compliance, schema extension.

### What Worked

**The agent correctly avoided force-fitting wrong-domain attributes.** The Weaver registry defines `commit_story.*` attributes for a journaling application. The test files are Kubernetes pipeline code. Rather than cramming `commit_story.commit.sha` onto a kubectl discovery function, the agent created domain-appropriate attributes (`discovery.*`, `kubectl.*`, `llm.*`). This shows the agent respects semantic meaning over mechanical schema compliance. (F21, SCH-004)

**No redundant schema entries.** The agent's attribute namespaces (`discovery.*`, `kubectl.*`, `llm.*`) are semantically disjoint from the registry namespaces (`commit_story.*`). No false matches, no confusion between domains. (SCH-004)

### What Didn't Work

**Weaver schema validation was explicitly disabled.** The agent-bridge sets `runWeaver: false` with the comment "Weaver may not be available in all environments." This means the core value proposition of schema-driven instrumentation — machine-validated attribute correctness — was never exercised. (F17)

**Schema Fidelity rubric scores reflect test setup, not agent quality.** SCH-001 and SCH-002 both failed because no span names or attribute keys match the `commit_story.*` registry. This is expected given the domain mismatch — Kubernetes pipeline code can't match a journaling schema. The 33% SCH dimension score is misleading without this context. (SCH-001, SCH-002)

### What Was Missing

**Schema-driven instrumentation was not exercised at all.** The combination of disabled Weaver validation (F17), domain mismatch in test setup, and no extension mechanism means we have zero data on how the agent handles schema-driven attribute creation. Every attribute name in the output is ad-hoc. We can't evaluate whether the agent would use registry-defined attributes when they exist, create proper extensions when they don't, or produce valid Weaver-compatible schema entries. This is the biggest evaluation gap. (F17, F21)

**No schema diff for reviewers.** Without schema extensions, the PR description (even if it weren't discarded — see F22) would have no schema diff section. Reviewers can't check whether the agent's attribute additions align with project conventions, because the agent treats every attribute as a free-form string. (F21)

---

## Code Quality

The correctness and maintainability of the instrumentation code the agent produces.

### What Worked

**100% Code Quality rubric score (CDQ: 6/6).** Every span is properly closed in a `finally` block. Every tracer is correctly acquired via `trace.getTracer()`. Every error uses the standard `recordException` + `setStatus` pattern. Async context is maintained via `startActiveSpan` callbacks. No expensive computations in attribute values. No unbounded attributes. (CDQ-001 through CDQ-007)

**Textbook OTel error handling pattern.** Every instrumented function uses the same correct structure: `try { ... } catch { recordException; setStatus; throw; } finally { span.end(); }`. Spans always close, errors always record, exceptions always re-throw. This pattern is consistent across all 4 instrumented files and all 7 spans. (CDQ-001, CDQ-003)

**API-only dependency model (API: 3/3).** All files import only from `@opentelemetry/api`. No SDK, exporter, or vendor-specific packages. `package.json` declares `@opentelemetry/api` as a `peerDependency`, which is correct for a distributable package. (API-001 through API-004)

**Non-destructiveness (NDS: 2/2).** All exported function signatures preserved exactly. No pre-existing error handling modified. The agent's changes are additive — wrapping existing logic in spans without altering its behavior. (NDS-004, NDS-005)

**CodeRabbit found zero issues with instrumentation code.** An automated code reviewer with no knowledge of the evaluation found nothing wrong with the telemetry the agent added. All 8 actionable comments and 2 nitpicks targeted pre-existing issues in the source files or evaluation setup artifacts. (CodeRabbit review)

### What Didn't Work

**Inconsistent tracer naming across files.** The agent used different tracer name patterns:
- `'commit-story'` (discovery.ts) — project name
- `'inference'` (inference.ts) — module name
- `'commit-story.tools.format-results'` (format-results.ts) — fully qualified path
- `'kubectl-get'` (kubectl-get.ts) — file name

CDQ-002 only checks that a name argument exists, not naming consistency. A consistent convention (all using package name, or all using module-relative paths) would improve trace analysis. This is a rubric gap and a quality observation. (CDQ-002)

### What Was Missing

Nothing significant. Code quality is the agent's strongest dimension. The instrumentation code itself is correct and maintainable — the problems lie in everything around it (validation, interfaces, schema integration).

---

## Coverage Decisions

What the agent chose to instrument, what it chose to skip, and whether those decisions were correct.

### What Worked

**All entry points instrumented (COV-001: 5/5).** `discoverResources`, `inferCapability`, `inferCapabilities`, `formatSearchResults`, and `kubectlGet` all received spans. The agent identified public API surfaces correctly. (COV-001)

**All outbound calls covered (COV-002: pass).** External subprocess calls (kubectl) flow through already-instrumented functions. The LLM call has a dedicated `llm.invoke` span. No outbound call site lacks a span. (COV-002)

**Error recording on every span (COV-003: 7/7).** All 7 spans have `recordException` + `setStatus` in catch blocks. Error visibility is complete. (COV-003)

**All async and I/O functions covered (COV-004: pass).** Every async function and I/O function has a span. No async gap. (COV-004)

**Correct restraint on internal helpers (RST-001: 0 violations).** 11 internal utility functions were correctly left uninstrumented. The agent didn't over-instrument — it targeted boundaries and entry points, not implementation details. (RST-001)

**Correct skip on already-instrumented code (RST-005: pass).** `utils/kubectl.ts` (explicit inline tracing) was identified and skipped. `tracing/index.ts` (infrastructure) was also skipped. (RST-005)

### What Didn't Work

**Zero registry-defined attributes (COV-005: 0/21 fail).** The agent created 21 ad-hoc attributes but used zero attributes from the Weaver registry. This is expected given the domain mismatch (the registry defines `commit_story.*`, the code is Kubernetes pipeline), but it means we have no data on whether the agent would use registry attributes when they match. (COV-005)

**Manual span where auto-instrumentation exists (COV-006: fail).** The LangChain `ChatAnthropic.invoke()` call in `inference.ts` could be covered by `@traceloop/instrumentation-langchain`. The agent's manual span provides more specific attributes, but the principle of preferring auto-instrumentation was violated. (COV-006)

**One debatable RST-004 violation.** `getCrdNames` is unexported (internal) but performs I/O (kubectl subprocess call). The rubric says unexported functions shouldn't get spans. But observability on I/O boundaries has genuine value regardless of export status. This exposes a tension in the rubric: "unexported = internal" is a good heuristic for pure logic but breaks down for internal functions that talk to external systems. (RST-004)

### What Was Missing

**No data on auto-instrumentation integration.** The agent knows auto-instrumentation libraries exist (it recommended one), but the spec doesn't define how the agent should interact with them — detect, defer, configure, or ignore? This is a gap in both the spec and the evaluation. (COV-006)

---

## Cross-Cutting Themes

Patterns that span multiple categories and represent the most important takeaways for PRD #3.

### Theme 1: The Spec Is Sound — The Build Didn't Match It

The most important finding across all categories: **the spec describes the right system; the first-draft implementation didn't build what the spec describes.**

Every critical failure traces to a build quality issue, not a spec gap:

| Spec Feature | Spec Status | Build Status |
|---|---|---|
| CLI with init and instrument commands | Fully specified | Arg parsing only, no handlers (F1, F2) |
| 3-attempt hybrid fix strategy | Fully specified | Single attempt, gives up on first failure (F16) |
| Validation chain | Fully specified | Architecture sound, individual checkers broken (F13, F19, F20) |
| Weaver schema validation | Fully specified | Explicitly disabled (F17) |
| Progress callbacks | Fully specified | Hooks defined, never wired (F7) |
| Prerequisite checks | Fully specified | 2 of 4 checks implemented (F8, F11) |
| Git workflow | Fully specified | No branch, commits, or PR (F23) |
| PR description | Fully specified | Assembled and discarded (F22) |

Only two actual spec changes emerged: JavaScript support (F4) and model configurability emphasis (F9). Everything else is already in the spec.

### Theme 2: 332 Tests Pass, Nothing Works

The first-draft implementation has 332 passing unit tests. Every real-world execution path failed. This is the clearest demonstration that **unit tests alone don't validate a system**.

The tests verify components in isolation — prompt construction, response parsing, validation logic. None test:
- Does the CLI actually call the coordinator?
- Does the validation chain accept the agent's own output on a real filesystem?
- Can you instrument one file and get a compilable result?

One end-to-end test ("instrument a real file, verify it compiles") would have caught findings 1, 2, 4, 8, 11, and 13 in a single run. The testing strategy missed the most important thing: does the tool work?

**Implication for PRD #3**: The spec should include builder success criteria that go beyond unit test counts. Specifically: an e2e smoke test, interface wiring verification, and validation chain integration test.

### Theme 3: Validation Checkers Individually Broken Despite Good Chain Design

The validation chain's architecture is well-designed (F18): sequential execution, short-circuit on failure, clear diagnostics. But three of four checkers have bugs that reject valid output:

1. **Syntax checker**: In-memory filesystem can't resolve `node_modules/`. Every file with `@opentelemetry/api` imports fails. (F13)
2. **Shadowing checker**: Blanket ban on variable names (`tracer`, `span`) instead of scope-based conflict detection. The agent's standard OTel pattern is incompatible with its own validator. (F19)
3. **Lint checker**: Rejects non-Prettier formatting. LLMs don't reliably match Prettier's exact rules. (F20)

The pattern: each checker was built and unit-tested in isolation, verified against synthetic inputs, and never tested against the agent's actual output. An integration test ("instrument a file, run validation, verify it passes") would have caught all three.

**Implication for PRD #3**: Validation checkers need integration tests against real agent output, not just unit tests against synthetic fixtures.

### Theme 4: Silent Failures Compound Into Wasted Cost

Multiple silent failure modes (F5, F10, F14) compounded during evaluation:
- Zero files found → exits 0, no output, no warning
- File path instead of directory → empty results, no error
- Repeated identical failures → processes every file, burning tokens

Across 8 runs, ~$5.50-6.50 was spent. Runs 1-7 produced zero usable output ($3.50-4.50 wasted). The cost ceiling tool reports token counts but not dollar estimates (F9), so cost awareness requires manual calculation.

**Implication for PRD #3**: Developer experience requires explicit attention. Silent failures, missing progress feedback, and absent cost estimates are not edge cases — they're the primary user experience during development and debugging.

### Theme 5: JavaScript Support Is the Critical Spec Change

The spec targets TypeScript (`**/*.ts` file discovery). The demo target — commit-story-v2 — is entirely JavaScript (21 `.js` files). This forced the evaluation to use copied TypeScript files from a different project, which:
- Introduced the schema domain mismatch (Kubernetes pipeline code vs. commit-story schema)
- Made Schema Fidelity scores meaningless (33% reflects the setup, not the agent)
- Prevented testing the agent against its actual intended target

JavaScript support changes file discovery, validation approach (no type checking, but `node --check` for syntax), and likely simplifies the agent's task (no type annotations to preserve). (F4)

**Implication for PRD #3**: This is the one spec change with the highest leverage. With JS support, the next evaluation can run against the real target, and schema fidelity becomes testable.

### Theme 6: The Rubric Can Serve Double Duty

The evaluation rubric was designed as a post-hoc scoring tool. But its output format is already machine-readable:

```text
COV-002 | fail | src/api-client.ts:42 | Outbound HTTP call to /users has no enclosing span
```

This means the rubric can be integrated into the agent's fix loop: after instrumenting a file, run rubric checks, feed failures back as guidance for the next attempt. The rubric transforms from "how we score after the fact" into "how the agent knows it's not done yet."

Specific rubric rules map to build phases:
- **Phase 1 gates**: NDS-001 (compiles), NDS-003 (only instrumentation lines), CDQ-001 (spans closed)
- **Phase 2 gates**: Full NDS dimension, API-001 (correct imports)
- **Phase 3 gates**: COV dimension (right things instrumented), RST dimension (wrong things skipped)
- **Phase 5 gates**: Full rubric pass on a real-world codebase

**Implication for PRD #3**: Ship the rubric alongside the spec as builder acceptance criteria. 332 unit tests passed, but zero rubric criteria would have passed pre-patch. The rubric catches what unit tests don't.

### Theme 7: Rubric Gaps Discovered

Two quality signals emerged that the rubric doesn't measure, and one rule needs refinement:

**Gap: Tracer naming consistency.** CDQ-002 checks that a tracer name argument exists but not that names follow a consistent pattern. The agent used 4 different naming conventions across 4 files. A consistency rule would improve trace analysis. (CDQ-002)

**Gap: Conditional attribute setting.** The agent's practice of only setting attributes when values are defined (avoiding `undefined` values) is a positive quality signal with no corresponding rubric rule. (CDQ-007 covers unbounded attributes but not conditional setting)

**Refinement: RST-004 exception for I/O.** "Unexported = internal = don't instrument" works for pure logic functions but breaks for internal functions that perform I/O or external calls. The rubric should exempt internal functions at I/O boundaries. (RST-004)

---

## Summary Table

| Category | Worked | Didn't Work | Missing |
|---|---|---|---|
| Architecture | Coordinator pattern, validation chain design, callback hooks, non-destructive rollback | CLI unwired, MCP entry point missing, progress hooks unwired, PR description discarded, git workflow unwired | Entry point script, handler wiring |
| Agent Behavior | Priority hierarchy, skip decisions, nested spans, conditional attributes, library recommendations | Pattern-dependent already-instrumented detection, no early abort, awareness without compliance (COV-006) | Fix loop, schema extensions |
| Schema Handling | Correct domain-mismatch handling, no redundant entries | Weaver disabled, misleading SCH scores | Schema-driven instrumentation entirely untested |
| Code Quality | 100% CDQ, textbook OTel, API-only, non-destructive, CodeRabbit clean | Inconsistent tracer naming | — |
| Coverage Decisions | Entry points, outbound calls, error recording, async coverage, correct restraint | No registry attributes, manual over auto-instrumentation, debatable RST-004 | Auto-instrumentation integration guidance |

**Bottom line**: The agent produces good telemetry code. Everything around the instrumentation — validation, interfaces, schema integration, developer experience — is where the work needs to happen. The spec is sound; the next build needs acceptance criteria that go beyond unit tests.
