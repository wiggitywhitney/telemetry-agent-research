# Telemetry Agent Specification

**Status:** Draft v2
**Created:** 2026-02-05
**Updated:** 2026-02-06
**Purpose:** AI agent that auto-instruments TypeScript code with OpenTelemetry based on a Weaver schema

### Revision History

| Version | Date | Changes |
|---------|------|---------|
| v1 | 2026-02-05 | Initial draft |
| v2 | 2026-02-06 | Incorporated consolidated technical review. Resolved Q5 (framework choice). Added research spikes, file revert protocol, Coordinator-managed SDK writes, dependency installation workflow, variable shadowing checks, periodic schema checkpoints, configurable limits, in-memory results, span density guardrails, and fix loop ceilings. |

---

## Vision

An AI agent that takes a Weaver schema and code files, then automatically instruments them with OpenTelemetry. The agent prioritizes semantic conventions, can extend the schema as needed, and validates its work through Weaver.

**Short-term goal:** Works on commit-story-v2 repository
**Long-term goal:** Distributable tool that works on any TypeScript codebase

### Why Schema-Driven?

**The gap:** No existing tool combines Weaver schema + AI + source transformation.

Tools like o11y.ai already do AI + TypeScript + PR generation. OllyGarden's tooling (Insights for instrumentation scoring, Tulip for OTel Collector distribution) is adding agentic instrumentation capabilities. But none of them validate against a schema contract. The difference is **"AI guesses what to instrument"** vs **"AI implements a contract that Weaver can verify."**

That verification loop is what makes this approach trustworthy enough to run autonomously. The schema defines the contract, the agent implements it, and Weaver validates the result. Without schema-driven validation, you're trusting AI judgment alone.

---

## Pre-Implementation Research Spikes

Before implementation begins, two research spikes are required. These are timeboxed investigations whose output is documentation, not code. The implementing agent must complete these before writing any agent logic.

### RS1: Prompt Engineering for Code Transformation

**Goal:** Understand what makes LLM prompts reliable for source code modification — not just generation, but transformation of existing code while preserving semantics.

**Deliverable:** A document covering:
- What prompt structures produce correct code transformations (examples, output format constraints, chain-of-thought vs. direct output)
- Common failure modes when LLMs modify existing code (lost context, incomplete edits, hallucinated imports)
- How to structure the "instrumented file" output so it's complete and unambiguous
- Whether the agent should output a full file replacement or a diff/patch
- Findings from any published research or practitioner experience on LLM-based code transformation

**Scope:** Focus on TypeScript instrumentation specifically. Evaluate against the patterns in this spec (wrapping functions with spans, adding imports, try/catch/finally blocks).

### RS2: Validation/Fix Loop Design

**Goal:** Understand how existing AI coding tools handle the "generate → validate → fix → retry" cycle, and design the fix loop mechanics for this agent.

**Deliverable:** A document covering:
- How tools like Cursor, Aider, Claude Code, and similar handle iterative code correction
- What context to include in fix-loop prompts (full file? just the error? original + current?)
- How to prevent the agent from oscillating between different broken states
- Recommended max retry counts and when to bail out
- Whether the fix loop should be a single multi-turn conversation or separate API calls
- Concrete fix loop design recommendation for this agent

**Implementation-time decision:** The fix loop can be modeled as either (a) a single multi-turn conversation where each retry adds the error to context, or (b) separate API calls where each retry starts fresh with just the file, schema, and error. Option (a) preserves what the agent tried; option (b) prevents context bloat. RS2 should evaluate which works better empirically.

**Scope:** Focus on the per-file validation chain (syntax → lint → Weaver). The end-of-run validation is separate and doesn't involve fix loops.

---

## Architecture

### Coordinator + Agents Architecture

The system has a **Coordinator** (deterministic script) that manages workflow and delegates to **AI Agents** for the parts that need intelligence.

#### Coordinator (Not AI)
- **What it is:** A deterministic TypeScript script using Node.js
- **Responsibilities:**
  - Branch management (create feature branch)
  - File iteration (glob for files to process)
  - **File snapshots** before handing to agent (for revert on failure)
  - Spawn AI agent instances
  - Collect results from each agent (in-memory)
  - **Periodic schema checkpoints** (Weaver validation every N files)
  - **Aggregate library requirements** from all agents and perform single SDK init file write
  - **Bulk dependency installation** (`npm install` for all discovered libraries)
  - Run end-of-run validation (Weaver live-check)
  - Assemble PR with summary
- **Why not AI:** These are mechanical tasks that don't need intelligence

#### Schema Builder Agent (Future — Not in PoC)
- **Purpose:** Discover codebase structure, auto-generate Weaver schema
- **Status:** Descoped from PoC — "detecting service boundaries" and "mapping directories to spans" is too ambiguous
- **PoC approach:** Schema must already exist. User or Claude Code creates it manually.
- **Future:** Research how to automatically map directory structure to span patterns

#### Instrumentation Agent (Per-File)
- **Purpose:** Instrument a single file according to existing schema
- **Runs:** Fresh instance per file (prevents laziness)
- **Allowed to:** Read schema + single file, extend schema within guardrails
- **Not allowed to:** Scan codebase, restructure existing schema definitions, modify the SDK init file, run `npm install`
- **Outputs:** In-memory result object returned to Coordinator (see Result Data)

**What "fresh instance" means:** A new LLM API call with a clean context. The context contains only: the system prompt, the resolved Weaver schema, the single file to instrument, and trace context (trace ID + parent span ID for observability). No conversation history, result data, or context from previous files carries over. This is the key mechanism that prevents quality degradation across files — each file gets the agent's full attention as if it were the only file. (Trace context is operational metadata, not task context — it doesn't undermine the quality argument.)

This separation solves the "scope problem": discovery happens once in init, instrumentation follows established patterns. The Coordinator handles the mechanical orchestration.

### Interfaces (cosmetic wrappers around core)
- **MCP server** (PoC) — invoked from Claude Code
- **CLI** (future) — `telemetry-agent instrument src/`
- **GitHub Action** (future) — runs on PR/push

