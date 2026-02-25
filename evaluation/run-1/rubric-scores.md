# Rubric Scores: Evaluation Run 1

**Date**: 2026-02-25
**Rubric version**: PRD #1 evaluation rubric (`research/evaluation-rubric.md`)
**Agent output**: [commit-story-v2-eval#1](https://github.com/wiggitywhitney/commit-story-v2-eval/pull/1)
**Run conditions**: Validation bypassed (Patch 3); see `evaluation/run-1/README.md` for full context

## Files Evaluated

| File | Action | Spans Added | Attributes Added |
|------|--------|-------------|-----------------|
| `pipeline/discovery.ts` | Instrumented | 2 (`discovery.resources`, `discovery.crd_names`) | 4 |
| `pipeline/inference.ts` | Instrumented | 3 (`inference.infer_capability`, `llm.invoke`, `inference.infer_capabilities`) | 11 |
| `tools/format-results.ts` | Instrumented | 1 (`format.search.results`) | 2 |
| `tools/kubectl-get.ts` | Instrumented | 1 (`kubectl.get`) | 3 |
| `pipeline/types.ts` | Skipped (correct) | 0 | 0 |
| `tracing/index.ts` | Skipped (correct) | 0 | 0 |
| `utils/kubectl.ts` | Skipped (correct) | 0 | 0 |

---

## Gate Checks

All gates must pass for quality scoring to be meaningful.

| Gate | Scope | Result | Evidence |
|------|-------|--------|----------|
| NDS-001 | Per-run | **CONDITIONAL PASS** | Validation was bypassed (Patch 3) due to in-memory FS bug (Finding 13). The agent's TypeScript output is syntactically correct — proper imports, type annotations, and OTel API usage. Not independently verified via `tsc --noEmit`. |
| NDS-002 | Per-run | **NOT VERIFIED** | Agent only modified `src/ts-samples/` files (copied from cluster-whisperer). Original 320 tests cover different code (`src/*.js`). No tests exist for the TS sample files. Risk of regression is minimal but not formally validated. |
| NDS-003 | Per-file | **PASS (4/4)** | All 4 instrumented files: diffs contain only OTel imports, tracer acquisition, `startActiveSpan` wrapping, `span.setAttribute` calls, `span.recordException`/`setStatus`/`end` calls, and try/catch/finally blocks for span lifecycle. No original business logic modified. |
| API-001 | Per-file | **PASS (4/4)** | All 4 files import only from `@opentelemetry/api`. No SDK, exporter, or instrumentation package imports. |

**Gate assessment**: NDS-001 and NDS-002 are technically unverified, which limits confidence. NDS-003 and API-001 pass cleanly. Quality scoring proceeds with the caveat that two gates were not independently validated.

---

## Dimension 1: Non-Destructiveness (NDS)

| Rule | Scope | Result | Evidence |
|------|-------|--------|----------|
| NDS-004 | Per-file | **PASS (4/4)** | All exported function signatures preserved exactly. `discoverResources`, `inferCapability`, `inferCapabilities`, `formatSearchResults`, `kubectlGet` — parameters, return types, and export declarations unchanged. Pure utility exports (`parseApiResources`, `extractGroup`, `buildFullyQualifiedName`, `filterResources`, `LlmCapabilitySchema`, `kubectlGetSchema`, etc.) also unchanged. |
| NDS-005 | Per-file | **PASS (4/4)** | No pre-existing error handling modified. `discovery.ts`: original `getCrdNames` try/catch preserved inside new span wrapper. `inference.ts`: original try/catch around `model.invoke` preserved inside span wrapper. `format-results.ts` and `kubectl-get.ts`: no pre-existing error handling to modify. All added try/catch/finally blocks re-throw after recording. |

**Dimension summary**: 2/2 rules pass. The agent preserved all public contracts and error semantics.

---

## Dimension 2: Coverage (COV)

| Rule | Scope | Result | Detail |
|------|-------|--------|--------|
| COV-001 | Per-instance | **PASS (5/5)** | All entry points instrumented: `discoverResources` → `discovery.resources`, `inferCapability` → `inference.infer_capability`, `inferCapabilities` → `inference.infer_capabilities`, `formatSearchResults` → `format.search.results`, `kubectlGet` → `kubectl.get`. |
| COV-002 | Per-instance | **PASS** | Outbound calls covered: kubectl subprocess calls flow through `executeKubectl` which has its own spans (in already-instrumented `utils/kubectl.ts`). LLM call has dedicated `llm.invoke` span. No outbound call site lacks a span. |
| COV-003 | Per-instance | **PASS (7/7)** | All 7 spans have `span.recordException(error)` + `span.setStatus({ code: SpanStatusCode.ERROR })` in catch blocks. |
| COV-004 | Per-instance | **PASS** | All async functions (`discoverResources`, `inferCapability`, `inferCapabilities`, `kubectlGet`) and I/O functions (`getCrdNames` — kubectl call) have spans. |
| COV-005 | Per-instance | **FAIL (0/21)** | Zero registry-defined attributes used. Agent created 21 ad-hoc attributes (`discovery.*`, `resource.*`, `llm.*`, `kubectl.*`, `search.*`, `resources.*`). Weaver registry defines `commit_story.*` attributes. See [Schema Fidelity notes](#surprises-and-unexpected-findings) for domain mismatch context. |
| COV-006 | Per-instance | **FAIL (0/1)** | `inference.ts:217`: Manual `llm.invoke` span wraps `model.invoke()` — a LangChain `ChatAnthropic` call coverable by `@traceloop/instrumentation-langchain`. Agent acknowledged this in its notes (recommended the library), but still added a manual span. |

**Dimension summary**: 4/6 rules pass. Failures are in schema-driven attributes (domain mismatch artifact) and auto-instrumentation preference (agent was aware of the alternative).

---

## Dimension 3: Restraint (RST)

| Rule | Scope | Result | Detail |
|------|-------|--------|--------|
| RST-001 | Per-instance | **PASS (0 violations)** | 11 utility functions correctly left uninstrumented: `parseApiResources`, `extractGroup`, `buildFullyQualifiedName`, `filterResources`, `getPromptTemplate`, `getDefaultModel`, `buildHumanMessage`, `describeSimilarity`, `formatMetadata`, `redactSensitiveArgs`, `extractKubectlMetadata`. |
| RST-002 | Per-instance | **N/A** | No getter/setter/accessor patterns in evaluated files. |
| RST-003 | Per-instance | **PASS (0 violations)** | No thin wrappers have spans. `kubectlGet` builds arguments before delegating — not a single-call wrapper. |
| RST-004 | Per-instance | **FAIL (1 violation)** | `discovery.ts:304`: `getCrdNames` is unexported (internal implementation detail) but has a `discovery.crd_names` span. Mitigating factor: the function performs I/O (kubectl subprocess call), so observability has genuine value. See [surprises](#surprises-and-unexpected-findings). |
| RST-005 | Per-instance | **PASS (0 violations)** | `utils/kubectl.ts` (already instrumented with `startActiveSpan`, `SpanKind.CLIENT`, full attribute set) correctly identified and skipped. `tracing/index.ts` (infrastructure code) also correctly skipped. |

**Dimension summary**: 3/4 applicable rules pass. One internal function instrumented (debatable — it wraps I/O).

---

## Dimension 4: API-Only Dependency (API)

| Rule | Scope | Result | Detail |
|------|-------|--------|--------|
| API-002 | Per-run | **PASS** | `package.json`: `@opentelemetry/api: ^1.9.0` declared in `peerDependencies`. Correct for a distributable npm package. |
| API-003 | Per-run | **PASS** | No vendor-specific instrumentation SDKs in dependencies. (`@opentelemetry/sdk-node` is the standard OTel SDK, not vendor-specific.) |
| API-004 | Per-file | **PASS (4/4)** | All 4 files: import only `trace` and `SpanStatusCode` from `@opentelemetry/api`. No `@opentelemetry/sdk-*`, `@opentelemetry/exporter-*`, or `@opentelemetry/instrumentation-*` imports. |

**Dimension summary**: 3/3 rules pass. Dependency model is correct.

---

## Dimension 5: Schema Fidelity (SCH)

| Rule | Scope | Result | Detail |
|------|-------|--------|--------|
| SCH-001 | Per-instance | **FAIL (0/7)** | Zero span names match registry operations. Agent span names: `discovery.resources`, `discovery.crd_names`, `inference.infer_capability`, `llm.invoke`, `inference.infer_capabilities`, `format.search.results`, `kubectl.get`. Registry defines `commit_story.*` operations. Domain mismatch: Weaver registry defines commit-story journaling operations; instrumented code is Kubernetes pipeline logic from cluster-whisperer. |
| SCH-002 | Per-instance | **FAIL (0/21)** | Zero attribute keys match registry names. Agent uses `discovery.*`, `resource.*`, `llm.*`, `kubectl.*`, `search.*`, `resources.*`. Registry defines `commit_story.commit.*`, `commit_story.context.*`, `commit_story.journal.*`, `commit_story.filter.*`, `commit_story.ai.*`. |
| SCH-003 | Per-instance | **N/A** | No registry attributes used — nothing to validate types/constraints against. |
| SCH-004 | Per-instance | **PASS** | No redundant entries. Agent-added attribute namespaces (`discovery.*`, `kubectl.*`, `llm.*`) are semantically disjoint from registry namespaces (`commit_story.*`). No string/token similarity matches above threshold. |

**Dimension summary**: 1/3 applicable rules pass. SCH-001 and SCH-002 fail due to domain mismatch between the Weaver registry (commit-story) and the instrumented code (cluster-whisperer). The agent correctly avoided force-fitting commit-story attributes onto unrelated code but also did not extend the schema (Finding 21: `schema_extensions: []` for every file).

---

## Dimension 6: Code Quality (CDQ)

| Rule | Scope | Result | Detail |
|------|-------|--------|--------|
| CDQ-001 | Per-instance | **PASS (7/7)** | Every `startActiveSpan` callback includes `span.end()` in a `finally` block. Pattern: `tracer.startActiveSpan('name', async (span) => { try { ... } catch { span.recordException(...); throw; } finally { span.end(); } })`. |
| CDQ-002 | Per-file | **PASS (4/4)** | All `trace.getTracer()` calls include a library name string: `'commit-story'`, `'inference'`, `'commit-story.tools.format-results'`, `'kubectl-get'`. No version argument (recommended but not required). |
| CDQ-003 | Per-instance | **PASS (7/7)** | All catch blocks use the standard OTel pattern: `span.recordException(error as Error)` + `span.setStatus({ code: SpanStatusCode.ERROR, message: ... })`. No ad-hoc `span.setAttribute('error', ...)` patterns. |
| CDQ-005 | Per-instance | **PASS (7/7)** | All spans use the `startActiveSpan` callback pattern (context automatically maintained). No `startSpan()` manual pattern requiring `context.with()`. Async functions correctly use `async (span) => { ... }` callbacks. |
| CDQ-006 | Per-instance | **PASS (0 violations)** | All `setAttribute` values are simple property accesses (`.length`, `.size`, `.name`), string properties, or numeric values. No function calls, method chains, serialization, or computed values. No `isRecording()` guard needed. |
| CDQ-007 | Per-instance | **PASS (0 violations)** | All 21 attributes are bounded: counts (number), resource identifiers (bounded strings), enum values (`complexity`). No `JSON.stringify`, object spreads, or unbounded arrays. No PII-pattern attribute keys. Conditional attributes (`kubectl.namespace`, `kubectl.name`) only set when values present — avoids `undefined` attributes. |

**Dimension summary**: 6/6 rules pass. The OTel instrumentation patterns are textbook-correct.

---

## Instrumentation Score (IS) — Static Analysis

Most IS rules evaluate runtime OTLP streams. This evaluation is static (source code review only). Per the rubric, 6 of 19 IS rules are statically automatable; the rest require runtime or are N/A.

| Rule | Static Result | Notes |
|------|--------------|-------|
| RES-005 | **PARTIAL** | `service.name` not verified — SDK init file (`src/telemetry/setup.ts`) was created during manual setup, not by the agent. |
| SPA-003 | **PASS** | All 7 span names are string literals. No template literal interpolation or dynamic values. Bounded cardinality. |
| MET-002 | **N/A** | No metrics in agent output. |
| MET-005 | **N/A** | No metrics in agent output. |
| MET-006 | **N/A** | No metrics in agent output. |
| SDK-001 | **PASS** | `package.json` engines `>=18.0.0`; `@opentelemetry/sdk-node: ^0.212.0` compatible. |

**IS static summary**: 2 rules pass, 1 partial, 3 N/A. Remaining 13 IS rules require runtime OTLP data not available in this evaluation.

---

## Score Summary

### Gate Results

| Gate | Result |
|------|--------|
| NDS-001 (Compilation) | Conditional pass (not independently verified) |
| NDS-002 (Tests pass) | Not verified (no tests for modified files) |
| NDS-003 (Non-instrumentation lines unchanged) | **Pass** |
| API-001 (API-only imports) | **Pass** |

### Dimension Profiles

| Dimension | Rules Applicable | Rules Passed | Pass Rate |
|-----------|-----------------|--------------|-----------|
| Non-Destructiveness (NDS) | 2 | 2 | 100% |
| Coverage (COV) | 6 | 4 | 67% |
| Restraint (RST) | 4 | 3 | 75% |
| API-Only Dependency (API) | 3 | 3 | 100% |
| Schema Fidelity (SCH) | 3 | 1 | 33% |
| Code Quality (CDQ) | 6 | 6 | 100% |
| **Total** | **24** | **19** | **79%** |

### Per-File Breakdown

| File | Gate | NDS | COV | RST | API | SCH | CDQ |
|------|------|-----|-----|-----|-----|-----|-----|
| `discovery.ts` | Pass | 2/2 | COV-005 fail, rest pass | RST-004 fail (getCrdNames) | Pass | SCH-001,002 fail | 6/6 |
| `inference.ts` | Pass | 2/2 | COV-005,006 fail, rest pass | Pass | Pass | SCH-001,002 fail | 6/6 |
| `format-results.ts` | Pass | 2/2 | COV-005 fail, rest pass | Pass | Pass | SCH-001,002 fail | 6/6 |
| `kubectl-get.ts` | Pass | 2/2 | COV-005 fail, rest pass | Pass | Pass | SCH-001,002 fail | 6/6 |
| `types.ts` | N/A (skipped) | — | — | — | — | — | — |
| `tracing/index.ts` | N/A (skipped) | — | — | — | — | — | — |
| `utils/kubectl.ts` | N/A (skipped) | — | — | RST-005 pass | — | — | — |

---

## Surprises and Unexpected Findings

### 1. Schema Fidelity is structurally untestable in this eval setup

The biggest scoring gap (SCH dimension at 33%) is an artifact of the evaluation design, not agent behavior. The Weaver registry defines `commit_story.*` attributes for a JavaScript journaling tool. The agent instrumented TypeScript Kubernetes pipeline code copied from a different project. No amount of agent intelligence can match registry attributes that describe a different domain.

The agent's actual behavior was correct: it did not force-fit `commit_story.commit.sha` onto a Kubernetes discovery function. It created domain-appropriate names (`discovery.resource_count`, `kubectl.resource`). The failure is that it didn't *extend* the schema with new entries — but Weaver validation was explicitly disabled in the implementation (Finding 17: `runWeaver: false`), so the mechanism to create extensions didn't exist.

**Implication for PRD #3**: Schema fidelity can only be meaningfully evaluated when the instrumented code matches the registry domain. Future evaluations should either (a) use the actual commit-story JavaScript files as the test target, or (b) create a cluster-whisperer Weaver registry.

### 2. Two gate checks provide no confidence

NDS-001 (compilation) was not independently verified because the validation chain was broken (Finding 13) and bypassed (Patch 3). NDS-002 (tests pass) was not verified because the modified files have no tests in the eval repo. This means the two most fundamental safety checks — "did the agent break the build?" and "did the agent break existing behavior?" — provide zero signal.

This is not the agent's fault. The gates failed because (a) the implementation's own validation chain was broken, and (b) the eval setup copied files from a different project that had no test coverage in the target repo. But it limits how much confidence we can place in the overall scores.

**Implication for PRD #3**: The spec should require that gate checks are independently runnable. An implementation that can't verify its own output doesn't provide safety guarantees.

### 3. Perfect Code Quality despite everything else

CDQ scored 100% (6/6 rules). Every span is properly closed, every tracer is correctly acquired, every error uses the standard recording pattern, async context is maintained, no expensive computations, no unbounded attributes. This matches Finding 24's observation that "instrumentation quality is good when validation is bypassed."

The agent produces textbook-correct OTel patterns. The problems are upstream (schema integration, auto-instrumentation awareness) and downstream (validation chain, git workflow), not in the instrumentation code itself.

### 4. COV-006 failure shows awareness without compliance

The agent added a manual `llm.invoke` span around a LangChain `ChatAnthropic.invoke()` call, which is coverable by `@traceloop/instrumentation-langchain`. However, the agent's notes explicitly recommended this library with a nuanced observation about auto vs. manual instrumentation. The agent was *aware* of the principle while violating it — arguably better than being unaware, and the manual span provides more specific attributes than auto-instrumentation would.

### 5. RST-004 highlights a rubric tension

`getCrdNames` is unexported (internal) but performs I/O (kubectl subprocess). RST-004 says "unexported = internal, instrumentation not reasonable." But observability on I/O boundaries has real value regardless of export status. The rubric's heuristic (export as proxy for "public API") works well for pure logic but breaks down for I/O operations that happen to be internal.

**Implication for PRD #3**: Consider refining RST-004 to exempt internal functions that perform I/O or external calls, since these are natural observability boundaries regardless of visibility.

### 6. Tracer naming is inconsistent (quality observation, not rubric violation)

The agent used different tracer names across files: `'commit-story'` (discovery.ts), `'inference'` (inference.ts), `'commit-story.tools.format-results'` (format-results.ts), `'kubectl-get'` (kubectl-get.ts). A consistent naming convention (e.g., all using the package name, or all using module-relative names) would improve trace analysis. CDQ-002 only checks that a name argument exists, not naming consistency — a potential rubric gap.

### 7. Conditional attribute setting shows quality awareness

In `kubectl-get.ts:92-97`, the agent only sets `kubectl.namespace` and `kubectl.name` when values are defined (`!== undefined`). This avoids spurious `undefined` attribute values, which is a quality signal beyond what CDQ-007 checks. The rubric doesn't have a specific rule for this, but it's a positive behavior worth noting for PRD #3 acceptance criteria.
