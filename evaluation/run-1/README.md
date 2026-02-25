# Evaluation Run 1: commit-story-v2-eval

**Date**: 2026-02-24
**Target**: `wiggitywhitney/commit-story-v2-eval` @ `evaluation/run-1` branch
**Baseline SHA**: `9c6e4a1`
**Agent**: first-draft implementation (`telemetry-agent-spec-v3`)
**Result**: 7 files processed, 7 spans added (after 3 patches to unblock validation)

## Summary

The first-draft implementation's CLI commands (`init` and `instrument`) are defined in
yargs but have no handler functions wired. Both commands parse arguments, exit 0, and
produce no output or side effects. The only functional interface is the MCP server.

After discovering the CLI was non-functional, we configured the MCP server interface
and ran instrumentation via MCP tool calls. The agent discovered 7 TypeScript files
(copied from cluster-whisperer for testing, since commit-story-v2 is JavaScript).
Runs 2-7 all failed due to cascading validation issues (missing tsconfig, in-memory FS,
shadowing bans, lint formatting). After three patches (MCP entry point, real filesystem,
validation bypass), run 8 succeeded: 7 files processed, 7 spans added across 4 files,
3 files correctly skipped (type-only, infrastructure, already-instrumented).

## Setup

### Prerequisites satisfied

- Weaver CLI v0.21.2 installed and in PATH
- `@opentelemetry/api` added as peerDependency
- `@opentelemetry/sdk-node` added as dependency
- SDK init file created at `src/telemetry/setup.ts`
- Weaver schema registry present at `telemetry/registry/`
- Reference implementation built successfully (`npm install && npm run build`)
- MCP server entry point created at `bin/telemetry-agent-mcp.js`
- MCP server added to `.mcp.json` with vals for API key injection

### Configuration

Manually created `telemetry-agent.yaml` (minimal, required fields only):

```yaml
schemaPath: ./telemetry/registry
sdkInitFile: ./src/telemetry/setup.ts
```

### Test files

4 uninstrumented TypeScript files copied from cluster-whisperer into
`src/ts-samples/` (commit-story-v2 is JavaScript, agent only discovers `**/*.ts`):

- `pipeline/discovery.ts` (300 lines) — Kubernetes resource discovery
- `pipeline/inference.ts` (268 lines) — LLM inference pipeline
- `tools/format-results.ts` (110 lines) — result formatting utility
- `tools/kubectl-get.ts` (102 lines) — kubectl get wrapper

### Schema mismatch (intentional)

The Weaver schema (`telemetry/registry/`) defines commit-story domain attributes
(`commit_story.commit.*`, `commit_story.context.*`, `commit_story.journal.*`, etc.).
The test files are from cluster-whisperer (Kubernetes pipeline domain). This tests
how the agent handles schema-code domain mismatch: does it extend the schema with
new attributes, or force-fit existing ones?

## Execution

### CLI attempts (both failed — unwired handlers)

```bash
# Attempt 1: init command
telemetry-agent init -o json
# EXIT 0, no output, no config file created

# Attempt 2: instrument command
telemetry-agent instrument src/ --config telemetry-agent.yaml --yes --verbose
# EXIT 0, no output, no file modifications
```

### MCP interface (functional)

```text
# Cost ceiling (no LLM calls)
get-cost-ceiling { path: "src/ts-samples/" }
→ 4 files, 26,290 bytes, 320,000 token ceiling

# Instrumentation (real LLM calls)
instrument { path: "src/ts-samples/" }
→ 4 files processed, all 4 failed validation
→ ~1m 24s runtime
```

## Findings

### Finding 1: CLI `init` command has no handler (critical)

**Observed**: `telemetry-agent init` exits 0 with no output and no config file created.

**Root cause**: The `init` command is registered in yargs (`src/interfaces/cli/cli.ts:42`)
with argument definitions but no `.handler()` callback. The init flow logic exists as
library code (`prerequisite-checker.ts`, `project-detector.ts`, `sdk-init-locator.ts`,
`config-writer.ts`) but is never called from the CLI.

