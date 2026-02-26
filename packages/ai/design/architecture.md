# `packages/ai` — Architecture & Design

## Overview

A **unified, multi-provider LLM streaming API** providing a single abstraction over 9 API backends and 20+ providers, with tool calling, reasoning/thinking, image input, and cost tracking.

---

## Directory Layout

```
packages/ai/src/
├── index.ts                   # Public API exports
├── types.ts                   # Core types (Message, Tool, Model, Context, etc.)
├── stream.ts                  # Entry points: stream(), complete(), streamSimple()
├── models.ts                  # Model registry (getModel, getModels, calculateCost)
├── api-registry.ts            # Provider registration system
├── env-api-keys.ts            # Environment variable key resolution (20+ providers)
├── models.generated.ts        # Auto-generated model definitions (~3000 lines)
├── providers/                 # 15 files — one per API backend
│   ├── register-builtins.ts   # Registers all built-in providers
│   ├── anthropic.ts           # Anthropic Messages API
│   ├── openai-completions.ts  # OpenAI Chat Completions (+ compatible providers)
│   ├── openai-responses.ts    # OpenAI Responses API
│   ├── google.ts / google-shared.ts / google-vertex.ts / google-gemini-cli.ts
│   ├── amazon-bedrock.ts      # AWS Bedrock
│   ├── transform-messages.ts  # Cross-provider message transformation
│   └── simple-options.ts      # Unified reasoning level mapping
└── utils/
    ├── event-stream.ts        # Generic async event stream (queue + iterator)
    ├── validation.ts          # AJV-based tool argument validation
    ├── overflow.ts            # Context overflow detection (15+ error patterns)
    ├── json-parse.ts          # Partial JSON parser for streaming tool args
    ├── sanitize-unicode.ts    # Surrogate pair removal
    └── oauth/                 # OAuth flows for 5+ providers
```

---

## Core Type Hierarchy

```
Api (9 known)  ──►  Provider (20+ known)  ──►  Model<TApi>
                                                   ├── id, name, api, provider, baseUrl
                                                   ├── reasoning, input, cost
                                                   ├── contextWindow, maxTokens
                                                   └── compat? (OpenAI-compat overrides)

Context = { systemPrompt?, messages: Message[], tools?: Tool[] }

Message = UserMessage | AssistantMessage | ToolResultMessage
  └── AssistantMessage.content = (TextContent | ThinkingContent | ToolCall)[]

AssistantMessageEvent = start | text_delta | thinking_delta | toolcall_delta | done | error
```

### Core Types (from `types.ts`)

```typescript
// API types — identifies which LLM API to use
type Api = KnownApi | (string & {})
type KnownApi =
  | "anthropic-messages"
  | "openai-completions"
  | "openai-responses"
  | "azure-openai-responses"
  | "openai-codex-responses"
  | "google-generative-ai"
  | "google-gemini-cli"
  | "google-vertex"
  | "bedrock-converse-stream"

// Provider types — identifies the service provider
type Provider = KnownProvider | (string & {})
type KnownProvider =
  | "openai" | "azure-openai-responses" | "anthropic"
  | "google" | "google-vertex" | "google-gemini-cli" | "google-antigravity"
  | "amazon-bedrock" | "github-copilot" | "openai-codex"
  | "xai" | "groq" | "cerebras" | "openrouter" | "mistral" | "minimax" | ...

// Thinking/Reasoning
type ThinkingLevel = "minimal" | "low" | "medium" | "high" | "xhigh"

// Streaming options common to all providers
interface StreamOptions {
  temperature?: number
  maxTokens?: number
  signal?: AbortSignal
  apiKey?: string
  transport?: Transport
  cacheRetention?: CacheRetention
  sessionId?: string
  onPayload?: (payload: unknown) => void
  headers?: Record<string, string>
  maxRetryDelayMs?: number
  metadata?: Record<string, unknown>
}

// Content types
interface TextContent     { type: "text"; text: string; textSignature?: string }
interface ThinkingContent { type: "thinking"; thinking: string; thinkingSignature?: string }
interface ImageContent    { type: "image"; data: string; mimeType: string }
interface ToolCall        { type: "toolCall"; id: string; name: string; arguments: Record<string, any> }

// Message types
interface UserMessage       { role: "user"; content: string | (TextContent | ImageContent)[]; timestamp: number }
interface AssistantMessage  { role: "assistant"; content: (TextContent | ThinkingContent | ToolCall)[]; api; provider; model; usage; stopReason; timestamp }
interface ToolResultMessage { role: "toolResult"; toolCallId: string; toolName: string; content: (TextContent | ImageContent)[]; isError: boolean; timestamp }

type Message = UserMessage | AssistantMessage | ToolResultMessage

// Tool definition using TypeBox schemas
interface Tool<TParameters extends TSchema = TSchema> {
  name: string
  description: string
  parameters: TParameters
}

// Model definition
interface Model<TApi extends Api> {
  id: string; name: string; api: TApi; provider: Provider; baseUrl: string
  reasoning: boolean; input: ("text" | "image")[]
  cost: { input: number; output: number; cacheRead: number; cacheWrite: number }
  contextWindow: number; maxTokens: number
  headers?: Record<string, string>
  compat?: OpenAICompletionsCompat | OpenAIResponsesCompat
}

// Usage tracking
interface Usage {
  input: number; output: number; cacheRead: number; cacheWrite: number; totalTokens: number
  cost: { input: number; output: number; cacheRead: number; cacheWrite: number; total: number }
}

type StopReason = "stop" | "length" | "toolUse" | "error" | "aborted"
```

