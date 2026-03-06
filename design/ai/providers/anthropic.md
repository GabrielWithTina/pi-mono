# anthropic.ts — Anthropic Claude Provider

**Source:** `packages/ai/src/providers/anthropic.ts`

## Purpose

Streaming provider for Anthropic's Claude API (`anthropic-messages`). Handles OAuth/API key/Copilot auth, adaptive thinking (Opus/Sonnet 4.6), budget-based thinking (older models), and fine-grained tool streaming.

## Constants

| Name | Line | Value |
|---|---|---|
| `claudeCodeVersion` | 65 | `"2.1.2"` |
| `claudeCodeTools` | 70 | Array of canonical Claude Code tool names |
| `ccToolLookup` | 90 | Map for case-insensitive tool name lookup |

## Exported Types

- **`AnthropicEffort`** (line 155): `"low" | "medium" | "high" | "max"`
- **`AnthropicOptions`** (line 157): Extends StreamOptions with `thinkingEnabled`, `thinkingBudgetTokens`, `effort`, `interleavedThinking`, `toolChoice`

## `streamAnthropic()` (line 193)

```typescript
export function streamAnthropic(model, context, options?): AssistantMessageEventStream
```

```mermaid
flowchart TD
    A["streamAnthropic()"] --> B["createClient(apiKey, model)"]
    B --> C["convertMessages(context, model)"]
    C --> D["convertTools(context.tools)"]
    D --> E["buildParams(model, msgs, tools, options)"]
    E --> F["client.messages.stream(params)"]

    F --> G{SSE Event}
    G -->|message_start| H["Extract usage from response"]
    G -->|content_block_start| I["Create text/thinking/toolCall block"]
    G -->|content_block_delta| J["Append delta to block"]
    G -->|content_block_stop| K["Emit _end event"]
    G -->|message_delta| L["Update usage + stopReason"]
    G -->|message_stop| M["Emit done event"]
    G -->|error| N["Emit error event"]
```

Processes Anthropic SSE events into unified `AssistantMessageEvent` stream. Handles `input_json` tool call deltas, thinking block signatures, and cache usage metrics.

## `streamSimpleAnthropic()` (line 447)

Wraps `streamAnthropic` with reasoning level mapping:
- Calls `supportsAdaptiveThinking()` (line 416) — checks for Opus/Sonnet 4.6 model IDs
- Adaptive models: maps reasoning to `AnthropicEffort` via `mapThinkingLevelToEffort()` (line 430)
- Older models: uses `adjustMaxTokensForThinking()` for budget-based thinking

## Key Internal Functions

### Authentication

- **`isOAuthToken()`** (line 488): Checks if key contains `"sk-ant-oat"` prefix
- **`createClient()`** (line 492): Creates Anthropic SDK client with auth detection:

```mermaid
flowchart TD
    A["createClient()"] --> B{Key type?}
    B -->|"sk-ant-oat"| C["OAuth Bearer + Claude Code headers"]
    B -->|copilotToken| D["Bearer + Copilot dynamic headers"]
    B -->|standard| E["API Key auth"]
    C --> F["new Anthropic({...})"]
    D --> F
    E --> F
```

### Message Conversion

- **`convertMessages()`** (line 662): Converts `Message[]` to Anthropic `MessageParam[]`. Handles system prompt caching, merges consecutive tool results into user messages (Anthropic requirement), preserves thinking signatures for same model.
- **`convertContentBlocks()`** (line 106): Converts TextContent/ImageContent to Anthropic format with base64 image encoding.
- **`convertTools()`** (line 819): Converts Tool[] to Anthropic tool format with JSON schema.

### Request Building

- **`buildParams()`** (line 573): Constructs `MessageCreateParamsStreaming`. Configures thinking (adaptive with effort or enabled with budget), adds cache control to last user message with optional TTL for long retention.
- **`normalizeToolCallId()`** (line 658): Sanitizes to 64 chars max, alphanumeric + underscore/hyphen only.
- **`mapStopReason()`** (line 837): Maps Anthropic stop reasons to normalized `StopReason`.

### Cache Control

- **`resolveCacheRetention()`** (line 39): Resolves from options or `PI_CACHE_RETENTION` env var, defaults to "short"
- **`getCacheControl()`** (line 49): Returns ephemeral cache with optional 1h TTL for api.anthropic.com

---

## Message & Tool Conversion Details

### `convertContentBlocks()` (lines 106–153)

```typescript
function convertContentBlocks(content: (TextContent | ImageContent)[]): string | ContentBlockArray
```

