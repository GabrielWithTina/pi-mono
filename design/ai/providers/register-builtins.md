# register-builtins.ts — Built-in Provider Registration

**Source:** `packages/ai/src/providers/register-builtins.ts`

## Purpose

Central registry for all built-in provider implementations.

## `registerBuiltInApiProviders()` (lines 12–66)

Registers all 9 built-in providers:

| API | Stream Function | Simple Function | Line |
|---|---|---|---|
| `anthropic-messages` | `streamAnthropic` | `streamSimpleAnthropic` | 13 |
| `openai-completions` | `streamOpenAICompletions` | `streamSimpleOpenAICompletions` | 19 |
| `openai-responses` | `streamOpenAIResponses` | `streamSimpleOpenAIResponses` | 25 |
| `azure-openai-responses` | `streamAzureOpenAIResponses` | `streamSimpleAzureOpenAIResponses` | 31 |
| `openai-codex-responses` | `streamOpenAICodexResponses` | `streamSimpleOpenAICodexResponses` | 37 |
| `google-generative-ai` | `streamGoogle` | `streamSimpleGoogle` | 43 |
| `google-gemini-cli` | `streamGoogleGeminiCli` | `streamSimpleGoogleGeminiCli` | 49 |
| `google-vertex` | `streamGoogleVertex` | `streamSimpleGoogleVertex` | 55 |
| `bedrock-converse-stream` | `streamBedrock` | `streamSimpleBedrock` | 61 |

## `resetApiProviders()` (lines 68–71)

Clears all providers and re-registers built-ins. Used for testing.

## Auto-Registration (line 73)

```typescript
registerBuiltInApiProviders();
```

Called at module load time. Since `stream.ts` imports this file, all providers are ready before any `stream()` call.