**Impact**: Users cannot bootstrap configuration via the CLI. Config must be created manually.

### Finding 2: CLI `instrument` command has no handler (critical)

**Observed**: `telemetry-agent instrument src/ --config telemetry-agent.yaml --yes --verbose`
exits 0 with no output and no file modifications.

**Root cause**: Same pattern as init — the `instrument` command is registered in yargs
(`src/interfaces/cli/cli.ts:51`) with full argument definitions but no handler.
`runCoordinator()` is only imported and called from the MCP server interface
(`src/interfaces/mcp/server.ts:166`), not from the CLI.

**Impact**: The CLI is non-functional. The only way to run the agent is via MCP.

### Finding 3: `init` command lacks `--verbose` flag

**Observed**: `telemetry-agent init --verbose` fails with "Unknown argument: verbose".

**Details**: The `instrument` command defines `--verbose` and `--debug` flags, but `init`
only supports `--output`. Minor inconsistency — `init` should support verbose output for
prerequisite check results and detection logic.

### Finding 4: JavaScript files not discoverable (by design)

**Observed**: File discovery uses hardcoded `**/*.ts` glob
(`src/coordinator/file-processor.ts:36`).

**Impact**: The agent cannot instrument JavaScript projects. commit-story-v2 is entirely
JavaScript (21 `.js` source files). This is a spec limitation, not a bug — the spec
targets TypeScript — but should be documented as a scope constraint.

### Finding 5: Silent success on zero files

**Observed**: When the agent discovers 0 files, it exits 0 with no output (no warning,
no summary, no "0 files found" message). Also occurs via MCP — JSON output is empty.

**Impact**: Users get no feedback that the agent found nothing to instrument. Should at
minimum log a warning when the discovered file set is empty.

### Finding 6: MCP server is the only wired interface

**Observed**: `runCoordinator()` is imported in `src/interfaces/mcp/server.ts` and called
at line 166. It is not imported anywhere in the CLI code. `startMcpServer()` is exported
but never called from any entry point — we had to create `bin/telemetry-agent-mcp.js`.

**Implication**: The first-draft implementation was built MCP-first. The CLI and GitHub
Action interfaces exist as argument parsers but aren't connected to the core logic. This
is consistent with the implementation having 332 passing tests — the library code works,
the CLI is a stub.

### Finding 7: No progress feedback via MCP

**Observed**: During the ~1m 24s instrumentation run, the MCP tool call showed only
"Running..." with no indication of which file was being processed or overall progress.

**Root cause**: The coordinator has callback hooks (`onFileStart`, `onFileComplete`) but
the MCP server handler (`server.ts:150-213`) never wires them. The MCP protocol supports
progress notifications (`notifications/progress`), but the server doesn't use them.

**Impact**: Poor user experience. Users have no way to gauge whether the agent is working,
stuck, or almost done. Particularly problematic for large codebases where runs take minutes.

### Finding 8: No tsconfig.json prerequisite check

**Observed**: The prerequisite checker (`prerequisite-checker.ts`) validates `package.json`,
`@opentelemetry/api`, and Weaver CLI — but not `tsconfig.json`, even though the
validation step requires it.

**Impact**: The agent burns through API tokens instrumenting files, then fails at
validation. A tsconfig.json check should be added to prerequisites.

### Finding 9: Cost ceiling lacks dollar estimates

**Observed**: The `get-cost-ceiling` tool returns token counts (320,000 max) but no
dollar estimate. The `confirmEstimate` feature in the config is designed for cost
confirmation, but without dollar amounts users can't make informed decisions.

**Impact**: The cost confirmation UX requires users to calculate costs themselves from
token counts and model pricing. The ceiling should include an estimated dollar range.

### Finding 10: No early abort on repeated identical failures

**Observed**: All 4 files failed validation with the same error (missing tsconfig.json).
The agent processed each file sequentially, making full LLM calls for each, despite the
validation environment being broken.

