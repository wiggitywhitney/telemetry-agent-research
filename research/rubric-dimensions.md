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

## Extensions: Code-Level Dimensions

The Instrumentation Score spec evaluates OTLP data streams at runtime. The following dimensions extend it to evaluate the **source code** produced by an AI instrumentation agent. These cover concerns that only exist when reviewing generated code, not telemetry output.

### Dimension 1: Non-Destructiveness

**Definition**: The instrumented codebase compiles, passes all existing tests, and preserves existing behavior. The agent's changes do not break anything.

**Rationale**: This is the most fundamental quality gate. Instrumentation that breaks the application is worse than no instrumentation at all. Every vendor and practitioner source treats this as an implicit requirement.

**What it measures**:
- TypeScript compilation succeeds after instrumentation
- All pre-existing tests still pass
- No runtime behavior changes (same outputs for same inputs)
- No removed or modified business logic

**Grounded in**: Universal across all sources (implicit prerequisite). Academic survey found "incomplete or erroneous traces" commonly caused by instrumentation that disrupted existing code ([PMC8629732](https://pmc.ncbi.nlm.nih.gov/articles/PMC8629732/)).

### Dimension 2: Coverage

**Definition**: The agent instruments the appropriate code paths — the ones that provide meaningful observability value. Important operations are covered; trivial ones are not.

**Rationale**: Coverage measures whether the agent chose the *right things* to instrument. The OTel library guidelines specify instrumenting "public APIs making network calls or performing long-running I/O operations" and "request/message handlers," while explicitly excluding thin wrappers and internal implementation details.

**What it measures**:
- Network-calling code paths are instrumented
- Request/message handlers are instrumented
- Operations that can fail have error visibility
- The agent did not skip important code paths

**Grounded in**: [OTel Library Instrumentation Guidelines](https://opentelemetry.io/docs/concepts/instrumentation/libraries/) ("Public APIs, handlers, operations that can fail"). Academic survey found 90-100% coverage targets in industrial systems ([PMC8629732](https://pmc.ncbi.nlm.nih.gov/articles/PMC8629732/)).

### Dimension 3: Restraint

**Definition**: The agent avoids over-instrumentation. Not every function gets a span. Trivial utility functions, getters/setters, and thin wrappers are left alone.

**Rationale**: This is the inverse of coverage — measuring what should NOT be instrumented. Over-instrumentation is the most commonly cited anti-pattern across sources. Honeycomb's practitioner guide calls wrapping everything in spans "the most common failure mode." Elastic warns against instrumenting trivial functions that create "a huge amount of telemetry data with very low additional value."

**What it measures**:
- No spans on trivial utility functions (getters, setters, formatters)
- No spans on thin wrappers where the underlying call is already instrumented
- No spans on internal implementation details users cannot map to their code
- Span depth is appropriate (not excessively nested)

**Grounded in**: [Honeycomb Practitioner's Guide](https://jeremymorrell.dev/blog/a-practitioners-guide-to-wide-events/) ("wrapping everything in its own span is the most common failure mode"). [Elastic OTel Best Practices](https://www.elastic.co/observability-labs/blog/best-practices-instrumenting-opentelemetry) ("huge amount of telemetry data with very low additional value"). [OTel Library Guidelines](https://opentelemetry.io/docs/concepts/instrumentation/libraries/) (do not instrument thin wrappers or internal details). SPA-001 and SPA-005 from the Instrumentation Score spec also address this at the runtime level.

### Dimension 4: API-Only Dependency

**Definition**: The instrumented code depends only on the OpenTelemetry API package, not the SDK. In Node.js, `@opentelemetry/api` is a `peerDependency`.

**Rationale**: This is a hard rule from the OTel specification. Libraries "should only use the OpenTelemetry API" — the SDK is the deployer's choice. Multiple API instances in `node_modules` cause silent trace loss via no-op fallbacks. Sigelman emphasizes portability: vendor-specific instrumentation is a quality defect.

**What it measures**:
- `package.json` dependencies include `@opentelemetry/api` as a `peerDependency` (for libraries) or `dependency` (for applications)
- No direct dependency on `@opentelemetry/sdk-*` packages in library code
- No vendor-specific instrumentation SDKs in library dependencies

**Grounded in**: [OTel Client Design Principles](https://opentelemetry.io/docs/specs/otel/library-guidelines/) ("API and SDK MUST be provided as independent artifacts"). [OTel Library Guidelines](https://opentelemetry.io/docs/concepts/instrumentation/libraries/) ("Libraries should only use the OpenTelemetry API").

### Dimension 5: Schema Fidelity

**Definition**: The instrumentation conforms to the project's Weaver semantic conventions registry. Span names, attribute keys, and metric names match the schema definitions.

**Rationale**: commit-story-v2 has a Weaver registry at `telemetry/registry/` that defines the project's telemetry schema. The agent should use these definitions rather than inventing ad-hoc span names and attributes. OTel Weaver enables automated validation of instrumentation against a registry.

**What it measures**:
- Span names match registry-defined operations
- Attribute keys use registry-defined names
- Attribute values conform to registry-defined types and constraints
- No ad-hoc telemetry that bypasses the registry

**Grounded in**: [OTel Weaver Blog](https://opentelemetry.io/blog/2025/otel-weaver/) (policy-based validation, "telemetry like a public API"). [Grafana: Instrumentation Quality](https://grafana.com/docs/grafana-cloud/monitor-applications/application-observability/setup/instrumentation-quality/) (quality checks against conventions).

### Dimension 6: Code Quality

**Definition**: The instrumented code is clean, idiomatic, and maintainable. Instrumentation is integrated naturally into the existing code style, not bolted on.

**Rationale**: Sigelman argues instrumentation "should be part of the code base like unit tests." Poorly integrated instrumentation becomes technical debt that developers remove rather than maintain. The OTel library guidelines include specific patterns for proper span management (try/finally, isRecording checks, context propagation).

**What it measures**:
- Spans are properly closed (try/finally or equivalent patterns)
- Instrumentation follows existing code style and conventions
- Tracer acquisition follows OTel patterns (library name and version provided)
- No unnecessary complexity added to existing code
- Error recording follows OTel conventions (span status, error.type attribute)

**Grounded in**: Ben Sigelman via [SE Radio](https://se-radio.net/2018/09/se-radio-episode-337-ben-sigelman-on-distributed-tracing/) ("instrumentation should be part of the code base like unit tests"). [OTel Library Guidelines](https://opentelemetry.io/docs/concepts/instrumentation/libraries/) (span management patterns, isRecording checks). [Better Stack Best Practices](https://betterstack.com/community/guides/observability/opentelemetry-best-practices/) (proper closure, active span context).

---

## Dimension Summary

| # | Dimension | Source | Evaluates |
|---|---|---|---|
| — | Instrumentation Score (20 rules) | [instrumentation-score/spec](https://github.com/instrumentation-score/spec) | Runtime telemetry quality |
| 1 | Non-Destructiveness | All sources (implicit) | Code still works |
| 2 | Coverage | OTel guidelines, academic survey | Right things instrumented |
| 3 | Restraint | Honeycomb, Elastic, OTel guidelines | Wrong things NOT instrumented |
| 4 | API-Only Dependency | OTel spec, Sigelman | Correct dependency model |
| 5 | Schema Fidelity | OTel Weaver, Grafana | Matches project registry |
| 6 | Code Quality | Sigelman, OTel guidelines, Better Stack | Clean integration |
