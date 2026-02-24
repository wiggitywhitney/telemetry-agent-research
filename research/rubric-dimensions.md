# Evaluation Rubric: Dimensions (Working Draft)

> **Superseded by [`evaluation-rubric.md`](./evaluation-rubric.md)**, which is the finalized, codebase-agnostic evaluation framework. This file is preserved as a working draft for historical context.

This rubric adopts the [Instrumentation Score Specification](https://github.com/instrumentation-score/spec) as its foundation for telemetry quality assessment, then extends it with code-level dimensions specific to evaluating AI-generated instrumentation.

## Foundation: Instrumentation Score Specification

The Instrumentation Score is a community-driven, vendor-neutral standard (0-100) created by [OllyGarden](https://ollygarden.com/) with contributions from Dash0, New Relic, Splunk, Datadog, and Grafana Labs. It evaluates OTLP telemetry streams against boolean rules weighted by impact level.

### Scoring Formula

```text
Score = SUM(P_i x W_i) / SUM(T_i x W_i) x 100
```

| Impact Level | Weight |
|---|---|
| Critical | 40 |
| Important | 30 |
| Normal | 20 |
| Low | 10 |

| Score Range | Category |
|---|---|
| 90-100 | Excellent |
| 75-89 | Good |
| 50-74 | Needs Improvement |
| 0-49 | Poor |

### Adopted Rules (19 rules, 5 categories)

These rules evaluate the telemetry output produced by the instrumented code. We adopt them as-is from the spec.

#### RES: Resource Attributes

| Rule | Description | Impact |
|---|---|---|
| RES-001 | `service.instance.id` must be present | Normal |
| RES-002 | `service.instance.id` must be unique across instances of the same `service.name` | Important |
| RES-003 | `k8s.pod.uid` must be present when running in Kubernetes | Important |
| RES-004 | Semantic convention attributes must appear at their designated OTLP level (resource, span, log, metric) | Important |
| RES-005 | `service.name` must be present with a non-empty string value | Critical |

#### SPA: Spans

| Rule | Description | Impact |
|---|---|---|
| SPA-001 | No more than 10 INTERNAL spans per trace per service | Normal |
| SPA-002 | No orphan spans (every span with a parent_span_id must have a matching parent) | Normal |
| SPA-003 | Span names must have bounded cardinality (no embedded literals like user IDs or URLs) | Important |
| SPA-004 | Root spans must not be of kind CLIENT (indicates missing instrumentation or context loss) | Important |
| SPA-005 | No more than 20 spans per trace with duration < 5ms | Important |

#### MET: Metrics

| Rule | Description | Impact |
|---|---|---|
| MET-001 | Metric attribute keys must have < 10,000 unique values per hour | Important |
| MET-002 | All metrics must have a non-default unit compliant with UCUM | Important |
| MET-003 | All time series for a metric name must use the same unit over 14 days | Important |
| MET-004 | All histogram time series for a metric name must use the same buckets over 14 days | Normal |
| MET-005 | Metric names must not contain the unit (e.g., `collection.duration.seconds` is wrong) | Normal |
| MET-006 | Metric names must not equal semantic convention attribute keys | Important |

#### LOG: Logs

| Rule | Description | Impact |
|---|---|---|
| LOG-001 | Debug-level logs must not persist in production beyond 14 days | Important |
| LOG-002 | Log records must have `severityNumber` configured (no UNSET severity) | Important |

#### SDK: SDK Configuration

| Rule | Description | Impact |
|---|---|---|
| SDK-001 | Language and runtime versions must be within SDK-supported values | Low |

---

## Code-Level Evaluation

The code-level dimensions evaluate the **source code** produced by an AI instrumentation agent. Unlike the Instrumentation Score (which monitors runtime OTLP streams continuously), code-level evaluation assesses a point-in-time code diff with the goal of iterating on the agent — fixing prompts, adjusting guardrails, tuning behavior.

### Two-Score Model

| Score | Evaluates | Input | Methodology |
|---|---|---|---|
| Instrumentation Score | Runtime telemetry quality | OTLP data streams | Weighted boolean formula (per IS spec) |
| Code-Level Evaluation | Generated code quality | Source code diff | Gate checks + dimension profiles (below) |

These are reported separately. The Instrumentation Score uses its own weighted formula as designed — it was built for that purpose. The Code-Level Evaluation uses a different structure optimized for agent iteration.

### Evaluation Structure

Code-level rules are organized into three layers:

**Layer 1 — Gate Checks**: Binary must-pass preconditions. If any gate fails, the file (or run) is marked as failed and quality scoring is skipped — there is nothing meaningful to score. Gates represent "the agent broke something" or "the agent fundamentally misunderstood its role," not quality gradations.

**Layer 2 — Dimension Profiles**: For files/runs that pass all gates, each dimension is scored independently as a pass rate (e.g., "Coverage: 4/6 rules passed"). The per-dimension profile is the primary diagnostic output — it tells you where the agent is strong and where it struggles.

**Layer 3 — Per-File Detail**: Each file gets its own dimension profile. The run-level view aggregates across files (e.g., "COV-002 failed on 3 of 12 files: api-client.ts, db-connector.ts, queue-publisher.ts"). This tells you exactly which files and rules to investigate when iterating on the agent.

### Impact Levels

Rules retain impact levels (Critical, Important, Normal, Low) as **prioritization hints** within each dimension — "fix this first" — not as weights in a scoring formula. When a dimension has multiple failures, address higher-impact rules first.

### Evaluation Scope

Each rule specifies its evaluation scope:

- **Per-run**: Evaluated once for the entire agent run (e.g., NDS-001: compilation succeeds)
- **Per-file**: Evaluated for each file the agent modifies (e.g., CDQ-002: tracer acquired correctly)
- **Per-instance**: Evaluated for each relevant code site within a file (e.g., COV-002: each outbound call site)

Per-instance rules are reported as ratios at the file level (e.g., "3 of 4 outbound calls have spans in api-client.ts") and aggregated at the run level. Pass/fail thresholds for per-instance ratios are defined in the scoring criteria (see Milestone 3 in the PRD).

---

## Gate Checks

The following rules are binary preconditions. If any gate fails, quality scoring is not meaningful for the affected scope.

| Gate | Description | Scope | Failure Means |
|---|---|---|---|
| NDS-001 | TypeScript compilation succeeds after instrumentation | Per-run | Agent broke the build |
| NDS-002 | All pre-existing tests pass without modification | Per-run | Agent broke existing behavior |
| NDS-003 | Non-instrumentation lines in the agent's diff are identical to the original file | Per-file | Agent modified business logic |
| API-001 | Instrumented code imports only from `@opentelemetry/api`, not SDK packages | Per-file | Agent used wrong dependency model |

---

## Dimensions: Quality Rules

The following dimensions evaluate instrumentation quality for files/runs that pass all gates. The Instrumentation Score spec evaluates OTLP data streams at runtime. These dimensions extend it to evaluate the **source code** produced by an AI instrumentation agent — concerns that only exist when reviewing generated code, not telemetry output.

### Dimension 1: Non-Destructiveness (NDS)

**Definition**: The instrumented codebase preserves existing behavior beyond what gate checks verify. The agent's changes do not introduce subtle breakage that passes compilation and tests.

**Rationale**: Gate checks catch hard failures (won't compile, tests fail, business logic altered). These rules catch softer violations where the code still works but the agent made changes it shouldn't have — altered error propagation, changed function signatures, or modified public API contracts.

**What it measures**:
- Public API contracts preserved (parameters, return types, exports)
- Error handling behavior unchanged (exceptions not swallowed, re-thrown differently, or converted)

**Grounded in**: Universal across all sources (implicit prerequisite). Academic survey of 10 industrial microservice systems found "the quality of the trace data is low and problems such as incomplete or erroneous traces are popular," attributed to system complexity and heterogeneous technology stacks ([PMC8629732](https://pmc.ncbi.nlm.nih.gov/articles/PMC8629732/)).

**Scoring Rules**:

| Rule | Description | Impact | Scope |
|---|---|---|---|
| NDS-004 | No existing public API signatures changed (parameters, return types, exports) | Important | Per-file |
| NDS-005 | No existing error handling behavior altered (exceptions not swallowed, re-thrown differently, or converted by span management code) | Important | Per-file |

*NDS-001, NDS-002, and NDS-003 are gate checks (see above).*

### Dimension 2: Coverage (COV)

**Definition**: The agent instruments the appropriate code paths — the ones that provide meaningful observability value — using the appropriate method (auto-instrumentation library vs. manual span).

**Rationale**: Coverage measures whether the agent chose the *right things* to instrument and the *right method*. The OTel library guidelines specify instrumenting "public APIs making network calls or performing long-running I/O operations" and "request/message handlers," while explicitly excluding thin wrappers and internal implementation details. Auto-instrumentation libraries are preferred over manual spans when available, because libraries handle edge cases (connection pooling, middleware chains) that manual spans miss, and they auto-update with framework changes.

**What it measures**:
- Important code paths are instrumented
- Operations that can fail have error visibility
- Auto-instrumentation libraries preferred when available
- The agent did not skip important code paths

**Grounded in**: [OTel Library Instrumentation Guidelines](https://opentelemetry.io/docs/concepts/instrumentation/libraries/) ("Public APIs, handlers, operations that can fail"). Academic survey of 10 industrial systems observed 90-100% service coverage rates, with gaps attributed to non-core or legacy services ([PMC8629732](https://pmc.ncbi.nlm.nih.gov/articles/PMC8629732/)).

**Scoring Rules**:

| Rule | Description | Impact | Scope |
|---|---|---|---|
| COV-001 | Entry point operations (request handlers, exported service functions) have spans | Critical | Per-instance |
| COV-002 | Outbound calls (HTTP clients, database queries, external service calls) have spans | Important | Per-instance |
| COV-003 | Operations that can fail have any error visibility in the span (presence of error recording, not correctness of pattern) | Important | Per-instance |
| COV-004 | Long-running or asynchronous operations have spans | Normal | Per-instance |
| COV-005 | Spans on high-priority operations include attributes that capture domain-specific context beyond OTel defaults | Normal | Per-instance |
| COV-006 | Operations covered by available auto-instrumentation libraries do not have manual spans | Important | Per-instance |

### Dimension 3: Restraint (RST)

**Definition**: The agent avoids over-instrumentation. Not every function gets a span. Trivial utility functions, getters/setters, thin wrappers, and already-instrumented code are left alone.

**Rationale**: This is the inverse of coverage — measuring what should NOT be instrumented. Over-instrumentation is the most commonly cited anti-pattern across sources. Honeycomb's practitioner guide calls wrapping everything in spans "the most common failure mode." Elastic warns against instrumenting trivial functions that create "a huge amount of telemetry data with very low additional value."

**What it measures**:
- No spans on trivial utility functions (getters, setters, formatters)
- No spans on thin wrappers where the underlying call is already instrumented
- No spans on internal implementation details users cannot map to their code
- No re-instrumentation of already-instrumented code

**Grounded in**: [Honeycomb Practitioner's Guide](https://jeremymorrell.dev/blog/a-practitioners-guide-to-wide-events/) ("wrapping everything in its own span is the most common failure mode"). [Elastic OTel Best Practices](https://www.elastic.co/observability-labs/blog/best-practices-instrumenting-opentelemetry) ("huge amount of telemetry data with very low additional value"). [OTel Library Guidelines](https://opentelemetry.io/docs/concepts/instrumentation/libraries/) (do not instrument thin wrappers or internal details). SPA-001 and SPA-005 from the Instrumentation Score spec also address this at the runtime level.

**Scoring Rules**:

| Rule | Description | Impact | Scope |
|---|---|---|---|
| RST-001 | Pure utility functions (formatters, validators, type guards) do not have spans | Important | Per-instance |
| RST-002 | Getters, setters, and trivial property accessors do not have spans | Low | Per-instance |
| RST-003 | Thin wrappers around already-instrumented calls do not create duplicate spans | Important | Per-instance |
| RST-004 | Internal implementation details (private helpers, iteration callbacks) do not have spans | Normal | Per-instance |
| RST-005 | Functions that already contain tracer calls (`startActiveSpan`, `startSpan`) are not re-instrumented | Important | Per-instance |

### Dimension 4: API-Only Dependency (API)

**Definition**: The instrumented code depends only on the OpenTelemetry API package, not the SDK. In Node.js, `@opentelemetry/api` is a `peerDependency`.

**Rationale**: This is a hard rule from the OTel specification. Libraries "should only use the OpenTelemetry API" — the SDK is the deployer's choice. Multiple API instances in `node_modules` cause silent trace loss via no-op fallbacks. Sigelman emphasizes portability: vendor-specific instrumentation is a quality defect.

**What it measures**:
- Correct dependency declaration in `package.json`
- No vendor-specific instrumentation SDKs
- No SDK-internal imports

**Grounded in**: [OTel Client Design Principles](https://opentelemetry.io/docs/specs/otel/library-guidelines/) ("API and SDK MUST be provided as independent artifacts"). [OTel Library Guidelines](https://opentelemetry.io/docs/concepts/instrumentation/libraries/) ("Libraries should only use the OpenTelemetry API").

**Scoring Rules**:

| Rule | Description | Impact | Scope |
|---|---|---|---|
| API-002 | `@opentelemetry/api` is declared as the correct dependency type (`peerDependency` for libraries, `dependency` for applications) | Important | Per-run |
| API-003 | No vendor-specific instrumentation SDKs in dependencies | Important | Per-run |
| API-004 | No direct imports from SDK-internal modules (exporters, processors, samplers) | Important | Per-file |

*API-001 is a gate check (see above).*

### Dimension 5: Schema Fidelity (SCH)

**Definition**: The instrumentation conforms to the project's Weaver semantic conventions registry. Span names, attribute keys, and metric names match the schema definitions.

**Rationale**: commit-story-v2 has a Weaver registry at `telemetry/registry/` that defines the project's telemetry schema. The agent should use these definitions rather than inventing ad-hoc span names and attributes. OTel Weaver enables automated validation of instrumentation against a registry.

**What it measures**:
- Span names match registry-defined operations
- Attribute keys use registry-defined names
- Attribute values conform to registry-defined types and constraints
- No redundant schema entries when equivalents exist

**Grounded in**: [OTel Weaver Blog](https://opentelemetry.io/blog/2025/otel-weaver/) (policy-based validation, "telemetry like a public API"). [Grafana: Instrumentation Quality](https://grafana.com/docs/grafana-cloud/monitor-applications/application-observability/setup/instrumentation-quality/) (quality checks against conventions).

**Scoring Rules**:

| Rule | Description | Impact | Scope |
|---|---|---|---|
| SCH-001 | Span names match operations defined in the project's telemetry registry | Critical | Per-instance |
| SCH-002 | Attribute keys use registry-defined names | Important | Per-instance |
| SCH-003 | Attribute values conform to registry-defined types and constraints | Important | Per-instance |
| SCH-004 | Agent does not create new schema entries when semantically equivalent definitions already exist in the registry | Important | Per-instance |

### Dimension 6: Code Quality (CDQ)

**Definition**: The instrumented code is clean, correct, and maintainable. Instrumentation uses proper OTel patterns and does not introduce code-level risks.

**Rationale**: Sigelman argues instrumentation "should be part of the code base like unit tests." Poorly integrated instrumentation becomes technical debt that developers remove rather than maintain. The OTel library guidelines include specific patterns for proper span management (try/finally, isRecording checks, context propagation).

**What it measures**:
- Spans properly closed in all code paths
- OTel API patterns used correctly (tracer acquisition, error recording, context propagation)
- No unsafe attribute values

**Grounded in**: Ben Sigelman via [SE Radio](https://se-radio.net/2018/09/se-radio-episode-337-ben-sigelman-on-distributed-tracing/) ("instrumentation should be part of the code base like unit tests"). [OTel Library Guidelines](https://opentelemetry.io/docs/concepts/instrumentation/libraries/) (span management patterns, isRecording checks). [Better Stack Best Practices](https://betterstack.com/community/guides/observability/opentelemetry-best-practices/) (proper closure, active span context).

**Scoring Rules**:

| Rule | Description | Impact | Scope |
|---|---|---|---|
| CDQ-001 | Spans are closed in all code paths, including error paths (try/finally or equivalent) | Critical | Per-instance |
| CDQ-002 | Tracer is acquired with a library name argument; version argument recommended but not required per OTel API spec | Normal | Per-file |
| CDQ-003 | Error recording uses the OTel-standard pattern (`recordException` + `setStatus(ERROR)`), not ad-hoc attribute setting | Important | Per-instance |
| CDQ-005 | Async functions using `await` correctly maintain span context within the function body | Important | Per-instance |
| CDQ-006 | `span.isRecording()` checked before setting attributes whose values require function calls, iteration, or serialization | Low | Per-instance |
| CDQ-007 | Span attributes do not contain potentially unbounded values (full request/response bodies, large arrays) or PII | Important | Per-instance |

---

## Dimension Summary

| # | Dimension | Prefix | Gates | Quality Rules | Source | Evaluates |
|---|---|---|---|---|---|---|
| — | Instrumentation Score | RES/SPA/MET/LOG/SDK | — | 19 | [instrumentation-score/spec](https://github.com/instrumentation-score/spec) | Runtime telemetry quality |
| 1 | Non-Destructiveness | NDS | 3 | 2 | All sources (implicit) | Code still works |
| 2 | Coverage | COV | — | 6 | OTel guidelines, academic survey | Right things instrumented, right method |
| 3 | Restraint | RST | — | 5 | Honeycomb, Elastic, OTel guidelines | Wrong things NOT instrumented |
| 4 | API-Only Dependency | API | 1 | 3 | OTel spec, Sigelman | Correct dependency model |
| 5 | Schema Fidelity | SCH | — | 4 | OTel Weaver, Grafana | Matches project registry |
| 6 | Code Quality | CDQ | — | 6 | Sigelman, OTel guidelines, Better Stack | Clean integration |

**Total**: 19 Instrumentation Score rules + 30 Code-Level rules (4 gates + 26 quality rules) across 11 categories.

---

## Evaluation Method Classification

Each rule is classified by how it can be evaluated. The guiding question is not "can a human do better?" but **"what is the cost of a false positive or false negative?"** For agent iteration (the stated purpose of this rubric), false positives are cheap — you inspect the flagged rule, see it's wrong, and move on. False negatives are more costly but acceptable in early iterations. This tilts toward automating aggressively and accepting imperfect precision.

### Categories

| Category | Definition | Implication for PRD #2 |
|---|---|---|
| **Automatable** | Script produces a pass/fail signal with no human input; may have false positives, but the cost of inspecting and dismissing a false positive (seconds) is far lower than the cost of manual review on every iteration (hours) | Build the check; run it on every agent iteration |
| **Semi-automatable** | Script provides partial signal but cannot produce a meaningful pass/fail; human interprets the output | Used only for IS static analysis, where some rules fundamentally require runtime data |
| **Not checkable / Not evaluable** | No automated signal possible at this layer | Deferred to the other evaluation layer (runtime or static), or excluded from single-run evaluation |

28 of 30 code-level rules are fully automatable. Two code-level rules (NDS-005, SCH-004) are semi-automatable — the script flags structural changes or similarity matches, but determining whether the flagged item is a true problem requires semantic judgment. The not-checkable category applies only to Instrumentation Score rules at the static analysis layer (the IS rules were designed for runtime evaluation, so static analysis provides partial coverage by design).

### Code-Level Rules (30 rules)

#### Gate Checks

All gates are automatable. If a gate cannot be automated, it cannot serve as a precondition.

| Rule | Classification | Mechanism |
|---|---|---|
| NDS-001 | Automatable | Run `tsc --noEmit`; exit code 0 = pass |
| NDS-002 | Automatable | Run existing test suite; all tests pass = pass |
| NDS-003 | Automatable | Diff analysis: filter instrumentation-related additions (import lines, tracer acquisition, `startActiveSpan`/`startSpan` calls, `span.setAttribute`/`recordException`/`setStatus`/`end` calls, try/finally blocks wrapping span lifecycle); remaining diff lines must be empty |
| API-001 | Automatable | AST/grep: all `@opentelemetry/*` imports resolve to `@opentelemetry/api` only |

#### Non-Destructiveness (NDS)

| Rule | Classification | Mechanism | Notes |
|---|---|---|---|
| NDS-004 | Automatable | AST: diff all exported function signatures (parameters, return types, export declarations) before/after agent run | TypeScript `export` is unambiguous — exported = public, unexported = internal. No judgment needed. |
| NDS-005 | Semi-automatable | AST: detect structural changes to pre-existing `try`/`catch`/`finally` blocks, reordered catch clauses, merged error handling blocks, and modified throw statements in the agent's diff; flag any modification to existing error handling structure | The script catches the easy cases: new try/finally that re-throws (benign, expected instrumentation pattern) vs. agent never touching existing error handling (pass). The hard case is the agent *restructuring* existing error handling to accommodate instrumentation — merging try/catch blocks, reordering catch clauses, wrapping an existing throw in new logic. The AST detects that the structure changed, but whether the restructured version preserves original propagation semantics (same exception types, same re-throw behavior, same catch ordering) is not a definitive pass/fail. Subtle behavioral changes like swallowed exceptions or changed error types have a meaningful false-negative rate under pure structural analysis. |

#### Coverage (COV)

| Rule | Classification | Mechanism | Notes |
|---|---|---|---|
| COV-001 | Automatable | AST: detect framework from `package.json` dependencies, then find entry point operations using framework-specific patterns (Express: `app.get/post/put/delete()`, `router.*()` callbacks; Fastify: `fastify.get/post()`, route handlers; raw http: `createServer()` callback; exported async functions from service modules) ; verify each has a span | Framework detection from `package.json` makes entry point patterns enumerable. If the codebase uses an unrecognized framework, the check produces false negatives — acceptable for agent iteration. Discovered gaps are added to the pattern list. |
| COV-002 | Automatable | AST: detect outbound call sites using dependency-derived patterns (`fetch()`, `axios.*()`, `pg.query()`, `redis.*()`, `amqp.publish()`, database client method calls, HTTP client methods); verify each has a span | Same framework-detection approach as COV-001. The outbound call pattern list is enumerable per-dependency and maintained alongside the check. |
| COV-003 | Automatable | AST: for each COV-001/COV-002 site plus any operation in a pre-existing try/catch block, verify the enclosing span has any error recording call (`recordException`, `setStatus`, or `span.setAttribute` with error-related keys) | "Can fail" is operationalized as: outbound calls (COV-002 sites), entry points (COV-001 sites), and operations already in try/catch blocks. All three sources are automatable. Checks presence of error visibility, not correctness of pattern (CDQ-003 handles correctness). |
| COV-004 | Automatable | AST: find `async` functions, functions containing `await` expressions, and calls to known I/O libraries (fs, net, stream, database clients); verify each has a span | Edge case: CPU-bound computation that happens to be long-running is not detected by this heuristic. The false-negative rate is acceptable for agent iteration. |
| COV-005 | Automatable | Compare `setAttribute` calls in instrumented code against the project's Weaver registry: for each span, check whether required/recommended attributes from the registry definition are present | The registry encodes what domain-specific attributes belong on each span. The human judgment happened when the registry was designed, not at evaluation time. |
| COV-006 | Automatable | Check whether manual spans target operations covered by known auto-instrumentation libraries (express, pg, mysql, redis, http, grpc, etc.); flag manual spans on those operations | The OTel contrib repo publishes auto-instrumentation packages with well-defined scope. The mapping from framework to auto-instrumentation package is static and maintained upstream. If a library is missing from the list, the false negative is acceptable — discovered gaps are added to the list. |

#### Restraint (RST)

| Rule | Classification | Mechanism | Notes |
|---|---|---|---|
| RST-001 | Automatable | AST: flag spans on functions that are synchronous, under ~5 lines, unexported, and contain no I/O calls (no `await`, no calls to known I/O libraries) | The spec defines concrete heuristics for "utility function." All criteria are AST-checkable. |
| RST-002 | Automatable | AST: flag spans on `get`/`set` accessor declarations and trivial property accessor methods (single return statement returning a property) | Well-defined syntactic pattern. |
| RST-003 | Automatable | AST: flag spans on functions whose body is a single return statement calling another function (possibly with argument transformation) | "Single-call wrapper" is well-defined in AST terms. |
| RST-004 | Automatable | AST: flag spans on unexported functions and private class methods | Same principle as NDS-004: TypeScript `export` is unambiguous. Unexported = internal, exported = public. The "exported for testing" edge case is not worth a separate classification — the function chose to be part of the public API by being exported, and instrumentation is reasonable on public APIs. |
| RST-005 | Automatable | AST: detect functions that already contain `startActiveSpan`, `startSpan`, or `tracer.` calls in the pre-agent source; flag if the agent adds additional tracer calls | Pre-existing tracer calls are unambiguous in the AST. |

#### API-Only Dependency (API)

| Rule | Classification | Mechanism | Notes |
|---|---|---|---|
| API-002 | Automatable | Parse `package.json`: verify `@opentelemetry/api` is in `peerDependencies` (for libraries) or `dependencies` (for applications) | Library vs. application distinction may need a project-level config flag. |
| API-003 | Automatable | Parse `package.json`: check all dependencies against a list of known vendor-specific instrumentation packages (e.g., `dd-trace`, `@newrelic/telemetry-sdk`, `@splunk/otel`) | The vendor package list is static and maintainable. |
| API-004 | Automatable | AST/grep: flag imports from `@opentelemetry/sdk-*`, `@opentelemetry/exporter-*`, `@opentelemetry/instrumentation-*`, or any `@opentelemetry/*` path that is not `@opentelemetry/api` | Import paths are unambiguous string literals. |

#### Schema Fidelity (SCH)

| Rule | Classification | Mechanism | Notes |
|---|---|---|---|
| SCH-001 | Automatable | Extract span name string literals from `startActiveSpan`/`startSpan` calls; compare against operation names in the resolved registry JSON | Span names are string literals (or should be — dynamic span names would also fail SPA-003). |
| SCH-002 | Automatable | Extract attribute key strings from `setAttribute`/`setAttributes` calls; compare against attribute names in the resolved registry JSON | Attribute keys are string literals. |
| SCH-003 | Automatable | For registry attributes with defined types (enum, int, string) and constraints (allowed values, ranges), verify attribute values in code match; for variable values, use TypeScript type inference to resolve types | Type checking (string vs. number) is deterministic. Enum constraints (value in allowed set) are deterministic. Variable types resolve via TypeScript's type system. The remaining edge case (dynamic values whose types aren't inferrable) is rare in typed TypeScript codebases. |
| SCH-004 | Semi-automatable | Flag any agent-added attribute key NOT in the registry; for flagged keys, compute string/token similarity against all registry entries (e.g., Jaccard similarity on hyphen-split tokens > 0.5) and flag matches above threshold | String/token similarity catches obvious duplicates (`http.request.duration` vs. `http_request_duration`). It does not catch semantic equivalence across different naming conventions (`http.request.duration` vs. `request.latency`). "No flag" means "no obvious redundancy," not "no redundancy." The automated part (similarity flagging) is genuinely useful, but the absence of a flag is not a definitive pass — it's an absence of obvious failure. For agent iteration the false-negative cost is low (missed duplicates surface later), but this is not a definitive pass/fail check. |

#### Code Quality (CDQ)

| Rule | Classification | Mechanism | Notes |
|---|---|---|---|
| CDQ-001 | Automatable | AST: verify every `startActiveSpan`/`startSpan` call has a corresponding `span.end()` in a `finally` block (or the span is managed by a callback passed to `startActiveSpan`) | The two OTel span patterns (callback-based and manual) both have well-defined AST shapes. |
| CDQ-002 | Automatable | AST/grep: verify `trace.getTracer()` calls include a library name string argument | Presence of a string argument is unambiguous. Version argument recommended but not required. |
| CDQ-003 | Automatable | AST: in catch/error handling blocks, verify error recording uses `span.recordException(error)` + `span.setStatus({ code: SpanStatusCode.ERROR })`, not ad-hoc `span.setAttribute('error', ...)` | The standard pattern has a well-defined AST shape. Ad-hoc patterns are detectable by checking for `setAttribute` with error-related keys without a corresponding `recordException`. |
| CDQ-005 | Automatable | AST: for `startActiveSpan()` callback pattern, context is automatically managed — pass. For `startSpan()` manual pattern, verify `context.with()` is used to wrap any async operations within the span's scope | In Node.js with `AsyncLocalStorage` (the standard OTel context manager), the rules are deterministic. Callback-based `startActiveSpan` handles context automatically. Manual `startSpan` requires explicit `context.with()` for async. Both patterns have well-defined AST shapes. |
| CDQ-006 | Automatable | AST: detect `setAttribute` calls whose value argument contains function calls, method chains (`.map`, `.reduce`, `.join`, `.filter`), or serialization (`JSON.stringify`) without a preceding `span.isRecording()` check in the same scope | "Expensive" is enumerable: function calls, iteration methods, serialization in the value expression. Simple variables and literals are not expensive. Low impact rule — deprioritize for initial evaluation but the check itself is deterministic. |
| CDQ-007 | Automatable | AST: flag `setAttribute` calls where the value is a full object spread, `JSON.stringify` of a request/response object, or an array without bounded length; flag attribute keys matching known PII field patterns (`email`, `password`, `ssn`, `phone`, `creditCard`, `address`, `*_name` for person names) | Unbounded value detection (object spreads, `JSON.stringify`, unbounded arrays) is unambiguous. PII detection uses a maintained field-name pattern list. False positives (e.g., `service.name` matching `*_name`) are cheap to dismiss during agent iteration — better to over-flag than miss PII exposure. |

### Instrumentation Score Rules (20 rules)

These rules evaluate runtime OTLP data. The **runtime classification** is the primary perspective — these rules were designed for runtime evaluation via Weaver live-check or OTLP collector analysis. The **static analysis classification** is supplementary — what can be caught earlier from source code, before running tests.

| Rule | Runtime | Static | Notes |
|---|---|---|---|
| RES-001 | Automatable | Semi-automatable | **Runtime**: check OTLP resource attributes for `service.instance.id`. **Static**: grep SDK init/configuration for `service.instance.id` setup; may be set via environment variable or auto-detection, so absence in code doesn't mean absence at runtime. |
| RES-002 | Automatable (multi-instance) | Semi-automatable | **Runtime**: compare `service.instance.id` across instances of same `service.name`. Requires multiple instances running — single test run can verify presence but not uniqueness. **Static**: check if ID generation uses UUID or similar unique source. |
| RES-003 | Automatable (conditional) | Semi-automatable | **Runtime**: check OTLP resource for `k8s.pod.uid` when K8s environment detected. Not evaluable in local test runs. **Static**: verify K8s resource detector is configured in SDK setup. |
| RES-004 | Automatable | Semi-automatable | **Runtime**: validate attribute placement in OTLP data (resource-level vs. span-level vs. log-level). **Static**: AST can check that code sets attributes on the correct object type (resource builder vs. span), but some attributes are set dynamically. |
| RES-005 | Automatable | Automatable | **Runtime**: check OTLP resource for non-empty `service.name`. **Static**: grep SDK configuration for `service.name` setting — presence and non-empty value are both verifiable in code. |
| SPA-001 | Automatable | Semi-automatable | **Runtime**: count INTERNAL-kind spans per trace per service in OTLP output. **Static**: count `startActiveSpan` calls with `{ kind: SpanKind.INTERNAL }` or default kind (INTERNAL is default), but actual trace shape depends on execution paths — static count is an upper bound. |
| SPA-002 | Automatable | Not checkable | **Runtime**: verify every span with `parent_span_id` has a matching parent in the trace. **Static**: parent-child relationships are determined at runtime by context propagation; no static analysis can determine span topology. |
| SPA-003 | Automatable | Automatable | **Runtime**: check unique span name count over time window. **Static**: grep span name arguments for template literals with dynamic interpolation (`${variable}`); flag any span name that is not a string literal or constant. |
| SPA-004 | Automatable | Semi-automatable | **Runtime**: check span kind of root spans in OTLP output. **Static**: verify entry point handlers create SERVER or INTERNAL spans, but which spans become root spans depends on incoming context propagation at runtime. |
| SPA-005 | Automatable | Not checkable | **Runtime**: check span durations in OTLP output. **Static**: duration is fundamentally a runtime measurement; no static analysis can predict it. |
| MET-001 | Automatable (traffic-dependent) | Semi-automatable | **Runtime**: count unique values per metric attribute key over time window. Requires sufficient traffic volume to evaluate cardinality. **Static**: flag metric attributes sourced from high-cardinality inputs (user IDs, timestamps, request URLs). |
| MET-002 | Automatable | Automatable | **Runtime**: check metric definitions in OTLP for non-default unit. **Static**: grep metric creation calls (e.g., `meter.createHistogram()`) for `unit` argument presence and UCUM compliance. |
| MET-003 | Not evaluable (single run) | Semi-automatable | **Runtime**: requires 14-day observation window to verify unit consistency. Not evaluable from a single test run. **Static**: verify all code paths that create the same-named metric use the same unit argument. Catches inconsistency within a single codebase snapshot, though not temporal drift. |
| MET-004 | Not evaluable (single run) | Semi-automatable | **Runtime**: requires 14-day observation window for bucket consistency. Not evaluable from a single test run. **Static**: verify all histogram creations for the same metric name use the same bucket boundaries. Same caveat as MET-003. |
| MET-005 | Automatable | Automatable | **Runtime**: regex check on metric names for embedded unit strings. **Static**: grep metric name arguments for patterns containing unit strings (`.seconds`, `.bytes`, `.milliseconds`, etc.). |
| MET-006 | Automatable | Automatable | **Runtime**: cross-reference metric names with attribute keys in OTLP. **Static**: cross-reference metric name strings in code with attribute keys from the registry. |
| LOG-001 | Not evaluable (operational) | Not checkable | Requires production log pipeline retention configuration review. Not evaluable from a test run or source code — this is an infrastructure/operational concern. |
| LOG-002 | Automatable | Semi-automatable | **Runtime**: check OTLP log records for `severityNumber` presence. **Static**: grep log creation calls for severity level configuration; absence in code may mean the logging framework sets a default. |
| SDK-001 | Automatable | Automatable | **Runtime**: check resource attributes for language/runtime version. **Static**: check `package.json` `engines` field and `@opentelemetry/*` package versions against SDK compatibility matrix. |

### Summary

#### Code-Level Rules (30)

| Classification | Count | Rules |
|---|---|---|
| Automatable | 28 | NDS-001, NDS-002, NDS-003, NDS-004, API-001, COV-001, COV-002, COV-003, COV-004, COV-005, COV-006, RST-001, RST-002, RST-003, RST-004, RST-005, API-002, API-003, API-004, SCH-001, SCH-002, SCH-003, CDQ-001, CDQ-002, CDQ-003, CDQ-005, CDQ-006, CDQ-007 |
| Semi-automatable | 2 | NDS-005, SCH-004 |
| Human-only | 0 | — |

**93% automatable (28/30), 7% semi-automatable (2/30), 0% human-only.** The two semi-automatable rules share a common trait: both involve semantic equivalence that structural/syntactic analysis cannot definitively resolve. NDS-005 requires judging whether restructured error handling preserves propagation semantics. SCH-004 requires judging whether differently-named schema entries refer to the same concept. In both cases, the automated check (structural diff, string similarity) provides genuine signal, but "no flag" is not a definitive pass.

**LLM-assisted evaluation candidate.** Both semi-automatable rules are strong candidates for LLM-as-judge evaluation. NDS-005 (given a before/after diff, does the restructured error handling preserve propagation semantics?) and SCH-004 (given the registry and an agent-added attribute, is it semantically equivalent to an existing entry?) are exactly the kind of semantic reasoning LLMs handle reliably. A script + LLM judge pipeline could bring the effective automation rate to 30/30 — fully automatable with no specialized human knowledge required. That's a meaningful property: the entire evaluation rubric can run without a telemetry expert in the loop. Design of the LLM judge is deferred until PRD #2 results show actual agent behavior — which rules the agent fails, how it fails them, and whether the evaluation harness can serve as an inner-loop validation stage in the agent's own fix cycle (see PRD #1 Decision Log).

The 28 automatable rules succeed because their definitions can be operationalized into deterministic checks: framework-specific patterns are enumerable from `package.json`, TypeScript's `export` keyword makes public/internal unambiguous, the Weaver registry encodes domain knowledge, and field-name pattern lists enable over-flagging where the cost of a false positive (seconds to dismiss) is far lower than the cost of manual review on every agent iteration.

#### Instrumentation Score Rules (19)

*Note: The IS rules adopted from the spec total 19 (RES: 5, SPA: 5, MET: 6, LOG: 2, SDK: 1). The "20" in the Adopted Rules header above is a counting error to be corrected.*

| Runtime Classification | Count | Rules |
|---|---|---|
| Automatable | 16 | RES-001, RES-002, RES-003, RES-004, RES-005, SPA-001, SPA-002, SPA-003, SPA-004, SPA-005, MET-001, MET-002, MET-005, MET-006, LOG-002, SDK-001 |
| Not evaluable (single run) | 3 | MET-003, MET-004, LOG-001 |

Of the 16 automatable rules, three have evaluation scope caveats: RES-002 (requires multi-instance comparison for uniqueness), RES-003 (conditional on K8s environment — not applicable in local test runs), and MET-001 (requires sufficient traffic volume for cardinality assessment).

| Static Analysis Classification | Count | Rules |
|---|---|---|
| Automatable | 6 | RES-005, SPA-003, MET-002, MET-005, MET-006, SDK-001 |
| Semi-automatable | 10 | RES-001, RES-002, RES-003, RES-004, SPA-001, SPA-004, MET-001, MET-003, MET-004, LOG-002 |
| Not checkable | 3 | SPA-002, SPA-005, LOG-001 |

**Runtime**: 16 of 19 rules are automatable from a single test run with OTLP collection. Three rules (MET-003, MET-004, LOG-001) are fundamentally not evaluable without production-scale observation or infrastructure configuration review.

**Static**: 6 of 19 rules can catch issues before running tests. These are the cheapest checks and should run first during agent iteration. The 10 semi-automatable rules provide partial early signals (upper bounds, suspicious patterns) that are worth flagging even if not definitive.
