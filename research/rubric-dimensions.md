# Evaluation Rubric: Dimensions

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

### Adopted Rules (20 rules, 5 categories)

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

**Grounded in**: Universal across all sources (implicit prerequisite). Academic survey found "incomplete or erroneous traces" commonly caused by instrumentation that disrupted existing code ([PMC8629732](https://pmc.ncbi.nlm.nih.gov/articles/PMC8629732/)).

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

**Grounded in**: [OTel Library Instrumentation Guidelines](https://opentelemetry.io/docs/concepts/instrumentation/libraries/) ("Public APIs, handlers, operations that can fail"). Academic survey found 90-100% coverage targets in industrial systems ([PMC8629732](https://pmc.ncbi.nlm.nih.gov/articles/PMC8629732/)).

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
| — | Instrumentation Score | RES/SPA/MET/LOG/SDK | — | 20 | [instrumentation-score/spec](https://github.com/instrumentation-score/spec) | Runtime telemetry quality |
| 1 | Non-Destructiveness | NDS | 3 | 2 | All sources (implicit) | Code still works |
| 2 | Coverage | COV | — | 6 | OTel guidelines, academic survey | Right things instrumented, right method |
| 3 | Restraint | RST | — | 5 | Honeycomb, Elastic, OTel guidelines | Wrong things NOT instrumented |
| 4 | API-Only Dependency | API | 1 | 3 | OTel spec, Sigelman | Correct dependency model |
| 5 | Schema Fidelity | SCH | — | 4 | OTel Weaver, Grafana | Matches project registry |
| 6 | Code Quality | CDQ | — | 6 | Sigelman, OTel guidelines, Better Stack | Clean integration |

**Total**: 20 Instrumentation Score rules + 30 Code-Level rules (4 gates + 26 quality rules) across 11 categories.
