# Instrumentation Quality Research Survey

Raw findings from researching instrumentation quality standards across the OpenTelemetry community, observability vendors, and academic/industry frameworks. This document supports PRD #1 (Evaluation Rubric).

---

## 1. OpenTelemetry Community Standards

### 1.1 Instrumentation Types and When to Use Each

OTel defines three instrumentation approaches:

- **Code-based (manual)**: Using APIs/SDKs to generate telemetry directly. Best for "deeper insight and rich telemetry from your application itself."
- **Zero-code (auto)**: Automatic instrumentation without code changes. Best "when you can't modify the application" or for broad coverage at the edges.
- **Library instrumentation**: Native observability built into third-party libraries.

The docs explicitly state: "You can use both solutions simultaneously." Neither approach alone is sufficient.

**Source:** [Instrumentation | OpenTelemetry](https://opentelemetry.io/docs/concepts/instrumentation/)

### 1.2 API vs SDK Dependency Rules

**Critical rule**: "Libraries should only use the OpenTelemetry API" and never depend on the SDK. The API is "abstractions and non-operational implementations" with zero overhead when no SDK is present.

- The API includes a minimal no-op implementation ensuring applications function correctly without an SDK
- The no-op must "incur as little performance penalty as possible"
- API and SDK "MUST be provided as independent artifacts"
- In Node.js, `@opentelemetry/api` must be a `peerDependency` to avoid silent trace loss from multiple instances

**Source:** [Libraries | OpenTelemetry](https://opentelemetry.io/docs/concepts/instrumentation/libraries/), [OpenTelemetry Client Design Principles](https://opentelemetry.io/docs/specs/otel/library-guidelines/)

### 1.3 What to Instrument (and What Not To)

**Instrument:**
- Public APIs making network calls or performing long-running I/O
- Request/message handlers
- Operations that can fail and require visibility

**Do NOT instrument:**
- Thin wrappers over documented APIs where OTel already supports the underlying client
- Internal implementation details users can't map to their code
- Trivial utility functions (getters/setters) causing "a huge amount of telemetry data with very low additional value"

**Source:** [Libraries | OpenTelemetry](https://opentelemetry.io/docs/concepts/instrumentation/libraries/), [Elastic OTel Best Practices](https://www.elastic.co/observability-labs/blog/best-practices-instrumenting-opentelemetry)

### 1.4 Span Naming Conventions

**Core pattern: `{verb} {object}`** — produces low-cardinality names with unique details in attributes.

| Anti-pattern | Correct | Reason |
|---|---|---|
| `process_payment_for_user_jane_doe` | `process payment` | User ID belongs in attributes |
| `send_invoice_#98765` | `send invoice` | Invoice number creates cardinality explosion |
| `validation_failed` | `validate user_input` | Name the operation, put outcome in span status |

**Domain-specific patterns:**
- HTTP: `GET /api/users/:ID` (method + route template, not actual path)
- Database: `INSERT my_database.users` (operation + resource)
- RPC: `com.example.UserService/GetUser` (method within service)

**Source:** [How to Name Your Spans | OpenTelemetry](https://opentelemetry.io/blog/2025/how-to-name-your-spans/)

### 1.5 Semantic Conventions Naming Rules

**MUST requirements:**
- Every name must be a valid Unicode sequence
- Limited to printable Basic Latin characters (U+0021..U+007E)
- Names starting with `otel.*` are reserved

**SHOULD requirements:**
- Lowercase formatting
- Dot-delimited namespacing (e.g., `service.version`)
- Snake_case for multi-word components (e.g., `http.response.status_code`)
- Precise and descriptive, avoiding ambiguity

**Pluralization:** Single entities use singular (`host.name`); multiple entities use plural with arrays (`process.command_args`).

**Metrics naming patterns:**
- `entity.limit` (constant total amounts)
- `entity.usage` (amount consumed from known total)
- `entity.utilization` (fraction of usage vs limit)
- `entity.time` (passage of time)
- `entity.io` (bidirectional data flow)

**Source:** [Naming | OpenTelemetry](https://opentelemetry.io/docs/specs/semconv/general/naming/), [Metrics Semantic Conventions](https://opentelemetry.io/docs/specs/semconv/general/metrics/)

### 1.6 Semantic Convention Requirement Levels

Each convention has a requirements level: **Required**, **Conditionally Required**, **Recommended**, or **Opt-In**. These guide how instrumentation libraries should implement and comply.

**Source:** [Semantic Conventions | OpenTelemetry](https://opentelemetry.io/docs/concepts/semantic-conventions/)

### 1.7 Quality Standards for Writing Conventions

- New conventions MUST have a group of codeowners
- Conventions not used by instrumentations MUST NOT be declared stable
- Prototyping in real instrumentation is strongly recommended before standardization
- Reuse existing attributes before defining new ones
- Avoid unbounded values (strings >1KB, arrays >1000 elements)
- Flag attributes containing PII with warnings
- When NOT to define spans: point-in-time events (use events), short operations without out-of-process calls

**Source:** [How to Write Semantic Conventions | OpenTelemetry](https://opentelemetry.io/docs/specs/semconv/how-to-write-conventions/)

### 1.8 OTel Weaver for Quality Validation

Weaver provides policy-based validation that "enforces best practices—naming, stability, immutability, and more."

- **Static registry checks**: Validates convention definitions follow patterns
- **Live-check**: Monitors running applications and generates compliance reports against registry specs
- **Custom Rego policies**: Enable organization-specific checks; "every attribute and signal is passed through the policy engine as it's received"
- Treats "telemetry like a public API" to prevent breaking changes

**Source:** [Observability by Design: OTel Weaver | OpenTelemetry](https://opentelemetry.io/blog/2025/otel-weaver/)

### 1.9 Performance and Anti-Patterns

**Performance rules:**
- Check `span.isRecording()` before calculating expensive attributes
- Supply sampling-critical attributes during span creation
- Use batch processing (typically 512 spans per batch)
- Set reasonable timeouts and queue limits

**Anti-patterns:**
- Creating instruments too frequently (expensive; meant for reuse)
- Synchronous span export (use batch processor)
- Creating spans in tight loops
- Unbounded attribute sizes
- Not closing spans (resource leaks, incomplete traces)
- Initializing OTel after importing instrumented libraries (misses early spans)
- 100% trace collection without sampling

**Source:** [Better Stack OTel Best Practices](https://betterstack.com/community/guides/observability/opentelemetry-best-practices/), [Libraries | OpenTelemetry](https://opentelemetry.io/docs/concepts/instrumentation/libraries/)

---

## 2. Observability Vendor Best Practices

### 2.1 Honeycomb

**Wide events philosophy**: Maturely instrumented datasets are "often 200-500 dimensions wide." Default to including attributes — "the marginal cost of each extra attribute is very small."

**Key anti-pattern — excessive child spans**: "Wrapping absolutely everything in its own span is the most common failure mode I see when engineers first get access to tracing tools." Wide events with timing breakdowns on a single span are easier to aggregate than nested child spans.

**Instrumentation as code review discipline**: "Just as you wouldn't accept a pull-request without tests, you should never accept a pull-request unless you can answer the question, 'how will I know when this isn't working?'"

**Progressive model (5-Star OTel):**
1. Auto-instrumentation first
2. Enable everything initially, then dial back
3. Leverage the OTel Collector
4. Sample strategically (tail sampling preferred)
5. Deploy trace-aware sampling (Refinery)

**Sources:**
- [A Practitioner's Guide to Wide Events](https://jeremymorrell.dev/blog/a-practitioners-guide-to-wide-events/)
- [5-Star OTel: Best Practices | Honeycomb](https://www.honeycomb.io/blog/opentelemetry-best-practices)
- [Observability: the Present and Future | Pragmatic Engineer](https://newsletter.pragmaticengineer.com/p/observability-the-present-and-future)

### 2.2 Datadog

**Operation names must be static and low-cardinality**: "When writing custom spans, statically set the span name to ensure that your resources are grouped with the same primary operation." Dynamic detail goes in the resource name.

**Unified service tagging**: Three standard tags tie all telemetry together: `env`, `service`, `version`.

**Anti-patterns:**
- High-cardinality tags in span names — "drastically increase storage costs"
- Environment values in service names (e.g., `prod-web-store`) — use primary tags
- Inconsistent sampling across services — causes incomplete or misleading traces
- Late SDK initialization — causes broken traces

**Proprietary friction**: Datadog's operation name / resource name distinction does not map cleanly to OTel's `span.name` semantics. This creates documented bugs.

**Sources:**
- [Primary Operations in Services | Datadog](https://docs.datadoghq.com/tracing/guide/configuring-primary-operation/)
- [Tagging Best Practices | Datadog](https://www.datadoghq.com/blog/tagging-best-practices/)

### 2.3 Grafana Labs

**Only vendor with a shipped instrumentation quality report product.** The Quality Report tab "reports on the completeness of your instrumentation and recommends steps to improve it."

**Specific quality checks:**
- No slashes in `service.name` or `service.namespace`
- High-cardinality span names flagged
- Missing Kubernetes resource attributes (k8s.cluster.name, k8s.namespace.name, k8s.pod.name)
- Missing signals (logs, profiles alongside traces)
- Mixed histogram types per service (silently loses data)

**CI/CD validation**: Promotes OTel Weaver's live-check as a pipeline tool — shift-left approach to telemetry quality.

**Most OTel-aligned vendor**: Quality criteria map directly to semantic conventions compliance without proprietary divergence.

**Source:** [Instrumentation Quality | Grafana Cloud](https://grafana.com/docs/grafana-cloud/monitor-applications/application-observability/setup/instrumentation-quality/)

### 2.4 Lightstep / Ben Sigelman

**Instrumentation should be treated as code**: "should be part of the code base like unit tests" and is "as important if not more important than a lot of those other things that we've come to accept will be part of the maintained code base."

**Portability as a quality criterion**: Choose "something portable for the instrumentation integration piece" — vendor-specific instrumentation is a quality defect.

**"Three pillars" myth**: Treating metrics, logging, and tracing as independent pillars is fundamentally flawed. "A holistic approach must be taken."

**The impossible quartet**: Can only choose three of four — high throughput, high cardinality, lengthy retention, unsampled data.

**Sources:**
- [SE Radio: Ben Sigelman on Distributed Tracing](https://se-radio.net/2018/09/se-radio-episode-337-ben-sigelman-on-distributed-tracing/)
- [Three Pillars with Zero Answers | InfoQ](https://www.infoq.com/news/2019/02/rethinking-observability/)

---

## 3. Evaluation Frameworks and Scoring

### 3.1 Instrumentation Score Specification

**The closest thing the industry has to a standard scoring framework.** Created by [OllyGarden](https://ollygarden.com/), a startup founded in January 2025 by Juraci Paixao Krohling (OTel maintainer) and Yuri Oliveira Sa, dedicated to OpenTelemetry instrumentation quality scoring. OllyGarden raised $1.6M pre-seed and builds their Insights product around this spec. The spec itself is community-driven, vendor-neutral, and Apache 2.0 licensed, with contributions from Dash0, New Relic, Splunk, Datadog, and Grafana Labs. Hosted at [score.olly.garden](https://score.olly.garden/).

**Score: 0-100 numerical value.**

Formula: `Score = SUM(P_i x W_i) / SUM(T_i x W_i) x 100`

| Range | Category | Interpretation |
|---|---|---|
| 90-100 | Excellent | High standard of instrumentation quality |
| 75-89 | Good | Solid quality; minor improvements possible |
| 50-74 | Needs Improvement | Tangible issues requiring attention |
| 0-49 | Poor | Significant problems needing urgent action |

**Impact levels (weighted):**
- Critical: Fundamental issues (highest weight)
- Important: Significant concerns
- Normal: Standard expectations
- Low: Minor enhancements (lowest weight)

**Rule structure:**
- ID (e.g., `RES-001`, `SPAN-042`)
- Description and rationale
- Boolean criteria (pass/fail)
- Target type (Resource, TraceSpan, Metric, Log)
- Impact level

**Example rule:** "Resource attributes MUST contain 'service.name' key with non-empty value" (RES-001, Critical)

**Key distinction from semantic conventions**: "Provides a higher-level view that considers telemetry quality at the point of use, measuring how well telemetry meets specific observability needs."

**Sources:**
- [Instrumentation Score Specification | GitHub](https://github.com/instrumentation-score/spec)
- [What Does Good Telemetry Look Like? | New Relic](https://newrelic.com/blog/best-practices/what-does-good-telemetry-look-like)

### 3.2 AWS Observability Maturity Model

Four stages of maturity:

1. **Foundational**: Basic telemetry, disparate tools, different teams use different tools
2. **Intermediate**: Metrics/logs/traces collected with defined visualizations and alerting, but extended troubleshooting times
3. **Advanced**: Root cause identification through signal correlation, 360-degree visibility
4. **Proactive**: Predictive issue detection, AI/ML-driven automated root cause analysis

Assesses readiness across 14 core questions spanning logs, metrics, traces, dashboards, alerting, and organizational strategy.

**Source:** [AWS Observability Maturity Model](https://aws-observability.github.io/observability-best-practices/guides/observability-maturity-model/)

### 3.3 Academic: Industrial Survey of Microservice Tracing

Key finding: "The quality of the trace data is low and problems such as incomplete or erroneous traces are popular" in large-scale deployments.

**Three quality dimensions identified:**
1. **Completeness**: Incomplete traces from exceptions interrupting span logging
2. **Accuracy**: Erroneous traces from inconsistent instrumentation across teams
3. **Coverage**: Observed service instrumentation coverage ranges 90-100% across 8 surveyed systems, with gaps attributed to non-core or legacy services

Evaluates three instrumentation techniques:
- Manual coding: Flexible but error-prone and intrusive
- Tracing frameworks: Preferred balance of effort and reliability
- Dynamic binary instrumentation: Non-intrusive but language-dependent

**Source:** [Enjoy Your Observability: Industrial Survey of Microservice Tracing | PMC](https://pmc.ncbi.nlm.nih.gov/articles/PMC8629732/)

---

## 4. Cross-Cutting Themes for Rubric Dimensions

The following themes emerge consistently across all source categories:

### Theme → Rubric Dimension Mapping

| Theme | Sources | Automatable? |
|---|---|---|
| **Semantic conventions compliance** (naming, attributes, schema) | OTel docs, Grafana quality report, Instrumentation Score spec, Weaver | Yes (Weaver, static analysis) |
| **Span name cardinality** (low-cardinality names, details in attributes) | OTel blog, Datadog, Grafana, Honeycomb | Yes (static analysis) |
| **Resource attribute completeness** (service.name, version, env) | Instrumentation Score (RES-001), Grafana checks, Datadog unified tagging | Yes (rule-based) |
| **Coverage / appropriate scope** (right things instrumented, not over/under) | OTel library guidelines, Elastic, Honeycomb, academic survey | Semi (static analysis for presence; human for appropriateness) |
| **Restraint** (no over-instrumentation, no trivial spans) | Honeycomb ("most common failure mode"), Elastic, OTel guidelines | Human judgment |
| **Non-destructiveness** (existing code compiles, tests pass, behavior preserved) | All (implicit) | Yes (test suite, build) |
| **API-only dependency** (libraries use API, not SDK) | OTel spec, Sigelman, CLAUDE.md global rules | Yes (dependency check) |
| **Context propagation correctness** (parent-child, W3C trace context) | OTel docs, Better Stack, Datadog | Semi (integration test) |
| **Code quality** (clean, idiomatic, maintainable instrumentation) | Sigelman ("instrumentation is code"), OTel library guidelines | Human judgment |
| **Privacy / PII safety** (no sensitive data in attributes) | OTel semconv guidelines, Grafana database conventions | Semi (pattern matching) |
| **Trace completeness** (spans closed, no resource leaks) | OTel docs, Better Stack, academic survey | Yes (static analysis, runtime check) |
| **Schema fidelity** (matches Weaver registry definitions) | OTel Weaver, Grafana, project-specific | Yes (Weaver validation) |

---

## 5. Sources Index

### OpenTelemetry Official

- [Instrumentation Concepts](https://opentelemetry.io/docs/concepts/instrumentation/)
- [How to Name Your Spans](https://opentelemetry.io/blog/2025/how-to-name-your-spans/)
- [Naming Conventions](https://opentelemetry.io/docs/specs/semconv/general/naming/)
- [Metrics Semantic Conventions](https://opentelemetry.io/docs/specs/semconv/general/metrics/)
- [Semantic Conventions Overview](https://opentelemetry.io/docs/concepts/semantic-conventions/)
- [How to Write Conventions](https://opentelemetry.io/docs/specs/semconv/how-to-write-conventions/)
- [Library Instrumentation Guidelines](https://opentelemetry.io/docs/concepts/instrumentation/libraries/)
- [Client Design Principles](https://opentelemetry.io/docs/specs/otel/library-guidelines/)
- [OTel Weaver Blog](https://opentelemetry.io/blog/2025/otel-weaver/)

### Vendor

- [Better Stack OTel Best Practices](https://betterstack.com/community/guides/observability/opentelemetry-best-practices/)
- [Honeycomb: Practitioner's Guide to Wide Events](https://jeremymorrell.dev/blog/a-practitioners-guide-to-wide-events/)
- [Honeycomb: 5-Star OTel](https://www.honeycomb.io/blog/opentelemetry-best-practices)
- [Honeycomb: Naming Best Practices](https://www.honeycomb.io/blog/opentelemetry-best-practices-naming)
- [Honeycomb: Observability Maturity Model](https://www.honeycomb.io/blog/observability-maturity-model)
- [Datadog: Primary Operations](https://docs.datadoghq.com/tracing/guide/configuring-primary-operation/)
- [Datadog: Tagging Best Practices](https://www.datadoghq.com/blog/tagging-best-practices/)
- [Grafana: Instrumentation Quality](https://grafana.com/docs/grafana-cloud/monitor-applications/application-observability/setup/instrumentation-quality/)
- [Elastic: Instrumenting OTel Best Practices](https://www.elastic.co/observability-labs/blog/best-practices-instrumenting-opentelemetry)

### Frameworks and Scoring

- [Instrumentation Score Specification](https://github.com/instrumentation-score/spec)
- [New Relic: What Does Good Telemetry Look Like?](https://newrelic.com/blog/best-practices/what-does-good-telemetry-look-like)
- [AWS Observability Maturity Model](https://aws-observability.github.io/observability-best-practices/guides/observability-maturity-model/)

### Academic

- [Industrial Survey of Microservice Tracing and Analysis](https://pmc.ncbi.nlm.nih.gov/articles/PMC8629732/)

### Key Practitioner Perspectives (cited within vendor sections)

- Ben Sigelman via [SE Radio](https://se-radio.net/2018/09/se-radio-episode-337-ben-sigelman-on-distributed-tracing/) and [InfoQ](https://www.infoq.com/news/2019/02/rethinking-observability/)
- Charity Majors via [Pragmatic Engineer](https://newsletter.pragmaticengineer.com/p/observability-the-present-and-future)
