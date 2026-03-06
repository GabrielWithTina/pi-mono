# index.ts — Barrel Exports

**Source:** `packages/agent/src/index.ts`

## Purpose

Re-exports all public API from the package.

## Exports

| Line | Module | Key Exports |
|---|---|---|
| 2 | `./agent.js` | `Agent`, `AgentOptions` |
| 4 | `./agent-loop.js` | `agentLoop()`, `agentLoopContinue()` |
| 6 | `./proxy.js` | `streamProxy()`, `ProxyStreamOptions`, `ProxyAssistantMessageEvent` |
| 8 | `./types.js` | `AgentMessage`, `AgentEvent`, `AgentTool`, `AgentLoopConfig`, `AgentState`, `ThinkingLevel`, etc. |
