---
name: prd-phase
description: Create implementation phase PRDs from pre-existing research for the telemetry agent. Translates already-complete research into focused, bounded PRDs. Use when the user says "prd-phase N" or wants to create a PRD for a specific build phase (1-7). Do NOT use for new feature discovery — use prd-create for that.
---

# Phase PRD Creation

## Purpose

Create implementation phase PRDs for the telemetry agent by assembling content from existing research documents. All research is complete. The job is to route to the right documents, extract the right details, and produce a focused PRD.

## Critical Principles

1. **Bounded context**: Read ONLY what this phase needs. Do not read the full spec or full tech stack document. Read the specific sections listed in the routing tables below.
2. **No detail loss**: Version numbers, API patterns, config field names, and code snippets must be copied verbatim from source documents into the PRD. Do not summarize technical details.
3. **Excerpt, don't just reference**: Critical details (acceptance gates, API patterns, rubric rules) must be IN the PRD, not just referenced by section name.
4. **One phase at a time**: Never create multiple phase PRDs in one session. Each phase gets a fresh context.
5. **JavaScript and ESM**: The agent is JavaScript (`"type": "module"` in package.json). All code samples use ESM `import` syntax. ts-morph is used for AST analysis (it handles JS via `allowJs: true`), but the agent code itself is JavaScript.

## Process

### Step 1: Identify the Phase

Parse the phase number (1-7) from the user's command.

### Step 2: Read the Phase Definition

Read the specific phase section from `docs/specs/research/implementation-phasing.md`. Each phase contains:
- What Gets Built
- Acceptance Gate
- Rubric Rules That Apply
- Why This Boundary
- Spec Sections (table mapping to spec sections and subsections)

**Also read these cross-cutting sections** (they apply to every phase):
- "Two-Tier Validation Architecture" section
- "Cross-Cutting: The DX Principle" section

### Step 3: Read Source Documents (Routing)

Read ONLY the sections relevant to this phase.

#### 3a. Spec Sections

The phase definition contains a "Spec Sections" table. For each row:
- **Full** -> read the entire section from `docs/specs/telemetry-agent-spec-v3.6.md`
- **Subsection only** -> read only the named subsection
- **Fields only** -> extract just the named configuration fields
- **Row only** -> extract just the named row from the Technology Stack table

**Note**: The spec was renamed from v3.5.md to v3.6.md to match its content version.

**Spec section landmarks** (line numbers for the 1,634-line spec):

| Section | Line | Key Subsections |
|---------|------|-----------------|
| Architecture | 52 | Coordinator+Agents 54, Coordinator API 93, Error Handling 123, Interfaces 154, Tech Stack 181, Weaver Integration 200 |
| Init Phase (Required) | 237 | What Init Does 241, Prerequisites 271 |
| Complete Workflow | 281 | (diagram through ~343) |
| How It Works | 344 | Input 346, Processing (per file) 351, Output 367, Agent Output Format 371, System Prompt Structure 385, Known Failure Modes 415, Elision Detection 431 |
| File/Directory Processing | 441 | File Revert Protocol 453, Periodic Schema Checkpoints 459 |
| What Gets Instrumented | 475 | Priority Hierarchy 477, Review Sensitivity 490, Patterns Not Covered 507 |
| What the Agent Actually Does to Code | 520 | Path 1: Auto-Instrumentation 524, Path 2: Manual Span 594, Decision: Which Path? 658 |
| Attribute Priority Chain | 668 | Schema Extension Guardrails 678 |
| Auto-Instrumentation Libraries | 692 | Why Libraries First 696, Detection Flow 702, Library Discovery 712 |
| Validation Chain | 812 | Per-File Validation (with fix loop) 816, End-of-Run Validation 872 |
| Agent Self-Instrumentation | 902 | |
| Handling Existing Instrumentation | 968 | |
| Schema as Source of Truth | 986 | |
| Result Data | 995 | Result Structure 1005, PR Summary 1091, Schema Hash Tracking 1105 |
| Configuration | 1111 | What Goes Where 1162, Dependency Strategy 1202, Cost Visibility 1217 |
| Minimum Viable Schema Example | 1231 | |
| PoC Scope | 1427 | In Scope 1429, Out of Scope 1476 |
| Evaluation & Acceptance Criteria | 1492 | Two-Tier Validation Architecture 1521, Required Verification Levels 1555 |

#### 3b. Tech Stack Sections

Read `references/tech-stack-by-phase.md` (bundled with this skill) to find which sections of `docs/architecture/tech-stack-evaluation.md` to read for this phase. Extract:
- Version numbers (exact)
- API patterns (code snippets, verbatim)
- Caveats and limitations
- Recommendation summaries

#### 3c. Rubric Rules

The phase definition lists rubric rule IDs (e.g., NDS-001, CDQ-003). Look up each rule's full definition in `research/evaluation-rubric.md`. The code-level rules are in **Part 2: Code-Level Evaluation** (line 136), organized as:

