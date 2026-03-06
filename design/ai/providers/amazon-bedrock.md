# amazon-bedrock.ts — Amazon Bedrock Provider

**Source:** `packages/ai/src/providers/amazon-bedrock.ts`

## Purpose

Amazon Bedrock Converse Stream API provider. Supports Claude, Qwen, Minimax, Moonshot. Handles adaptive/budget thinking, prompt caching, proxy, and HTTP/2.

## Exported Types

- **`BedrockOptions`** (line 48): Extends StreamOptions with `region`, `profile`, `toolChoice`, `reasoning`, `thinkingBudgets`, `interleavedThinking`

## `streamBedrock()` (line 62)

```mermaid
flowchart TD
    A["streamBedrock()"] --> B["Create BedrockRuntimeClient<br/>(region, profile, proxy)"]
    B --> C["convertMessages(context, model)"]
    C --> D["buildSystemPrompt(context, model)"]
    D --> E["convertToolConfig(tools, options)"]
    E --> F["buildAdditionalModelRequestFields(model, options)"]
    F --> G["ConverseStreamCommand(params)"]

    G --> H{Stream event}
    H -->|messageStart| I["Init output message"]
    H -->|contentBlockStart| J["handleContentBlockStart:<br/>create toolCall block"]
    H -->|contentBlockDelta| K["handleContentBlockDelta:<br/>text/thinking/toolCall delta"]
    H -->|contentBlockStop| L["handleContentBlockStop:<br/>emit _end events"]
    H -->|metadata| M["handleMetadata:<br/>extract usage metrics"]
    H -->|messageStop| N["Emit done"]
```

### Event Handlers

- **`handleContentBlockStart()`** (line 251): Creates `toolCall` block on `toolUse` start
- **`handleContentBlockDelta()`** (line 274): Updates text/toolCall/thinking blocks with delta content, creates text block if needed
- **`handleContentBlockStop()`** (line 347): Finalizes blocks, emits `_end` events, parses JSON for toolCalls
- **`handleMetadata()`** (line 332): Extracts usage metrics from metadata event

## `streamSimpleBedrock()` (line 207)

```mermaid
flowchart TD
    A["streamSimpleBedrock()"] --> B{Claude model?}
    B -->|"Opus/Sonnet 4.6"| C["Adaptive thinking<br/>effort mapping"]
    B -->|"Older Claude"| D["Budget-based thinking<br/>adjustMaxTokensForThinking"]
    B -->|"Other"| E["Pass reasoning as-is"]
```

- **`supportsAdaptiveThinking()`** (line 376): Checks for opus-4-6/sonnet-4-6 model IDs
- **`mapThinkingLevelToEffort()`** (line 385): Maps to `"low"/"medium"/"high"/"max"`

## Key Internal Functions

### AWS Configuration

Region from options or `AWS_REGION`/`AWS_DEFAULT_REGION`. Proxy via `HTTP_PROXY`/`HTTPS_PROXY` with `NodeHttpHandler`. Force HTTP/1.1 via `AWS_BEDROCK_FORCE_HTTP1=1`.

### Prompt Caching

- **`resolveCacheRetention()`** (line 408): Options or `PI_CACHE_RETENTION` env
- **`supportsPromptCaching()`** (line 422): Checks `cost.cacheRead/Write` or Claude 4.x/3.7-sonnet/3.5-haiku patterns
- Cache points added to system prompt and last user message

### Message Conversion

- **`convertMessages()`** (line 472): Converts to Bedrock format. Handles user text/images, assistant text/toolCalls/thinking with signatures, tool results. Adds cache points for supported models.
- **`supportsThinkingSignature()`** (line 443): Only `anthropic.claude` or `anthropic/claude` models
- **`normalizeToolCallId()`** (line 467): Alphanumeric + underscore/hyphen, max 64 chars

### Thinking Configuration

- **`buildAdditionalModelRequestFields()`** (line 667): For Claude models with reasoning:
  - Adaptive (Opus/Sonnet 4.6): `thinking.type: "adaptive"` with `output_config.effort`
  - Budget (older Claude): `thinking.type: "enabled"` with `budget_tokens`
  - Includes `anthropic_beta` for interleaved thinking

### Tool Conversion & Stop Reason

- **`mapStopReason()`** (line 652): Maps Bedrock stop reasons (END_TURN→"stop", MAX_TOKENS→"length", TOOL_USE→"toolUse")

---

## Message & Tool Conversion Details

### `convertMessages()` (lines 472–619)

```typescript
function convertMessages(
    context: Context, model: Model<"bedrock-converse-stream">,
    cacheRetention: CacheRetention
): Message[]
```

Converts internal `Message[]` to Bedrock `Message[]` format. First calls `transformMessages()` (line 478) for normalization.

