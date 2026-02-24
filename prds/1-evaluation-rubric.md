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

- [ ] **Research survey complete**: Web research on instrumentation quality standards from OTel community, observability vendors (Honeycomb, Datadog, Grafana, Lightstep), academic sources, and practitioner blogs. Raw findings captured in research notes.
- [ ] **Draft rubric dimensions defined**: Named list of evaluation dimensions with descriptions and rationale for inclusion. Each dimension has a clear definition of what it measures.
- [ ] **Scoring criteria per dimension**: For each dimension, define what constitutes poor/acceptable/good/excellent instrumentation. Include concrete examples where possible.
- [ ] **Automatable vs. human-judgment classification**: Each dimension tagged as automatable (can be checked by script), semi-automatable (script can flag, human confirms), or human-only (requires expert judgment).
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

## Dependencies

- None (this is the first phase)

## Blocks

- PRD #2 (Run & Evaluate) depends on this rubric
