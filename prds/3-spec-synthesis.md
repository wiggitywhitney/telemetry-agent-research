# PRD #3: Spec Synthesis & Design Recommendations

**Issue**: [#3](https://github.com/wiggitywhitney/telemetry-agent-research/issues/3)
**Status**: In Progress
**Priority**: Medium
**Blocked by**: PRD #2 (Run & Evaluate) — Complete
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

*Milestones refined 2026-02-25 based on PRD #2 evaluation findings. Implementation phasing added as a standalone milestone (Decision 5). Spec milestone split into rubric gap fixes + spec section (Decision 6). JS support and two-tier validation scheduled.*

- [x] **Evaluation patterns identified**: Group PRD #2 findings into themes — what worked, what didn't, what was missing. Categorize by: architecture, agent behavior, schema handling, code quality, coverage decisions. → `evaluation/patterns.md`
- [x] **Implementation phasing documented** (Decision 5): Dependency-graph-driven 7-phase build plan for the next implementation, with two-tier validation architecture and spec section maps per phase. Grounded in evaluation findings — each phase boundary addresses specific failure modes from PRD #2. Spec section maps ensure future phase PRDs reference the right spec content without losing details. → `docs/specs/research/implementation-phasing.md`
- [x] **Rubric gap fixes** (Decision 6): Update the evaluation rubric with gaps discovered during PRD #2 — CDQ-008 (tracer naming consistency), RST-004 refinement (I/O boundary exception), CDQ-007 update (conditional attribute setting). Do this before the spec update because the spec references the updated rubric.
- [ ] **Spec updated with evaluation criteria** (Decisions 6, 11): Add to the spec: evaluation philosophy (why unit tests aren't enough, grounded in Theme 2), rubric dimension summary, two-tier validation architecture as a spec-level commitment (structural + semantic tiers, both feeding the fix loop), required verification levels beyond unit tests (e2e smoke test, interface wiring, validation chain integration, progress verification), and JavaScript support (file discovery + language-neutral gate framing). References the rubric and phasing documents — does not duplicate them. **Edit constraints (Decision 11):** Spec edits are primarily additive (new Evaluation & Acceptance Criteria section, v3.6 revision history). Modifications limited to: file discovery (JS support), NDS-001 framing (language-neutral), validation chain (Tier 2 addition), model configurability (more prominent). All other sections stay verbatim. Edits done one at a time with diffs reviewed individually. Build section inventory before and after to verify untouched sections are unchanged.
- [ ] **Architectural recommendations documented**: Assessment of current architecture (Coordinator + Agent, SDK choice, Weaver integration) with specific recommendations for next iteration.
- [ ] **Tech stack evaluation complete**: Research and recommendation on orchestration frameworks, SDK approaches, and tooling for the next implementation. Grounded in what we learned from evaluating the current one.
- [ ] **Design document drafted**: Blueprint for the next implementation that preserves what works and addresses what doesn't. Includes architectural decisions, tech stack choices, and key design changes.
- [ ] **Repository README written** (Decision 4): Write the project README using `/write-docs` skill once all research phases are complete. Covers the full research arc, repo structure, navigation guide, and how findings connect to the public spec.

## Open Questions

- How much of the current architecture is worth preserving vs. rebuilding?
- Does LangGraph add value for agent orchestration, or is direct SDK sufficient?
- Should the Cluster Whisperer codebase serve as a secondary validation target for the next implementation?

### Resolved Questions

- ~~Should evaluation criteria live in the spec or in a separate evaluation framework document?~~ → Yes, in the spec. Decision 2: "Success metrics and quality dimensions are general enough to be part of the spec."
- ~~What's the right boundary between spec (what to build) and evaluation (how to judge it)?~~ → The spec gets the evaluation framework and two-tier validation architecture. The rubric document gets detailed rules. The phasing document gets build-phase gates. See Decisions 5, 6, and 7.
- ~~Should the evaluation harness serve double duty as an inner-loop validation stage in the agent's fix loop?~~ → Yes. Decision 7: rubric rules become Tier 2 of a two-tier validation chain. Tier 2 checks produce the same structured feedback format as Tier 1 and feed into the fix loop. Critical/Important rules are blocking; Normal/Low are advisory.

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
| 2026-02-25 | Implementation phasing separated into its own milestone | The 7-phase build plan is a design artifact that stands on its own — not a subsection of the spec's evaluation criteria. The phasing document captures dependency-graph analysis, phase boundaries grounded in evaluation failures, and acceptance gates with rubric rule mappings. The spec references it rather than defining phases inline. |
| 2026-02-25 | Spec milestone split: rubric gap fixes first, then spec section | The spec update references the rubric. Rubric gap fixes (CDQ-008, RST-004, CDQ-007) must land first so the spec references a complete rubric. Also: bundling rubric updates, spec sections, and phasing into one milestone mixes distinct workstreams that deserve separate review. |
| 2026-02-25 | Two-tier validation is a spec-level architectural decision | The rubric's automatable rules (COV-002, RST-001, CDQ-001, NDS-003) are concrete AST-based deterministic checks that produce the same structured feedback the fix loop consumes. The spec must define Tier 1 (structural: "does the code work?") and Tier 2 (semantic: "is the instrumentation correct?") as part of the validation chain architecture. Without this, the next builder will build structural-only validation and miss semantic issues. |
| 2026-02-25 | JavaScript support included in Phase 1 and spec update | The evaluation identified JS support as "the one spec change with the highest leverage" (Theme 5). It affects file discovery (`**/*.{ts,js}`), validation approach (`tsc --noEmit` vs `node --check`), and gate NDS-001 framing. Included in the spec update milestone and as a Phase 1 requirement in the phasing document. |
| 2026-02-25 | Basic Weaver validation in Phase 2, complex integration in Phase 5 | The spec defines `weaver registry check` as step 3 of the validation chain. Deferring all Weaver to Phase 5 repeats the first-draft's mistake (F17: Weaver disabled). Basic static validation belongs in Phase 2; schema extensions, checkpoints, drift detection, and live-check belong in Phase 5. |
| 2026-02-25 | Spec section maps added to phasing document | Each phase includes a table mapping it to exact spec sections and subsections. When a future phase PRD is created, the author copies the map as a checklist of what to reference. This prevents detail loss during PRD creation — the content stays in the spec (single source of truth), and the map ensures the right sections are included for each phase. |
| 2026-02-25 | Spec edit constraints for v3.6 update | Spec section maps protect future PRD creation but not the current spec edit. The v3.6 update must follow process discipline: edits are primarily additive (new sections). Modifications are explicitly scoped to file discovery (JS), NDS-001 (language-neutral), validation chain (Tier 2), model configurability. All other sections stay verbatim. Edits are sequential (one change at a time, diff reviewed), with a section inventory before/after to catch unintended deletions. |

## Dependencies

- PRD #2: Evaluation report must exist before synthesis begins
- The telemetry agent spec (v3.5) at `docs/specs/telemetry-agent-spec-v3.5.md` in this repo

## Blocks

- Future public implementation repo (not tracked here — separate project)
