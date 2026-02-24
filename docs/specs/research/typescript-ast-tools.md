# TypeScript AST Transformation Tools

**Research Date:** 2026-02-05

## Executive Summary

For building an agent that modifies TypeScript source code to add OpenTelemetry instrumentation, **ts-morph** is the recommended tool due to its TypeScript-native design, comprehensive API, active maintenance, and strong community adoption.

---

## Tool Comparison

| Tool | Ease of Use | Type Preservation | Format Preservation | TS Types Access | Instrumentation Fit |
|------|-------------|-------------------|---------------------|-----------------|---------------------|
| **ts-morph** | High | Excellent | Medium | Full | **Best** |
| TS Compiler API | Low | Perfect | Poor | Full | Medium-High |
| jscodeshift | Medium | Limited | Good | None | Medium |
| recast | Medium | Limited | Excellent | None | Medium |
| Babel | Medium | Poor | Basic | None | Low-Medium |
| magicast | Very High | Limited | Good | None | Low-Medium |

---

## ts-morph (Recommended)

**Overview:** High-level wrapper around TypeScript Compiler API that simplifies AST navigation and manipulation.

### Key Features
- Full TypeScript type system integration
- In-memory changes until explicit save
- Wrapped nodes maintain references between manipulations
- Built-in fixes like `organizeImports()` and `fixMissingImports()`

### API Examples
```typescript
// Adding imports
sourceFile.addImportDeclaration({
  namedImports: ["trace", "SpanStatusCode"],
  moduleSpecifier: "@opentelemetry/api",
});

// Inserting statements
functionDeclaration.insertStatements(0,
  "const span = tracer.startSpan('operation');"
);

// Working with async functions
const isAsync = functionDeclaration.isAsync();
```

### Stats
- ~6 million weekly npm downloads
- ~5,700 GitHub stars
- Actively maintained by David Sherret

**Source:** [ts-morph.com](https://ts-morph.com/)

---

## TypeScript Compiler API

**Overview:** Official low-level API from Microsoft for programmatic AST operations.

### Key Features
- Full program context (understands entire projects)
- Access to binder and type checker
- Factory functions (`ts.factory`) for creating nodes
- Transformation phases: before, after, afterDeclarations

### Pattern
```typescript
function transformer(context) {
  return (sourceFile) => {
    function visit(node) {
      if (ts.isFunctionDeclaration(node)) {
        return ts.factory.updateFunctionDeclaration(...);
      }
      return ts.visitEachChild(node, visit, context);
    }
    return ts.visitNode(sourceFile, visit);
  };
}
```

### Tradeoffs
- Very powerful but verbose
- Nodes are immutable; mutations require factory functions
- No built-in formatting preservation

**Source:** [TypeScript Compiler API Wiki](https://github.com/microsoft/TypeScript/wiki/Using-the-Compiler-API)

---

## jscodeshift

**Overview:** Facebook's codemod toolkit built on recast for large-scale refactoring.

### Key Features
- Collection-based API for batch operations
- Built-in runner for multiple files
- Uses recast for formatting preservation

### API Example
```javascript
export default function transformer(file, api) {
  const j = api.jscodeshift;

  return j(file.source)
    .find(j.CallExpression, { callee: { name: 'oldFunction' }})
    .replaceWith(path =>
      j.callExpression(j.identifier('newFunction'), path.node.arguments)
    )
    .toSource();
}
```

### Limitations
- Limited TypeScript type information
- Decorator handling issues with TC39 syntax

**Source:** [github.com/facebook/jscodeshift](https://github.com/facebook/jscodeshift)

---

## recast

**Overview:** JavaScript transformer that preserves original formatting through shadow copy tracking.

### Core Principle
`recast.print(recast.parse(source)).code === source`

### How It Works
1. `parse()` creates shadow copy of AST
2. Each node has `.original` property
3. `print()` compares current vs original
4. Unmodified nodes use original source text
5. Modified nodes fall back to automatic formatting

### Best for
- When preserving exact original style is critical
- Working with Babel transforms

**Source:** [github.com/benjamn/recast](https://github.com/benjamn/recast)

---

## magicast

**Overview:** Modern library providing JSON-like interface for code modification.

### Philosophy
"Modify files like JSON" - abstracts away AST complexity.

### API Example
```typescript
import { parseModule, generateCode } from 'magicast';

const mod = parseModule(code);
mod.exports.default.plugins.push('newPlugin');
const { code: output } = generateCode(mod);
```

### Limitations
- Designed for "static-ish" code
- "CAN NOT cover all possible cases"
- Best for configuration files, not complex code

**Source:** [github.com/unjs/magicast](https://github.com/unjs/magicast)

---

## Common Pitfalls

### 1. Decorator Handling
- TC39 proposal changed decorator placement
- jscodeshift's `find(j.Decorator)` may miss some decorators
- **Solution:** Use ts-morph for decorator-heavy code

### 2. Generics
- Must preserve type parameters during transformation
- Easy to lose type arguments when creating new nodes

### 3. Async/Await
- ts-morph has `isAsync()` method and `AsyncableNode` interface
- Don't add synchronous wrappers around async functions

### 4. Comment Preservation
- AST typically discards comments
- recast preserves but may move them
- Consider post-processing for critical comments

### 5. Import Deduplication
- Check for existing imports before adding
- Use `fixMissingImports()` in ts-morph

### 6. Scope and Variable Shadowing
- Injecting variables that shadow existing ones causes bugs
- Use TypeScript's binder to check scope

---

## Instrumentation Pattern

```typescript
// Before
async function processOrder(order: Order) {
  // business logic
}

// After
async function processOrder(order: Order) {
  return tracer.startActiveSpan('processOrder', async (span) => {
    try {
      // business logic
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

### Key Operations for Agent
1. Add imports (`@opentelemetry/api`)
2. Wrap function bodies with span creation
3. Add attributes from Weaver schema
4. Handle errors with try/catch/finally

---

## Recommendation

**Primary:** ts-morph
- TypeScript-native with full type access
- Comprehensive API for import management and statement insertion
- Active maintenance, production-proven

**Combine with:** Prettier
- ts-morph doesn't preserve original formatting
- Run Prettier after transformation for consistent style

---

## Sources

- [ts-morph Documentation](https://ts-morph.com/)
- [TypeScript Compiler API Wiki](https://github.com/microsoft/TypeScript/wiki/Using-the-Compiler-API)
- [jscodeshift GitHub](https://github.com/facebook/jscodeshift)
- [recast GitHub](https://github.com/benjamn/recast)
- [magicast GitHub](https://github.com/unjs/magicast)
- [OpenTelemetry Auto-Instrumentation Blog](https://opentelemetry.io/blog/2025/demystifying-auto-instrumentation/)
