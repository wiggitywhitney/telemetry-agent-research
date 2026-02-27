# Telemetry Agent Research

Research repo for evaluating AI-powered telemetry instrumentation agents. Whitney Lee's [telemetry agent spec](docs/specs/telemetry-agent-spec-v3.8.md) describes an AI agent that auto-instruments JavaScript codebases with OpenTelemetry. This repo evaluated a first-draft implementation of that spec, scored the results against a formal rubric, and synthesized findings back into the spec and a design document for the next iteration.

## What This Research Produced

The research ran over three days (February 24–26, 2026) across three PRDs. The headline finding: **the agent produces good telemetry code, but everything around the instrumentation is broken.** The spec describes the right system; the first-draft implementation didn't build what the spec describes.

Key outputs:

- **Spec v3.8** — The telemetry agent spec evolved from v3.5 to v3.8 through this research. New content includes evaluation criteria, a two-tier validation architecture (structural + semantic), JavaScript PoC target, expanded tech stack, and cross-phase interface contracts. See [docs/specs/telemetry-agent-spec-v3.8.md](docs/specs/telemetry-agent-spec-v3.8.md).
- **Evaluation rubric** — A 30-rule rubric for scoring AI-generated instrumentation quality, grounded in OpenTelemetry community standards, vendor best practices, and practitioner experience. Gate checks, dimension profiles, and per-file detail. See [research/evaluation-rubric.md](research/evaluation-rubric.md).
- **Design document** — Blueprint for the next implementation: cross-phase interface contracts, module organization, and a consolidated decision register (52 entries). See [docs/architecture/design-document.md](docs/architecture/design-document.md).
- **7-phase build plan** — Dependency-graph-driven implementation phasing with acceptance gates per phase, grounded in specific failure modes from the evaluation. See [docs/specs/research/implementation-phasing.md](docs/specs/research/implementation-phasing.md).

## Research Arc

### PRD #1: Evaluation Rubric (Feb 24)

**Question**: What does "good instrumentation" look like, and how do you measure it?

Instrumentation quality is tribal knowledge — the industry hasn't codified it. This phase surveyed OpenTelemetry community standards, observability vendor best practices, academic literature, and practitioner experience to build a formal rubric.

The rubric defines 30 rules across three layers: gate checks (binary must-pass preconditions), dimension profiles (per-dimension pass rates across 6 quality dimensions), and per-file detail. Each rule carries an impact level (Critical/Important/Normal/Low) and distinguishes automatable checks from human-judgment checks. A companion mapping document applies the rubric to commit-story-v2 specifically, inventorying 50 exported functions, 15 error-handling sites, and the expected span tree.

See [prds/done/1-evaluation-rubric.md](prds/done/1-evaluation-rubric.md).

### PRD #2: Run & Evaluate (Feb 24–25)

**Question**: How well does the first-draft implementation actually work?

