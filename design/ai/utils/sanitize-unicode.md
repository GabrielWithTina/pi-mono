# sanitize-unicode.ts — Unicode Surrogate Sanitization

**Source:** `packages/ai/src/utils/sanitize-unicode.ts`

## Purpose

Removes unpaired Unicode surrogate characters that cause JSON serialization errors in API providers. Preserves valid emoji and properly paired surrogates.

## `sanitizeSurrogates()` (lines 21–25)

```typescript
export function sanitizeSurrogates(text: string): string
```

Single regex replacement (line 24):
```typescript
text.replace(/[\uD800-\uDBFF](?![\uDC00-\uDFFF])|(?<![\uD800-\uDBFF])[\uDC00-\uDFFF]/g, "")
```

- First alternation: High surrogate (U+D800–U+DBFF) NOT followed by low surrogate → remove
- Second alternation: Low surrogate (U+DC00–U+DFFF) NOT preceded by high surrogate → remove
- Uses negative lookahead and lookbehind for context-aware matching
- No-op for ASCII and properly paired Unicode (valid emoji preserved)
