# Optional Telemetry Patterns

**Research Date:** 2026-02-05

## Executive Summary

OpenTelemetry was designed from the ground up to support optional instrumentation. The key is the separation between API and SDK: libraries depend only on `@opentelemetry/api` (lightweight, zero-dependency), while the SDK is the consumer's responsibility.

---

## The Core Pattern: API vs SDK Separation

**Package sizes:**
| Package | Dependencies | Size |
|---------|-------------|------|
| `@opentelemetry/api` | 0 | ~8-10 KB |
| `@opentelemetry/sdk-node` | 24 packages | ~25+ MB |
| `@opentelemetry/auto-instrumentations-node` | ~50+ packages | ~15+ MB |

**Key insight:** The API package has zero dependencies and provides no-op implementations by default.

---

## 1. Peer Dependencies Pattern

### Standard Approach
```json
{
  "peerDependencies": {
    "@opentelemetry/api": "^1.3.0"
  }
}
```

### Making It Optional
```json
{
  "peerDependencies": {
    "@opentelemetry/api": "^1.3.0"
  },
  "peerDependenciesMeta": {
    "@opentelemetry/api": {
      "optional": true
    }
  }
}
```

This tells npm 7+ to NOT auto-install the dependency.

**Real-world examples:**
- `@opentelemetry/instrumentation-express`
- `@prisma/instrumentation`
- Azure SDK packages

---

## 2. Built-in No-Op Behavior

OpenTelemetry's design principle:

> "Unless your application imports the OpenTelemetry SDK, your instrumentation does nothing and does not impact application performance."

**How it works:**
- If you never call `setGlobalTracerProvider()`, `trace.getTracer()` returns a `ProxyTracer` that redirects to `NoopTracer`
- All API methods are near-zero overhead when no SDK is configured
- `OTEL_SDK_DISABLED=true` environment variable forces no-op mode

```typescript
import { trace } from '@opentelemetry/api';

// Without SDK initialization, this returns a no-op tracer
const tracer = trace.getTracer('my-library');

// These calls do nothing - no overhead
const span = tracer.startSpan('operation');
span.end();
```

---

## 3. Runtime Detection Pattern

### Try-Catch for Optional Module Loading
```typescript
// Default to no-op implementation
let createSpanFunction = createSpanNoop;

try {
  require("@opentelemetry/api");
  createSpanFunction = require("./createSpan").createSpanFunction;
} catch {
  // OpenTelemetry not installed - use no-op
}

export { createSpanFunction };
```

### ESM (Dynamic Imports)
```typescript
let otel: typeof import('@opentelemetry/api') | null = null;

async function initTelemetry() {
  try {
    otel = await import('@opentelemetry/api');
  } catch {
    // Not installed - otel remains null
  }
}

function startSpan(name: string) {
  if (otel) {
    return otel.trace.getTracer('my-lib').startSpan(name);
  }
  return { end: () => {}, setAttribute: () => {} };
}
```

---

## 4. Facade/Abstraction Pattern

Create your own abstraction layer:

```typescript
// telemetry.ts
interface Tracer {
  startSpan(name: string): Span;
}

interface Span {
  end(): void;
  setAttribute(key: string, value: string): void;
}

const noopSpan: Span = {
  end: () => {},
  setAttribute: () => {},
};

const noopTracer: Tracer = {
  startSpan: () => noopSpan,
};

let realTracer: Tracer = noopTracer;

try {
  const api = require('@opentelemetry/api');
  const otelTracer = api.trace.getTracer('my-library');
  realTracer = {
    startSpan: (name) => {
      const span = otelTracer.startSpan(name);
      return {
        end: () => span.end(),
        setAttribute: (k, v) => span.setAttribute(k, v),
      };
    },
  };
} catch {}

export const tracer = realTracer;
```

**Benefits:**
- Library code uses stable internal API
- Swappable implementations
- Zero coupling to specific vendor

---

## 5. Build-Time Stripping

### Using Define/Replace
```typescript
if (process.env.ENABLE_TELEMETRY === 'true') {
  import('@opentelemetry/api').then(otel => {
    // Instrumentation code
  });
}
```

**esbuild:**
```bash
esbuild --define:process.env.ENABLE_TELEMETRY=\"false\" --bundle src/index.ts
```

When minified, dead code branch is eliminated.

**Limitations:**
- Cross-module elimination can be imperfect
- Keep telemetry code in isolated modules
- Use dynamic imports so telemetry is never statically referenced

---

## Recommended Implementation

For the telemetry agent project:

### package.json
```json
{
  "peerDependencies": {
    "@opentelemetry/api": "^1.3.0"
  },
  "peerDependenciesMeta": {
    "@opentelemetry/api": {
      "optional": true
    }
  }
}
```

### Telemetry abstraction module
```typescript
// src/telemetry/index.ts
let api: typeof import('@opentelemetry/api') | null = null;

try {
  api = await import('@opentelemetry/api');
} catch {}

// Minimal no-op tracer for when OTel API is unavailable
const noopSpan = { end() {}, setAttribute() { return this; } };
const noopTracer = { startSpan() { return noopSpan; } };

export function getTracer(name: string) {
  if (api) {
    return api.trace.getTracer(name);
  }
  return noopTracer;
}
```

### Consumers who want telemetry
```bash
npm install @opentelemetry/api @opentelemetry/sdk-node
```

### Consumers who don't
Do nothing - code uses no-ops automatically.

---

## Anti-Patterns to Avoid

1. **Don't bundle the SDK** - Let consumers bring their own
2. **Don't use `@opentelemetry/auto-instrumentations-node`** - It's massive
3. **Don't have multiple copies of `@opentelemetry/api`** - Use peer dependencies
4. **Don't depend on SDK internals** - Only use public APIs

---

## Sources

- [OpenTelemetry Instrumentation for Libraries](https://opentelemetry.io/docs/concepts/instrumentation/libraries/)
- [OpenTelemetry Library Guidelines](https://opentelemetry.io/docs/specs/otel/library-guidelines/)
- [Azure SDK RFC: Make @opentelemetry/api opt-in](https://github.com/Azure/azure-sdk-for-js/issues/17068)
- [Bundlephobia - @opentelemetry/api](https://bundlephobia.com/package/@opentelemetry/api)
