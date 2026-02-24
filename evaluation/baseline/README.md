# Evaluation Baseline & Run Procedures

## Eval Repo

- **GitHub**: `wiggitywhitney/commit-story-v2-eval`
- **Local**: `~/Documents/Repositories/commit-story-v2-eval`
- **Baseline SHA**: `9c6e4a1` (see `git-state.md` for full details)

## Setup Checklist (already done for initial setup)

- [x] Repo created and code pushed from canonical commit-story-v2
- [x] `.vals.yaml` — secrets config (Anthropic API, GitHub token, DD keys)
- [x] `.mcp.json` — CodeRabbit MCP + self-referential commit-story MCP
- [x] `.claude/settings.local.json` — Claude Code permissions
- [x] `.claude/CLAUDE.md` — eval-specific project instructions
- [x] `npm install` — dependencies installed
- [x] Git post-commit hook — commit-story auto-generates journal entries
- [x] CodeRabbit — automatic via GitHub account (reviews all PRs)
- [x] Tests verified — 320 passing
- [x] Coverage captured — 83.19% statements

## Starting a New Evaluation Run

Each run gets its own branch. Historical branches are preserved.

```bash
cd ~/Documents/Repositories/commit-story-v2-eval

# Create a fresh evaluation branch from baseline
git checkout main
git checkout -b evaluation/run-N    # increment N for each run

# Run the telemetry agent against this branch
cd ~/Documents/Repositories/telemetry-agent-spec-v3
npm run build
cd ~/Documents/Repositories/commit-story-v2-eval
vals exec -f .vals.yaml -- /path/to/telemetry-agent instrument src/ --config telemetry-agent.yaml

# Create PR for CodeRabbit review
# PR from evaluation/run-N → main
```

## Verifying Baseline

```bash
cd ~/Documents/Repositories/commit-story-v2-eval
git checkout main
git rev-parse HEAD    # should be 9c6e4a14955ae4a30ca41163a1141a4e0645465f
npm test              # should be 320 passing
```

## After Each Run

1. Capture results in `evaluation/run-N/` directory in the research repo
2. Score against rubric (from PRD #1)
3. Capture CodeRabbit review
4. Keep the evaluation branch for historical comparison

## Baseline Artifacts (in this directory)

- `git-state.md` — commit SHA, test counts, coverage table, environment info
- `test-results.txt` — full test output
- `coverage-report.txt` — full coverage output