**Impact**: The agent wastes API tokens on files that will inevitably fail. A heuristic
like "abort after N consecutive files fail with the same error" would save money and
surface the systemic issue faster. The coordinator has abort mechanics (exit code 2 =
total failure) but doesn't detect systemic failure patterns.

### Finding 11: No TypeScript compiler prerequisite check

**Observed**: The prerequisite checker validates `package.json`, `@opentelemetry/api`,
and Weaver CLI — but not that `typescript` is installed. The eval repo (a JavaScript
project) didn't have TypeScript installed, causing validation to fail.

**Impact**: Another prerequisite gap. The validation chain requires tsc via ts-morph,
but the agent doesn't verify TypeScript is available before starting.

### Finding 12: Agent reverts on validation failure (positive)

**Observed**: `git diff` shows no changes to source files after 3 failed runs.
The agent rolls back its modifications when validation fails.

**Impact**: Good non-destructiveness behavior. Even when validation is broken,
the target codebase is not left in a corrupted state.

### Finding 13: Validation uses in-memory filesystem — can't resolve node_modules (critical)

**Observed**: The syntax checker (`src/validation/syntax-checker.ts:44`) creates a
ts-morph `Project` with `useInMemoryFileSystem: true`. This means the virtual
filesystem has no access to `node_modules/`. Every file that imports
`@opentelemetry/api` — which is what the agent adds to every instrumented file —
fails with "Cannot find module '@opentelemetry/api'".

**Root cause**: The in-memory FS was likely used for isolation and testability, but
it makes validation fundamentally incompatible with the agent's core output. The
agent adds OTel imports → validation rejects OTel imports → every file fails.

**Impact**: This is a blocking bug. No file can ever pass validation in a real
project because the agent always adds imports that the in-memory FS can't resolve.
Three separate instrumentation runs (burning ~$3-5 in API tokens) failed due to
this single issue before diagnosis.

**Patch applied**: Changed to real filesystem with `overwrite: true` on file
creation so the agent's in-memory content is used but `node_modules` resolves
from disk. See "Patches Applied" section below.

### Finding 14: Agent only accepts directory paths, not file paths

**Observed**: `instrument src/ts-samples/tools/format-results.ts` returned empty
results (0 files, 0 tokens, no errors). Passing the directory path
`src/ts-samples/tools/` correctly discovered both files.

**Root cause**: File discovery (`file-processor.ts:35-42`) resolves the target path
then globs `**/*.ts` relative to it. When the target is a file, the glob runs
against a file path as if it were a directory — finds nothing, returns silently.

**Impact**: Users passing a specific file path get empty results with no error
message. The CLI description says "Path to instrument (directory or glob)" but
single-file paths fail silently. Combined with Finding 5 (silent on zero files),
this is a confusing UX.

### Finding 15: No detection of already-instrumented code

**Observed**: `kubectl-get.ts` (copied from cluster-whisperer) already had OTel
instrumentation — it imports `SpanKind`, `SpanStatusCode` from `@opentelemetry/api`
and calls `getTracer()`. The agent added a second `const tracer = ...` variable,
which the shadowing checker correctly flagged as a conflict.

**Impact**: The agent should detect existing OTel instrumentation and either skip
the file or account for existing spans/imports in its output. Without this, running
the agent against a partially-instrumented codebase (common in practice) produces
conflicts.

### Finding 16: Hybrid fix strategy not implemented

**Observed**: Both files failed with `validation_attempts: 1` and
`validation_strategy_used: 'initial-generation'`. The spec describes a 3-attempt
hybrid strategy (initial generation → multi-turn fixing → fresh regeneration), but
the agent-bridge (`agent-bridge.ts:24-25`) explicitly states: *"For Phase 6
(Walking Skeleton), this is a single-attempt processor... The fix loop (Phase 8)
will wrap this."*

**Impact**: When validation fails for a fixable reason (e.g., variable shadowing
where the validator suggests renaming `tracer` to `otelTracer`), the agent gives
up instead of retrying with the feedback. The shadowing errors in this run were
exactly the kind of issue the fix loop was designed to handle — the validator even
provided the fix (`suggest "otelTracer"`).

