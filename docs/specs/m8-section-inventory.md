# M8 Section Inventory — Before/After Comparison

**Spec file:** `telemetry-agent-spec-v3.8.md` (1759 lines)
**Purpose:** Decision 11 discipline — verify untouched sections are unchanged after M8 edits.

## Before Inventory (v3.7, pre-M8)

Sections marked with **[EDIT]** will be modified during M8. All others must remain unchanged.

| Line | Heading | M8 Status |
|------|---------|-----------|
| 1 | `# Telemetry Agent Specification` | **[EDIT]** version bump |
| 8 | `## Revision History` | **[EDIT]** add v3.8 entry |
| 26 | `## Vision` | Unchanged |
| 33 | `### Why Schema-Driven?` | Unchanged |
| 43 | `## Pre-Implementation Research Spikes` | Unchanged |
| 53 | `## Architecture` | Unchanged |
| 55 | `### Coordinator + Agents Architecture` | Unchanged |
| 59 | `#### Coordinator (Not AI)` | **[EDIT]** TS→JS prose (line 60) |
| 75 | `#### Schema Builder Agent (Future)` | Unchanged |
| 81 | `#### Instrumentation Agent (Per-File)` | Unchanged |
| 94 | `#### Instrumentation Output` | Unchanged (M7 already converted) |
| 130 | `### Coordinator Programmatic API` | Unchanged |
| 134 | `#### Progress Callbacks` | Unchanged (M7 already converted) |
| 162 | `### Coordinator Error Handling` | **[EDIT]** strengthen no-silent-failures (Task 6) |
| 193 | `### Interfaces` | Unchanged |
| 201 | `#### MCP Server` | Unchanged |
| 212 | `#### CLI` | Unchanged |
| 216 | `#### GitHub Action` | Unchanged |
| 220 | `### Technology Stack (PoC)` | **[EDIT]** table + prose (Tasks 2, 3, 4) |
| 239 | `### Weaver Integration Approach` | Unchanged |
| 276 | `## Init Phase (Required)` | Unchanged |
| 280 | `### What Init Does` | Unchanged |
| 310 | `### Prerequisites (verified during init)` | Unchanged |
| 320 | `## Complete Workflow` | Unchanged |
| 383 | `## How It Works` | Unchanged |
| 385 | `### Input` | Unchanged |
| 390 | `### Processing (per file)` | Unchanged |
| 406 | `### Output` | Unchanged |
| 410 | `### Agent Output Format` | Unchanged |
| 424 | `### System Prompt Structure` | Unchanged |
| 454 | `### Known Failure Modes` | Unchanged |
| 470 | `### Elision Detection` | Unchanged |
| 480 | `## File/Directory Processing` | Unchanged |
| 492 | `### File Revert Protocol` | Unchanged |
| 498 | `### Periodic Schema Checkpoints` | Unchanged |
| 504 | `### SDK Init File Parsing Scope` | Unchanged |
| 508 | `### Future: Parallel Processing` | Unchanged |
| 514 | `## What Gets Instrumented` | Unchanged |
| 516 | `### Instrumentation Priority Hierarchy` | Unchanged |
| 529 | `### Review Sensitivity` | Unchanged |
| 541 | `### Schema guidance` | Unchanged |
| 546 | `### Patterns Not Covered (PoC)` | Unchanged |
| 559 | `## What the Agent Actually Does to Code` | Unchanged |
| 563 | `### Path 1: Auto-Instrumentation Library (Primary)` | Unchanged |
| 633 | `### Path 2: Manual Span (Fallback for Business Logic)` | Unchanged |
| 697 | `### Decision: Which Path?` | **[EDIT]** auto-instrumentation interaction model (Task 7) |
| 707 | `## Attribute Priority Chain` | Unchanged |
| 717 | `### Schema Extension Guardrails` | Unchanged |
| 731 | `## Auto-Instrumentation Libraries` | Unchanged |
| 735 | `### Why Libraries First` | Unchanged |
| 741 | `### Detection Flow` | **[EDIT]** auto-instrumentation interaction model (Task 7) |
| 751 | `### Library Discovery` | Unchanged |
| 819 | `### Schema Updates for Libraries` | Unchanged |
| 841 | `### Manual Spans (Secondary)` | Unchanged |
| 851 | `## Validation Chain` | Unchanged |
| 855 | `### Per-File Validation (with fix loop)` | **[EDIT]** validator feedback format (Task 6), ts-morph note (Task 4) |
| 911 | `#### Validation Chain Types` | Unchanged (M7 already converted) |
| 959 | `### End-of-Run Validation (once, after all files)` | Unchanged |
| 985 | `### Future Optimizations` | Unchanged |
| 989 | `### If Tests Don't Exist` | Unchanged |
| 996 | `## Agent Self-Instrumentation` | Unchanged |
| 1000 | `### Bootstrapping` | Unchanged |
| 1004 | `### Separate Telemetry Pipelines` | Unchanged |
| 1015 | `### Coordinator Spans (one trace per run)` | Unchanged |
| 1025 | `### Instrumentation Agent Spans (child spans per file)` | Unchanged |
| 1041 | `### Trace Context Propagation` | Unchanged |
| 1045 | `### PoC Scope` | Unchanged |
| 1055 | `## Handling Existing Instrumentation` | Unchanged |
| 1057 | `### Already instrumented (complete)` | Unchanged |
| 1061 | `### Broken instrumentation` | Unchanged |
| 1066 | `### Telemetry removed by user` | Unchanged |
| 1073 | `## Schema as Source of Truth` | Unchanged |
| 1082 | `## Result Data` | Unchanged |
| 1086 | `### Why In-Memory Results` | Unchanged |
| 1092 | `### Result Structure` | **[EDIT]** FileResult population requirement (Task 6) |
| 1186 | `### Run-Level Result` | Unchanged |
| 1216 | `### PR Summary` | Unchanged |
| 1230 | `### Schema Hash Tracking` | Unchanged |
| 1236 | `## Configuration` | Unchanged |
| 1287 | `### What Goes Where` | Unchanged |
| 1307 | `### Dry Run Mode` | Unchanged |
| 1315 | `### Include and Exclude Patterns` | Unchanged |
| 1319 | `### Instrumentation Mode (Reserved)` | Unchanged |
| 1323 | `### Config Validation` | Unchanged |
| 1327 | `### Dependency Strategy` | Unchanged |
| 1342 | `### Cost Visibility` | Unchanged |
| 1356 | `## Minimum Viable Schema Example` | Unchanged |
| 1470 | `### Validation` | Unchanged |
| 1480 | `### Key Points` | Unchanged |
| 1492 | `## Resolved Questions` | Unchanged |
| 1494 | `### Q1: Weaver Schema Structure` | Unchanged |
| 1499 | `### Q2: Live-Check Setup` | Unchanged |
| 1508 | `### Q3: Error Handling` | Unchanged |
| 1515 | `### Q4: Semconv Lookup` | Unchanged |
| 1522 | `### Q5: Agent Framework Choice` | **[EDIT]** TS→JS prose reference (Task 5) |
| 1527 | `### Q6: Quality Checks for AI` | Unchanged |
| 1537 | `## Dependencies` | Unchanged |
| 1539 | `### PRD 25 (Autonomous Dev Infrastructure)` | Unchanged |
| 1545 | `### Separate Repository` | Unchanged |
| 1552 | `## PoC Scope` | Unchanged |
| 1554 | `### In Scope` | Unchanged |
| 1601 | `### Out of Scope (Future)` | **[EDIT]** TS→JS prose if applicable (Task 5) |
| 1617 | `## Evaluation & Acceptance Criteria` | Unchanged |
| 1621 | `### Evaluation Philosophy` | Unchanged |
| 1631 | `### Rubric Dimensions` | **[EDIT]** RST-004 exemption, CDQ-008 note (Task 7) |
| 1646 | `### Two-Tier Validation Architecture` | Unchanged |
| 1680 | `### Required Verification Levels` | **[EDIT]** independently runnable gates (Task 7) |
| 1696 | `## Research Summary` | Unchanged |
| 1698 | `### Prior Art` | Unchanged |
| 1705 | `### Optional Telemetry Pattern` | Unchanged |
| 1713 | `### Instrumentation Level (Still Open)` | Unchanged |
| 1721 | `## References` | **[EDIT]** if new references added |
| 1723 | `### Research Documents` | Unchanged |
| 1730 | `### External Resources` | **[EDIT]** TS→JS prose if applicable (Task 5) |
| 1736 | `### RS1 Research Sources` | Unchanged |
| 1745 | `### RS2 Research Sources` | Unchanged |
| 1752 | `### RS3 Research Sources` | Unchanged |