### Compatibility Overrides

```typescript
// For OpenAI-compatible completions APIs
interface OpenAICompletionsCompat {
  supportsStore?: boolean
  supportsDeveloperRole?: boolean
  supportsReasoningEffort?: boolean
  supportsUsageInStreaming?: boolean
  maxTokensField?: "max_completion_tokens" | "max_tokens"
  requiresToolResultName?: boolean
  requiresAssistantAfterToolResult?: boolean
  requiresThinkingAsText?: boolean
  requiresMistralToolIds?: boolean
  thinkingFormat?: "openai" | "zai" | "qwen"
  supportsStrictMode?: boolean
}
```

---

## Key Design Patterns

| Pattern | Where | Purpose |
|---|---|---|
| **Strategy** | Provider system | Each API backend is a swappable strategy |
| **Registry** | `api-registry.ts` | Dynamic provider registration with `sourceId` tracking |
| **Adapter** | `transform-messages.ts` | Cross-provider message format normalization |
| **Observer/Iterator** | `EventStream<T,R>` | Async push/pull decoupling for streaming |
| **Template Method** | All provider files | Common flow: create client -> build params -> stream -> emit events |
| **Factory** | `AssistantMessageEventStream` | Creates typed event streams for each provider call |
| **Options Builder** | `StreamOptions` hierarchy | Unified options with provider-specific extensions |
| **Visitor** | `transformMessages()` | Content block transformation via `flatMap` per block type |

---

## Provider System

Each provider implements the `ApiProvider` interface:

```typescript
interface ApiProvider<TApi extends Api = Api, TOptions extends StreamOptions = StreamOptions> {
  api: TApi
  stream: StreamFunction<TApi, TOptions>
  streamSimple: StreamFunction<TApi, SimpleStreamOptions>
}

type StreamFunction<TApi extends Api, TOptions> = (
  model: Model<TApi>,
  context: Context,
  options?: TOptions
) => AssistantMessageEventStream
```

### Registration Flow

```
ApiProvider { api, stream(), streamSimple() }
           ↓ registered in
     api-registry (Map<Api, ApiProvider>)
           ↓ resolved by
     stream(model, context, options)  ←── public API
```

### Provider Registration (`api-registry.ts`)

```typescript
registerApiProvider(provider: ApiProvider, sourceId?: string): void
getApiProvider(api: Api): ApiProviderInternal | undefined
getApiProviders(): ApiProviderInternal[]
unregisterApiProviders(sourceId: string): void
clearApiProviders(): void
```

### Built-in Providers (9 backends)

| Provider | API | File |
|---|---|---|
| Anthropic | `anthropic-messages` | `anthropic.ts` (~600 lines) |
| OpenAI Completions | `openai-completions` | `openai-completions.ts` (~550 lines) |
| OpenAI Responses | `openai-responses` | `openai-responses.ts` + `openai-responses-shared.ts` |
| Azure OpenAI | `azure-openai-responses` | `azure-openai-responses.ts` |
| OpenAI Codex | `openai-codex-responses` | `openai-codex-responses.ts` |
| Google GenAI | `google-generative-ai` | `google.ts` (~450 lines) + `google-shared.ts` |
| Google Gemini CLI | `google-gemini-cli` | `google-gemini-cli.ts` |
| Google Vertex | `google-vertex` | `google-vertex.ts` |
| Amazon Bedrock | `bedrock-converse-stream` | `amazon-bedrock.ts` (~600 lines) |

### Provider Implementation Pattern

Every provider follows the same async structure:

