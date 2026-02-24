# PRD #3: Spec Synthesis & Design Recommendations

**Issue**: [#3](https://github.com/wiggitywhitney/telemetry-agent-research/issues/3)
**Status**: Open
**Priority**: Medium
**Blocked by**: PRD #2 (Run & Evaluate)
**Created**: 2026-02-24

## Problem

After evaluating the first-draft implementation, we need to capture what worked, what didn't, and what should change. These findings should feed back into the telemetry agent spec and inform the architecture of a future implementation.

This is a research synthesis phase — no code output. The deliverables are documents: updated spec sections, architectural recommendations, and a design document for the next iteration.

## Solution

Analyze the evaluation report from PRD #2, identify patterns (strengths to preserve, weaknesses to address, gaps to fill), and produce:

1. Spec updates — evaluation criteria and success metrics added to the spec itself
2. Architectural recommendations — what to keep, what to change, tech stack considerations
3. Design document — blueprint for the next implementation iteration

## Scope

**In scope:**
- Analyzing PRD #2 evaluation results for patterns and themes
- Identifying what the current implementation does well (preserve in next iteration)
- Identifying what needs to change (architectural issues, missing capabilities, wrong abstractions)
- Tech stack evaluation (current: direct Anthropic SDK. Consider: LangGraph, other orchestration)
- Updating the telemetry agent spec with evaluation criteria and success metrics
- Producing a design document for the next implementation

**Out of scope:**
- Writing code or building a new implementation
- Forking or modifying the first-draft implementation
- Running additional evaluation passes (unless PRD #2 results are inconclusive)
- Making findings public (this repo is private; public artifacts come from a future implementation repo)

## Context

### What We'll Have From PRD #2
- Scored evaluation across all rubric dimensions
- CodeRabbit review analysis
- Raw observations on agent behavior
- Non-destructiveness validation results
- Specific examples of good and bad instrumentation decisions

### What This Phase Produces
- **Spec updates**: The evaluation rubric and success metrics are general enough to belong in the spec itself. Future implementations (by anyone) should be measured against these criteria.
- **Architectural assessment**: The Coordinator + per-file Agent architecture, direct SDK usage, Weaver CLI integration — what worked and what didn't at an architectural level.
- **Tech stack analysis**: Should the next iteration use LangGraph or similar orchestration? What about the Anthropic SDK choice? Are there better approaches for schema validation?
- **Design document**: Not a full spec rewrite, but a "here's what v2 of the implementation should look like" document that captures lessons learned.

## Milestones

*These milestones are preliminary and will be refined once PRD #2's evaluation report exists.*

- [ ] **Evaluation patterns identified**: Group PRD #2 findings into themes — what worked, what didn't, what was missing. Categorize by: architecture, agent behavior, schema handling, code quality, coverage decisions.
- [ ] **Spec updated with evaluation criteria**: Add evaluation rubric dimensions and success metrics to the telemetry agent spec. These become the acceptance criteria for any future implementation.
- [ ] **Architectural recommendations documented**: Assessment of current architecture (Coordinator + Agent, SDK choice, Weaver integration) with specific recommendations for next iteration.
- [ ] **Tech stack evaluation complete**: Research and recommendation on orchestration frameworks, SDK approaches, and tooling for the next implementation. Grounded in what we learned from evaluating the current one.
- [ ] **Design document drafted**: Blueprint for the next implementation that preserves what works and addresses what doesn't. Includes architectural decisions, tech stack choices, and key design changes.
- [ ] **Repository README written** (Decision 4): Write the project README using `/write-docs` skill once all research phases are complete. Covers the full research arc, repo structure, navigation guide, and how findings connect to the public spec.

## Open Questions (to resolve during or after PRD #2)

- How much of the current architecture is worth preserving vs. rebuilding?
- Does LangGraph add value for agent orchestration, or is direct SDK sufficient?
- Should evaluation criteria live in the spec or in a separate evaluation framework document?
- What's the right boundary between spec (what to build) and evaluation (how to judge it)?
- Should the Cluster Whisperer codebase serve as a secondary validation target for the next implementation?
- Should the evaluation harness serve double duty as an inner-loop validation stage in the agent's fix loop? (PRD #1 Decision: harness checks produce structured, machine-readable output — rule ID, pass/fail, file path, line number, actionable message. If PRD #2 shows the agent fails specific rules consistently, the harness could feed those failures back to the agent during instrumentation, not just after.)

## Success Criteria

- The spec is updated with concrete, measurable evaluation criteria
- Architectural recommendations are grounded in specific evidence from PRD #2
- The design document is actionable — someone could start a new implementation from it
- All findings stay in this private repo; only spec updates flow to the public spec
- Repository README written using the `/write-docs` skill, documenting the complete research arc, repo structure, and how to navigate findings (Decision 4)

## Decision Log

| Date | Decision | Rationale |
|------|----------|-----------|
| 2026-02-24 | No code output from this phase | This is research synthesis. Code comes from a future public implementation repo. |
| 2026-02-24 | Evaluation criteria should go into the spec | Success metrics and quality dimensions are general enough to be part of the spec, not just this evaluation. |
| 2026-02-24 | Private findings, public spec updates | This repo stays private. Spec updates (evaluation criteria, architectural guidance) can flow to the public spec. |
| 2026-02-24 | README deferred to PRD #3 completion | Too soon during PRD #1 — repo structure and findings will evolve through PRDs 2 and 3. Writing the README after synthesis means it documents the complete research arc, not a speculative snapshot. Use `/write-docs` skill to ensure validated content. |

## Dependencies

- PRD #2: Evaluation report must exist before synthesis begins
- The telemetry agent spec (v3.5) at `docs/specs/telemetry-agent-spec-v3.5.md` in this repo

## Blocks

- Future public implementation repo (not tracked here — separate project)