**PoC call chain:** Claude Code invokes MCP server tools → MCP server wraps the Coordinator → Coordinator runs the deterministic workflow (branch, file iteration, validation, PR) and spawns fresh Instrumentation Agent instances for each file. The MCP server is a thin interface layer; the Coordinator is where the orchestration logic lives.

### Technology Stack (PoC)

| Component | Technology | Rationale |
|-----------|------------|-----------|
| Coordinator | Plain TypeScript (Node.js) | Deterministic orchestration doesn't need a framework |
| Instrumentation Agent | Direct Anthropic API via `@anthropic-ai/sdk` | Single provider, maximum control, simplest debugging |
| AST manipulation | ts-morph | TypeScript-native, full type access, scope analysis |
| Schema validation | Weaver CLI (with awareness of Weaver MCP server in v0.21.2) |
| Code formatting | Prettier | Post-transformation formatting |
| MCP interface | MCP TypeScript SDK | Thin wrapper over Coordinator |

**Why direct Anthropic SDK over LangChain/LangGraph:** The agent architecture is simple — the Coordinator is a linear loop, and each Instrumentation Agent is one (or a few) LLM API calls per file. There is no complex state graph, no multi-turn tool-use chains, no branching decision trees. LangGraph solves problems (state machines, checkpointing, complex agent graphs) that this architecture deliberately avoids. The direct SDK provides full control over prompts and API calls with no abstraction overhead, which is critical during prompt iteration. If future complexity demands it (parallel agents with shared state, complex multi-step tool use), migrating to LangGraph is a straightforward refactor — the Coordinator becomes a graph, agent calls become nodes.

**Why not Vercel AI SDK:** Provider-agnostic abstraction adds a layer with no benefit when using a single provider (Claude). The direct Anthropic SDK gives the most transparent debugging experience.

**Note on Weaver MCP server:** Weaver v0.21.2 introduced `weaver registry mcp` — an MCP server providing search, get, and live-check tools directly. Since the PoC architecture already uses MCP (Claude Code → MCP server → Coordinator), the agent could interact with Weaver's native MCP server for schema operations instead of shelling out to CLI commands. Weaver v0.21.2 also added `weaver serve` (REST API + web UI) which could help during development/debugging.

**Implementation-time decision:** Weaver CLI vs MCP server. The CLI approach (`weaver registry check`, `weaver registry resolve`) is simpler — just shell out and parse output. The MCP approach could provide tighter integration but adds architectural complexity (the Coordinator would need to maintain an MCP client connection to Weaver alongside its own MCP server for Claude Code). Start with CLI for PoC; consider MCP if schema operations become a bottleneck.

### Key Tools
- **ts-morph** for AST manipulation (TypeScript-native, full type access)
- **Weaver** for schema validation and live-check
- **Prettier** for post-transformation formatting

---

## Init Phase (Required)

Before instrumentation can begin, user must run `telemetry-agent init`. This is mandatory.

### What Init Does

1. **Verify prerequisites**
   - `package.json` exists → extracts project name for namespace
   - `@opentelemetry/api` in dependencies (or offers to add it)
   - OTel SDK initialization exists somewhere → **records path in config** (e.g., `src/telemetry/setup.ts`)
   - OTLP endpoint configured
   - Test suite exists (warns if missing, continues anyway)
   - Verify localhost port availability for Weaver live-check (:4317 gRPC, :4320 HTTP). Note: if running in Docker, ensure the container can bind to these ports.
   - **Implementation-time note:** If ports 4317/4320 are already in use (e.g., local OTel Collector, another Weaver instance), init should detect this and fail with a clear message. Consider whether to support configurable ports (Weaver may not support this — verify) or require the user to free the ports.

2. **Validate Weaver schema**
   - Schema must already exist (PoC requirement)
   - Run `weaver registry check` to validate
   - If invalid or missing → fail with helpful error

3. **Create config file**
   - Writes `telemetry-agent.yaml` with schema path, SDK init file path, and settings
   - Config file is the gate for instrumentation phase

### Prerequisites (verified during init)

1. **`package.json`** — provides namespace
2. **OTel SDK initialized** — user's responsibility, not agent's job. Init phase records the file path.
3. **`@opentelemetry/api` in dependencies** — minimum SDK version: compatible with OTel JS SDK 2.0+ (see `@opentelemetry/auto-instrumentations-node` v0.68.0 compatibility notes)
4. **OTLP endpoint configured** — for production use (Datadog, Jaeger, etc.). During validation, agent temporarily overrides to point at Weaver.
5. **Test suite exists** — for validation (agent warns if missing). The `testCommand` config accepts any runner (npm test, vitest, jest, nx test, etc.) — fine for PoC with npm, note for post-PoC that arbitrary runners should be well-supported.

---

## Complete Workflow

```text
┌─────────────────────────────────────────────────────────────────┐
│  telemetry-agent init (REQUIRED, ONE-TIME)                      │
│                                                                 │
│  1. Verify prerequisites (package.json, OTel deps, etc.)        │
│  2. Locate and record SDK init file path                        │
│  3. Validate existing schema (weaver registry check)            │
│     - Schema must exist (PoC requirement)                       │
│     - If missing → fail with error                              │
│  4. Create telemetry-agent.yaml config                          │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│  telemetry-agent instrument <path> (REQUIRES INIT)              │
│                                                                 │
│  COORDINATOR (deterministic script):                            │
│  1. Create feature branch                                       │
│  2. Glob for files to process                                   │
│  3. For each file:                                              │
│     a. Snapshot file (copy for revert on failure)               │
│     b. Spawn Instrumentation Agent (fresh instance)             │
│     c. Collect result (in-memory)                               │
│     d. If agent failed → revert file to snapshot                │
│     e. If agent succeeded → commit code + schema changes        │
│     f. Every N files → periodic schema checkpoint               │
│  4. After all files:                                            │
│     a. Aggregate libraries_needed from all results              │
│     b. npm install all discovered libraries (bulk)              │
│     c. Write SDK init file once (register all libraries)        │
│     d. Commit SDK + package.json changes                        │
│  5. Run end-of-run validation (tests + Weaver live-check)       │
│  6. Create PR with summary rendered from collected results      │
│                                                                 │
│  INSTRUMENTATION AGENT (per file, fresh instance):              │
│  1. Read config → get schema path                               │
│  2. Read schema → understand patterns                           │
│  3. Analyze imports → what libraries/frameworks are used?        │
│  4. Check schema for libraries, discover new via allowlist/npm  │
│  5. Record libraries_needed in result (do NOT modify SDK file)  │
│  6. Check for variable shadowing before inserting new variables │
│  7. Add manual spans ONLY for business logic gaps               │
│  8. Extend schema if needed (within guardrails)                 │
│  9. Per-file validation (fix loops): syntax → lint → Weaver     │
│  10. Return result object to Coordinator                        │
└─────────────────────────────────────────────────────────────────┘
```