Forked [commit-story-v2](https://github.com/wiggitywhitney/commit-story-v2) as a test target, ran the first-draft implementation against it, collected CodeRabbit PR reviews, and scored everything against the rubric.

It took 8 runs and 3 manual patches before the agent produced any output. The CLI commands had no handlers. The MCP server was the only functional interface — and it was missing an entry point. The validation chain rejected the agent's own output. But when validation was bypassed, the agent processed all 7 files correctly: 4 instrumented, 3 skipped, all decisions sound. CodeRabbit found zero issues with the instrumentation code itself.

25 findings documented across architecture, agent behavior, schema handling, code quality, and coverage. See [evaluation/report.md](evaluation/report.md) for the full evaluation and [evaluation/run-1/rubric-scores.md](evaluation/run-1/rubric-scores.md) for scored results.

See [prds/done/2-run-and-evaluate.md](prds/done/2-run-and-evaluate.md).

### PRD #3: Spec Synthesis (Feb 25–26)

**Question**: What should the next implementation look like, and what does the spec need to change?

Analyzed the 25 findings into cross-cutting themes ([evaluation/patterns.md](evaluation/patterns.md)), then produced four documents: architectural recommendations (7 preserve, 3 change), tech stack evaluation (12 technology assessments with version-pinned recommendations), implementation phasing (7 phases with acceptance gates), and a design document (cross-phase interfaces, module organization, decision register).

Fed findings back into the spec across three version bumps: v3.6 (evaluation criteria, JS target), v3.7 (JSDoc notation, interface additions), and v3.8 (tech stack expansion, SDK capabilities, evaluation refinements). 19 PRD #3 decisions documented in the decision log.

See [prds/done/3-spec-synthesis.md](prds/done/3-spec-synthesis.md).

## Repository Structure

```text
telemetry-agent-research/
├── docs/
│   ├── specs/
│   │   ├── telemetry-agent-spec-v3.8.md    # Current spec (authoritative)
│   │   ├── archive/                         # Historical spec versions (v1, v2)
│   │   ├── research/                        # Research artifacts
│   │   │   ├── implementation-phasing.md    #   7-phase build plan with acceptance gates
│   │   │   ├── telemetry-agent-prior-art.md #   Survey of existing telemetry agents
│   │   │   ├── typescript-ast-tools.md      #   AST tooling comparison (ts-morph, etc.)
│   │   │   ├── weaver-typescript.md         #   Weaver CLI integration research
│   │   │   ├── optional-telemetry-patterns.md    # Optional instrumentation patterns
│   │   │   ├── instrumentation-level-strategies.md # Instrumentation granularity
│   │   │   └── *.json                       #   Section/field inventories (edit verification)
│   │   └── m8-section-inventory.md          # Before/after inventory for spec v3.8 edit
│   └── architecture/
│       ├── recommendations.md               # What to preserve vs change (7 preserve, 3 change)
│       ├── tech-stack-evaluation.md         # 12 technology assessments, version-pinned
│       └── design-document.md               # Cross-phase interfaces, module org, decision register
├── evaluation/
│   ├── report.md                            # Full evaluation report (PRD #2 output)
│   ├── patterns.md                          # Findings grouped into cross-cutting themes
│   ├── baseline/                            # Pre-instrumentation state of the test target
│   └── run-1/
│       ├── README.md                        # Run log: 8 attempts, 3 patches, final success
│       └── rubric-scores.md                 # Scored results against 30-rule rubric
├── research/
│   ├── evaluation-rubric.md                 # 30-rule instrumentation quality rubric
│   ├── rubric-dimensions.md                 # Quality dimension definitions
│   ├── instrumentation-quality-survey.md    # Industry survey informing the rubric
│   └── rubric-codebase-mapping.md           # Rubric applied to commit-story-v2
├── prds/
│   ├── done/                                # Completed PRDs
│   │   ├── 1-evaluation-rubric.md
│   │   ├── 2-run-and-evaluate.md
│   │   └── 3-spec-synthesis.md
│   └── 7-implementation-repo-setup.md       # Next PRD
└── journal/
    └── entries/2026-02/                     # Daily development journal
        ├── 2026-02-24.md
        ├── 2026-02-25.md
        └── 2026-02-26.md
```

## Finding What You Need

| If you want to... | Start here |
|---|---|
| Understand what the agent should do | [Telemetry agent spec v3.8](docs/specs/telemetry-agent-spec-v3.8.md) |
| See how the first-draft implementation scored | [Evaluation report](evaluation/report.md) |
| Review the rubric rules and scores | [Rubric](research/evaluation-rubric.md) and [scored results](evaluation/run-1/rubric-scores.md) |
| Understand what worked and what didn't | [Evaluation patterns](evaluation/patterns.md) |
| See what to preserve vs change architecturally | [Recommendations](docs/architecture/recommendations.md) |
| Choose libraries and versions | [Tech stack evaluation](docs/architecture/tech-stack-evaluation.md) |
| Plan the build order for the next implementation | [Implementation phasing](docs/specs/research/implementation-phasing.md) |
| Start building the next implementation | [Design document](docs/architecture/design-document.md) |
| Read the raw run log (8 attempts, 3 patches) | [Run 1 log](evaluation/run-1/README.md) |
| See how the spec evolved | [Spec version history](#spec-version-history) below |
| Trace the day-by-day research narrative | [Journal entries](journal/entries/2026-02/) |

## How Findings Connect to the Public Spec

The telemetry agent spec is the bridge between this research and the next implementation.

**Research artifacts (this repo):**
- Evaluation data (rubric scores, run logs, findings)
- Architectural analysis (recommendations, tech stack evaluation, design document)
- Implementation planning (phasing, module organization, decision register)
- Development journal

**Changes fed back into the spec:**
- Evaluation criteria and success metrics (added in v3.6)
- Two-tier validation architecture (added in v3.6)
- JavaScript PoC target and file discovery patterns (added in v3.6)
- Interface contracts and type definitions (added in v3.7)
- Tech stack requirements and SDK capabilities (added in v3.8)

The spec is the single source of truth for *what* the agent does. This repo documents *how we decided what it should do* and *how the first attempt went*. The next implementation will live in a separate public repo, with phase PRDs referencing spec sections directly.

## Spec Version History

The spec entered this research at v3.5 and exited at v3.8. Earlier versions (v1, v2) predate this repo and are archived for context.

| Version | Date | What Changed |
|---|---|---|
| v1 | Feb 5 | Initial draft |
| v2 | Feb 6 | Consolidated technical review |
| v3–v3.5 | Feb 7–23 | Cost visibility, dependency strategy, prompt engineering, Weaver integration, fix loop design |
| **v3.6** | **Feb 25** | Evaluation criteria, two-tier validation architecture, JavaScript PoC target (PRD #3 M4) |
| **v3.7** | **Feb 26** | JSDoc notation, camelCase field names, 7 new interface types from design document work (PRD #3 M7) |
| **v3.8** | **Feb 26** | Tech stack expansion (Node.js 24.x, simple-git, Vitest, yaml, Zod), SDK capabilities, evaluation refinements (PRD #3 M8) |

Historical versions: [docs/specs/archive/](docs/specs/archive/)