### Finding 17: Weaver schema validation disabled

**Observed**: The agent-bridge (`agent-bridge.ts:57`) explicitly sets
`runWeaver: false` with the comment *"Weaver may not be available in all
environments"*.

**Impact**: The agent never validates its schema extensions against the Weaver
registry. Schema correctness — the core value proposition of schema-driven
instrumentation — is not enforced. This means the agent could generate attributes
that violate Weaver conventions, and no check would catch it.

### Finding 18: Validation chain architecture is sound (positive)

**Observed**: The validation chain correctly short-circuits on first failure
(elision → syntax → shadowing → lint → weaver). Diagnostic messages include
the failing step name, error count, line numbers, and fix suggestions. The
chain correctly skips downstream checks when an earlier check fails.

**Impact**: The chain's design is well-architected. The problems are in the
individual checkers (findings 19-20) and the missing fix loop (finding 16),
not the chain's orchestration logic.

### Finding 19: Shadowing checker bans variable names instead of detecting real shadowing

**Observed**: The shadowing checker (`shadowing-checker.ts:25-28`) maintains a
hardcoded blocklist: any variable declaration or parameter named `tracer` or
`span` anywhere in the file is flagged as a shadowing issue. This is not scope-
based shadowing detection — it's a blanket name ban.

The agent generates `const tracer = trace.getTracer(...)`, which is the standard
OTel pattern. The validator then immediately rejects it because `tracer` is on
the banned list. **The agent's own output is incompatible with its own validator.**

Files with NO existing OTel code (`discovery.ts`, `inference.ts`) also fail
because the agent introduces `tracer` as a new variable — not shadowing anything.

**Root cause**: The checker was likely designed to prevent the agent from
introducing variables that shadow user code. But it checks the agent's OUTPUT
(which always contains `tracer`), not the DIFF between original and output.
The result is that the standard OTel instrumentation pattern can never pass.

**Impact**: Every file the agent instruments will fail shadowing validation.
Combined with the missing fix loop (finding 16), this is a systematic blocker.
The validator suggests `otelTracer` as a fix, but the agent doesn't know to
use that name and has no retry mechanism to learn from the feedback.

### Finding 20: Lint checker rejects non-Prettier-formatted output

**Observed**: `types.ts` (a pure types file) passed elision, syntax, and
shadowing checks, but failed at lint: *"File does not match Prettier formatting
(original was formatted, agent output is not)"*.

**Root cause**: The lint checker (`lint-checker.ts`) compares the agent's output
against Prettier formatting. If the original file was Prettier-formatted and the
agent's output isn't, it fails. LLMs don't reliably produce Prettier-compliant
output — whitespace, trailing commas, and line wrapping often differ.

**Impact**: Even when instrumentation logic is correct, formatting mismatches
block the file. This is another issue the fix loop would handle (run Prettier
on the output before lint checking). Without the loop, it's a systematic blocker
that affects every file.

### Finding 21: Schema extensions never created — schema-driven instrumentation not exercised

**Observed**: Every file reported `schema_extensions: []` and `schema_defined: 0`
in span categories. The agent created custom attributes (`discovery.resource_count`,
`kubectl.resource`, `search.collection`, etc.) but never registered them in the
Weaver schema registry.

**Root cause**: Two factors combine:
1. Weaver validation is disabled (`runWeaver: false`, Finding 17), so the agent
   has no mechanism to create or validate schema entries.
2. The schema domain mismatch — the Weaver registry defines `commit_story.*`
   attributes, but the test files are Kubernetes pipeline code. The agent correctly
   didn't force-fit commit-story attributes, but also didn't extend the schema
   with new `cluster_whisperer.*` or generic `k8s.*` entries.

**Impact**: The core value proposition of schema-driven instrumentation — attributes
defined in a Weaver registry that are machine-validated for correctness — is not being
exercised. The agent produces standard manual OTel instrumentation with ad-hoc attribute
names. Without schema extensions, reviewers have no schema diff to check, attribute naming
is unconstrained, and semantic conventions aren't enforced.

