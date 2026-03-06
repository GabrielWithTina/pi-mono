# index.ts — Package Public API

**Source:** `packages/ai/src/index.ts`

## Purpose

Barrel export file. Single entry point for all public symbols from `@mariozechner/pi-ai`.

## Re-exports (lines 1–22)

| Line | Source | What |
|---|---|---|
| 1–2 | `@sinclair/typebox` | `Static`, `TSchema`, `Type` |
| 4 | `./api-registry.js` | registerApiProvider, getApiProvider, etc. |
| 5 | `./env-api-keys.js` | getEnvApiKey |
| 6 | `./models.js` | getModel, getProviders, calculateCost, etc. |
| 7–13 | `./providers/*.js` | All provider stream functions and types |
| 14 | `./providers/register-builtins.js` | registerBuiltInApiProviders, resetApiProviders |
| 15 | `./stream.js` | stream, complete, streamSimple, completeSimple |
| 16 | `./types.js` | All type definitions |
| 17 | `./utils/event-stream.js` | EventStream, AssistantMessageEventStream |
| 18 | `./utils/json-parse.js` | parseStreamingJson |
| 19 | `./utils/oauth/index.js` | OAuth providers and utilities |
| 20 | `./utils/overflow.js` | isContextOverflow |
| 21 | `./utils/typebox-helpers.js` | StringEnum |
| 22 | `./utils/validation.js` | validateToolCall, validateToolArguments |

Side-effect: importing this module triggers `register-builtins.ts` and `http-proxy.ts` via `stream.js`.