| Dimension | Location | Rules |
|-----------|----------|-------|
| Gate Checks | ~line 140 | NDS-001, NDS-002, NDS-003, API-001 |
| Non-Destructiveness | ~line 189 | NDS-004, NDS-005 |
| Coverage | ~line 218 | COV-001 through COV-006 |
| Restraint | ~line 282 | RST-001 through RST-005 |
| API-Only Dependency | ~line 337 | API-002, API-003, API-004 |
| Schema Fidelity | ~line 374 | SCH-001 through SCH-004 |
| Code Quality | ~line 422 | CDQ-001 through CDQ-008 |

For each rule, extract: Rule ID, name, impact level, evaluation scope, full description including evaluation method, and automation classification (summary at ~line 514).

For Phase 2+, classify rules as Tier 1 (structural/blocking) vs Tier 2 (semantic/blocking or advisory) using the "Two-Tier Validation Architecture" section of implementation-phasing.md.

#### 3d. Architectural Context (if needed)

Read relevant sections of `docs/architecture/recommendations.md`:
- Phases 1-3: "Fix Loop Feedback Quality" subsection
- Phase 2: "Two-Tier Validation (New Architecture)" section
- Phases 4-7: "DX and Error Handling Architecture" section
- All phases: Summary table at end for quick reference

### Step 4: Assemble the PRD

Read `references/phase-prd-template.md` (bundled with this skill) for the output template.

Fill in each section using content gathered in Steps 2-3. Key rules:

**Tech Stack section:**
- Copy version numbers exactly (e.g., "Vitest 4.0.18", not "latest")
- Copy API code snippets verbatim — all JavaScript, ESM imports
- Include caveats that affect THIS phase
- Include only entries relevant to this phase

**Rubric Rules section:**
- Include the full rule definition, not just the ID
- Group by Gate vs Dimension
- Note which are Tier 1 vs Tier 2 (for Phase 2+)
- Include impact level and automation classification

**Acceptance Gate section:**
- Copy the gate verbatim from the phasing document
- Add specific, testable verification steps
- Map each gate criterion to the rubric rules that verify it

**Milestones:**
- Define 5-8 milestones that build toward the acceptance gate incrementally
- Each milestone should be independently testable
- Final milestone: "Acceptance gate passes end-to-end"
- Include a milestone for the DX cross-cutting requirement specific to this phase

**Spec Reference section:**
- Reproduce the spec section map table from the phasing document
- Add line number ranges from the landmarks table above

### Step 5: Create GitHub Issue and PRD File

Follow the same workflow as `/prd-create`:

1. **Create GitHub Issue** with label "PRD":
   ```markdown
   ## PRD: Phase N — [Phase Name]

   **Problem**: Implementation phase N of the telemetry agent — [what gets built]

   **Solution**: [1-2 sentence summary from phasing document]

   **Detailed PRD**: Will be added after PRD file creation

   **Priority**: High
   ```

2. **Create PRD file** at `prds/[issue-id]-phase-N-[phase-name].md`

3. **Update GitHub Issue** with PRD link

4. **Commit** with `[skip ci]` flag:
   ```text
   docs(prd-[issue-id]): create Phase N PRD — [phase-name] [skip ci]
   ```

### Step 6: Post-Creation Verification

Before presenting the PRD to the user, verify:

- [ ] Every rubric rule ID listed in the phasing document appears with its full definition
- [ ] Every spec section from the phase's spec section map is referenced with line numbers
- [ ] Tech stack version numbers are exact (not "latest" or "current")
- [ ] API code snippets are present and use JavaScript ESM syntax
- [ ] The acceptance gate is copied verbatim from the phasing document
- [ ] The DX cross-cutting requirement for this phase is included
- [ ] No content from other phases leaked in
- [ ] Milestones build incrementally toward the acceptance gate
- [ ] All code samples are JavaScript (not TypeScript)

## Source Documents

| Document | Path | Purpose |
|----------|------|---------|
| Implementation Phasing | `docs/specs/research/implementation-phasing.md` | Phase definitions, spec section maps, cross-cutting architecture |
| Telemetry Agent Spec | `docs/specs/telemetry-agent-spec-v3.6.md` | Source of truth for what to build |
| Tech Stack Evaluation | `docs/architecture/tech-stack-evaluation.md` | Version numbers, API patterns, library choices |
| Evaluation Rubric | `research/evaluation-rubric.md` | Full rule definitions for acceptance criteria |
| Architectural Recommendations | `docs/architecture/recommendations.md` | Preserve/change verdicts, evidence base |
| Tech Stack by Phase | `references/tech-stack-by-phase.md` (skill-bundled) | Routing table: which tech stack sections per phase |
| Phase PRD Template | `references/phase-prd-template.md` (skill-bundled) | Output template |

## Important Notes

- **One phase per session.** Multiple phase PRDs in one context risks detail loss.
- **The spec is the source of truth.** If the phasing document and spec disagree, the spec wins. Flag the discrepancy for the user.
- **Don't include evaluation findings.** The implementing AI needs what to build, not why the last attempt failed.
- **Don't include rejected alternatives.** No LangGraph comparisons, no Babel evaluations.
- **Trust the routing tables.** Don't add "extra context that might be helpful."
- **JavaScript only.** The agent code is JavaScript with ESM modules. ts-morph handles JS via `allowJs: true`. All samples, all paths, all config — JavaScript.

**Note**: If any `gh` command fails with "command not found", inform the user that GitHub CLI is required: https://cli.github.com/