### Finding 22: PR description assembled but discarded by MCP handler

**Observed**: The coordinator assembles a detailed PR description at step 7
(`coordinator.ts:194`) using `assemblePrDescription()`, which builds markdown with
summary stats, per-file results table, span categories breakdown, schema diff, failed
files detail, agent notes, token usage, and review sensitivity. But the MCP server
handler (`server.ts:193`) calls `formatJsonOutput()` instead, which only formats
the results array as JSON. The `prDescription` field in the coordinator result is
computed and thrown away.

**Impact**: The PR report — which the spec describes as the primary review artifact —
is never surfaced to the user. The MCP user sees raw JSON results but not the formatted
review document designed for human consumption.

### Finding 23: No git branch, commits, or PR created

**Observed**: The spec describes per-file commits to a feature branch, with the PR
description as the final output. The `branch-manager.ts` has git operations (`checkout -b`,
`commit`, `branch --show-current`, `branch -D`), but they're never called from the
coordinator. The coordinator processes files and returns results; creating git artifacts
is deferred to the interface layer — and neither the MCP nor the CLI interface does it.

**Impact**: The agent modifies files in-place on whatever branch the user is on.
There's no branch isolation, no per-file commits for revert granularity, and no PR.
The entire git workflow described in the spec is unwired.

### Finding 24: Instrumentation quality is good when validation is bypassed (positive)

**Observed**: With validation disabled (Patch 3), the agent produced well-structured
OTel instrumentation across 4 files:

| File | Spans | Attributes | Quality |
|------|-------|------------|---------|
| `discovery.ts` | 2 | 6 | Correct: wraps entry point + external kubectl call |
| `inference.ts` | 3 | 11 | Correct: nested spans for entry points + LLM call |
| `format-results.ts` | 1 | 2 | Correct: wraps public API surface only |
| `kubectl-get.ts` | 1 | 3 | Correct: wraps external kubectl call |

Positive behaviors observed:
- **Standard OTel patterns**: `trace.getTracer()`, `tracer.startActiveSpan()`,
  `span.setAttribute()`, `span.recordException()`, `span.setStatus()`, `span.end()`
- **Proper error handling**: try/catch/finally with error recording on spans
- **Reasonable attribute names**: `discovery.resource_count`, `kubectl.resource`,
  `search.collection`, `llm.model` — descriptive, snake_case, dot-namespaced
- **Nested spans**: `llm.invoke` properly nested inside `inference.infer_capability`
- **Correct skip decisions**: types.ts (type-only), tracing/index.ts (infrastructure),
  kubectl.ts (already instrumented) — all correctly left unchanged
- **Library suggestion**: recommended `@traceloop/instrumentation-langchain` for
  LangChain/Anthropic calls with a note that the coordinator should evaluate
  auto-instrumentation vs the manual span
- **Thoughtful notes**: each file includes explanation of what was instrumented
  and why, with references to priority rules

### Finding 25: Already-instrumented detection works (partially contradicts Finding 15)

**Observed**: `utils/kubectl.ts` was correctly identified as already instrumented and
left unchanged with 0 spans added. The agent noted: *"File is already fully instrumented.
The exported executeKubectl function already wraps the spawnSync external call with
tracer.startActiveSpan()..."*

However, `tools/kubectl-get.ts` — which also already had OTel imports and tracing —
got additional instrumentation added. The agent recognized `utils/kubectl.ts` because
it has explicit `tracer.startActiveSpan()` patterns. But `kubectl-get.ts` imports from
a central `getTracer()` function, which is a different pattern the agent didn't flag
as "already instrumented."

**Impact**: Already-instrumented detection is pattern-dependent, not comprehensive.
Files with explicit inline tracer setup are detected; files that import tracer factories
from other modules are not. This is nuanced behavior — partially correct but inconsistent.

## Patches Applied

These modifications to the first-draft implementation were necessary to unblock
evaluation. They are documented here for transparency and to inform PRD #3.

