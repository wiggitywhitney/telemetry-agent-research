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
- JavaScript (ESM) with JSDoc, Node.js

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
- [x] **Rubric mapped to commit-story-v2**: All 30 code-level rules mapped to specific code sites with evaluator actions. 50 exported signatures, 15 error handling sites, 39 utility functions, 8 entry points, 25+ outbound call sites, 27 registry attributes inventoried. Expected instrumentation topology documented. 6 review feedback items addressed. 4 new decision log entries added.
- [x] **Rubric document finalized**: Codebase-agnostic evaluation rubric at `research/evaluation-rubric.md` (564 lines, 19 IS rules + 30 code-level rules with mechanisms). External review addressed 8 issues: academic citation accuracy, IS spec version pinning (v0.1, commit `52c14ba`), CDQ-004 removal note, SCH-001 dual-mode classification, NDS-003 filter limitation, scoring approach clarification. Working draft `rubric-dimensions.md` marked as superseded. PRD #2 uses this rubric for "what to check" and the codebase mapping for "where to check it."

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
| 2026-02-24 | Rubric assumes target codebase has a test suite; NDS-002 is vacuously true without one | NDS-002 ("all pre-existing tests pass") is the only gate that catches behavioral regressions from instrumentation. Without tests, the gate passes but proves nothing — "not evaluable" rather than "pass." NDS-001 (compilation) catches syntax errors; NDS-003 (unchanged lines) catches accidental edits; neither catches semantic breakage. Finding: the telemetry agent spec should list a test suite as a prerequisite, with a harsh warning and explicit user approval required to proceed without one (not a hard block — real-world codebases without tests exist, but the user must accept the risk). |
| 2026-02-24 | Rubric-to-codebase mapping is an evaluator reference, not agent input | The mapping of rubric rules to specific commit-story-v2 code sites (entry points, outbound calls, utility functions, etc.) is an answer key for scoring agent output. The agent never sees it. Discovering the codebase structure is part of what we evaluate — giving the agent the mapping would invalidate the evaluation. |
| 2026-02-24 | NDS-001 gate generalizes beyond TypeScript | commit-story-v2 is JavaScript (ESM), not TypeScript. `tsc --noEmit` does not apply. The gate is "code compiles/parses after instrumentation" — for JS this means `node --check` or, more robustly, running the test suite (which also satisfies NDS-002). The rubric description should use language-neutral framing: "compilation or syntax validation succeeds." If the agent misidentifies the language (e.g., adds `.ts` files to a JS project), that is itself a gate failure. |
| 2026-02-24 | commit-story-v2 registry defines attributes but not operation names | The Weaver registry has 5 attribute groups but no explicit span name / operation definitions. SCH-001 ("span names match registry operations") cannot be strictly evaluated against the registry. For this evaluation, SCH-001 checks naming quality — bounded cardinality, consistent convention, meaningful mapping to code operations — rather than exact registry conformance. This is a gap in the target codebase's registry, not in the rubric. |

## Spec Update (Follow-up)

- **Target**: telemetry-agent-spec v3.5 prerequisites section
- **Change**: Add "target codebase should have a test suite" as a prerequisite check. The agent already validates prerequisites before proceeding. When no test suite is detected: issue a harsh warning explaining that non-destructiveness cannot be verified, require explicit user approval to continue, and log that NDS-002 was skipped.
- **Rationale**: Without tests, the agent cannot prove instrumentation preserves existing behavior. This is not a hard block (the agent can still add correct instrumentation), but the user must consciously accept the risk.

## Dependencies

- None (this is the first phase)

## Blocks

- PRD #2 (Run & Evaluate) depends on this rubric
