# PRD #7: Implementation Repo Setup

**Issue**: [#7](https://github.com/wiggitywhitney/telemetry-agent-research/issues/7)
**Status**: In Progress
**Priority**: High
**Blocked by**: None (PRD #3 M8 complete as of 2026-02-26)
**Blocks**: Phase 1 implementation (first PRD in new repo)
**Created**: 2026-02-26

## Problem

PRD #3 produces all the research artifacts, tooling, and spec updates needed to build the telemetry instrumentation agent, but there is no implementation repo to receive them. The prd-phase skill needs updates before it can be copied (stale spec filename, wrong line numbers in landmarks table, missing Step 3e for the design document). The developer workflow (journal hook, CodeRabbit review, secrets management, project instructions) needs to be configured so that Phase 1 can begin with a fully-equipped environment.

## Solution

Update the prd-phase skill in the research repo (where the source of truth lives), then create a public GitHub repo with all research artifacts at their correct paths, developer tooling configured, and a project CLAUDE.md adapted for implementation work. Verify the setup by running `prd-phase 1` as the first act in the new repo.

## Scope

**In scope:**
- Updating prd-phase skill in this research repo (spec filename, landmarks table, Step 3e, Source Documents table, Step 2b for previous-phase PRD read, milestone-to-rubric guidance)
- Creating a public GitHub repo with package.json (`"type": "module"`), .gitignore, LICENSE
- Copying 7 skill-routed documents, rubric-codebase-mapping, and prd-phase skill directory to correct paths
- Configuring vals (ANTHROPIC_API_KEY, GITHUB_TOKEN only), CodeRabbit MCP server, commit-story-v2 journal hook
- Writing a project CLAUDE.md for the implementation repo (TypeScript + ESM, document layering, prd-phase usage)
- Verifying setup by generating Phase 1 PRD

**Out of scope:**
- Phase 1 implementation (that's the first PRD in the new repo)
- Writing the implementation repo README (comes after Phase 1 proves the setup)
- Modifying any research documents beyond M0's language decision propagation — once updated by M0, they are copied as-is
- Setting up CI/CD (comes with Phase 1 when there's code to test)

## Context

### Why This Needs a PRD

The previous conversation concluded this was "just a checklist." That was before adding commit-story-v2, CodeRabbit MCP, vals, and the CLAUDE.md — which introduce configuration decisions and a verification surface larger than a simple copy task. The prd-phase skill updates are also non-trivial (landmarks table rebuild requires reading the v3.9 spec and recording every section's line number).

### Document Inventory (What Gets Copied)

**Skill-routed documents (the prd-phase skill reads these):**

| Document | New Repo Path | Skill Route |
|----------|--------------|-------------|
| telemetry-agent-spec-v3.9.md | docs/specs/telemetry-agent-spec-v3.9.md | Step 3a |
| implementation-phasing.md | docs/specs/research/implementation-phasing.md | Step 2 |
| tech-stack-evaluation.md | docs/architecture/tech-stack-evaluation.md | Step 3b |
| evaluation-rubric.md | research/evaluation-rubric.md | Step 3c |
| recommendations.md | docs/architecture/recommendations.md | Step 3d |
| design-document.md | docs/architecture/design-document.md | Step 3e (new) |

**Not skill-routed (still needed):**

| Document | New Repo Path | Purpose |
|----------|--------------|---------|
| rubric-codebase-mapping.md | research/rubric-codebase-mapping.md | Integration test answer key (agent never sees this) |

**Skill files (copy entire directory):**

| File | New Repo Path |
|------|--------------|
| SKILL.md | .claude/skills/prd-phase/SKILL.md |
| references/tech-stack-by-phase.md | .claude/skills/prd-phase/references/tech-stack-by-phase.md |
| references/phase-prd-template.md | .claude/skills/prd-phase/references/phase-prd-template.md |

### What Stays in the Research Repo

PRDs 1-3, patterns.md, report.md, rubric-scores.md, section inventory JSONs, survey, superseded drafts. These are research record.

### prd-phase Skill Updates Required

All updates happen in the research repo before copying:

1. **Spec filename**: References `telemetry-agent-spec-v3.6.md` in SKILL.md (lines 46, 51, 188) and template (line 98). Update to `v3.9`.
2. **Landmarks table** (SKILL.md lines 53-74): Line numbers are for v3.6 (1,634 lines). The v3.9 spec will be longer. Rebuild by reading v3.9 and recording every section heading's line number.
3. **Step 3e — Design Document**: Route each phase to relevant design-document.md sections:
   - All phases: Module organization (directory structure, dependency rules), Decision register
   - Phase 1: instrumentFile interface contract, InstrumentationOutput type
   - Phase 2: validateFile interface contract, ValidationResult/CheckResult types, ValidateFileInput
   - Phase 3: instrumentWithRetry interface contract, FileResult evolution, schema re-resolution
   - Phase 4: coordinate interface contract, RunResult type, CoordinatorCallbacks
   - Phase 5: Schema integration extensions
   - Phase 6: CostCeiling, CoordinatorCallbacks (wiring focus)
   - Phase 7: RunResult consumption for git/PR, advisoryAnnotations flow
4. **Source Documents table**: Add row for design-document.md.
5. **Phase-to-module mapping**: Reference the design document's Phase-to-Module Mapping table (either in Step 3e or Step 4).
6. **Step 2b — Previous Phase PRD Read** (Phase 2+ only): Before generating a new phase PRD, read the previous phase's PRD — specifically the decision log and any noted interface changes. The design document defines planned contracts, but implementation decisions may have changed them. Extract deviations and include them in the new PRD's Dependencies section as "Phase N-1 delivered X (with these modifications from plan: ...)". Phase 1 has no predecessor, so this step is skipped.
7. **Milestone-to-rubric guidance**: In the milestone guidance (Step 4 or template), add: "Where a milestone directly satisfies a rubric rule, note the rule ID. Not every milestone maps to a rubric rule — Phase 1 milestones about prompt engineering and file I/O have no clean rubric connection, and stretching for one adds noise. Phase 2 (validation chain) maps naturally (e.g., 'Tier 1 syntax check works' → NDS-001, 'Tier 2 CDQ-001 works' → CDQ-001). Include the mapping where it's real; omit it where it isn't."

## Milestones

- [x] **M0: Agent language decision updated in research documents**: Propagate the TypeScript decision (Decision 9) across all affected research documents using a pre/post inventory for safety. Sub-steps: (1) Pre-migration inventory JSON (each entry: id, file, line, snippet, classification as "convert"/"keep"/"unchanged"). (2) Update tech-stack-evaluation.md — reopen Agent Language resolved question, document Node.js 24.x native type-stripping, update decision log entry 12. (3) Bump spec v3.8 → v3.9 — rename file, status/purpose lines, tech stack table (line 227), revision history, agent code fences javascript→typescript (target file examples stay javascript), keep all target file refs (.js paths, \*\*/\*.js globs). (4) Convert design-document.md — line 27 notation declaration, all JSDoc @typedef blocks → TypeScript interfaces (~12 code blocks), module org .js → .ts for agent modules. (5) Update implementation-phasing.md — clarify agent code is TypeScript, keep target file refs as JS. (6) Update prd-phase skill — SKILL.md lines 18/181/202, tech-stack-by-phase.md language declarations and code fence tags. (7) Post-migration inventory JSON and diff — every "convert" now TS, every "keep"/"unchanged" identical to before. (8) Commit. *No blockers. Executes in research repo on `feature/prd-7-implementation-repo-setup`.*
- [ ] **M1: prd-phase skill updated in research repo**: Spec filename → v3.9, landmarks table rebuilt against v3.9 spec, Step 3e added for design document routing, Source Documents table updated, phase-to-module mapping referenced, Step 2b added for previous-phase PRD decision log read (Phase 2+), milestone guidance updated with rubric-rule mapping where natural (not forced). Committed to `feature/prd-7-implementation-repo-setup` branch. *Blocked by M0 (needs v3.9 spec to exist).*
- [ ] **M2: Public repo created with project foundation**: GitHub repo created (name decided with human approval — see Open Questions), initialized with package.json (`"type": "module"`, `"engines": { "node": ">=24.0.0" }`), .gitignore (node_modules/, journal/, .env, *.local), LICENSE. commit-story-v2 installed as devDependency, journal hook initialized. *No blockers — can run in parallel with M1.*
- [ ] **M3: Research artifacts copied to correct paths**: All 7 skill-routed documents, rubric-codebase-mapping, and prd-phase skill directory copied to the paths specified in the Document Inventory tables above. After copying, spot-check the landmarks table against the copied v3.9 spec: open the spec, confirm 3-5 section headings (Architecture, Technology Stack, Interface Contracts, Evaluation Criteria, Revision History) appear at the line numbers the landmarks table claims. This catches line-number drift before M6. *Blocked by M0 (documents must be updated first), M1 (needs updated skill), and M2 (needs the repo to exist).*
- [ ] **M4: Developer tooling configured**: `.vals.yaml` with ANTHROPIC_API_KEY and GITHUB_TOKEN (Google Secret Manager refs). `.claude/settings.local.json` with CodeRabbit MCP server enabled. commit-story journal hook verified working (make a test commit, confirm journal entry appears in `journal/entries/`). *Blocked by M2.*
- [ ] **M5: Project CLAUDE.md written**: Adapted from research repo's CLAUDE.md for implementation context — TypeScript + ESM (erasableSyntaxOnly, native Node.js type stripping, tsc --noEmit CI gate), Vitest, document layering explanation (spec → tech stack → recommendations → design doc → rubric), prd-phase skill usage instructions, attribution rules, vals usage. Not a copy — a purpose-built document for the implementation repo. *Blocked by M2.*
- [ ] **M6: Setup verified with prd-phase 1**: Run `/prd-phase 1` in the new repo. The skill should resolve all file paths, read the correct spec sections, route to the right tech stack and rubric subsections, and produce a valid Phase 1 PRD. This is the acceptance test for the entire setup. *Blocked by M1-M5.*

## Open Questions

- **Repo name?** Decide at M2 execution with human approval. Direction: spider-lineage name (Weaver is the CLI the spec names; the agent that produces instrumentation threads could inherit from that lineage). Candidates to discuss: `spinneret`, `arachne`, `loom`, `silkworm`, or straight `telemetry-agent` if the thematic naming doesn't stick.
- **License?** MIT and Apache 2.0 are both common for developer tooling. The spec is Whitney Lee's original work; the implementation repo's license applies to the agent code only.

## Success Criteria

- All prd-phase skill routes resolve to existing files at expected paths
- `prd-phase 1` produces a complete Phase 1 PRD without manual path fixes
- commit-story journal hook generates entries on commit
- vals injects ANTHROPIC_API_KEY successfully
- CodeRabbit MCP server is accessible from Claude Code sessions
- CLAUDE.md accurately describes the implementation repo's conventions

## Decision Log

| # | Decision | Rationale | Date |
|---|----------|-----------|------|
| 1 | Lightweight PRD (not a checklist) | commit-story, CodeRabbit MCP, vals, and CLAUDE.md authoring add enough configuration decisions and verification surface to justify tracking | 2026-02-26 |
| 2 | Skill updates happen in research repo before copying | Research repo is the source of truth for the skill. Updating after copying means two divergent copies. | 2026-02-26 |
| 3 | vals trimmed to ANTHROPIC_API_KEY + GITHUB_TOKEN only | DD_API_KEY/DD_APP_KEY are Datadog-specific to commit-story-v2, not needed for the agent implementation | 2026-02-26 |
| 4 | Public repo | The spec is intended to be public; the implementation should be too | 2026-02-26 |
| 5 | Verify with prd-phase 1 as acceptance test | Exercises every file path and skill route in one shot. If something is misconfigured, you find out immediately rather than discovering it during Phase 1 implementation. | 2026-02-26 |
| 6 | Explicit milestone dependency annotations | M2 can parallelize with M1 (no dependency on skill updates). M3 blocked by both M1 and M2. M4/M5 blocked by M2. M6 blocked by all. Prevents wrong execution order. | 2026-02-26 |
| 7 | Landmarks spot-check in M3 before M6 | M6 catches failures but gives opaque errors ("wrong content at line 181"). A spot-check of 3-5 section headings against landmarks table line numbers after copying gives immediately actionable diagnostics ("Architecture moved from 512 to 538"). | 2026-02-26 |
| 8 | Repo name decided at execution time with human approval | Spider-lineage naming direction (related to Weaver). Decision deferred to M2 rather than locked in the PRD. | 2026-02-26 |
| 9 | Agent code is TypeScript with erasableSyntaxOnly and native Node.js 24.x type stripping | Zero build step (`node src/index.ts` directly), `tsc --noEmit` as CI gate, Zod gets full `z.infer<typeof Schema>` type inference. Target files remain JavaScript. Supersedes design document Decision 12. | 2026-02-27 |