```typescript
export const streamProvider: StreamFunction<"api-name", ProviderOptions> = (
  model, context, options
): AssistantMessageEventStream => {
  const stream = new AssistantMessageEventStream()

  ;(async () => {
    try {
      // 1. Resolve API key
      const apiKey = options?.apiKey || getEnvApiKey(model.provider) || ""

      // 2. Create SDK client
      const client = createClient(model, apiKey, options?.headers)

      // 3. Build request parameters
      const params = buildParams(model, context, options)
      options?.onPayload?.(params)

      // 4. Make streaming API call
      const providerStream = await client.stream(params, { signal: options?.signal })

      // 5. Emit start event
      stream.push({ type: "start", partial: output })

      // 6. Process chunks, emit delta events
      for await (const chunk of providerStream) {
        // Parse chunk, emit text_delta/thinking_delta/toolcall_delta
        // Update output.content and output.usage
      }

      // 7. Emit done event
      stream.push({ type: "done", reason: output.stopReason, message: output })
    } catch (error) {
      stream.push({ type: "error", reason: "error", error: output })
    }
  })()

  return stream // Return immediately (non-blocking)
}
```

### Provider-Specific Details

**Anthropic** — Cache control with ephemeral + 1h TTL for long retention. Tool name normalization to canonical names. Interleaved thinking support.

**OpenAI Completions** — Supports all OpenAI-compatible providers via `baseUrl`. Mistral tool ID format (9 alphanumeric chars). Developer role for reasoning models vs system for others.

**Google** — Thought signatures for context preservation. Tool call ID normalization. Functional calling configuration.

**Bedrock** — Cache points for input/output caching. Interleaved thinking for Claude 4.x. Tool result status (success/error).

---

## Streaming Architecture

### Entry Points (`stream.ts`)

```typescript
// Standard API — provider-specific options
function stream<TApi extends Api>(model: Model<TApi>, context: Context, options?: ProviderStreamOptions): AssistantMessageEventStream
async function complete<TApi extends Api>(model: Model<TApi>, context: Context, options?: ProviderStreamOptions): Promise<AssistantMessage>

// Simplified API — unified reasoning support
function streamSimple<TApi extends Api>(model: Model<TApi>, context: Context, options?: SimpleStreamOptions): AssistantMessageEventStream
async function completeSimple<TApi extends Api>(model: Model<TApi>, context: Context, options?: SimpleStreamOptions): Promise<AssistantMessage>
```

### Event Stream (`utils/event-stream.ts`)

```typescript
class EventStream<T, R = T> implements AsyncIterable<T> {
  push(event: T): void        // Queue event, deliver to waiting consumer
  end(result?: R): void        // Signal completion
  result(): Promise<R>         // Get final result
  async *[Symbol.asyncIterator]()  // for await...of support
}

class AssistantMessageEventStream extends EventStream<AssistantMessageEvent, AssistantMessage> {
  // Detects "done"/"error" events as terminal
  // Extracts AssistantMessage from terminal event
}
```

Uses a **queue + waiter** pattern: producers push events; consumers either dequeue immediately or register a waiter callback.

### Streaming Event Types

```typescript
type AssistantMessageEvent =
  | { type: "start"; partial: AssistantMessage }
  | { type: "text_start" | "text_delta" | "text_end"; contentIndex: number; ... }
  | { type: "thinking_start" | "thinking_delta" | "thinking_end"; ... }
  | { type: "toolcall_start" | "toolcall_delta" | "toolcall_end"; ... }
  | { type: "done"; reason: StopReason; message: AssistantMessage }
  | { type: "error"; reason: StopReason; error: AssistantMessage }
```

---

## Model System

### Model Registry (`models.ts`)

```typescript
// Type-safe model access with full IDE autocomplete
function getModel<TProvider extends KnownProvider, TModelId extends keyof (typeof MODELS)[TProvider]>(
  provider: TProvider, modelId: TModelId
): Model<ModelApi<TProvider, TModelId>>

function getModels<TProvider extends KnownProvider>(provider: TProvider): Model<...>[]
function getProviders(): KnownProvider[]
function calculateCost(model: Model<Api>, usage: Usage): Usage["cost"]
function supportsXhigh(model: Model<Api>): boolean
function modelsAreEqual(a: Model, b: Model): boolean
```

### Model Generation (`scripts/generate-models.ts`)

Fetches models from multiple sources:
- **Hard-coded** — Anthropic, Amazon, direct API providers
- **OpenRouter API** — Dynamic tool-capable model discovery
- **Vercel AI Gateway API** — Dynamic tool-capable model discovery
- **Copilot static headers**

Generates `models.generated.ts` (~3000 lines) with full metadata per model: cost/million tokens, context window, max tokens, input modalities, reasoning flag, compatibility overrides.