---

## How It Works

### Input
- Config file (`telemetry-agent.yaml`) — must exist (created by init)
- Weaver schema (resolved from path in config)
- File or directory path to instrument

### Processing (per file)
1. **Check imports** — what libraries/frameworks does this file use?
2. **Check schema for libraries** — does schema already specify instrumentation?
3. **Discover new libraries** — if not in schema, check allowlist then npm registry
4. **Record library needs** — add to result's `libraries_needed` (Coordinator handles installation and SDK registration later)
5. **Find business logic gaps** — code paths that libraries can't instrument
6. **Skip already-instrumented functions** — pattern match for `tracer.startActiveSpan` etc.
7. **Variable shadowing check** — before inserting `span`, `tracer`, or other OTel variables, use ts-morph scope analysis to check for existing variables with the same name. If collision detected, use suffixed names (`otelSpan`, `otelTracer`).
8. **For business logic gaps only:**
   - Determine needed attributes
   - Check semconv first, then existing schema, then create new
   - Add manual span
9. **Update Weaver schema** if new libraries/attributes/spans added
10. **Per-file validation** — syntax → lint → Weaver static (fix loops, max attempts enforced)
11. **Return result** to Coordinator

### Output
- Single PR with all changes (code + schema updates + SDK init + package.json)
- Validation results rendered in PR description

---

## File/Directory Processing

- **User specifies:** file or directory
- **Directory processing:** sequential, one file at a time
- **New AI instance per file:** prevents laziness, ensures quality
- **Schema changes propagate:** via git commits on feature branch
- **SDK init file:** written once by Coordinator after all agents complete
- **Dependency installation:** one bulk `npm install` by Coordinator after all agents complete
- **Single PR at end:** contains all instrumented files, schema updates, SDK init changes, and package.json updates
- **Configurable file limit:** Default 50 files per run (configurable via `maxFilesPerRun`). This is a cost/time guardrail, not an architectural constraint — the Coordinator's design (centralized SDK writes, in-memory results, independent agents) supports higher file counts without structural changes. If the glob returns more files than the limit, the Coordinator fails with an error suggesting the user adjust the limit or target a subdirectory.

### File Revert Protocol

