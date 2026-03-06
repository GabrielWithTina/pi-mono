# typebox-helpers.ts — TypeBox Utilities

**Source:** `packages/ai/src/utils/typebox-helpers.ts`

## Purpose

TypeBox helper for creating string enum schemas compatible with providers that don't support `anyOf`/`const` patterns (e.g., Google's API).

## `StringEnum()` (lines 14–24)

```typescript
export function StringEnum<T extends readonly string[]>(
    values: T,
    options?: { description?: string; default?: T[number] }
): TUnsafe<T[number]>
```

Produces `{ type: "string", enum: [...] }` instead of TypeBox's default `anyOf`/`const` output:

```typescript
// Standard TypeBox: Type.Union(Type.Literal("a"), Type.Literal("b"))
// → { anyOf: [{ const: "a" }, { const: "b" }] }  ❌ breaks some providers

// StringEnum(["a", "b"])
// → { type: "string", enum: ["a", "b"] }  ✅ universally supported
```

Uses `Type.Unsafe<T[number]>()` to bypass TypeBox's schema generation and emit raw JSON schema. Optional `description` and `default` fields spread into the schema.
