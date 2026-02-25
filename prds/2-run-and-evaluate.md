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

### Test Target: commit-story-v2-eval (separate repo)
- Canonical commit-story-v2 must stay uninstrumented (reserved for KubeCon demo)
- New public repo `wiggitywhitney/commit-story-v2-eval` with full history pushed from canonical
- CodeRabbit installed; evaluation PRs from `evaluation/run-N` branches target main
- Self-referential: commit-story git hook + MCP server point at itself — every commit auto-generates journal entries, and after instrumentation, the repo produces its own telemetry
- 320 tests, full Vitest suite — strong baseline for non-destructiveness checks
- Weaver schema already exists at `telemetry/registry/`
- Local clone at `~/Documents/Repositories/commit-story-v2-eval`

### Reference Implementation
- Located at `~/Documents/Repositories/telemetry-agent-spec-v3/`
- 44 TypeScript files, 332 tests
- Coordinator + per-file Agent architecture (fresh LLM instance per file, no context carryover)
- Direct Anthropic SDK, Weaver CLI for schema validation
- CLI: `telemetry-agent instrument <path> --config telemetry-agent.yaml`
- Requires `telemetry-agent.yaml` in target repo with `schemaPath` and `sdkInitFile` at minimum
- Requires `ANTHROPIC_API_KEY` env var and Weaver CLI >= v0.21.2 in PATH
- Hybrid 3-attempt fix strategy: initial generation → multi-turn fixing → fresh regeneration
- Schema re-resolution between files (agents see extensions from prior agents)
- Produces instrumented code, schema extensions, SDK init file, and PR description markdown
- Exit codes: 0 (success), 1 (partial failure), 2 (total failure), 3 (user abort)

### Evaluation Framework
- Rubric from PRD #1 (dimensions, scoring criteria, automatable vs. human checks)
- CodeRabbit review as an additional signal (automated code review on the PR)

## Milestones

- [x] **Test target ready**: commit-story-v2-eval repo created on GitHub with full history. Baseline captured: test results (320 passing), coverage (83.19%), git state (`9c6e4a1`). Self-referential commit-story hook + MCP. Reset/run procedure documented in `evaluation/baseline/`.
- [x] **Reference implementation runs successfully**: Agent executed via MCP interface (8 runs; runs 1-7 failed due to validation chain issues, run 8 succeeded with validation bypassed). Configuration documented in `evaluation/run-1/README.md`. 25 findings documented including 3 patches applied to unblock evaluation. Setup issues (CLI unwired, in-memory FS, shadowing checker, lint checker) resolved and documented.
- [x] **First evaluation run complete**: Agent output captured (7 files, 7 spans, 22 attributes, 25 findings). Schema extensions documented as empty (Finding 21). PR created: [commit-story-v2-eval#1](https://github.com/wiggitywhitney/commit-story-v2-eval/pull/1). CodeRabbit review: 8 actionable + 2 nitpicks, zero issues with instrumentation itself.
- [x] **Results scored against rubric**: 24 quality rules scored across 6 dimensions (79% pass rate). Gate checks: 2 pass, 2 not independently verified. Strongest: CDQ 100%, API 100%, NDS 100%. Weakest: SCH 33% (domain mismatch artifact). 7 surprises documented with PRD #3 implications. Full scores in `evaluation/run-1/rubric-scores.md`.
- [ ] **Evaluation report drafted**: Single document covering: what the agent did, how it scored, what worked well, what didn't, and specific observations that should inform PRD #3.

*Note: Milestones may be refined once PRD #1's rubric is complete — additional scoring dimensions may suggest additional evaluation steps.*

## Open Questions (to resolve during PRD #1 or early PRD #2)

- ~~How does the first-draft implementation discover the Weaver schema? Does it expect a specific path?~~ **Resolved**: Via `schemaPath` in `telemetry-agent.yaml` config (e.g., `./telemetry/registry`).
- ~~What configuration does the first-draft implementation need beyond API keys and target path?~~ **Resolved**: Requires `telemetry-agent.yaml` with `schemaPath` and `sdkInitFile` at minimum. Also needs `ANTHROPIC_API_KEY` env var and Weaver CLI >= v0.21.2 in PATH. Full config schema includes agent model, fix attempts, token limits, exclude patterns, dependency strategy.
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
| 2026-02-24 | New repo (commit-story-v2-eval) instead of GitHub fork | Can't fork your own repo to the same account. New repo with pushed history satisfies all fork requirements (separate repo, CodeRabbit PRs, resettable). |
| 2026-02-24 | Self-referential eval repo with commit-story hook | Eval repo's `.mcp.json` commit-story server points at itself, and git post-commit hook installed. Every commit auto-generates journal entries documenting the instrumentation process. After instrumentation, the repo becomes a live telemetry producer — validating that instrumented code actually works. |
| 2026-02-24 | Evaluation runs on branches, PRs against main | Main branch stays as the clean baseline. Agent works on `evaluation/run-N` branches, PRs target main for CodeRabbit reviews. Historical branches are kept to track progress across runs; new runs start fresh branches from main. |

## Dependencies

- PRD #1: Evaluation rubric must exist before formal scoring begins
- Anthropic API key (for first-draft implementation LLM calls)
- GitHub access (for fork creation and CodeRabbit reviews)

## Blocks

- PRD #3 (Spec Synthesis) depends on this evaluation report
