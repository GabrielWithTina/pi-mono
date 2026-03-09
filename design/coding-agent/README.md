# packages/coding-agent — Design Documentation

Full coding agent CLI (`pi` command). Wraps `@mariozechner/agent` with session persistence, tool implementations, extension system, compaction, and three run modes (interactive TUI, print, RPC).

## Architecture

```mermaid
flowchart TD
    subgraph "Entry"
        CLI["cli.ts → main.ts<br/>CLI entry + arg parsing"]
    end

    subgraph "Core"
        SESSION["agent-session.ts<br/>Session orchestrator"]
        SM["session-manager.ts<br/>JSONL persistence + tree"]
        SDK["sdk.ts<br/>Session factory"]
        MSG["messages.ts<br/>Custom message types"]
        SP["system-prompt.ts<br/>Prompt builder"]
        MR["model-resolver.ts<br/>Model resolution"]
        MREG["model-registry.ts<br/>Model registry + custom models"]
        CONFIG["config.ts<br/>Paths + configuration"]
    end

    subgraph "Extensions"
        TYPES["extensions/types.ts<br/>Extension API types"]
        RUNNER["extensions/runner.ts<br/>Event dispatch"]
        LOADER["extensions/loader.ts<br/>Discovery + loading"]
    end

    subgraph "Compaction"
        COMPACT["compaction/compaction.ts<br/>Context compaction"]
        BRANCH["compaction/branch-summarization.ts<br/>Branch summaries"]
    end

    subgraph "Tools"
        TOOLS["tools/index.ts<br/>bash, read, edit, write,<br/>grep, find, ls"]
    end

    subgraph "Modes"
        INTERACTIVE["interactive-mode.ts<br/>Full TUI (4396 lines)"]
        PRINT["print-mode.ts<br/>Single-shot output"]
        RPC["rpc-mode.ts<br/>JSON-RPC headless"]
    end

    subgraph "Dependencies"
        AGENT["@mariozechner/agent<br/>Agent class + loop"]
        AI["@mariozechner/ai<br/>LLM streaming"]
        TUI["@mariozechner/tui<br/>Terminal UI"]
    end

    CLI --> SDK
    SDK --> SESSION
    SDK --> SM
    SDK --> MREG
    SDK --> MR
    SESSION --> AGENT
    SESSION --> SM
    SESSION --> SP
    SESSION --> MSG
    SESSION --> COMPACT
    SESSION --> BRANCH
    SESSION --> RUNNER
    RUNNER --> LOADER
    RUNNER --> TYPES
    SESSION --> TOOLS

    CLI -->|"dispatch"| INTERACTIVE
    CLI -->|"dispatch"| PRINT
    CLI -->|"dispatch"| RPC

    INTERACTIVE --> SESSION
    INTERACTIVE --> TUI
    PRINT --> SESSION
    RPC --> SESSION
```

## Data Flow

```mermaid
flowchart LR
    USER["User Input"] -->|"prompt()"| SESSION["AgentSession"]
    SESSION -->|"steer/followUp"| AGENT["Agent"]
    SESSION -->|"convertToLlm()"| MSG["messages.ts"]
    MSG --> AGENT
    AGENT -->|"streamSimple()"| LLM["LLM Provider"]
    LLM -->|"events"| AGENT
    AGENT -->|"AgentEvent"| SESSION
    SESSION -->|"append*()"| SM["SessionManager<br/>(JSONL)"]
    SESSION -->|"AgentSessionEvent"| MODE["Interactive/Print/RPC"]
```

## Document Index

### Core
| Document | Source | Description |
|---|---|---|
| [core/agent-session.md](core/agent-session.md) | `core/agent-session.ts` | Session orchestrator — prompting, steering, compaction, retry, extensions |
| [core/session-manager.md](core/session-manager.md) | `core/session-manager.ts` | JSONL persistence, tree structure, branching, context building |
| [core/sdk.md](core/sdk.md) | `core/sdk.ts` | `createAgentSession()` factory — wires all dependencies |
| [core/messages.md](core/messages.md) | `core/messages.ts` | Custom message types + `convertToLlm()` |
| [core/system-prompt.md](core/system-prompt.md) | `core/system-prompt.ts` | System prompt builder |
| [core/model-resolver.md](core/model-resolver.md) | `core/model-resolver.ts` | Model pattern matching + resolution |
| [core/model-registry.md](core/model-registry.md) | `core/model-registry.ts` | Model registry with custom models.json support |
| [config.md](config.md) | `config.ts` | Path resolution + app configuration |

### Extensions
| Document | Source | Description |
|---|---|---|
| [core/extensions/overview.md](core/extensions/overview.md) | `extensions/*.ts` | Extension system — types, loader, runner |

### Compaction
| Document | Source | Description |
|---|---|---|
| [core/compaction/overview.md](core/compaction/overview.md) | `compaction/*.ts` | Context compaction + branch summarization |

### Tools
| Document | Source | Description |
|---|---|---|
| [core/tools/overview.md](core/tools/overview.md) | `tools/*.ts` | Built-in tool implementations |

### Entry & Modes
| Document | Source | Description |
|---|---|---|
| [main.md](main.md) | `main.ts` | CLI entry point + mode dispatch |
| [modes/interactive.md](modes/interactive.md) | `modes/interactive/interactive-mode.ts` | Interactive TUI mode |
| [modes/print.md](modes/print.md) | `modes/print-mode.ts` | Non-interactive single-shot mode |
| [modes/rpc/overview.md](modes/rpc/overview.md) | `modes/rpc/*.ts` | Headless JSON-RPC mode |