---

## Public API Surface (`index.ts`)

```typescript
// Core types
export type { Static, TSchema } from "@sinclair/typebox"
export { Type } from "@sinclair/typebox"

// Provider & model system
export * from "./api-registry.js"
export * from "./models.js"
export * from "./types.js"

// Streaming API
export * from "./stream.js"

// Provider implementations (all 9 backends)
export * from "./providers/anthropic.js"
export * from "./providers/google.js"
export * from "./providers/openai-completions.js"
// ... etc.

// Utilities
export * from "./env-api-keys.js"
export * from "./utils/event-stream.js"
export * from "./utils/json-parse.js"
export * from "./utils/oauth/index.js"
export * from "./utils/overflow.js"
export * from "./utils/validation.js"
export * from "./utils/typebox-helpers.js"
```

### Typical Usage

```typescript
import { getModel, stream, Type, Tool } from "@mariozechner/pi-ai"

const model = getModel("openai", "gpt-4o-mini")

const tools: Tool[] = [{
  name: "get_time",
  description: "Get current time",
  parameters: Type.Object({ timezone: Type.Optional(Type.String()) }),
}]

const context = {
  systemPrompt: "You are helpful",
  messages: [{ role: "user", content: "What time is it?", timestamp: Date.now() }],
  tools,
}

const s = stream(model, context)
for await (const event of s) {
  if (event.type === "text_delta") process.stdout.write(event.delta)
}
const message = await s.result()
```

---

## Cross-Cutting Concerns

### Environment Variable Resolution (`env-api-keys.ts`)

Resolves API keys for 20+ providers with special handling:
- **Vertex AI** — Application Default Credentials (ADC) + env vars
- **Amazon Bedrock** — AWS credentials (profile, keys, bearer tokens, ECS/IRSA)
- **Anthropic** — OAuth token and API key
- **GitHub Copilot** — GitHub/Copilot tokens
- **OAuth providers** — Return `undefined` (handled by OAuth flow)

### Cross-Provider Handoffs (`transform-messages.ts`)

- Converts thinking blocks to text when switching providers
- Normalizes tool call IDs for Mistral, Anthropic, Google
- Adapts message formats between different API conventions

### Context Overflow Detection (`overflow.ts`)

- Regex patterns for 15+ provider error messages
- Detects when input exceeds context window
- Handles silent overflow (z.ai) via token comparison

### Tool Validation (`validation.ts`)

- AJV for JSON Schema validation against TypeBox schemas
- Browser extension CSP restrictions handling (no eval)
- Type coercion support

### Partial JSON Parsing (`json-parse.ts`)

- `partial-json` library for streaming tool arguments
- Fallback to empty object on parse failure

---

## Test Structure

| Category | Files | Scope |
|---|---|---|
| Core integration | `stream.test.ts` | Text generation, tool handling, images, thinking across all providers |
| Token/usage | `tokens.test.ts`, `total-tokens.test.ts`, `cache-retention.test.ts` | Token counting, cost calculation, cache modes |
| Provider-specific | `anthropic-tool-name-normalization.test.ts`, `google-thinking-signature.test.ts`, etc. | Provider quirks and edge cases |
| Edge cases | `context-overflow.test.ts`, `unicode-surrogate.test.ts`, `empty.test.ts`, `abort.test.ts` | Error handling and boundary conditions |
| Special features | `interleaved-thinking.test.ts`, `xhigh.test.ts`, `image-tool-result.test.ts` | Advanced capabilities |

Test config: `vitest` with 30s timeout for API calls, node environment.

---

## Build Configuration

- **Compiler**: `tsgo` (TypeScript native preview) for fast compilation
- **Module system**: ESM with `.js` extensions (Node16 resolution)
- **Build steps**: `generate-models` (fetch + codegen) then `tsgo -p tsconfig.build.json`

---

## Strengths

1. **Clean abstraction** — Callers use `stream(model, context)` without knowing provider internals
2. **Extensible** — Extensions can register custom providers via the registry
3. **Type-safe models** — Full TypeScript inference from provider + model to API type
4. **Consistent streaming** — Same event types and iteration pattern for all providers
5. **Battle-tested edge cases** — Unicode sanitization, partial JSON parsing, abort handling, tool ID normalization

## Complexity Areas

1. **`models.generated.ts`** (~3000 lines) is the largest file, but auto-generated and not manually maintained
2. **Compatibility overrides** (`compat` field) handle substantial variation across OpenAI-compatible providers
3. **Provider-specific auth** — 20+ env var patterns, OAuth flows, ADC detection, and AWS credential chains