### Patch 1: MCP server entry point

**File created**: `bin/telemetry-agent-mcp.js`

The MCP server had no entry point — `startMcpServer()` was exported but never
called from any bin script. Created a 4-line entry point to launch the stdio server.

### Patch 2: Syntax checker — real filesystem for node_modules resolution

**File modified**: `src/validation/syntax-checker.ts`

Changed `useInMemoryFileSystem: true` to use the real filesystem so that
`@opentelemetry/api` and other node_modules imports resolve correctly. The agent's
output content is still loaded via `createSourceFile()` with `overwrite: true`
rather than reading from disk.

### Patch 3: Validation bypass — disabled to see agent output

**File modified**: `src/coordinator/agent-bridge.ts`

Changed the validation gate (`if (!validation.passed)`) to `if (false && ...)` so
the agent's instrumented output is always accepted regardless of validation results.
This allows evaluating the quality of the agent's instrumentation logic separately
from the broken validation chain. Validation still runs (for diagnostic data) but
failures no longer block output.

## Rubric Scoring

Scored after 8 runs (7 failed, 1 successful with validation bypassed via Patch 3).

| Dimension | Score | Notes |
|-----------|-------|-------|
| Non-destructiveness | Good | Agent reverts files on validation failure (Finding 12) |
| Setup experience | Poor | `init` unwired; manual config; no tsconfig/TS prerequisite check |
| Error reporting | Poor | Silent on 0 files; silent on file paths; no progress; no early abort |
| Cost management | Mixed | Cost ceiling tool works but lacks dollar estimates; ~$5-6 spent total, $3.50+ with 0 output |
| Interface completeness | Poor | CLI non-functional; MCP works but no progress feedback; PR description discarded (Finding 22) |
| Validation chain | Broken | Architecture sound (Finding 18) but in-memory FS (13), blanket shadowing (19), and lint (20) block all output |
| Fix loop | Missing | Single attempt only; hybrid 3-attempt strategy not implemented (Finding 16) |
| Schema validation | Disabled | Weaver check explicitly skipped; no schema extensions created (Findings 17, 21) |
| Existing instrumentation | Partial | Detects inline tracing but not imported tracer patterns (Finding 25) |
| Git workflow | Missing | No branch, commits, or PR created (Finding 23) |
| Instrumentation quality | Good | Correct span placement, proper error handling, reasonable attributes, thoughtful skip decisions (Finding 24) |
| Skip decisions | Good | Types-only, infrastructure, and already-instrumented files correctly left unchanged |
| Library suggestions | Good | Recommended `@traceloop/instrumentation-langchain` with nuanced note about auto vs manual |
| Agent notes/reasoning | Good | Detailed per-file notes explaining what was instrumented and why, referencing priority rules |

## Token Usage (API cost)

| Run | Interface | Files | Input Tokens | Output Tokens | Result |
|-----|-----------|-------|-------------|---------------|--------|
| 1 | CLI | 0 | 0 | 0 | Silent exit (CLI unwired) |
| 2 | MCP | 4 | 16,068 | 15,636 | All failed (no tsconfig — pre-patch) |
| 3 | MCP | 4 | 16,068 | 14,664 | All failed (in-memory FS — pre-patch) |
| 4 | MCP | 4 | 16,068 | 14,837 | All failed (missing local modules — post-patch) |
| 5 | MCP | 0 | 0 | 0 | Empty result (file path, not directory) |
| 6 | MCP | 2 | 6,425 | 3,998 | All failed (shadowing — post-patch, post-stubs) |
| 7 | MCP | 3 | 14,252 | 12,919 | 2 failed shadowing, 1 failed lint (pipeline dir) |
| 8 | MCP | 7 | 27,870 | 21,617 | **7 passed** (validation bypassed — Patch 3) |
| **Total** | | | **96,751** | **83,671** | **7 successful (run 8 only)** |

