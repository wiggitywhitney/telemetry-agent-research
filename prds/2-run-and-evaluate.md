# PRD #2: Run & Evaluate Reference Implementation

**Issue**: [#2](https://github.com/wiggitywhitney/telemetry-agent-research/issues/2)
**Status**: Open
**Priority**: High
**Blocked by**: PRD #1 (Evaluation Rubric)
**Created**: 2026-02-24

## Problem

We have a first-draft telemetry instrumentation agent and an evaluation rubric. We need to run the agent against a real codebase, observe what it produces, and score the results systematically.

## Solution

Fork commit-story-v2 on GitHub as a test target, run the first-draft implementation against it, collect CodeRabbit PR reviews on the instrumented changes, and score everything against the rubric from PRD #1. Produce a detailed evaluation report.

## Scope

**In scope:**
- Fork commit-story-v2 on GitHub as a persistent test target
- Configure and run the first-draft implementation against the fork
- Capture all output: file changes, schema extensions, agent logs, CodeRabbit reviews
- Score results against the evaluation rubric from PRD #1
- Document raw observations and scored evaluation report
- Multiple iterations if needed (run → observe → refine approach → re-run)

**Out of scope:**
- Modifying the first-draft implementation code
- Merging instrumented code back to canonical commit-story-v2
- Building automation tooling (manual evaluation first)
- Modifying the telemetry agent spec (that's PRD #3)

## Context

### Test Target: commit-story-v2 (fork)
- Canonical commit-story-v2 must stay uninstrumented (reserved for KubeCon demo)
- GitHub fork allows CodeRabbit PR reviews on instrumented changes
- Instrumented branches being public is fine
- 320 tests, full Vitest suite — strong baseline for non-destructiveness checks
- Weaver schema already exists at `telemetry/registry/`

### Reference Implementation
- 44 TypeScript files, 332 tests
- Coordinator + per-file Agent architecture
- Direct Anthropic SDK, Weaver CLI for schema validation
- Weaver CLI v0.21.2 installed locally
- Runs against a target codebase and produces instrumented code + schema extensions
- Creates feature branch and PR with summary

### Evaluation Framework
- Rubric from PRD #1 (dimensions, scoring criteria, automatable vs. human checks)
- CodeRabbit review as an additional signal (automated code review on the PR)

## Milestones

- [ ] **Test target ready**: commit-story-v2 forked on GitHub. Baseline captured: test results, coverage report, git state. Documented so the fork can be reset to baseline between runs.
- [ ] **Reference implementation runs successfully**: Agent executes against the fork without errors. Configuration documented (API keys, schema paths, target paths). Any setup issues (Weaver CLI, dependencies) resolved and documented.
- [ ] **First evaluation run complete**: Agent output captured — all file changes, schema extensions, agent logs. PR created on fork. CodeRabbit review received and captured.
- [ ] **Results scored against rubric**: Each rubric dimension scored with evidence. Raw observations documented alongside scores. Surprises and unexpected findings highlighted.
- [ ] **Evaluation report drafted**: Single document covering: what the agent did, how it scored, what worked well, what didn't, and specific observations that should inform PRD #3.

*Note: Milestones may be refined once PRD #1's rubric is complete — additional scoring dimensions may suggest additional evaluation steps.*

## Open Questions (to resolve during PRD #1 or early PRD #2)

- How does the first-draft implementation discover the Weaver schema? Does it expect a specific path?
- What configuration does the first-draft implementation need beyond API keys and target path?
- Should we run against the full commit-story-v2 codebase or a subset of files?
- How do we handle cost management for LLM calls during evaluation runs?
- Cluster Whisperer as a secondary target codebase — defer to after commit-story-v2 evaluation, and possibly defer to own implementation (PRD #3 follow-up).

## Success Criteria

- At least one complete evaluation run with all rubric dimensions scored
- CodeRabbit review captured and analyzed
- Non-destructiveness validated (tests still pass after instrumentation)
- Evaluation report is detailed enough to inform PRD #3 design decisions
- The fork can be reset and the evaluation repeated

## Decision Log

| Date | Decision | Rationale |
|------|----------|-----------|
| 2026-02-24 | GitHub fork (not local branch) for test target | Enables CodeRabbit PR reviews as an evaluation signal. |
| 2026-02-24 | Full runs instead of dry-run mode | Dry-run still costs tokens but produces less to evaluate. Real diffs + CodeRabbit reviews give better signal. |
| 2026-02-24 | Manual evaluation first, automation later | Need to understand what matters before automating. First couple iterations should be hands-on. |
| 2026-02-24 | Never merge back to canonical commit-story-v2 | Canonical repo stays uninstrumented for KubeCon demo. Fork is disposable. |

## Dependencies

- PRD #1: Evaluation rubric must exist before formal scoring begins
- Anthropic API key (for first-draft implementation LLM calls)
- GitHub access (for fork creation and CodeRabbit reviews)

## Blocks

- PRD #3 (Spec Synthesis) depends on this evaluation report
