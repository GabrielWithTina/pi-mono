# proxy.ts — LLM Proxy Stream Client

**Source:** `packages/agent/src/proxy.ts`

## Purpose

Proxy stream function for apps that route LLM calls through a server. The server manages auth and proxies requests to LLM providers. Strips `partial` field from events to reduce bandwidth — client reconstructs it locally.

## Internal Classes

### `ProxyMessageEventStream` (line 20)

```typescript
class ProxyMessageEventStream extends EventStream<AssistantMessageEvent, AssistantMessage>
```

Extends `EventStream`. Terminates on `done` or `error` events. Extracts `AssistantMessage` from terminal events.

## Exported Types

### `ProxyAssistantMessageEvent` (line 36)

```typescript
export type ProxyAssistantMessageEvent =
    | { type: "start" }
    | { type: "text_start"; contentIndex: number }
    | { type: "text_delta"; contentIndex: number; delta: string }
    | { type: "text_end"; contentIndex: number; contentSignature?: string }
    | { type: "thinking_start"; contentIndex: number }
    | { type: "thinking_delta"; contentIndex: number; delta: string }
    | { type: "thinking_end"; contentIndex: number; contentSignature?: string }
    | { type: "toolcall_start"; contentIndex: number; id: string; toolName: string }
    | { type: "toolcall_delta"; contentIndex: number; delta: string }
    | { type: "toolcall_end"; contentIndex: number }
    | { type: "done"; reason: StopReason; usage: AssistantMessage["usage"] }
    | { type: "error"; reason: StopReason; errorMessage?: string; usage: AssistantMessage["usage"] }
```

Bandwidth-optimized event format. Omits the `partial` field present in standard `AssistantMessageEvent` — the client reconstructs partial messages locally using `contentIndex`.

### `ProxyStreamOptions` (line 59)

```typescript
export interface ProxyStreamOptions extends SimpleStreamOptions {
    authToken: string;
    proxyUrl: string;
}
```

Extends `SimpleStreamOptions` with proxy-specific auth token and server URL.

## `streamProxy()` (line 85)

```typescript
export function streamProxy(
    model: Model<any>, context: Context, options: ProxyStreamOptions
): ProxyMessageEventStream
```

Stream function that proxies LLM calls through a server. Drop-in replacement for `streamSimple` via the `streamFn` option on `Agent`.

```mermaid
flowchart TD
    A["streamProxy()"] --> B["Initialize empty partial<br/>AssistantMessage (lines 90-106)"]
    B --> C["Register abort handler<br/>(lines 110-118)"]
    C --> D["POST to {proxyUrl}/api/stream<br/>(lines 121-137)"]

    D --> E{response.ok?}
    E -->|no| F["Parse error JSON<br/>Throw with message"]
    E -->|yes| G["Read response body<br/>as stream (line 152)"]

    G --> H["Decode chunks<br/>Buffer partial lines<br/>(lines 156-166)"]
    H --> I{SSE line?}
    I -->|"data: ..."| J["Parse JSON →<br/>ProxyAssistantMessageEvent<br/>(line 172)"]
    J --> K["processProxyEvent()<br/>(line 173)"]
    K --> L["stream.push(event)"]
    L --> H

    H -->|stream done| M["stream.end()"]

    F --> N["Error handler:<br/>Set stopReason,<br/>Emit error event<br/>(lines 187-197)"]
```

**Request body (lines 127–136):** Sends `model`, `context`, and selected `options` (temperature, maxTokens, reasoning) as JSON.

**SSE parsing (lines 168–179):** Reads response body as stream, splits on newlines, extracts `data:` prefixed lines, parses as JSON `ProxyAssistantMessageEvent`.

**Abort handling (lines 110–118):** Registers abort listener that cancels the reader. Cleanup in `finally` block (lines 198–202).

## `processProxyEvent()` (line 211)

```typescript
function processProxyEvent(
    proxyEvent: ProxyAssistantMessageEvent,
    partial: AssistantMessage,
): AssistantMessageEvent | undefined
```

Reconstructs full `AssistantMessageEvent` objects from bandwidth-optimized proxy events by maintaining and mutating the `partial` message.

```mermaid
flowchart TD
    A["processProxyEvent()"] --> B{event type?}

    B -->|start| C["Return {type:'start', partial}"]

    B -->|text_start| D["Create {type:'text', text:''}<br/>at contentIndex (line 220)"]
    B -->|text_delta| E["Append delta to text<br/>(line 225)"]
    B -->|text_end| F["Set textSignature<br/>(line 240)"]

    B -->|thinking_start| G["Create {type:'thinking', thinking:''}<br/>at contentIndex (line 252)"]
    B -->|thinking_delta| H["Append delta to thinking<br/>(line 258)"]
    B -->|thinking_end| I["Set thinkingSignature<br/>(line 272)"]

    B -->|toolcall_start| J["Create {type:'toolCall', id, name,<br/>arguments:{}, partialJson:''}<br/>at contentIndex (lines 284-290)"]
    B -->|toolcall_delta| K["Append delta to partialJson<br/>parseStreamingJson() → arguments<br/>Spread for reactivity (lines 296-298)"]
    B -->|toolcall_end| L["Delete partialJson<br/>(line 312)"]

    B -->|done| M["Set stopReason + usage<br/>Return done event (lines 324-326)"]
    B -->|error| N["Set stopReason + errorMessage + usage<br/>Return error event (lines 329-332)"]
```

**Text events (lines 219–249):** Create text block at `contentIndex`, accumulate deltas, attach signature on end.

**Thinking events (lines 251–281):** Same pattern as text but for `thinking` blocks with `thinkingSignature`.

**Tool call events (lines 283–321):**
- `toolcall_start` (line 284): Creates toolCall with `partialJson` tracking field
- `toolcall_delta` (line 296): Appends to `partialJson`, calls `parseStreamingJson()` to parse partial JSON into `arguments`. Spreads content for reactivity.
- `toolcall_end` (line 312): Deletes temporary `partialJson` field

**Exhaustive check (lines 334–337):** Uses TypeScript `never` type to ensure all event types are handled.