Estimated cost: ~$5.50-6.50 (Sonnet 4.6 pricing). Runs 1-7 ($3.50-4.50) produced
zero usable output. Run 8 ($1.50-2.00) produced all successful results after
validation was bypassed.

## PRD #3 Implications

### Key insight: the spec is sound, the build didn't match it

The most important takeaway from this evaluation is that **the spec describes the
right system**. The architecture (coordinator + per-file agents), the validation
chain (elision → syntax → shadowing → lint → weaver), the hybrid fix strategy,
the CLI and MCP interfaces, the progress callbacks — all of these are well-designed
in the spec.

The failures are implementation quality issues, not spec gaps:

| Finding | Spec says | Implementation does |
|---------|-----------|---------------------|
| CLI handlers (1, 2) | CLI with `init` and `instrument` commands | Arg parsing only, no handlers wired |
| Fix loop (16) | 3-attempt hybrid strategy | Single attempt, gives up on first failure |
| Validation FS (13) | Validate instrumented output compiles | In-memory FS can't resolve any imports |
| Weaver check (17) | Schema validation via Weaver CLI | Explicitly disabled (`runWeaver: false`) |
| Progress (7) | Callback hooks for `onFileStart`/`onFileComplete` | Hooks defined but never wired in MCP |
| Shadowing (19) | Detect variable name collisions | Blanket ban on `tracer`/`span` — rejects agent's own output |
| Prerequisites (8, 11) | Check project readiness before running | Missing tsconfig.json and TypeScript checks |
| Schema extensions (21) | Agent extends Weaver registry with new attributes | No schema entries created; all `schema_extensions: []` |
| PR report (22) | Detailed markdown PR with schema diff, agent notes, token usage | PR description assembled but discarded by MCP handler |
| Git workflow (23) | Feature branch, per-file commits, PR creation | No branch, commits, or PR; files modified in-place |

This means PRD #3's primary job is NOT rewriting the spec — it's ensuring the
next build actually implements what the spec already describes.

### The rubric as acceptance criteria

Shipping the evaluation rubric alongside the spec solves this. If the next builder
(AI or human) has concrete acceptance criteria from day one, they can't ship
something that only works in unit tests:

- **"Must pass an e2e smoke test against a real project"** → catches findings
  1, 2, 4, 8, 11, 13 in a single test run
- **"CLI must produce visible output"** → catches findings 1, 2, 5, 14
- **"Validation must resolve real imports"** → catches finding 13
- **"Fix loop must retry on fixable failures"** → catches findings 16, 19, 20

The rubric transforms from a post-hoc evaluation tool into a build-time
acceptance test suite. 332 unit tests passed, but zero rubric criteria would
have passed. That's the gap.

### Actual spec changes needed

Only two real spec changes emerged from this evaluation:

1. **JavaScript support** — the spec currently targets TypeScript only
   (`**/*.ts` glob hardcoded in file discovery). JS is the bigger ecosystem;
   scoping it in (or explicitly out with rationale) is a spec-level decision.

2. **Model configurability emphasis** — `agentModel` is already configurable
   in the config schema, but cost management and model selection deserve more
   prominent treatment in the spec given real-world budget constraints.

Everything else — CLI wiring, fix loops, validation, progress feedback,
prerequisites, error reporting — is already in the spec. It just needs to be
built correctly, and the rubric ensures that.

### Testing directives for the spec

The first-draft implementation has 332 passing tests, yet every real-world
execution path failed. The spec should include testing requirements:

- **Validation tests must include at least one integration test against a real
  filesystem** with real `node_modules` imports. Unit tests with in-memory FS
  are fine for speed, but if 100% of validation tests use in-memory FS,
  resolution failures are invisible.
- **CLI and MCP interface tests must verify handler wiring** — not just argument
  parsing. A test that calls `instrument` and asserts the coordinator was invoked
  would have caught the unwired handlers immediately.
- **End-to-end smoke test**: instrument a single real file in a real project,
  verify it compiles. This one test would have caught findings 1, 2, 4, 8, 11,
  and 13 in a single run.
