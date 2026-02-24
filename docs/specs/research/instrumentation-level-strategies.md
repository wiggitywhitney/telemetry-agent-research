# Instrumentation Level Strategies

**Research Date:** 2026-02-05

## Executive Summary

The industry consensus has moved away from "instrument everything" toward selective, purposeful instrumentation. However, heavy development-time instrumentation for AI-assisted debugging is a valid use case distinct from production telemetry.

---

## "Instrument Everything + Filter" Approach

### What It Looks Like
- Enable auto-instrumentation for all libraries/frameworks
- Collect all log levels
- Generate spans for every operation
- Filter at collector level

### The Reality: It's a Trap

> "Feels right at first, but soon turns into a costly mess of observability noise."
> — Chronosphere

**Problems:**
- Cost explosion (storage, ingestion, query costs)
- Signal buried in noise
- Slower debugging (finding relevant data takes longer)
- Scaling compounds the problem

### Overhead Impact
- **CPU:** ~35% increase (OTel benchmarks)
- **Memory:** High cardinality tags significantly increase usage
- **Agent overhead:** 5-10% CPU typical

### When It Can Work
- Early in observability journey (unsure what's important)
- Backend handles high volumes at reasonable cost
- Debugging complex distributed issues
- Development-time (not production)

---

## Selective Instrumentation Approach

### The Golden Signals Framework (Google SRE)

"If you can only measure four metrics of your user-facing system, focus on these four":

1. **Latency** - Time to service a request (track success and error separately)
2. **Traffic** - Demand on system (requests/second)
3. **Errors** - Request failure rate
4. **Saturation** - System capacity utilization

### Business Value Heuristic

> "Will knowing the duration and outcome of this operation help me debug or optimize?"

**Sweet spot:** 5-15 custom spans per request for complex services

### High-Value Instrumentation Targets
- Business-critical paths (payment, order fulfillment)
- Customer-facing operations with SLOs
- Cross-service boundaries
- Database and external API calls
- Key state transitions

### Avoiding Under-Instrumentation

Under-instrumentation creates blind spots where:
- Problems occur but aren't detected
- Root cause analysis is impossible
- Debugging requires adding instrumentation, redeploying, waiting

**Mitigation:** Multiple instrumentation sources together (framework, network, targeted manual)

---

## Industry Best Practices

### Honeycomb's Philosophy

> "Observability is absolutely, utterly about instrumentation"

- Each event should be "as wide as possible, with as many high-cardinality dimensions as possible"
- Focus on "high-cardinality information, timing around network hops, and queries"
- "Think about future you, not current you"

### OpenTelemetry's Recommended Approach

**Start with auto-instrumentation:**
- Provides excellent baseline coverage
- Captures HTTP, database, external API calls
- No code changes required

**Layer manual instrumentation strategically:**
- Business-critical paths
- Custom code not covered by auto-instrumentation
- Domain-specific metrics

### Three-Tier Automation Strategy
1. **Framework-level:** OTel auto-instrumentation
2. **Kernel-level:** eBPF tools (Pixie, Cilium)
3. **Network-level:** Service meshes (Istio, Linkerd)

Then add manual instrumentation for business logic gaps.

---

## Filtering at the Collector Level

### OTel Collector Processors

**Filter Processor:**
- Uses OTTL (OpenTelemetry Transformation Language)
- Selectively drop telemetry based on attributes
- Example: Drop health check endpoints

**Attributes Processor:**
- Insert, update, delete attributes
- Add context (account_id, environment)

**Transform Processor:**
- Advanced customizations via OTTL
- Modify span status, extract attributes

### Sampling Strategies

**Head-Based Sampling:**
- Decision at trace start
- Lightweight, less resource-intensive
- Cannot consider entire trace outcome

**Tail-Based Sampling:**
- Decision after entire trace received
- Can prioritize errors, slow requests
- Requires buffering, more complex

**Best Practice:** Combine head + tail sampling + selective enrichment

---

## Configurable Instrumentation

### Environment-Based

**Log Levels:**
- Development: TRACE/DEBUG
- Production: INFO and above

**Sampling:**
- Development: 100%
- Production: 1-10%

### OpenTelemetry Environment Variables
```bash
export OTEL_TRACES_SAMPLER="parentbased_traceidratio"
export OTEL_TRACES_SAMPLER_ARG="0.1"  # 10% of traces
```

### Feature Flags
- Toggle verbose tracing for specific users/requests
- Enable debug instrumentation for specific services
- Control sampling rates dynamically

### Dynamic Instrumentation (Datadog)
Add instrumentation to running production systems without restarts:
- Dynamic Logs: Capture method parameters
- Dynamic Metrics: Analyze method usage
- Dynamic Spans: Add APM granularity

---

## Development vs Production

| Aspect | Development | Production |
|--------|-------------|------------|
| Log level | DEBUG/TRACE | INFO+ |
| Sampling | 100% | 1-10% |
| Collection | Direct to backend | Via Collector |
| All spans | Recorded | Sampled |
| Overhead | Acceptable | Minimized |

---

## Key Takeaways

1. **"Instrument everything" is NOT best practice** for production

2. **Consensus approach:**
   - Auto-instrumentation for baseline
   - Manual for business-critical paths
   - Filter early at collector
   - Use sampling

3. **Decision framework:**
   - Golden Signals
   - Business-critical operations
   - Cross-service boundaries
   - "Will this help me answer 'why'?"

4. **Development-time heavy instrumentation IS valid** for AI-assisted debugging (commit-story v1 use case)

5. **The agent should support configurable modes** - dev-heavy vs production-selective

---

## Sources

- [OpenTelemetry Instrumentation Concepts](https://opentelemetry.io/docs/concepts/instrumentation/)
- [OpenTelemetry Sampling](https://opentelemetry.io/docs/concepts/sampling/)
- [Google SRE Book - Monitoring](https://sre.google/sre-book/monitoring-distributed-systems/)
- [Honeycomb Observability Best Practices](https://www.honeycomb.io/blog/best-practices-for-observability)
- [Chronosphere - Observability Pipelines](https://chronosphere.io/learn/the-complete-guide-to-observability-pipelines-transform-your-telemetry-strategy/)
- [BetterStack OpenTelemetry Best Practices](https://betterstack.com/community/guides/observability/opentelemetry-best-practices/)
