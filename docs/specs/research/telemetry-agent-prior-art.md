# Prior Art: AI Instrumentation Tools

**Research Date:** 2026-02-05

## Executive Summary

The landscape of automated instrumentation tools includes AI-powered code analysis, schema-driven code generation, runtime auto-instrumentation, compile-time source transformation, and kernel-level eBPF observation. No existing tool combines Weaver schema + AI understanding + source code transformation.

---

## AI/LLM-Based Instrumentation Tools

### o11y.ai
- **What:** AI-powered observability assistant that scans GitHub repos, calculates "o11y Score", generates PRs to add OTel instrumentation
- **How:** Remote coding agent with hand-tuned prompts, analyzes frameworks in use
- **Languages:** TypeScript (others "coming soon")
- **Status:** Commercial SaaS with free beta
- **Gap:** Not schema-driven, uses prompts rather than Weaver schema

**Source:** [o11y.ai](https://www.o11y.ai/)

### OllyGarden Rose
- **What:** AI instrumentation agent for PR review, catches issues in manual instrumentation
- **How:** GitHub integration, context-aware analysis across codebase + OTel specs
- **Features:**
  - Automated PR reviews (available now)
  - One-click fixes (available now)
  - Agentic Instrumentation (coming soon)
- **Catches:** Missing context propagation, high cardinality, PII leaks
- **Status:** Research preview, launched October 2025

**Source:** [ollygarden.com/rose](https://ollygarden.com/rose)

### GitHub Copilot Custom Agents
- **What:** Specialized agents for observability tasks (Dynatrace, Elasticsearch)
- **How:** Model Context Protocol (MCP) integration
- **Status:** Commercial (Copilot subscription)

---

## Compile-Time / Source Transformation Tools

### Orchestrion (Datadog)
- **What:** Automatic compile-time instrumentation for Go
- **How:** AST manipulation based on imports, aspect-oriented programming pattern
- **Languages:** Go only
- **Status:** Open source (566 GitHub stars)
- **Key Insight:** Zero runtime overhead, full compiler verification

**Source:** [github.com/DataDog/orchestrion](https://github.com/DataDog/orchestrion)

### go-instrument
- **What:** CLI tool to add OTel spans to Go functions
- **How:** Go's standard `ast` library, finds functions with context.Context parameter
- **Languages:** Go only
- **Status:** Open source

**Source:** [github.com/nikolaydubina/go-instrument](https://github.com/nikolaydubina/go-instrument)

---

## LLM Observability Tools

### OpenLLMetry (Traceloop)
- **What:** OTel extensions for LLM applications
- **How:** Monkey patching at runtime, one-line init
- **Languages:** Python (primary), TypeScript, Go, Ruby
- **Frameworks:** OpenAI, Anthropic, LangChain, LlamaIndex, vector DBs
- **Status:** Open source (6.8k stars)

**Source:** [github.com/traceloop/openllmetry](https://github.com/traceloop/openllmetry)

### OpenLIT
- **What:** OTel-native GenAI observability with Kubernetes operator
- **How:** SDKs + K8s operator for zero-code instrumentation
- **Languages:** Python, TypeScript
- **Status:** Open source (2.2k stars)

**Source:** [github.com/openlit/openlit](https://github.com/openlit/openlit)

### OpenInference (Arize Phoenix)
- **What:** Semantic conventions for AI applications
- **How:** Standardized attribute names and span structures
- **Languages:** Python, JavaScript, Java
- **Status:** Open source (844 stars)

**Source:** [github.com/Arize-ai/openinference](https://github.com/Arize-ai/openinference)

### Langfuse
- **What:** LLM observability with decorator-based Python integration
- **How:** `@observe()` decorator, uses Contextvars for async-safe state
- **Languages:** Python, TypeScript
- **Status:** Open source (YC W23)

**Source:** [langfuse.com](https://langfuse.com/)

---

## eBPF-Based Tools

### Grafana Beyla
- **What:** Zero-code auto-instrumentation using eBPF
- **How:** Kernel-level probes, inspects executables and networking
- **Languages:** Language-agnostic
- **Requirements:** Linux kernel 5.8+
- **Status:** Open source (Grafana Labs)

**Source:** [grafana.com/oss/beyla-ebpf](https://grafana.com/oss/beyla-ebpf/)

### Odigos
- **What:** eBPF-based OTel automation for Kubernetes
- **How:** Instrumentation Devices as Kubernetes resources
- **Status:** Open source with enterprise tier

**Source:** [docs.odigos.io](https://docs.odigos.io/)

### Pixie
- **What:** eBPF observability for Kubernetes
- **How:** Kernel probes on syscalls, stores data locally
- **Status:** Open source (CNCF project)

**Source:** [px.dev](https://px.dev/)

---

## Key Insights

### Gap in the Market
No existing tool combines:
- OTel Weaver schema reading
- AI understanding of code semantics
- Source code transformation (not just runtime hooks)
- Schema extension capabilities

### Instrumentation Approaches Comparison

| Approach | Best For | Tradeoff |
|----------|----------|----------|
| Monkey Patching | Dynamic languages (Python, JS) | Runtime overhead |
| AST/Compile-time | Static languages (Go, Rust) | Build complexity |
| eBPF | Network/HTTP, Kubernetes | Linux-only, less business logic visibility |
| Proxy | LLM API observability | Only sees external calls |
| Decorators | Python/TS | Requires code changes |

### Common Patterns
1. Schema-first is emerging as best practice
2. LLM observability has standardized quickly (OpenLLMetry, OpenInference)
3. Kubernetes operators enable zero-code at scale
4. AI-assisted instrumentation is early stage (o11y.ai, Rose)