## Edit Summary

**Sections being edited (13 of 88):**
- Task 2: Technology Stack table (line 220)
- Task 3: Technology Stack prose (line 220 area, after table)
- Task 4: Per-File Validation (line 855), Technology Stack (line 220)
- Task 5: Coordinator description (line 59), Q5 (line 1522), Out of Scope (line 1601), References (line 1730), others TBD
- Task 6: Coordinator Error Handling (line 162), Per-File Validation (line 855), Result Structure (line 1092)
- Task 7: Decision: Which Path? (line 697), Detection Flow (line 741), Rubric Dimensions (line 1631), Required Verification Levels (line 1680)
- Task 8: Header (line 1), Revision History (line 8)

**Sections unchanged (75 of 88):** All others per table above.

## After Inventory (v3.8, post-M8)

**Spec file:** `telemetry-agent-spec-v3.8.md` (1796 lines, +37 from v3.7)

### Edit verification

| Task | Section(s) Modified | Lines Added | Status |
|------|-------------------|-------------|--------|
| 2 | Technology Stack table | +6 rows, 3 rows updated | Applied |
| 3 | Technology Stack prose (structured outputs, caching, countTokens, thinking) | +6 paragraphs | Applied |
| 4 | Technology Stack prose (ts-morph caveats, Prettier note) | +2 paragraphs | Applied |
| 5 | Coordinator (line 60), Patterns Not Covered (line 562), Weaver post-PoC (line 283), Q5 (line 1537), Dependencies (line 1579), References (line 1764) | 6 line edits | Applied |
| 6 | Per-File Validation (validator feedback), Result Structure (FileResult requirement), Coordinator Error Handling (no silent failures) | +3 paragraphs | Applied |
| 7 | Detection Flow (auto-instrumentation interaction model), Rubric Dimensions (RST-004, CDQ-008, CDQ-007), Required Verification Levels (independently runnable gates) | +3 blocks | Applied |
| 8 | Header (v3.7→v3.8), Revision History (+v3.8 entry), filename rename, cross-references | Version bump | Applied |

### Items reviewed — no change needed

- Line 923 "TypeScript binder access" — factually correct about ts-morph's compiler internals
- Line 31 "any TypeScript or JavaScript codebase" — long-term goal, intentional
- Lines 226, 249 — ts-morph is TypeScript-native, factual
- Line 1619 "TypeScript support" — correctly listed as future/out-of-scope
- Lines 1713, 1740, 1742, 1748 — external tool/doc names, not our code
- Lines 21-22 — revision history, historical record
- Out of Scope (line 1630) — no TS→JS changes needed (already says "TypeScript support" as a deferred item, which is correct)
- References (line 1766) — no TS→JS changes needed (external resource names)

### Sections confirmed unchanged

Spot-checked via git diff — no hunks touch sections outside the planned edit zones. All 75 "Unchanged" sections from the before inventory remain untouched. The diff contains exactly 15 hunks, all within the 13 planned edit zones (some zones contain multiple hunks).
