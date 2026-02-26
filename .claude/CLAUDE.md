# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
npm install                  # Install all dependencies
npm run build                # Build all packages (sequential, dependency order)
npm run check                # Lint (biome) + type check (tsgo --noEmit + tsc for web-ui). Requires build first.
./test.sh                    # Run all tests without API keys (safe for CI)
./pi-test.sh                 # Run pi coding agent from source (must run from repo root)
```

Run a specific test from its package directory:
```bash
cd packages/ai
npx tsx ../../node_modules/vitest/dist/cli.js --run test/specific.test.ts
```

**Important:**
- `npm run check` does not run tests. Always run it after code changes (not doc-only changes) and fix all errors, warnings, and infos before committing.
- NEVER run `npm run dev`, `npm run build`, or `npm test` from the root during development. Only run specific tests as shown above.

## Architecture

TypeScript ESM monorepo (`"type": "module"`) using npm workspaces. All packages versioned in lockstep under `@mariozechner/` scope.

### Package Dependency Graph

```
ai (LLM primitives)     tui (terminal UI primitives)
  └─► agent               ├─► coding-agent
       ├─► coding-agent ◄─┘
       │    └─► mom
       ├─► pods
       └─► web-ui ◄─── tui
```

- **`packages/ai`** — Unified multi-provider LLM streaming API. Provider implementations in `src/providers/`. Model list is code-generated (`src/models.generated.ts` from `scripts/generate-models.ts`).
- **`packages/agent`** — Stateful agent runtime with tool calling, steering/follow-up queues, and event streaming. Core loop in `agent-loop.ts`.
- **`packages/tui`** — Terminal UI framework with differential rendering, Kitty keyboard protocol, and component model. Components in `src/components/`.
- **`packages/coding-agent`** — Full coding agent CLI (`pi` command). Tools in `src/core/tools/`, extension system in `src/core/extensions/`, TUI components in `src/modes/interactive/components/`. Three run modes: interactive TUI, print, RPC.
- **`packages/mom`** — Slack bot delegating to pi coding agent. Each Slack channel gets its own `AgentRunner` instance.
- **`packages/web-ui`** — Browser Web Components (Lit/mini-lit + Tailwind CSS v4) for AI chat. Does NOT extend base tsconfig; uses `moduleResolution: bundler` and DOM libs.
- **`packages/pods`** — CLI for managing vLLM deployments on GPU pods via SSH.

### Build Order

`tui → ai → agent → coding-agent → mom → web-ui → pods`

### Key Patterns

- **ESM with `.js` extensions**: All imports use `.js` extensions (Node16 module resolution). No inline/dynamic imports allowed — always use top-level `import` statements.
- **tsgo for compilation**: All packages except web-ui use `@typescript/native-preview` (tsgo). Web-ui uses standard `tsc`.
- **Biome for linting/formatting**: Tabs, indent width 3, line width 120. Config in root `biome.json`.
- **Session persistence**: JSON lines format with versioned schema and migrations. Compaction summarizes old context to stay within token limits.
- **Extension system**: Coding-agent loads TypeScript extensions dynamically via `jiti`. Extensions hook into agent lifecycle events and can register tools, commands, and keybindings.
- **Generated models**: `packages/ai/src/models.generated.ts` is auto-generated during build. Do not edit manually.

## Code Quality Rules (from AGENTS.md)

- No `any` types unless absolutely necessary.
- Check `node_modules` for external API type definitions instead of guessing.
- **NEVER use inline imports** — no `await import("./foo.js")`, no `import("pkg").Type` in type positions. Always use top-level imports.
- Never remove or downgrade code to fix type errors; upgrade the dependency instead.
- All keybindings must be configurable via `DEFAULT_EDITOR_KEYBINDINGS` or `DEFAULT_APP_KEYBINDINGS` — never hardcode key checks.
- Always ask before removing functionality or code that appears intentional.

## Git and Commit Rules

- NEVER use `git add -A` or `git add .` — always stage specific files.
- NEVER use `git commit --no-verify`.
- Include `fixes #<number>` or `closes #<number>` in commit messages when there's a related issue.
- Commit message style: `fix(ai): description`, `feat(coding-agent): description`, etc.
- Do not edit `CHANGELOG.md` unless explicitly asked. Entries go under `## [Unreleased]` in each package's own CHANGELOG.

## Adding a New LLM Provider

Requires changes across multiple files. See AGENTS.md for the full checklist covering: core types (`packages/ai/src/types.ts`), provider implementation (`packages/ai/src/providers/`), stream integration (`packages/ai/src/stream.ts`), model generation (`packages/ai/scripts/generate-models.ts`), tests (11+ test files), coding-agent integration, and documentation.

## Releasing

Lockstep versioning — all packages share the same version. `npm run release:patch` (fixes/features) or `npm run release:minor` (breaking changes). The script handles version bump, changelog finalization, commit, tag, publish, and push.