- **Validation chain integration test**: instrument a file, run the full
  validation chain, verify the output passes. Would have caught findings 13,
  19, and 20.

## Instrumentation Quality Analysis (Run 8)

### Span placement decisions

The agent correctly applied the spec's priority hierarchy:

1. **External calls** (highest priority): `executeKubectl()` calls in `kubectl-get.ts`
   and `discovery.ts` both got spans wrapping the external subprocess execution.
2. **Service entry points**: `discoverResources()`, `inferCapability()`,
   `inferCapabilities()`, and `formatSearchResults()` all instrumented as entry points.
3. **Internal helpers skipped**: `parseApiResources()`, `extractGroup()`,
   `buildFullyQualifiedName()`, `filterResources()`, `describeSimilarity()`,
   `formatMetadata()`, `buildHumanMessage()`, `getPromptTemplate()`,
   `getDefaultModel()` — all correctly left uninstrumented.

### Attribute naming

All attributes use descriptive snake_case with dot-namespaced prefixes:

| Span | Attributes |
|------|------------|
| `discovery.resources` | `discovery.resource_count`, `discovery.filtered_count`, `discovery.total_api_resources`, `discovery.crd_count` |
| `discovery.crd_names` | `discovery.crd_count` |
| `inference.infer_capability` | `resource.name`, `resource.kind`, `resource.api_version`, `resource.group`, `llm.result.complexity`, `llm.result.confidence`, `llm.result.capabilities_count` |
| `llm.invoke` | `llm.model`, `llm.resource_name` |
| `inference.infer_capabilities` | `resources.count`, `resources.processed_count`, `resources.skipped_count` |
| `format.search.results` | `search.collection`, `search.results.count` |
| `kubectl.get` | `kubectl.resource`, `kubectl.namespace`, `kubectl.name` |

Naming is reasonable but entirely ad-hoc — none of these match the Weaver schema
(`commit_story.*` namespace) and none were registered as schema extensions. This is
the gap between "produces working OTel code" and "produces schema-driven OTel code."

### Error handling patterns

All instrumented functions follow the same pattern:

```typescript
return tracer.startActiveSpan('span.name', async (span) => {
  try {
    span.setAttribute('key', value);
    // ... original function body ...
    return result;
  } catch (error) {
    span.recordException(error as Error);
    span.setStatus({ code: SpanStatusCode.ERROR, message: (error as Error).message });
    throw error;
  } finally {
    span.end();
  }
});
```

This is correct and follows OTel best practices. The `finally` block ensures
`span.end()` is always called. Error paths record the exception and set status.

### Schema domain mismatch handling

The agent handled the mismatch well:
- Did NOT force-fit `commit_story.*` attributes onto Kubernetes pipeline code
- Created domain-appropriate custom attributes (`discovery.*`, `kubectl.*`, `llm.*`)
- Did NOT extend the Weaver schema (all `schema_extensions: []`)
- The ideal behavior would be to extend the schema with the new attributes, but
  gracefully degrading to custom attributes is acceptable given Weaver is disabled

### Notable behaviors

**Nested spans**: `inference.ts` uses a nested `llm.invoke` span inside
`inference.infer_capability`, which correctly captures the LLM call as a child
operation. This shows the agent understands span hierarchy.

**Conditional attributes**: `kubectl-get.ts` only sets `kubectl.namespace` and
`kubectl.name` when the values are present (checking `!== undefined`), avoiding
spurious `undefined` attribute values. This is a quality signal.

**Already-instrumented handling**: `utils/kubectl.ts` was correctly identified and
skipped. The agent's notes show it analyzed the existing spans, error handling, and
attributes before deciding no changes were needed. However, `kubectl-get.ts` (which
also had OTel code via an imported tracer) was not detected as already-instrumented.

## Next Steps

- Create PR on eval repo with instrumented changes for review
- Consider cluster-whisperer as a full evaluation target (native TypeScript, no stubs)
- Score against formal rubric dimensions once PRD #1 rubric is finalized
- Feed findings into PRD #3 spec synthesis
