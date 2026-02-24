# PRD #1: Instrumentation Quality Evaluation Rubric

**Issue**: [#1](https://github.com/wiggitywhitney/telemetry-agent-research/issues/1)
**Status**: Open
**Priority**: High
**Created**: 2026-02-24

## Problem

We have a telemetry agent spec (v3.5) and a first-draft implementation ready to evaluate. But "evaluate" requires knowing what "good" looks like. Without a principled rubric grounded in industry standards, evaluation devolves into subjective impressions.

The instrumentation quality dimensions we care about are not well-codified in the industry — telemetry is often treated as an afterthought, and "good instrumentation" is tribal knowledge. We need to make it explicit.

## Solution

Research instrumentation quality from multiple angles (OpenTelemetry community standards, observability vendor best practices, academic literature, practitioner experience) and synthesize findings into a formal evaluation rubric with:

- Named dimensions (e.g., coverage, correctness, schema fidelity)
- Scoring criteria per dimension (what does a 1 vs. 3 vs. 5 look like?)
- Weighting guidance (which dimensions matter most?)
- Clear distinction between automatable checks and human-judgment checks

## Scope

**In scope:**
- Research survey of instrumentation quality standards
- Evaluation rubric document with scoring criteria
- Identification of which dimensions can be automated vs. require human review
- Mapping rubric dimensions to concrete checks against commit-story-v2 (320 tests, Weaver schema at `telemetry/registry/`)

**Out of scope:**
- Running the agent (that's PRD #2)
- Building automation tooling (that's a future concern)
- Modifying any codebase

## Context

### Target Codebase: commit-story-v2
- 320 tests across 11 files, all passing (~630ms)
- Vitest test framework with v8 coverage
- Weaver semantic conventions schema at `telemetry/registry/` (manifest, attributes, resolved JSON)
- Telemetry documentation at `docs/telemetry/`
- TypeScript/ESM, Node.js

### First-Draft Implementation
- 44 TypeScript files, 332 tests
- Coordinator + per-file Agent architecture
- Direct Anthropic SDK (no LangGraph)
- Weaver CLI for schema validation (v0.21.2 installed locally)
- Has dry-run mode (reverts file changes but still costs tokens — not useful for formal evaluation since we want real diffs and CodeRabbit PR reviews)

### Quality Dimensions to Research (initial list)
- **Coverage**: Are the right things instrumented? Are important code paths covered?
- **Correctness**: Are spans, attributes, and metrics used correctly per OTel conventions?
- **Schema fidelity**: Does instrumentation match the Weaver semantic conventions schema?
- **Restraint**: Does the agent avoid over-instrumenting? (Not every function needs a span.)
- **Non-destructiveness**: Does existing code still compile, pass tests, and behave correctly?
- **Library usage**: Does the agent use OTel SDK/API libraries rather than hand-rolling telemetry?
- **Code quality**: Is the instrumented code clean, idiomatic, and maintainable?
- **Semantic correctness**: Are span names, attribute keys, and metric names meaningful and consistent?

## Milestones

- [x] **Research survey complete**: Web research on instrumentation quality standards from OTel community, observability vendors (Honeycomb, Datadog, Grafana, Lightstep), academic sources, and practitioner blogs. Raw findings captured in research notes.
- [x] **Draft rubric dimensions defined**: Named list of evaluation dimensions with descriptions and rationale for inclusion. Each dimension has a clear definition of what it measures.
- [x] **Scoring criteria per dimension**: 30 binary rules (4 gates + 26 quality rules) across 6 dimensions with 3-layer evaluation structure (Gates → Dimension Profiles → Per-File Detail). Each rule has ID, description, impact level, and evaluation scope. Two external review cycles completed.
- [x] **Automatable vs. human-judgment classification**: 28 of 30 code-level rules fully automatable via AST/diff/registry checks; 2 semi-automatable (NDS-005, SCH-004) requiring semantic judgment — candidates for LLM-assisted evaluation. 19 IS rules classified for both runtime (16 automatable, 3 not evaluable) and static analysis (6 automatable, 10 semi-automatable, 3 not checkable).
- [ ] **Rubric mapped to commit-story-v2**: Each dimension has concrete examples of what to look for in commit-story-v2 specifically, referencing its test suite, schema, and code structure.
- [ ] **Rubric document finalized**: Single document in this repo that PRD #2 will use as its evaluation framework.

## Success Criteria

- The rubric is specific enough that two reviewers evaluating the same instrumentation output would arrive at similar scores.
- Every dimension is grounded in at least one external reference (not just opinion).
- The rubric accounts for the commit-story-v2 context (existing schema, test suite, code patterns).

## Decision Log

| Date | Decision | Rationale |
|------|----------|-----------|
| 2026-02-24 | Rubric before running the agent | Need to know what we're measuring before we measure it. Avoids rationalizing whatever output we get. |
| 2026-02-24 | Three separate PRDs (rubric → evaluate → synthesize) | Each phase has distinct deliverables and success criteria. Smaller PRDs are easier to track. |
| 2026-02-24 | Include automation classification in rubric | Future iterations should be repeatable. Knowing which checks can be automated informs PRD #3 design. |
| 2026-02-24 | Adopt Instrumentation Score spec as foundation, extend with code-level dimensions | Community-driven 0-100 scoring standard (by OllyGarden, with New Relic/Splunk/Dash0/Datadog/Grafana contributions) already covers runtime telemetry quality via 20 boolean rules. We extend it with 6 code-level dimensions for evaluating AI-generated instrumentation source code. Avoids reinventing what the industry has already standardized. |
| 2026-02-24 | Binary rules + 3-layer evaluation instead of qualitative scale | Binary rules maximize inter-rater reliability (PRD success criterion). The IS formula was designed for continuous runtime monitoring; code diff evaluation needs gates (preconditions) + dimension profiles (diagnostic output) + per-file detail (actionable). Qualitative scales add abstraction that hides useful signal for agent iteration. |
| 2026-02-24 | Removed CDQ-004 (incidental modifications) — redundant with NDS-003 gate | NDS-003 gate checks that non-instrumentation lines are identical to original. CDQ-004 couldn't fail independently. Edge case (unnecessary instrumentation lines) is a restraint concern, not code quality. |
| 2026-02-24 | CDQ-002 demoted to Normal; version argument optional | OTel API spec makes version argument optional. Telemetry agent spec's own examples omit version. Rule shouldn't penalize following the spec's examples. |
| 2026-02-24 | IS rule count corrected from 20 to 19 | Recount of adopted Instrumentation Score rules (RES: 5, SPA: 5, MET: 6, LOG: 2, SDK: 1) totals 19, not 20. |
| 2026-02-24 | Classification uses false-positive-cost framing, not "can a human do better?" | For agent iteration, false positives are cheap (seconds to inspect and dismiss). This tilts toward automating aggressively and accepting imperfect precision. Result: 28/30 code-level rules are fully automatable. |
| 2026-02-24 | Two semi-automatable rules (NDS-005, SCH-004) are candidates for LLM-assisted evaluation | Both rules involve semantic equivalence judgments that LLMs handle well (error handling preservation, schema entry deduplication). Script + LLM judge could make the rubric fully automatable (30/30) with no specialized human knowledge required. Design deferred until PRD #2 results show actual agent behavior — which rules fail, how they fail, and whether the harness can serve as an inner-loop validation stage. |
| 2026-02-24 | Evaluation harness checks should produce structured, machine-readable output | Output format: rule ID, pass/fail, file path, line number, actionable message. This enables future use as an inner-loop validation stage in the agent's own fix loop, where rubric feedback supplements the existing syntax/lint/Weaver chain. The harness is not just for evaluating the agent after the fact — it can guide the agent during instrumentation. Design of the feedback loop depends on PRD #2 results showing which rules the agent actually fails. |

## Dependencies

- None (this is the first phase)

## Blocks

- PRD #2 (Run & Evaluate) depends on this rubric
