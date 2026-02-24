# Weaver TypeScript Capabilities

**Research Date:** 2026-02-05

## Executive Summary

OpenTelemetry Weaver does NOT natively support TypeScript code generation, but provides excellent workarounds through its resolved JSON/YAML output that can be consumed by AI agents or custom code generators. For our use case, this is actually preferable to static code generation.

---

## Weaver Code Generation Support

### Officially Supported Languages

| Language | Status |
|----------|--------|
| Go | Production |
| Java | Production |
| Rust | Production |
| Python | Production |
| Erlang | Community |
| Markdown | Production (docs) |
| **TypeScript** | **Not supported** |

The `@opentelemetry/semantic-conventions` npm package exists but is NOT generated via Weaver - it uses a separate process.

---

## Weaver Registry Format

### Directory Structure
```text
telemetry/registry/
├── registry_manifest.yaml
├── attributes.yaml
└── signals.yaml
```

### registry_manifest.yaml
```yaml
name: my_app
description: Custom semantic conventions
semconv_version: 0.1.0
schema_base_url: https://example.com/schemas/

dependencies:
  - name: otel
    registry_path: https://github.com/open-telemetry/semantic-conventions/archive/refs/tags/v1.37.0.zip[model]
```

### Attribute Definitions
```yaml
groups:
  - id: registry.my_app.http
    type: attribute_group
    display_name: HTTP Attributes
    brief: Custom HTTP attributes
    attributes:
      - id: my_app.http.request_id
        type: string
        stability: development
        brief: Unique request identifier
        examples: ["abc-123", "xyz-789"]
```

### Span Definitions
```yaml
groups:
  - id: span.my_app.process
    type: span
    stability: development
    brief: Main processing operation
    span_kind: internal
    attributes:
      - ref: my_app.http.request_id
        requirement_level: required
      - ref: gen_ai.request.model  # OTel semconv reference
        requirement_level: recommended
```

---

## Weaver CLI Commands

| Command | Purpose |
|---------|---------|
| `weaver registry check` | Validate registry syntax and references |
| `weaver registry resolve` | Produce fully-resolved output (JSON/YAML) |
| `weaver registry generate` | Generate code/docs from templates |
| `weaver registry live-check` | Validate live OTLP telemetry against schema |
| `weaver registry diff` | Compare two registry versions |

### Key Commands for AI Agent

**Static validation:**
```bash
weaver registry check -r ./telemetry/registry
```

**Resolve to JSON (for AI consumption):**
```bash
weaver registry resolve -r ./telemetry/registry -f json -o resolved.json
```

**Live validation:**
```bash
weaver registry live-check -r ./telemetry/registry --otlp-grpc-port 4317
```

---

## Resolved Output for AI Agents

### What It Contains
```json
{
  "groups": [
    {
      "id": "span.commit_story.ai.generate",
      "type": "span",
      "stability": "development",
      "brief": "AI content generation operation",
      "span_kind": "client",
      "attributes": [
        {
          "id": "gen_ai.request.model",
          "type": "string",
          "stability": "development",
          "brief": "The name of the GenAI model",
          "requirement_level": "required",
          "examples": ["gpt-4", "claude-3-opus"]
        }
      ]
    }
  ]
}
```

### Information Available to AI

| Field | AI Agent Use |
|-------|--------------|
| `id` | Identify the signal/attribute |
| `type` | Know what kind (span, metric, attribute_group) |
| `brief` | Understand what to instrument |
| `attributes[].id` | The attribute key to set |
| `attributes[].type` | Expected data type |
| `attributes[].requirement_level` | required vs recommended |
| `attributes[].examples` | Sample values for context |
| `span_kind` | client/server/internal |

---

## Workarounds for TypeScript

### Option 1: Schema-Only (Recommended)

Use Weaver purely for validation. AI agent reads resolved JSON directly:

1. Define conventions in Weaver YAML
2. Run `weaver registry resolve -f json -o resolved.json`
3. AI agent reads `resolved.json`
4. AI generates TypeScript code directly

**Advantages:**
- No template maintenance
- AI adapts to any codebase patterns
- Schema remains single source of truth

### Option 2: Custom Jinja2 Templates

Create TypeScript templates:
```jinja2
{# templates/typescript/attributes.j2 #}
export const {{ ctx.root_namespace | screaming_snake_case }}_ATTRIBUTES = {
{% for attr in ctx.attributes %}
  {{ attr.name | screaming_snake_case }}: '{{ attr.name }}',
{% endfor %}
} as const;
```

**Weaver filters available:**
- `snake_case`, `camel_case`, `pascal_case`, `screaming_snake_case`
- `map_text` (type conversion)
- `comment` (formatting)

### Option 3: Post-Process Resolved JSON

Simple Node.js script:
```typescript
import resolved from '../resolved.json';

const attributes = resolved.groups
  .filter(g => g.type === 'attribute_group')
  .flatMap(g => g.attributes);

const output = `export const ATTRIBUTES = {
${attributes.map(a => `  ${toConstantName(a.id)}: '${a.id}'`).join(',\n')}
} as const;`;
```

### Option 4: Use @opentelemetry/semantic-conventions

For standard OTel attributes:
```typescript
import { ATTR_GEN_AI_REQUEST_MODEL } from '@opentelemetry/semantic-conventions/incubating';

span.setAttribute(ATTR_GEN_AI_REQUEST_MODEL, 'claude-3-opus');
```

For custom attributes, define constants matching Weaver schema.

---

## Why Schema-Only Works Best

For an AI instrumentation agent:

1. **AI can read JSON directly** - No need for code generation
2. **More flexible** - AI adapts to project patterns
3. **No template maintenance** - Weaver schema is enough
4. **Schema is source of truth** - Validation still works
5. **Weaver MCP server** - Potential for programmatic access

---

## Additional Weaver Features

### Weaver MCP Server
```bash
weaver registry mcp -r ./telemetry/registry
```
Exposes registry commands via Model Context Protocol. AI assistants can interact programmatically.

### Policy Enforcement
Custom Rego policies for validation:
```rego
deny[msg] {
  attr := input.groups[_].attributes[_]
  not startswith(attr.id, "my_app.")
  msg := sprintf("Attribute %s must start with 'my_app.'", [attr.id])
}
```

---

## Sources

- [OpenTelemetry Weaver GitHub](https://github.com/open-telemetry/weaver)
- [Weaver Usage Documentation](https://github.com/open-telemetry/weaver/blob/main/docs/usage.md)
- [OpenTelemetry Blog: Observability by Design](https://opentelemetry.io/blog/2025/otel-weaver/)
- [@opentelemetry/semantic-conventions npm](https://www.npmjs.com/package/@opentelemetry/semantic-conventions)
