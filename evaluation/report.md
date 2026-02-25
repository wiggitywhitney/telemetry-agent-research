# Evaluation Report: First-Draft Telemetry Agent

**Date**: 2026-02-25
**PRD**: #2 (Run & Evaluate Reference Implementation)
**Evaluation target**: commit-story-v2-eval repo ([PR #1](https://github.com/wiggitywhitney/commit-story-v2-eval/pull/1))

## Background

### What we're building

We are building an AI agent that automatically adds observability instrumentation to JavaScript codebases. When you point it at your source code, it analyzes each file, decides which functions need telemetry (and which don't), and adds [OpenTelemetry](https://opentelemetry.io/) tracing code — the spans, attributes, and error recording that let you see what your application is doing at runtime.

The agent is designed for a KubeCon demo where it instruments [commit-story-v2](https://github.com/wiggitywhitney/commit-story-v2), a JavaScript application that auto-generates developer journal entries from git commits.

### What the spec describes

Whitney Lee wrote a detailed specification for this agent. The spec covers:

- A **coordinator** that discovers files and orchestrates instrumentation
- **Per-file agents** (fresh LLM instances) that analyze each file and add telemetry
- A **validation chain** that checks each file compiles, has no variable conflicts, passes lint, and conforms to the project's telemetry schema
- A **fix loop** that retries failed files up to 3 times with different strategies
- **CLI and MCP interfaces** for running the agent
- A **git workflow** that creates branches, per-file commits, and a pull request with a detailed review document

### What happened

The first-draft implementation was built from this spec by an autonomous code-building system. This evaluation ran that implementation against a test codebase to see what worked, what didn't, and what we should do differently next time.

### What we learned (the short version)

The agent produces good telemetry code. When it instruments a function, the spans are correctly placed, errors are properly recorded, and the patterns follow OpenTelemetry best practices. An automated code reviewer ([CodeRabbit](https://coderabbit.ai/)) found zero issues with the instrumentation itself.

Everything *around* the instrumentation is broken. The CLI doesn't work. The validation chain rejects the agent's own output. Features described in the spec — the fix loop, progress feedback, git workflow, schema validation — aren't wired up. It took 7 failed runs and 3 manual patches before the agent produced any output at all.

The biggest discovery: the spec was written for TypeScript, but the target application (commit-story-v2) is JavaScript. This is the most important change for the next iteration.

---

## What the Agent Produced

### Test setup

We couldn't run the agent directly against commit-story-v2 because the agent only discovers `**/*.ts` files and commit-story-v2 is entirely JavaScript (21 `.js` source files). To get any evaluation data, we copied 4 TypeScript files from a different project ([cluster-whisperer](https://github.com/wiggitywhitney/cluster-whisperer), a Kubernetes pipeline tool) into the eval repo as test targets.

This workaround let us evaluate the agent's instrumentation quality, but it also introduced a mismatch: the eval repo's telemetry schema defines attributes for commit-story's journaling features (`commit_story.commit.*`, `commit_story.journal.*`), while the test files are Kubernetes pipeline code. This matters for the schema scoring — see [Rubric Learnings](#rubric-learnings).

### What it instrumented

The agent processed 7 files. It correctly instrumented 4 and correctly skipped 3:

| File | Decision | Why |
|------|----------|-----|
| `pipeline/discovery.ts` | Instrumented | Entry points and external kubectl calls |
| `pipeline/inference.ts` | Instrumented | Entry points and LLM calls |
| `tools/format-results.ts` | Instrumented | Public API function |
| `tools/kubectl-get.ts` | Instrumented | External kubectl call |
| `pipeline/types.ts` | Skipped | Type definitions only — no runtime code |
| `tracing/index.ts` | Skipped | Telemetry infrastructure — not application code |
| `utils/kubectl.ts` | Skipped | Already has OpenTelemetry instrumentation |

Across the 4 instrumented files, the agent added 7 spans and 22 attributes.

### What it got right

**Span placement follows the spec's priority hierarchy.** External calls (kubectl subprocesses, LLM API calls) got spans first. Service entry points (`discoverResources`, `inferCapabilities`, etc.) got spans next. Internal helper functions (11 of them, including `parseApiResources`, `extractGroup`, `formatMetadata`) were all correctly left alone.

**Error handling is textbook.** Every span uses the same correct pattern: try/catch/finally with `span.recordException()` and `span.setStatus()` in the catch block, and `span.end()` in the finally block. This means spans always get closed, even when errors occur.

**Skip decisions show real understanding.** The agent didn't just skip files without explanation — its notes for `utils/kubectl.ts` describe analyzing the existing spans, error handling, and attributes before concluding no changes were needed. For `types.ts`, it correctly identified that type-only files have no runtime behavior to instrument.

**Nested spans show hierarchy awareness.** In `inference.ts`, the agent created a child `llm.invoke` span inside the parent `inference.infer_capability` span, correctly representing the LLM call as a sub-operation of the inference pipeline.

**Conditional attributes show attention to data quality.** In `kubectl-get.ts`, the agent only sets `kubectl.namespace` and `kubectl.name` when the values are defined, avoiding spurious `undefined` attribute values. This is a subtle quality signal.

### Rubric scores

We scored the agent's output against the evaluation rubric from PRD #1. The rubric has 4 gate checks (binary pass/fail prerequisites) and 24 quality rules across 6 dimensions.

**Gate checks** (must pass before quality scoring is meaningful):

| Gate | Result | What it checks |
|------|--------|----------------|
| NDS-001: Code compiles | Conditional pass (not independently verified) | Agent didn't break the build |
| NDS-002: Tests pass | Not verified (no tests exist for the modified files) | Agent didn't break existing behavior |
| NDS-003: Only instrumentation lines changed | **Pass** | Agent didn't modify business logic |
| API-001: Only API imports | **Pass** | Agent used the right dependency model |

Two gates couldn't be verified — more on this in [Developer Experience](#developer-experience-observations).

**Quality dimensions:**

| Dimension | What it measures | Pass Rate | Summary |
|-----------|-----------------|-----------|---------|
| Non-Destructiveness | Agent preserved existing code behavior | 2/2 (100%) | All function signatures and error handling intact |
| Coverage | Agent instrumented the right things | 4/6 (67%) | Good span placement; missed schema attributes and auto-instrumentation |
| Restraint | Agent avoided over-instrumentation | 3/4 (75%) | One internal function instrumented (debatable — it does I/O) |
| API-Only Dependency | Agent used correct OpenTelemetry packages | 3/3 (100%) | Only `@opentelemetry/api` — no SDK or vendor packages |
| Schema Fidelity | Instrumentation matches the project's telemetry schema | 1/3 (33%) | Low score caused by test setup mismatch — see [Rubric Learnings](#5-rubric-learnings) |
| Code Quality | Instrumentation code is correct and maintainable | 6/6 (100%) | Perfect OTel patterns across all files |

**Overall: 19/24 rules passed (79%).** Strongest dimensions: Code Quality (100%), Non-Destructiveness (100%), API Dependency (100%). Weakest: Schema Fidelity (33%), but this is misleading — explained below.

### CodeRabbit review

CodeRabbit, an automated code review tool, reviewed the [pull request](https://github.com/wiggitywhitney/commit-story-v2-eval/pull/1) containing the agent's changes. It left 8 actionable comments and 2 nitpicks. **None of them were about the telemetry code.** Every comment targeted pre-existing issues in the cluster-whisperer source files (like potential secret leaks in kubectl argument handling) or evaluation setup artifacts (like dependency placement in package.json).

This is a strong signal: an automated reviewer with no knowledge of our evaluation found nothing wrong with the instrumentation the agent added.

---

## What Broke and Why

### Features described in the spec but not connected

The spec describes a complete system. The first-draft implementation has the pieces, but many aren't wired together. The pattern is consistent: the library code exists and passes unit tests, but the interface layer never calls it.

| Feature | What the spec says | What the build does |
|---------|-------------------|---------------------|
| CLI `init` command | Bootstrap configuration for a new project | Parses arguments, exits successfully, does nothing |
| CLI `instrument` command | Run the agent against a codebase | Parses arguments, exits successfully, does nothing |
| Progress feedback | Show which file is being processed, overall progress | MCP shows "Running..." for the entire duration (84 seconds) with no updates |
| PR description | Detailed markdown document with per-file results, schema diff, token usage | Document is assembled inside the agent, then thrown away by the MCP handler |
| Git workflow | Feature branch, per-file commits, pull request creation | Files modified in-place on whatever branch you're on; no branch, commits, or PR |

The only way to run the agent is through the MCP (Model Context Protocol) server interface, and even that required creating an entry point file that didn't exist.

### Broken validation chain

The spec describes a validation chain that checks each instrumented file in sequence: did the code compile? Are there variable name conflicts? Does it pass lint? Does it match the telemetry schema?

The chain's *architecture* is well-designed — it short-circuits on first failure and provides clear diagnostic messages. But three of the four checkers have bugs that reject valid output:

1. **The syntax checker uses an in-memory filesystem** that can't see `node_modules/`. Since the agent adds `import { trace } from '@opentelemetry/api'` to every file, and the in-memory filesystem can't find that package, every single file fails syntax checking. This is a blocking bug — no file can ever pass validation in a real project.

2. **The shadowing checker bans common variable names** instead of detecting actual conflicts. It maintains a hardcoded list of banned names (`tracer`, `span`). The agent generates `const tracer = trace.getTracer(...)` — the standard OpenTelemetry pattern — and the checker immediately rejects it. The agent's own output is incompatible with its own validator.

3. **The lint checker rejects non-Prettier formatting.** LLMs don't reliably produce output that matches Prettier's exact formatting rules (whitespace, trailing commas, line wrapping). Every file fails even when the instrumentation logic is correct.

We had to manually patch the implementation to bypass validation before the agent could produce any output at all.

### Missing features

| Feature | What the spec says | What the build does |
|---------|-------------------|---------------------|
| Fix loop | 3-attempt strategy: initial generation → multi-turn fixing with validator feedback → fresh regeneration | Single attempt; gives up on first failure |
| Schema validation | Validate attributes against the project's Weaver telemetry schema | Explicitly disabled (`runWeaver: false`) |
| Schema extensions | Register new attributes in the schema when instrumenting new domains | Never creates schema entries; all files report empty schema extensions |
| Prerequisite checks | Verify project readiness before running (package.json, OTel API, TypeScript compiler, tsconfig) | Checks package.json and OTel API; skips TypeScript compiler and tsconfig |
| Early abort | Stop after repeated identical failures | Processes every file sequentially even when all fail with the same error |

### Partial implementations

**Already-instrumented detection** works sometimes. The agent correctly identified `utils/kubectl.ts` as already instrumented (it has explicit `tracer.startActiveSpan()` calls) and skipped it. But `kubectl-get.ts`, which also has OpenTelemetry code imported from a shared utility module, was not detected — the agent added a second set of tracing on top of it. The detection is pattern-dependent: it catches inline tracing but not imported tracer patterns.

---

## Developer Experience Observations

Running this agent as a user — not reading the code, just trying to get it to work — was a frustrating experience. These observations are separate from the rubric scores and speak to the quality of the tool as a product.

### Silent failures hide everything

The agent fails silently in multiple scenarios:

- **Zero files found**: When the agent discovers no files to instrument (because the target is a JavaScript project and the agent only looks for TypeScript), it exits successfully with no output and no warning. The user gets no feedback that anything went wrong.
- **File path instead of directory**: Passing a specific file path (`instrument src/file.ts`) returns empty results with no error. The agent expects a directory path, but the CLI description says it accepts "directory or glob."
- **Unwired CLI commands**: Both `init` and `instrument` CLI commands exit 0 (success) and produce nothing. A user running these commands would reasonably assume the tool is broken, with no indication of what to try instead.

These silent failures compounded during evaluation. On the first several attempts, it wasn't clear whether the agent was failing, misconfigured, or working correctly on zero files.

### Wasted cost with no signal

Across 8 evaluation runs, the agent consumed approximately 180,000 tokens (~$5.50-6.50 at Sonnet pricing). Runs 1 through 7 produced zero usable output — $3.50-4.50 spent before we even knew the agent could generate valid instrumentation. Run 8, after manually patching the implementation, produced all successful results for ~$1.50-2.00.

The cost problem was amplified by the lack of early abort. When all 4 files in a run failed with the same error (missing tsconfig.json), the agent still processed each file sequentially, making full LLM calls for each one. A heuristic like "stop after 3 files fail with the same error" would have saved most of those wasted tokens.

The cost ceiling tool (`get-cost-ceiling`) reports token counts (320,000 max) but not dollar estimates. Users are left to calculate costs themselves from token counts and current model pricing — not practical for budget decisions.

### Three patches to get any output

To produce even a single instrumented file, we had to:

1. **Create an MCP entry point** (`bin/telemetry-agent-mcp.js`) — the MCP server's `startMcpServer()` function was exported but never called from any executable
2. **Fix the syntax checker** to use the real filesystem instead of an in-memory filesystem, so `@opentelemetry/api` imports could resolve
3. **Bypass the entire validation chain** — because even after fixing the filesystem, the shadowing checker and lint checker rejected every file

Without these patches, the agent had a 100% failure rate across 7 runs. With them, it had a 100% success rate on run 8. The instrumentation quality was good all along — it just couldn't get past its own validators.

### 332 tests pass, nothing works

The first-draft implementation has 332 passing unit tests. Every real-world execution path failed. This is the clearest indicator that the testing strategy missed the most important thing: does the tool actually work end-to-end?

The unit tests verify individual components in isolation — prompt construction, response parsing, validation logic. None of them test:
- Does the CLI actually call the coordinator when you run `instrument`?
- Does the validation chain accept the agent's own output on a real filesystem?
- Can you instrument a single file and get a compilable result?

One end-to-end test — "instrument a real file, verify it compiles" — would have caught most of the critical bugs (unwired CLI, in-memory filesystem, shadowing ban, missing prerequisites) in a single run.

---

## Rubric Learnings

Scoring the agent's output against the rubric revealed several issues with the rubric itself and the evaluation setup.

### Schema scores reflect the test setup, not the agent

The Schema Fidelity dimension scored 33% (1 of 3 applicable rules passed), making it the weakest dimension by far. But this score doesn't reflect poor agent behavior — it reflects a mismatch in the test setup.

Here's what happened: the eval repo's telemetry schema defines attributes for commit-story's features — things like `commit_story.commit.sha`, `commit_story.journal.entry`, `commit_story.context.diff`. But the files the agent actually instrumented were Kubernetes pipeline code from a completely different project. The agent was asked to match a schema that describes journaling software while instrumenting code that discovers Kubernetes resources.

The agent handled this correctly — it didn't force-fit journaling attributes onto Kubernetes code. Instead, it created sensible domain-specific attributes like `discovery.resource_count` and `kubectl.resource`. But because none of those match the commit-story schema, the rubric scored them as failures.

For future evaluations, the schema can only be meaningfully scored when the instrumented code matches the schema's domain. That means either (a) instrumenting the actual commit-story JavaScript files (which requires the agent to support JavaScript first), or (b) creating a Kubernetes-specific telemetry schema for the test files.

### One rule punishes good judgment

RST-004 (No Spans on Internal Implementation Details) says unexported functions shouldn't get spans — they're internal details, not public API. The agent added a span to `getCrdNames`, an unexported function, which the rubric flagged as a violation.

But `getCrdNames` makes an external subprocess call to kubectl. Observability on I/O boundaries has genuine value regardless of whether the function is exported. The rubric's heuristic — "exported = public API = worth instrumenting" — works well for pure logic functions but breaks down for internal functions that talk to external systems.

This suggests the rubric should have an exception: unexported functions that perform I/O or external calls are reasonable instrumentation targets.

### One anti-pattern shows awareness without compliance

The agent added a manual span around a LangChain `ChatAnthropic.invoke()` call. LangChain calls have a dedicated auto-instrumentation library (`@traceloop/instrumentation-langchain`) that would handle this automatically — so the rubric scored it as a COV-006 failure (prefer auto-instrumentation over manual spans).

However, the agent's notes explicitly recommended that library and included a nuanced observation about auto vs. manual instrumentation tradeoffs. The agent *knew* the right answer and explained it, but still added the manual span. This is arguably better than being unaware of the principle, and the manual span provides more specific attributes than the auto-instrumentation library would.

### Gaps the rubric doesn't cover

Two quality signals emerged that the current rubric doesn't measure:

1. **Tracer naming consistency.** The agent used different tracer names across files: `'commit-story'`, `'inference'`, `'commit-story.tools.format-results'`, `'kubectl-get'`. A consistent naming convention would improve trace analysis. The rubric checks that a tracer name *exists* (CDQ-002) but not that names follow a consistent pattern.

2. **Conditional attribute setting.** The agent's practice of only setting attributes when values are defined (avoiding `undefined` values) is a positive quality signal. The rubric has no rule for this, but it matters for data quality in observability backends.

---

## Implications for PRD #3

PRD #3 (Spec Synthesis) takes the findings from this evaluation and produces a revised specification. Based on what we learned, PRD #3 has four major workstreams.

### 1. Spec changes

Only two actual changes to the specification emerged from evaluation. Everything else is a build quality issue, not a spec gap.

**JavaScript support is the critical change.** The spec currently targets TypeScript (`**/*.ts` file discovery, TypeScript-specific validation). The demo target — commit-story-v2 — is JavaScript. This changes file discovery patterns, validation approach (no type checking needed, but `node --check` for syntax), and likely simplifies the agent's task (no type annotations to preserve).

**Model configurability needs more prominence.** The agent model is already configurable in the spec's config schema, but cost management and model selection deserve more emphasis given the real-world budget impact we saw ($5.50+ for a 4-file evaluation, most of it wasted).

### 2. Builder success criteria

The first-draft implementation had 332 passing tests and a 0% real-world success rate. The next build needs acceptance criteria that go beyond unit tests.

These criteria are for the autonomous system that *builds* the agent, not for the agent itself. They answer: "How do we know the builder actually implemented what the spec describes?"

**End-to-end smoke test.** Instrument one real file in a real project. Verify it still runs without errors. This single test would have caught the unwired CLI, the in-memory filesystem bug, the shadowing ban, and the missing prerequisite checks — 6 of the 25 findings — in one run.

**Interface wiring verification.** Every CLI command and MCP tool must invoke the coordinator, not just parse arguments. A test that calls `instrument` and asserts the coordinator was invoked would have immediately caught the two most critical bugs (Findings 1 and 2).

**Validation chain integration test.** Instrument a file, run the full validation chain, verify the output passes. This would have caught the in-memory filesystem, shadowing ban, and lint formatting issues — the three bugs that created a 100% validation failure rate.

**Progress and feedback verification.** Running the agent must produce visible output at every stage: file discovery results, per-file progress, per-file results, and a summary. Silent success and silent failure are both unacceptable.

**Cost tracking verification.** The agent must report actual cost (not just token counts) and the cost ceiling tool must include dollar estimates.

### 3. Case for phased builds

The first-draft implementation tried to build everything at once from a large spec. Many features exist as library code with passing tests but aren't connected to the interfaces users actually interact with. This suggests the spec should be broken into sequential build phases, where each phase has its own acceptance gate.

A rough phasing based on what the evaluation revealed:

| Phase | What gets built | Acceptance gate |
|-------|----------------|-----------------|
| 1. Core loop | File discovery → LLM instrumentation → write output | Instrument one file, output compiles |
| 2. Validation | Syntax check → lint → schema validation | Agent output passes its own validation chain on a real filesystem |
| 3. Fix loop | Multi-turn fixing with validator feedback | Agent retries and fixes a file that initially fails validation |
| 4. CLI | `init` and `instrument` commands wired to coordinator | CLI produces the same results as calling the library directly |
| 5. Git workflow | Branch creation, per-file commits, PR description | Full git workflow produces a reviewable PR |
| 6. Developer experience | Progress feedback, cost estimates, early abort, error messages | User can monitor progress and understand failures without reading source code |

Each phase builds on the previous one. Phase 1 is the minimum viable agent — it can instrument a file. Phase 2 ensures the output is validated. Phase 3 makes the agent self-correcting. Phases 4-6 make it usable as a product.

The key insight: if the first-draft implementation had been built phase by phase with these gates, the unwired CLI (Phase 4 failure) could never have masked the validation chain bugs (Phase 2 failure), because Phase 2 would have been verified before Phase 4 was started.

### 4. Rubric as build-time acceptance criteria

The evaluation rubric was designed as a post-hoc scoring tool — run the agent, then grade the output. But the rubric can serve double duty as build-time acceptance criteria.

The rubric's output format is already machine-readable:
```text
COV-002 | fail | src/api-client.ts:42 | Outbound HTTP call to /users has no enclosing span
```

This means the rubric evaluation can be integrated into the agent's own fix loop: after instrumenting a file, run rubric checks, feed failures back as guidance for the next attempt. The rubric transforms from "how we score after the fact" into "how the agent knows it's not done yet."

For the builder, specific rubric rules map to specific build phases:
- **Phase 1 gates**: NDS-001 (compiles), NDS-003 (only instrumentation lines changed), CDQ-001 (spans closed)
- **Phase 2 gates**: Full NDS dimension, API-001 (correct imports)
- **Phase 3 gates**: COV dimension (right things instrumented), RST dimension (wrong things left alone)
- **Phase 5 gates**: Full rubric pass on a real-world codebase

---

## Detailed Run Data

All raw evaluation data is available in the companion documents:

- **[evaluation/run-1/README.md](run-1/README.md)** — 25 detailed findings, setup procedure, patches applied, token usage, CodeRabbit review, instrumentation quality analysis
- **[evaluation/run-1/rubric-scores.md](run-1/rubric-scores.md)** — per-rule scores across all 6 dimensions, per-file breakdown, gate check results
- **[evaluation/baseline/](baseline/)** — test results (320 passing), coverage (83.19%), git state, run procedures

---

## Summary

| Area | Verdict | Detail |
|------|---------|--------|
| Instrumentation quality | Good | Correct span placement, proper error handling, thoughtful skip decisions, standard OTel patterns |
| Spec accuracy | Sound | The spec describes the right system; failures are build quality, not spec gaps |
| Build quality | Poor | Most spec features exist as library code with tests but aren't wired to interfaces |
| Developer experience | Poor | Silent failures, no progress feedback, wasted cost, three patches needed |
| Critical spec change | TypeScript → JavaScript | The demo target is JavaScript; the spec targets TypeScript |
| Primary PRD #3 task | Builder success criteria | Ensure the next build actually implements what the spec describes, phase by phase |