```mermaid
flowchart TD
    A["convertMessages()"] --> PRE["transformMessages()<br/>(normalize IDs, thinking)"]
    PRE --> LOOP["For each message"]

    LOOP --> R{role?}

    R -->|user| U1{content type?}
    U1 -->|string| U2["{text: sanitized}"]
    U1 -->|array| U3["text → {text}<br/>image → {image: createImageBlock()}"]

    R -->|assistant| A1["Skip if content empty"]
    A1 --> A2{block type?}
    A2 -->|text| A3["Skip if empty<br/>{text: sanitized}"]
    A2 -->|toolCall| A4["{toolUse: {toolUseId,<br/>name, input: arguments}}"]
    A2 -->|thinking| A5{supportsThinkingSignature?}
    A5 -->|yes| A6["{reasoningContent:<br/>{reasoningText:<br/>{text, signature}}}"]
    A5 -->|no| A7["{reasoningContent:<br/>{reasoningText: {text}}}"]

    R -->|toolResult| T1["Collect current result"]
    T1 --> T2["Lookahead: merge ALL<br/>consecutive toolResults"]
    T2 --> T3["Push single USER message<br/>with all toolResult blocks"]

    LOOP --> CP{cacheRetention != 'none'<br/>AND supportsPromptCaching?}
    CP -->|yes| CP1["Add cachePoint to last<br/>USER message content"]
    CP -->|no| DONE[Return]
    CP1 --> DONE
```

**User messages (lines 484–501):** String → `{text: sanitized}`. Array → maps text/image, images converted via `createImageBlock()` (line 712).

**Assistant messages (lines 502–553):**
- Skips entirely if `content.length === 0` (aborted requests)
- **Text** → skip if empty, push `{text: sanitized}`
- **Tool calls** → `{toolUse: {toolUseId, name, input: arguments}}`
- **Thinking** (lines 527–544): If `supportsThinkingSignature(model)` (line 443 — checks for `anthropic.claude` or `anthropic/claude`) → includes `signature` in `reasoningText`. Otherwise → omits signature to avoid API errors with non-Anthropic models.

**Tool results (lines 555–598):**
- **Consecutive merging** (lines 574–589): Lookahead loop collects ALL consecutive toolResult messages into single USER message
- Each result: `{toolUseId, content: [...blocks], status: ERROR|SUCCESS}`
- Images converted via `createImageBlock()`

**Cache points (lines 605–616):**
- Added to last USER message if `cacheRetention !== "none"` AND `supportsPromptCaching(model)`
- `CachePointType.DEFAULT` with optional `CacheTTL.ONE_HOUR` for `cacheRetention === "long"`

### `buildSystemPrompt()` (lines 448–465)

```typescript
function buildSystemPrompt(
    systemPrompt: string | undefined, model, cacheRetention
): SystemContentBlock[] | undefined
```

```mermaid
flowchart LR
    A[systemPrompt] --> B["{text: sanitized}"]
    B --> C{cache enabled +<br/>model supports?}
    C -->|yes| D["Append cachePoint block<br/>type:DEFAULT, ttl?:ONE_HOUR"]
    C -->|no| E["Return [textBlock]"]
    D --> E
```

### `convertToolConfig()` (lines 621–650)

```typescript
function convertToolConfig(
    tools: Tool[] | undefined, toolChoice: BedrockOptions["toolChoice"]
): ToolConfiguration | undefined
```

```mermaid
flowchart TD
    A["convertToolConfig()"] --> B{tools exist AND<br/>toolChoice != 'none'?}
    B -->|no| C[Return undefined]
    B -->|yes| D["Map each tool →<br/>{toolSpec: {name, description,<br/>inputSchema: {json: parameters}}}"]
    D --> E{toolChoice?}
    E -->|"auto"| F["{auto: {}}"]
    E -->|"any"| G["{any: {}}"]
    E -->|"{type:'tool', name}"| H["{tool: {name}}"]
    E -->|undefined| I["No toolChoice"]
    F --> J["Return {tools, toolChoice}"]
    G --> J
    H --> J
    I --> J
```

- **Lines 627–633**: Maps `Tool[]` to Bedrock `BedrockTool[]` — wraps parameters in `{json: tool.parameters}` (unlike other providers that pass schema directly)
- **Lines 635–647**: Converts `toolChoice` to Bedrock-specific format (`{auto: {}}`, `{any: {}}`, or `{tool: {name}}`)

### `buildAdditionalModelRequestFields()` (lines 667–710)

Configures thinking for Anthropic Claude models on Bedrock.

```mermaid
flowchart TD
    A["buildAdditionalModelRequestFields()"] --> B{reasoning enabled<br/>AND model.reasoning?}
    B -->|no| C[Return undefined]
    B -->|yes| D{Anthropic Claude model?}
    D -->|no| E[Return undefined]
    D -->|yes| F{supportsAdaptiveThinking?}
    F -->|"Opus/Sonnet 4.6"| G["{thinking:{type:'adaptive'},<br/>output_config:{effort}}"]
    F -->|"Older Claude"| H["{thinking:{type:'enabled',<br/>budget_tokens}}"]
    H --> I{interleavedThinking?}
    I -->|"yes (default)"| J["Add anthropic_beta:<br/>['interleaved-thinking-2025-05-14']"]
    I -->|"no"| K["No beta flag"]
```

- **Adaptive** (Opus/Sonnet 4.6): `effort` mapped from reasoning level (xhigh→"max" for Opus, xhigh→"high" for Sonnet)
- **Budget-based** (older Claude): Default budgets minimal=1024, low=2048, medium=8192, high=16384. Custom budgets from `options.thinkingBudgets` override defaults.