```mermaid
flowchart TD
    A["convertContentBlocks()"] --> B{Any images?}
    B -->|no| C["Concatenate text<br/>(sanitizeSurrogates)<br/>→ return string"]
    B -->|yes| D["Map to block array"]
    D --> E["TextContent → {type:'text', text}"]
    D --> F["ImageContent → {type:'image',<br/>source:{type:'base64', media_type, data}}"]
    D --> G{Only images, no text?}
    G -->|yes| H["Prepend placeholder:<br/>'(see attached image)'"]
    G -->|no| I["Return block array"]
    H --> I
```

- **Line 119–123**: No images → return concatenated text as plain string (simpler format)
- **Line 126–141**: Images present → return array of text/image blocks
- **Line 144–150**: If only images and no text, prepends placeholder text `"(see attached image)"` to avoid API rejection
- All text passed through `sanitizeSurrogates()` for Unicode safety

### `convertMessages()` (lines 662–817)

```typescript
function convertMessages(
    messages: Message[], model: Model<"anthropic-messages">,
    isOAuthToken: boolean, cacheControl?: { type: "ephemeral"; ttl?: "1h" }
): MessageParam[]
```

Converts internal `Message[]` to Anthropic `MessageParam[]` format. First calls `transformMessages()` (line 671) for cross-provider normalization with `normalizeToolCallId`.

```mermaid
flowchart TD
    A["convertMessages()"] --> PRE["transformMessages()<br/>(normalize IDs, handle thinking)"]
    PRE --> LOOP["For each message"]

    LOOP --> U{role?}
    U -->|user| U1{content type?}
    U1 -->|string| U2["Push {role:'user', content: sanitized}"]
    U1 -->|array| U3["Map to content blocks<br/>filter images if unsupported<br/>filter empty text"]

    U -->|assistant| A1["For each content block"]
    A1 --> A2{block type?}
    A2 -->|text| A3["Skip if empty<br/>Push {type:'text', text}"]
    A2 -->|thinking| A4{thinkingSignature?}
    A4 -->|missing| A5["Convert to plain text block<br/>(prevent API rejection)"]
    A4 -->|present| A6["Push thinking block<br/>with signature"]
    A2 -->|toolCall| A7["toClaudeCodeName() if OAuth<br/>Push {type:'tool_use', id, name, input}"]

    U -->|toolResult| T1["Collect current result"]
    T1 --> T2["Lookahead: merge ALL<br/>consecutive toolResults"]
    T2 --> T3["Push single user message<br/>with all tool_result blocks"]

    LOOP --> CC{cacheControl provided?}
    CC -->|yes| CC1["Add cache_control to last block<br/>of last user message"]
    CC -->|no| DONE[Return params]
    CC1 --> DONE
```

Key behaviors by role:

**User messages (lines 676–714):**
- String content → simple user message with sanitized text
- Array content → content blocks with image filtering (line 702: skip images if model doesn't support)
- Empty blocks skipped entirely

**Assistant messages (lines 715–755):**
- **Thinking blocks** (lines 725–741): If `thinkingSignature` is missing → converts to plain text (line 732–733, prevents API rejection). If present → keeps as thinking block with both text and signature
- **Tool calls** (lines 742–748): Name converted via `toClaudeCodeName()` for OAuth tokens (line 746). Output as `tool_use` block with `id`, `name`, `input`
- Skips message if no blocks remain (line 751)

**Tool results (lines 756–788):**
- **Consecutive merging**: Lookahead loop (lines 769–779) collects ALL consecutive `toolResult` messages into a single user message — Anthropic requires tool results as user messages
- Each result converted via `convertContentBlocks()` (line 763)

**Cache control (lines 792–813):**
- Added to last block of last user message
- If last user message is a string, converts to content block array first (lines 804–811)

### `convertTools()` (lines 819–835)

```typescript
function convertTools(tools: Tool[], isOAuthToken: boolean): Anthropic.Messages.Tool[]
```

```mermaid
flowchart TD
    A["convertTools()"] --> B{tools exists?}
    B -->|no| C[Return early]
    B -->|yes| D["Map each tool"]
    D --> E["Extract JSON schema from<br/>tool.parameters (TypeBox)"]
    E --> F{isOAuthToken?}
    F -->|yes| G["name = toClaudeCodeName(tool.name)"]
    F -->|no| H["name = tool.name"]
    G --> I["Build input_schema:<br/>{type:'object', properties, required}"]
    H --> I
```

- **Line 823**: TypeBox parameters are already JSON Schema — extracts `properties` and `required` directly
- **Line 826**: OAuth tokens get tool names converted to Claude Code canonical casing
- **Lines 828–832**: Wraps in `{type: "object", properties: ..., required: ...}` with safe fallbacks (`|| {}`, `|| []`)
