# Baseline Git State — commit-story-v2-eval

## Snapshot

- **Baseline commit**: `9c6e4a14955ae4a30ca41163a1141a4e0645465f`
- **Short SHA**: `9c6e4a1`
- **Commit message**: "Merge pull request #41 from wiggitywhitney/chore/archive-spec-pointer"
- **Branch**: `main`
- **Source repo**: `wiggitywhitney/commit-story-v2` (canonical, pushed to eval repo)
- **Eval repo**: `wiggitywhitney/commit-story-v2-eval`
- **Local clone**: `~/Documents/Repositories/commit-story-v2-eval`
- **Captured**: 2026-02-24

## Test Baseline

- **Test count**: 320 tests across 11 test files
- **Status**: All passing
- **Duration**: ~736ms
- **Framework**: Vitest v4.0.18

## Coverage Baseline

| Category | % Stmts | % Branch | % Funcs | % Lines |
|----------|---------|----------|---------|---------|
| **All files** | **83.19** | **83.29** | **81.42** | **83.25** |
| collectors | 78.57 | 78.84 | 86.95 | 77.66 |
| generators | 99.40 | 90.08 | 100 | 99.36 |
| integrators | 69.04 | 47.61 | 66.66 | 69.04 |
| integrators/filters | 97.14 | 93.10 | 100 | 98.98 |
| managers | 93.71 | 84.09 | 100 | 93.58 |
| mcp | 0 | 100 | 0 | 0 |
| mcp/tools | 0 | 100 | 0 | 0 |
| utils | 44.11 | 32.35 | 58.82 | 45.45 |

## File Statistics

- **Source files**: 21 JavaScript files in `src/`
- **Test files**: 11 test files in `tests/`
- **Weaver schema**: `telemetry/registry/` (attributes.yaml, registry_manifest.yaml, resolved.json)

## Environment

- **Weaver CLI**: v0.21.2
- **Node.js runtime**: ES modules
- **Dependencies**: LangGraph, LangChain, Anthropic SDK, dotenv