The Coordinator snapshots each file before handing it to the Instrumentation Agent. If the agent returns a `"failed"` status (e.g., syntax errors it couldn't fix after max attempts), the Coordinator reverts the file to its pre-agent state before continuing to the next file. This ensures the feature branch remains compilable even with partial failures.

Implementation: The Coordinator copies the file (and its corresponding schema state if the agent modified it) to a temp location before processing. On failure, it restores from the snapshot and discards the agent's changes. On success, it commits the agent's changes to the feature branch.

### Periodic Schema Checkpoints

To catch schema drift early instead of discovering it only at end-of-run, the Coordinator runs `weaver registry check` every `schemaCheckpointInterval` files (default: 5).

If a checkpoint fails, the Coordinator stops processing, reports which files were processed since the last successful checkpoint, and lets the user decide whether to fix and continue or abort. This bounds the blast radius of schema drift to at most N files of work.

### Future: Parallel Processing

The architecture supports parallelism without structural changes: agents are independent (no shared state, no cross-file context), results are in-memory, and shared resources (SDK init, package.json) are written by the Coordinator after all agents complete. The only hard problem is schema merging — two parallel agents might create conflicting schema entries. A Coordinator-level merge strategy (collect all schema changes, deduplicate, write once) would address this. Raising `maxFilesPerRun` beyond the default requires no architecture changes, just longer run times.

---

## What Gets Instrumented

### Heuristics (agent decides where to instrument)
- Exported async functions
- Functions in `services/`, `handlers/`, `api/` directories
- Functions making external calls (DB, HTTP, etc.)
- Skip: utilities, formatters, pure helpers, functions under ~5 lines, pure synchronous utilities

### Span Density Guardrails

LLMs tend to over-instrument when told to "add telemetry." To prevent excessive span creation:

- **Per-file cap:** Maximum 5 manual spans per file (configurable via `maxSpansPerFile`). If the agent needs more, it should prioritize the highest-value instrumentation points and note what was skipped in the result.
- **Project-wide cap:** The Coordinator sums `spans_added` across all results. If the total exceeds `maxSpansPerRun` (default: 50), the Coordinator flags the run for human review.
- **System prompt constraint:** The agent's prompt explicitly limits instrumentation to entry points, external calls (DB/HTTP/gRPC), and complex business logic. It prohibits instrumenting simple utilities, formatters, or pure functions.

These are guardrails, not hard failures — exceeding them triggers a warning and human review, not an abort.

### Schema guidance
- Schema defines attribute groups and naming patterns
- Schema can define specific spans for critical paths (overrides heuristics)
- Agent applies schema conventions to discovered instrumentation points

### Patterns Not Covered (PoC)

The PoC focuses on request-path functions (handlers, services, external calls). These async/event-driven patterns are common in TypeScript services but require different instrumentation strategies — future work:

- Event handlers / event emitters
- Pub/sub callbacks
- Cron jobs / scheduled tasks
- Queue consumers

---

## What the Agent Actually Does to Code

Two paths: auto-instrumentation (common) and manual spans (fallback).

### Path 1: Auto-Instrumentation Library (Primary)

Agent detects a framework import and records the library need. The Coordinator handles registration.

**Scenario:** File imports `pg` (PostgreSQL client)

**Step 1: Agent detects import in target file**
```typescript
// src/services/user-service.ts
import { Pool } from 'pg';

export async function getUser(id: string) {
  const result = await pool.query('SELECT * FROM users WHERE id = $1', [id]);
  return result.rows[0];
}
```

**Step 2: Agent records library need in result**
```json
{
  "libraries_needed": ["@opentelemetry/instrumentation-pg"]
}
```

The agent does NOT modify the SDK init file. The Coordinator handles this after all agents complete.

**Step 3: Coordinator writes SDK init file (once, after all agents)**
```typescript
// src/telemetry/setup.ts (path from config's sdkInitFile)
import { NodeSDK } from '@opentelemetry/sdk-node';
import { PgInstrumentation } from '@opentelemetry/instrumentation-pg';

const sdk = new NodeSDK({
  serviceName: 'commit-story',
  instrumentations: [
    new PgInstrumentation(),
  ],
});
sdk.start();
```

**Step 4: Agent updates Weaver schema**
```yaml
# telemetry/registry/signals.yaml (added)
groups:
  - id: span.commit_story.db.pg
    type: span
    brief: PostgreSQL database operations (via instrumentation-pg)
    span_kind: client
    note: Auto-instrumentation library handles span creation
    attributes:
      - ref: db.system
        requirement_level: required
      - ref: db.namespace
        requirement_level: recommended
      - ref: db.operation.name
        requirement_level: recommended
      - ref: db.statement
        requirement_level: recommended
```

**Key point:** The target file (`user-service.ts`) is NOT modified. The library handles instrumentation automatically once registered.

**Schema vs. Runtime Dependencies:** The schema entry (Step 4) and `libraries_needed` (Step 2) serve different purposes and both are required. The schema defines the telemetry contract — what spans will be emitted and what attributes they'll have. This enables Weaver live-check to validate that the running code produces what the schema says it should. The `libraries_needed` array tells the Coordinator which npm packages to install and register in the SDK init file. You can't have validation without the schema entry, and you can't have working instrumentation without the library installed. They're complementary, not redundant.

### Path 2: Manual Span (Fallback for Business Logic)

Agent wraps business logic that no library can instrument.

**Scenario:** Custom journal generation function

**Before:**
```typescript
// src/generators/summary.ts
export async function generateSummary(context: Context): Promise<string> {
  const filtered = filterContext(context);
  const prompt = buildPrompt(filtered);
  const response = await callAI(prompt);
  return response.content;
}
```

**After:**
```typescript
// src/generators/summary.ts
import { trace, SpanStatusCode } from '@opentelemetry/api';

const tracer = trace.getTracer('commit-story');

export async function generateSummary(context: Context): Promise<string> {
  return tracer.startActiveSpan('commit_story.journal.generate_summary', async (span) => {
    try {
      span.setAttribute('commit_story.context.messages_count', context.messages.length);

      const filtered = filterContext(context);
      const prompt = buildPrompt(filtered);
      const response = await callAI(prompt);

      span.setAttribute('commit_story.journal.word_count', response.content.split(' ').length);
      return response.content;
    } catch (error) {
      span.recordException(error);
      span.setStatus({ code: SpanStatusCode.ERROR });
      throw error;
    } finally {
      span.end();
    }
  });
}
```

**Variable shadowing note:** Before inserting `span` and `tracer`, the agent uses ts-morph's scope analysis to verify these names don't collide with existing variables. If `span` already exists in scope, the agent uses `otelSpan` instead (and adjusts all references). If `tracer` exists, it uses `otelTracer`. This prevents silent runtime bugs that would pass all validation.

**Schema entry created:**
```yaml
# telemetry/registry/signals.yaml (added)
groups:
  - id: span.commit_story.journal.generate_summary
    type: span
    stability: development
    brief: AI-powered journal summary generation
    span_kind: internal
    attributes:
      - ref: commit_story.context.messages_count
        requirement_level: recommended
      - ref: commit_story.journal.word_count
        requirement_level: recommended
```

### Decision: Which Path?

| Import Detected | OTel Library Exists? | Action |
|-----------------|---------------------|--------|
| `pg`, `express`, `http`, etc. | Yes | Path 1: Record library need for Coordinator |
| Custom business logic | N/A | Path 2: Wrap with manual span |
| Already instrumented | N/A | Skip |

---

## Attribute Priority Chain

When agent needs an attribute:

1. **Check OTel semantic conventions** — use semconv reference if exists
2. **Check existing Weaver schema** — use if already defined
3. **Neither exists** — create new custom attribute under project namespace

**Agent has full authority to extend schema** (create spans, attributes, groups) — within the guardrails below.

### Schema Extension Guardrails

The agent can extend the schema, but must follow existing patterns:

1. **Namespace prefix is mandatory** — New attributes MUST use the project namespace prefix as defined in the existing schema. If the schema uses `commit_story.*`, all new attributes must follow that prefix.

2. **Follow existing structural patterns** — New attribute groups should match the naming and structural conventions already present in the user-provided schema.

3. **Add or create, but stay consistent** — Agent can add to existing groups or create new ones, but must observe and follow the conventions in the existing schema.

4. **Coordinator enforces via drift detection** — The Coordinator sums `attributes_created` and `spans_added` across all results. Unreasonable totals (e.g., 30 new attributes for a single file) get flagged for human review.

---

## Auto-Instrumentation Libraries

**Libraries are the PRIMARY approach.** Manual spans are only for gaps.

### Why Libraries First
- Auto-instrumentation libraries are battle-tested
- They handle framework internals correctly (middleware, connection pooling, etc.)
- Less code to maintain
- Follows OTel best practices

### Detection Flow
1. **Check schema first** — does schema already reference an auto-instrumentation library for this file's imports?
   - If yes → record library need (schema is source of truth)
2. **Discovery** — if file uses a framework not in schema (e.g., `import express from 'express'`):
   - Check allowlist first, then query npm registry as fallback
   - If found and `autoApproveLibraries: true`:
     - Record library in result's `libraries_needed`
     - Add library's spans/attributes to Weaver schema (semconv references)
   - If found and `autoApproveLibraries: false` → prompt user

### Library Discovery

**Approach:** Hardcoded allowlist first, npm registry as fallback.

**Trusted Allowlist (PoC)**

Common OTel JS instrumentation packages, sourced from `@opentelemetry/auto-instrumentations-node`. Note: this allowlist should be reviewed quarterly against the upstream package list, as packages may be deprecated or replaced (see Fastify below).

| Framework/Library | Instrumentation Package |
|-------------------|------------------------|
| `http` / `https` | `@opentelemetry/instrumentation-http` |
| `express` | `@opentelemetry/instrumentation-express` |
| `pg` | `@opentelemetry/instrumentation-pg` |
| `mysql` / `mysql2` | `@opentelemetry/instrumentation-mysql` / `mysql2` |
| `mongodb` | `@opentelemetry/instrumentation-mongodb` |
| `redis` / `ioredis` | `@opentelemetry/instrumentation-redis` / `ioredis` |
| `grpc` / `@grpc/grpc-js` | `@opentelemetry/instrumentation-grpc` |
| `koa` | `@opentelemetry/instrumentation-koa` |
| `fastify` | `@fastify/otel` (note: `@opentelemetry/instrumentation-fastify` is deprecated as of auto-instrumentations-node v0.68.0, Feb 2026) |
| `nestjs` | `@opentelemetry/instrumentation-nestjs-core` |
| `mongoose` | `@opentelemetry/instrumentation-mongoose` |
| `kafkajs` | `@opentelemetry/instrumentation-kafkajs` |
| `pino` | `@opentelemetry/instrumentation-pino` |
| `@anthropic-ai/sdk` | `@traceloop/instrumentation-anthropic` |
| `openai` | `@traceloop/instrumentation-openai` |

**Note on OpenLLMetry packages:** The `@traceloop/instrumentation-*` packages are from OpenLLMetry, OTel extensions for LLM observability created by Traceloop. Their work contributed to the official GenAI semantic conventions now in OTel. These packages provide auto-instrumentation for LLM SDK calls (Anthropic, OpenAI, etc.) and emit `gen_ai.*` semconv attributes. For codebases with LLM integrations, these provide the same benefits as other auto-instrumentation libraries — no manual span wrapping needed for LLM calls.

**npm Registry Search (Fallback)**

If a file imports a framework not in the allowlist, the agent queries npm as a discovery mechanism. This is best-effort — the allowlist is the trusted set.

**Future:** Vector database synced with OTel ecosystem

Sources: [@opentelemetry/auto-instrumentations-node](https://github.com/open-telemetry/opentelemetry-js-contrib/tree/main/packages/auto-instrumentations-node)

### Schema Updates for Libraries
When a library is added, the schema should reference what it produces:
```yaml
# Example: express instrumentation added
groups:
  - id: span.myapp.http.server
    type: span
    brief: HTTP server spans from express instrumentation
    span_kind: server
    attributes:
      - ref: http.request.method
        requirement_level: required
      - ref: url.path
        requirement_level: required
      - ref: http.response.status_code
        requirement_level: required
      - ref: http.route
        requirement_level: recommended
```

This enables Weaver live-check to validate library output.

### Manual Spans (Secondary)
Only add manual spans for:
- Business logic that libraries can't see
- Custom operations with no framework equivalent
- Schema-defined critical paths

Agent does NOT add manual spans for things libraries already handle.

---

## Validation Chain

Validation stays in the Instrumentation Agent so it can fix what it breaks. The Coordinator enforces limits and handles failures.

### Per-File Validation (with fix loops)

Each check runs in a loop: validate → fix errors → retry until clean or max attempts reached.

1. **Syntax** — TypeScript compiler / ts-morph (code must compile)
2. **Lint** — Prettier/ESLint (code must be properly formatted)
3. **Weaver registry check** — static schema validation

Order matters: syntax first (must compile), then lint, then schema.

**Fix loop limits:** The agent retries up to `maxFixAttempts` (default: 3) per validation stage. If the agent cannot produce clean code within the limit, it returns a `"failed"` status and the Coordinator reverts the file. The `maxTokensPerFile` budget (default: 50,000) provides a hard ceiling on total token usage per file across all attempts.

**Variable shadowing check:** Before inserting new variables (`span`, `tracer`, etc.), the agent uses ts-morph's scope analysis (TypeScript binder access) to check for existing variables with the same name in the target scope. If a collision is detected, the agent uses suffixed names (`otelSpan`, `otelTracer`) or reports the collision and skips instrumentation for that function. This check happens before the validation loop — it's a pre-condition, not something that gets "fixed" in a retry.

### End-of-Run Validation (once, after all files)

These are slow, so they run once before creating the PR. They run after the Coordinator has completed SDK init file writes and dependency installation.

1. **Run tests with Weaver live-check**

Weaver acts as an OTLP receiver itself — no external collector needed for validation.

```text
Agent workflow:
1. Start Weaver: weaver registry live-check -r ./registry
   → Listens on localhost:4317 (gRPC) and :4320 (HTTP /stop)

2. Run tests with endpoint override:
   OTEL_EXPORTER_OTLP_ENDPOINT=localhost:4317 npm test
   → Tests exercise code, telemetry goes to Weaver

3. Stop Weaver: curl localhost:4320/stop
   → Weaver outputs compliance report

4. Parse results:
   → Coverage stats, violations, advisories by severity
```

The agent temporarily overrides the OTLP endpoint during validation. Normal production telemetry goes to the user's configured backend.

**Implementation-time note:** The `testCommand` config value (e.g., `"npm test"`) is executed with the OTLP endpoint override via environment variable. Verify during implementation that the test runner correctly inherits environment variables when spawned. Some runners or CI configurations may require explicit env passing. If interpolation is needed (e.g., `OTEL_EXPORTER_OTLP_ENDPOINT=${endpoint} npm test`), document the pattern in the config schema.

### Future Optimizations
- Smart test discovery (only tests touching changed files)
- Backend verification (query Datadog/Jaeger to confirm data arrived)

### If Tests Don't Exist
- Agent warns user
- Skips live-check validation
- Per-file validations still run

---

## Agent Self-Instrumentation

How the telemetry agent instruments its own operations.

### Bootstrapping

The agent cannot instrument itself using itself — that's circular. Its own instrumentation is hand-written, a fixed known set of spans covering the agent's workflow. This is distinct from the instrumentation it adds to target codebases.

### Separate Telemetry Pipelines

The agent maintains two independent telemetry streams that must not cross:

| Pipeline | Destination | Purpose |
|----------|-------------|---------|
| Target codebase telemetry | Weaver (validation) → user's backend (production) | What the agent creates |
| Agent operational telemetry | Developer's observability backend (e.g., Datadog) | Debugging the agent itself |

The OTLP endpoint override during Weaver live-check applies only to the target codebase's SDK, not to the agent's own exporter.

### Coordinator Spans (one trace per run)

| Span | Attributes |
|------|------------|
| `telemetry_agent.run` | Parent span for entire instrumentation run |
| | `files.attempted`, `files.succeeded`, `files.failed` |
| | `duration_ms` |
| | `schema_drift.attributes_created`, `schema_drift.spans_added` (totals from results) |
| | `validation.tests_passed`, `validation.weaver_compliance` (end-of-run outcomes) |

### Instrumentation Agent Spans (child spans per file)

Nested under the Coordinator's trace:

| Span | Purpose |
|------|---------|
| `telemetry_agent.file.process` | Parent span per file |
| `telemetry_agent.file.analyze` | File analysis (imports detected, frameworks found) |
| `telemetry_agent.file.discover_libraries` | Library lookup (allowlist hit or npm fallback, what was found) |
| `telemetry_agent.file.transform` | Code transformation (manual spans added, libraries recorded) |
| `telemetry_agent.file.validate` | Validation loop iteration (syntax/lint/Weaver, pass or fail) |

`telemetry_agent.file.validate` should include attributes for `retry_count` and `check_failed` (which check failed if any).

LLM calls should use `gen_ai.*` semconv attributes: `gen_ai.request.model`, `gen_ai.usage.input_tokens`, `gen_ai.usage.output_tokens`, latency.

### Trace Context Propagation

The Coordinator creates the root span and must propagate trace context to each Instrumentation Agent instance. Since each agent is a fresh LLM API call (direct Anthropic SDK), the Coordinator passes the trace context (trace ID + parent span ID) as parameters in the system prompt or tool input, and wraps the API call with the appropriate child span on the Coordinator side. There is no built-in OTel context propagation for LLM API calls — this must be handled manually by the Coordinator.

### PoC Scope

**Minimum:** Coordinator-level spans and LLM call spans with `gen_ai.*` attributes.

**Nice-to-have:** Per-phase child spans within the Instrumentation Agent.

The Weaver schema for the agent itself (as opposed to target codebases) is a separate registry that lives in the agent's own repo.

---

## Handling Existing Instrumentation

### Already instrumented (complete)
- Pattern match for `tracer.startActiveSpan`, `tracer.startSpan`, etc.
- Skip the function

### Broken instrumentation
- Agent is **additive only** — doesn't try to detect/fix broken patterns
- Weaver validation catches issues
- Agent fixes what Weaver reports

### Telemetry removed by user
- If user deletes instrumentation but schema still defines it
- Agent re-instruments according to schema (schema is source of truth)
- If user wants telemetry gone: don't feed file to agent

---

## Schema as Source of Truth

- Weaver schema defines what SHOULD be instrumented
- Agent implements the contract
- Agent can EXTEND schema (with semconv check)
- Code follows schema, not the other way around

---

## Result Data

Each Instrumentation Agent returns a result object to the Coordinator. Results are held in-memory during the run — no filesystem directory, no committed JSON files.

### Why In-Memory Results

Results are collected by the Coordinator as it processes each file sequentially. Since the Coordinator is a single process running a simple loop, it already has all results in memory by the time it needs them. No inter-process communication, no filesystem coordination, no cleanup.

For debugging, the Coordinator can optionally write results to a gitignored directory when run with `--verbose` or `--debug`. This is a debug aid, not architecture.

### Result Structure

```typescript
interface FileResult {
  path: string;
  status: "success" | "failed";
  spans_added: number;
  libraries_needed: string[];       // Coordinator handles installation
  schema_extensions: string[];       // IDs of new schema entries
  attributes_created: number;
  validation_retries: number;
  reason?: string;                   // present if failed
  last_error?: string;               // present if failed
}
```

Success example:
```json
{
  "path": "src/services/payment.ts",
  "status": "success",
  "spans_added": 3,
  "libraries_needed": ["@opentelemetry/instrumentation-pg"],
  "schema_extensions": ["span.commit_story.payment.process"],
  "attributes_created": 2,
  "validation_retries": 1
}
```

Failure example:
```json
{
  "path": "src/services/crypto.ts",
  "status": "failed",
  "spans_added": 0,
  "libraries_needed": [],
  "schema_extensions": [],
  "attributes_created": 0,
  "validation_retries": 3,
  "reason": "syntax errors after 3 fix attempts",
  "last_error": "Unexpected token at line 42"
}
```

### PR Summary

The Coordinator renders results into the PR description as a human-readable summary table. This is the primary way reviewers see what the agent did. The PR description includes per-file status, total spans added, libraries discovered, schema extensions, and any failures with reasons. This builds trust with reviewers without polluting the git history with machine-readable artifacts.

### Future: Schema State Tracking

Each agent can extend the schema, and extensions propagate via git commits. If Agent C's extension conflicts with Agent B's, periodic schema checkpoints and end-of-run Weaver validation catch it.

For future debugging, could add schema version/hash per agent run to track what schema state each agent started with. Not needed for PoC.

---

## Configuration

The config file is created during `telemetry-agent init` and serves as the gate for instrumentation. If no config file exists, the Instrumentation Agent refuses to run.

```yaml
# telemetry-agent.yaml (created by init, checked into repo)

# Required
schemaPath: ./telemetry/registry
sdkInitFile: ./src/telemetry/setup.ts    # recorded during init

# Agent behavior
autoApproveLibraries: true    # false = prompt before adding OTel libraries
testCommand: "npm test"        # command to run test suite (supports any runner)

# Limits and guardrails
maxFilesPerRun: 50             # cost/time guardrail, user adjustable
maxFixAttempts: 3              # bail out after N failed validation cycles per file
maxTokensPerFile: 50000        # hard token budget per file (all attempts combined)
maxSpansPerFile: 5             # manual span cap per file (triggers review if exceeded)
maxSpansPerRun: 50             # project-wide manual span cap (triggers review if exceeded)
schemaCheckpointInterval: 5    # run Weaver validation every N files
```

### What Goes Where

| In Config | In Schema |
|-----------|-----------|
| Schema path | Namespace (authoritative) |
| SDK init file path | Semconv version |
| Test command | Attribute definitions |
| Agent behavior settings | Span definitions |
| Limits and guardrails | |

The config tells the agent **how to run**. The schema tells it **what telemetry looks like**.

Note: OTLP endpoint for production is configured in the user's OTel SDK setup, not here. During validation, the agent temporarily uses Weaver as the receiver.

---

## Minimum Viable Schema Example

A complete, valid Weaver schema that passes `weaver registry check`. Based on commit-story-v2.

**Location:** `telemetry/registry/`

### registry_manifest.yaml

```yaml
# The root manifest — defines namespace and pulls in OTel semconv
name: commit_story
description: OpenTelemetry semantic conventions for commit-story
semconv_version: 0.1.0
schema_base_url: https://commit-story.dev/schemas/

# Import official OTel semantic conventions
# The [model] suffix specifies the subdirectory containing registry files
dependencies:
  - name: otel
    registry_path: https://github.com/open-telemetry/semantic-conventions/archive/refs/tags/v1.37.0.zip[model]
```

**Semconv version note:** This example pins to v1.37.0. The latest release is v1.39.0, which includes significant GenAI semconv updates (`gen_ai.conversation.id`, reasoning content message parts, multimodal support, agent span kind guidance) and the database semconv stability push where `db.name` is being replaced by `db.namespace` (v1.38.0+). For the PoC, v1.37.0 is fine — but the `db.name` → `db.namespace` migration will be needed when upgrading. Consider bumping to v1.39.0 if GenAI conventions are important for commit-story-v2.

### attributes.yaml

```yaml
# Custom attributes for gaps not covered by OTel semconv
groups:
  # Attribute group with OTel refs + custom attributes
  - id: registry.commit_story.ai
    type: attribute_group
    display_name: AI Generation Attributes
    brief: Attributes for AI content generation
    attributes:
      # Reference OTel GenAI semconv (resolved from dependencies)
      - ref: gen_ai.request.model
        requirement_level: required
      - ref: gen_ai.operation.name
        requirement_level: required
      - ref: gen_ai.usage.input_tokens
        requirement_level: recommended
      - ref: gen_ai.usage.output_tokens
        requirement_level: recommended

      # Custom extension (not in OTel semconv)
      - id: commit_story.ai.section_type
        type:
          members:
            - id: summary
              value: summary
              brief: Daily summary generation
              stability: development
            - id: dialogue
              value: dialogue
              brief: Developer dialogue extraction
              stability: development
        stability: development
        brief: The type of journal section being generated

  # Pure custom attribute group (no OTel equivalent)
  - id: registry.commit_story.journal
    type: attribute_group
    display_name: Journal Attributes
    brief: Attributes for journal entry output
    attributes:
      - id: commit_story.journal.entry_date
        type: string
        stability: development
        brief: The date of the journal entry (YYYY-MM-DD)
        examples: ["2026-02-03"]

      - id: commit_story.journal.word_count
        type: int
        stability: development
        brief: Total word count of generated entry
        examples: [450, 1200]
```

### signals.yaml

```yaml
# Span definitions — what telemetry the system emits
groups:
  # Span using OTel semconv attributes (for library instrumentation)
  - id: span.commit_story.db.operations
    type: span
    brief: Database operations (via @opentelemetry/instrumentation-pg)
    span_kind: client
    note: Auto-instrumentation library handles span creation
    attributes:
      # Reference OTel db semconv via ref
      - ref: db.system
        requirement_level: required
      - ref: db.namespace
        requirement_level: recommended
      - ref: db.operation.name
        requirement_level: recommended

  # Custom span (for manual instrumentation)
  - id: span.commit_story.journal.generate_summary
    type: span
    stability: development
    brief: AI-powered journal summary generation
    span_kind: internal
    attributes:
      - ref: gen_ai.request.model
        requirement_level: required
      - ref: commit_story.ai.section_type
        requirement_level: required
      - ref: commit_story.journal.word_count
        requirement_level: recommended
```

### Validation

```bash
# Validate schema syntax and references
weaver registry check -r ./telemetry/registry

# Resolve and view (includes semconv from dependencies)
weaver registry resolve -r ./telemetry/registry -f json -o resolved.json
```

### Key Points

| File | Purpose |
|------|---------|
| `registry_manifest.yaml` | Namespace, semconv version, OTel dependency |
| `attributes.yaml` | Custom attributes + refs to OTel semconv |
| `signals.yaml` | Span definitions (library-backed and manual) |

The agent reads the resolved output, which includes all semconv references expanded inline.

---

## Resolved Questions

### Q1: Weaver Schema Structure
- **Scope problem:** Solved by requiring schema to exist before instrumentation (Schema Builder descoped from PoC)
- **Minimum viable schema:** See "Minimum Viable Schema Example" section above
- **commit-story-v2 reference:** `telemetry/registry/` contains a working example

### Q2: Live-Check Setup
Weaver acts as its own OTLP receiver:
- Agent starts `weaver registry live-check` (listens on :4317)
- Agent runs tests with `OTEL_EXPORTER_OTLP_ENDPOINT=localhost:4317`
- Agent stops Weaver via `curl localhost:4320/stop`
- Agent parses compliance report

No external collector needed for validation. Test suite needs no code changes — just environment variable override.

### Q3: Error Handling
**Approach:** Fail forward with file revert. Commit successful files, revert and skip failed ones, create PR with summary.

- Per-file failures → Coordinator reverts file to snapshot, continues to next
- End-of-run failures → note in PR, create anyway
- Partial success with clear reporting and a compilable branch

### Q4: Semconv Lookup
The agent doesn't need to look up semconv separately — it's already in the resolved schema.

- `registry_manifest.yaml` has a `dependencies` field that pulls in OTel semconv
- `weaver registry resolve` produces output with resolved semconv references
- Agent just reads the resolved JSON and has everything it needs

### Q5: Agent Framework Choice
**Resolved: Direct Anthropic TypeScript SDK (`@anthropic-ai/sdk`)**

The architecture is deliberately simple — a deterministic Coordinator loop and one-shot LLM API calls per file. This doesn't require the complex state management, graph execution, or checkpointing that LangGraph provides. The direct SDK gives maximum control over prompts, full transparency for debugging, and zero framework dependencies. See "Technology Stack (PoC)" section for full rationale.

### Q6: Quality Checks for AI
Four levers:

1. **Fresh instance per file** — Prevents laziness, each file gets full attention
2. **Weaver validation chain** — Catches errors via static check and live-check
3. **Schema drift detection** — Coordinator sums `attributes_created` and `spans_added` across results; periodic schema checkpoints catch drift early
4. **Span density guardrails** — Per-file and project-wide caps prevent over-instrumentation

---

## Dependencies

### PRD 25 (Autonomous Dev Infrastructure)
- Should be completed first
- Provides testing infrastructure for this repo
- Agent doesn't depend on it architecturally (self-contained validation)
- But this repo benefits from having it

### Separate Repository
- Agent should live in its own repo
- commit-story-v2 is first test subject, not home
- Enables distribution to other codebases

---

## PoC Scope

### In Scope
- Assumes target codebase already has OTel API and SDK installed, initialized, and a valid Weaver schema in place
- Pre-implementation research spikes (RS1: prompt engineering, RS2: fix loop design)
- MCP server interface
- Coordinator + agent architecture:
  - Coordinator (deterministic TypeScript script for orchestration)
  - Instrumentation Agent (per-file, via direct Anthropic SDK)
  - Schema Builder Agent descoped (schema must exist)
- Mandatory init phase with prerequisite verification and SDK init file path recording
- Coordinator-managed SDK init file writes (single write after all agents)
- Coordinator-managed dependency installation (bulk npm install)
- File revert protocol (snapshot before agent, revert on failure)
- In-memory result collection with PR description rendering
- Variable shadowing checks via ts-morph scope analysis
- TypeScript support
- Traces only (no metrics/logs yet)
- File/directory input
- Sequential processing with fresh agent instance per file
- Configurable file limit (default 50)
- Periodic schema checkpoints
- PR output
- Per-file validation with fix loops: syntax, lint, Weaver static (with max attempts and token budget)
- Span density guardrails (per-file and project-wide)
- End-of-run validation: tests, Weaver live-check
- Allowlist-first library discovery with npm registry fallback
- Schema extension with semconv priority

### Out of Scope (Future)
- Schema Builder Agent (auto-generate schema from codebase discovery)
- Async/event-driven patterns (event emitters, pub/sub, cron jobs, queue consumers)
- CLI and GitHub Action interfaces
- Multi-agent for different signal types (separate metrics/logs/traces agents)
- Other languages
- Smart test discovery
- Vector database for OTel knowledge
- Backend verification (query observability platform)
- Configurable instrumentation levels (dev-heavy vs production-selective)
- Parallel agent execution (architecture supports it; needs schema merge strategy)

---

## Research Summary

### Prior Art
- **o11y.ai** — AI-powered, TypeScript, generates PRs. Not schema-driven.
- **OllyGarden** — Insights (instrumentation scoring), Tulip (OTel Collector distribution, Oct 2025). Agentic instrumentation direction confirmed by co-founder Juraci at KubeCon NA 2025 and December 2025 panel.
- **Orchestrion (Datadog)** — Compile-time AST instrumentation for Go.

See "Why Schema-Driven?" in Vision section for why the gap matters.

### Optional Telemetry Pattern
- `@opentelemetry/api` has zero dependencies (~8-10KB)
- Use peer dependencies with optional flag
- Without SDK initialization, all API calls are automatic no-ops
- Agent can instrument code that works with or without OTel present

Relevant when the agent targets codebases that don't already have OTel installed (see Out of Scope — PoC assumes OTel is pre-installed).

### Instrumentation Level (Still Open)
- "Instrument everything" is NOT industry best practice for production
- Consensus: auto-instrumentation baseline + manual for business-critical paths
- v1's heavy dev instrumentation was intentional for AI debugging assistance
- Agent should eventually support configurable modes

---

## References

### Research Documents
- [Prior Art: AI Instrumentation Tools](./research/telemetry-agent-prior-art.md)
- [Optional Telemetry Patterns](./research/optional-telemetry-patterns.md)
- [TypeScript AST Tools](./research/typescript-ast-tools.md)
- [Instrumentation Level Strategies](./research/instrumentation-level-strategies.md)
- [Weaver TypeScript Capabilities](./research/weaver-typescript.md)

### External Resources
- [OllyGarden](https://ollygarden.com/)
- [OpenTelemetry Weaver](https://github.com/open-telemetry/weaver)
- [ts-morph](https://ts-morph.com/)
- [Anthropic TypeScript SDK](https://github.com/anthropics/anthropic-sdk-typescript)
