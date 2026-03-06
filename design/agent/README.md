# packages/agent — Design Documentation

Stateful agent runtime with tool calling, steering/follow-up queues, and event streaming. Wraps `@mariozechner/pi-ai` streaming into a turn-based agent loop.

## Architecture

```mermaid
flowchart TD
    subgraph "packages/agent"
        TYPES["types.ts<br/>AgentMessage, AgentEvent,<br/>AgentTool, AgentLoopConfig"]
        LOOP["agent-loop.ts<br/>agentLoop(), agentLoopContinue()<br/>runLoop(), executeToolCalls()"]
        AGENT["agent.ts<br/>Agent class<br/>State, queues, lifecycle"]
        PROXY["proxy.ts<br/>streamProxy()<br/>SSE proxy client"]
        INDEX["index.ts<br/>Barrel exports"]
    end

    subgraph "@mariozechner/pi-ai"
        AI_STREAM["streamSimple()"]
        AI_EVENT["EventStream"]
        AI_VALIDATE["validateToolArguments()"]
        AI_TYPES["Message, Tool, Context"]
    end

    TYPES --> LOOP
    TYPES --> AGENT
    LOOP --> AGENT
    AGENT --> LOOP
    PROXY --> AI_EVENT
    LOOP --> AI_STREAM
    LOOP --> AI_EVENT
    LOOP --> AI_VALIDATE
    AGENT --> AI_STREAM
    INDEX --> AGENT
    INDEX --> LOOP
    INDEX --> PROXY
    INDEX --> TYPES
```

## Data Flow

```mermaid
flowchart LR
    USER["User / App"] -->|"prompt()"| AGENT["Agent"]
    AGENT -->|"AgentMessage[]"| LOOP["agentLoop()"]
    LOOP -->|"transformContext()"| TC["Context Transform"]
    TC -->|"convertToLlm()"| CONV["AgentMessage[] → Message[]"]
    CONV -->|"streamSimple()"| LLM["LLM Provider"]
    LLM -->|"AssistantMessageEvent"| LOOP
    LOOP -->|"executeToolCalls()"| TOOLS["AgentTool.execute()"]
    TOOLS -->|"ToolResultMessage"| LOOP
    LOOP -->|"AgentEvent"| AGENT
    AGENT -->|"emit()"| USER
```

## Document Index

| Document | Source | Description |
|---|---|---|
| [types.md](types.md) | `src/types.ts` | Core type definitions — AgentMessage, AgentEvent, AgentTool, AgentLoopConfig, AgentState |
| [agent-loop.md](agent-loop.md) | `src/agent-loop.ts` | Agent loop engine — turn processing, LLM streaming, tool execution, steering/follow-up |
| [agent.md](agent.md) | `src/agent.ts` | Agent class — stateful wrapper with queues, lifecycle management, event emission |
| [proxy.md](proxy.md) | `src/proxy.ts` | Proxy stream function — SSE-based LLM proxy client with partial message reconstruction |
| [index.md](index.md) | `src/index.ts` | Barrel exports |
