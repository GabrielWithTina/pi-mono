# github-copilot-headers.ts — Copilot Dynamic Headers

**Source:** `packages/ai/src/providers/github-copilot-headers.ts`

## Purpose

Generates dynamic HTTP headers required for GitHub Copilot API requests.

## `inferCopilotInitiator()` (line 5)

```typescript
export function inferCopilotInitiator(messages: Message[]): "user" | "agent"
```

Returns `"user"` if last message has role `"user"`, otherwise `"agent"`.

## `hasCopilotVisionInput()` (line 11)

```typescript
export function hasCopilotVisionInput(messages: Message[]): boolean
```

Scans `user` and `toolResult` messages for image content blocks. Returns true if any found.

## `buildCopilotDynamicHeaders()` (line 23)

```typescript
export function buildCopilotDynamicHeaders(params: {
    messages: Message[], model: Model<Api>
}): Record<string, string>
```

Returns headers object:
- `X-Initiator`: result of `inferCopilotInitiator()`
- `Openai-Intent`: `"conversation-panel"`
- `Copilot-Vision-Request`: `"true"` (only if `hasCopilotVisionInput()` returns true)
