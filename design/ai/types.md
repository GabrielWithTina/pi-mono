# types.ts — Core Type Definitions

**Source:** `packages/ai/src/types.ts`

## Purpose

Central type definitions for the unified multi-provider LLM streaming API. Defines all contracts between the streaming layer, providers, and consumers.

## Type Overview

```mermaid
classDiagram
    class Message {
        <<union>>
    }
    class UserMessage {
        +role: "user"
        +content: string | (TextContent | ImageContent)[]
        +timestamp: number
    }
    class AssistantMessage {
        +role: "assistant"
        +content: (TextContent | ThinkingContent | ToolCall)[]
        +api: Api
        +provider: Provider
        +model: string
        +usage: Usage
        +stopReason: StopReason
        +errorMessage?: string
        +timestamp: number
    }
    class ToolResultMessage {
        +role: "toolResult"
        +toolCallId: string
        +toolName: string
        +content: (TextContent | ImageContent)[]
        +details?: any
        +isError: boolean
        +timestamp: number
    }
    Message <|-- UserMessage
    Message <|-- AssistantMessage
    Message <|-- ToolResultMessage

    class Content {
        <<union>>
    }
    class TextContent {
        +type: "text"
        +text: string
        +textSignature?: string
    }
    class ThinkingContent {
        +type: "thinking"
        +thinking: string
        +thinkingSignature?: string
    }
    class ImageContent {
        +type: "image"
        +data: string
        +mimeType: string
    }
    class ToolCall {
        +type: "toolCall"
        +id: string
        +name: string
        +arguments: Record~string, any~
        +thoughtSignature?: string
    }
    Content <|-- TextContent
    Content <|-- ThinkingContent
    Content <|-- ImageContent
    Content <|-- ToolCall

    AssistantMessage --> Content : contains
```

## API & Provider Identification (lines 5–41)

- **`KnownApi`** (line 5): Union of 9 API types — `"openai-completions"`, `"openai-responses"`, `"azure-openai-responses"`, `"openai-codex-responses"`, `"anthropic-messages"`, `"bedrock-converse-stream"`, `"google-generative-ai"`, `"google-gemini-cli"`, `"google-vertex"`
- **`Api`** (line 16): `KnownApi | (string & {})` — extensible for custom providers
- **`KnownProvider`** (line 18): Union of 20+ providers (openai, anthropic, google, amazon-bedrock, groq, mistral, etc.)
- **`Provider`** (line 41): `KnownProvider | string`

## Configuration Types (lines 43–112)

- **`ThinkingLevel`** (line 43): `"minimal" | "low" | "medium" | "high" | "xhigh"`
- **`ThinkingBudgets`** (line 46): Token allocations per thinking level
- **`CacheRetention`** (line 54): `"none" | "short" | "long"`
- **`Transport`** (line 56): `"sse" | "websocket" | "auto"`
- **`StreamOptions`** (line 58): Base options — temperature, maxTokens, signal, apiKey, transport, cacheRetention, sessionId, headers, onPayload, maxRetryDelayMs, metadata
- **`ProviderStreamOptions`** (line 105): `StreamOptions & Record<string, unknown>`
- **`SimpleStreamOptions`** (line 108): Extends StreamOptions with `reasoning` (ThinkingLevel) and `thinkingBudgets`
- **`StreamFunction<TApi, TOptions>`** (line 115): Generic provider stream signature

## Content Types (lines 121–145)

- **`TextContent`** (line 121): Text with optional `textSignature` (OpenAI responses message ID)
- **`ThinkingContent`** (line 127): Thinking with optional `thinkingSignature` (reasoning item ID)
- **`ImageContent`** (line 133): Base64-encoded image with MIME type
- **`ToolCall`** (line 139): Function call with id, name, arguments, optional `thoughtSignature` (Google-specific)

## Usage & Messages (lines 147–206)

- **`Usage`** (line 147): Token counts (input, output, cacheRead, cacheWrite, totalTokens) + nested cost object
- **`StopReason`** (line 162): `"stop" | "length" | "toolUse" | "error" | "aborted"`
- **`UserMessage`** (line 164): User role with string or mixed content, timestamp
- **`AssistantMessage`** (line 170): Assistant role with content blocks, api/provider/model, usage, stopReason
- **`ToolResultMessage`** (line 182): Tool result with toolCallId, toolName, content, isError
- **`Message`** (line 192): Union of all three message types
- **`Tool`** (line 196): Tool definition with name, description, TypeBox parameters schema
- **`Context`** (line 202): Request context — systemPrompt, messages, tools

## Streaming Events (lines 208–220)

**`AssistantMessageEvent`** — discriminated union:

```mermaid
stateDiagram-v2
    [*] --> start
    start --> text_start
    start --> thinking_start
    start --> toolcall_start

    text_start --> text_delta
    text_delta --> text_delta
    text_delta --> text_end

    thinking_start --> thinking_delta
    thinking_delta --> thinking_delta
    thinking_delta --> thinking_end

    toolcall_start --> toolcall_delta
    toolcall_delta --> toolcall_delta
    toolcall_delta --> toolcall_end

    text_end --> thinking_start
    text_end --> toolcall_start
    text_end --> done
    thinking_end --> text_start
    thinking_end --> toolcall_start
    thinking_end --> done
    toolcall_end --> toolcall_start
    toolcall_end --> text_start
    toolcall_end --> done

    start --> error
    text_delta --> error
    thinking_delta --> error
    toolcall_delta --> error
    done --> [*]
    error --> [*]
```

| Event | Key fields | Line |
|---|---|---|
| `start` | `partial: AssistantMessage` | 209 |
| `text_start/delta/end` | `contentIndex`, `delta`/`content` | 210–212 |
| `thinking_start/delta/end` | `contentIndex`, `delta`/`content` | 213–215 |
| `toolcall_start/delta/end` | `contentIndex`, `delta`/`toolCall` | 216–218 |
| `done` | `reason`, `message: AssistantMessage` | 219 |
| `error` | `reason`, `error: AssistantMessage` | 220 |

## Compatibility Interfaces (lines 222–308)

- **`OpenAICompletionsCompat`** (line 226): Override URL-based auto-detection for OpenAI-compatible APIs — supportsStore, supportsDeveloperRole, supportsReasoningEffort, thinkingFormat, requiresMistralToolIds, etc.
- **`OpenAIResponsesCompat`** (line 256): Reserved for future use
- **`OpenRouterRouting`** (line 265): Provider routing with `only`/`order` arrays
- **`VercelGatewayRouting`** (line 277): Same pattern for Vercel AI Gateway
- **`Model<TApi>`** (line 285): Complete model definition — id, name, api, provider, baseUrl, reasoning, input, cost ($/M tokens), contextWindow, maxTokens, headers, compat (conditional on TApi)
