# config.ts — Paths & Configuration

**Source:** `packages/coding-agent/src/config.ts` (241 lines)

## Purpose

Resolves file paths for package assets, user configuration, and app constants. Handles Bun binary vs npm/pnpm/yarn installations.

## App Constants

| Constant | Line | Description |
|---|---|---|
| `APP_NAME` | 167 | Application name from package.json `piConfig` (default: `"pi"`) |
| `CONFIG_DIR_NAME` | 168 | Config directory name (default: `".pi"`) |
| `VERSION` | 169 | Package version string |
| `isBunBinary` | 17 | Whether running as Bun compiled binary |
| `isBunRuntime` | 21 | Whether Bun is the runtime |

## Path Resolution

### Package Assets (built with executable)

| Function | Line | Returns |
|---|---|---|
| `getPackageDir()` | 80 | Base directory for themes, docs, etc. |
| `getThemesDir()` | 111 | Built-in themes directory |
| `getExportTemplateDir()` | 127 | HTML export template directory |
| `getPackageJsonPath()` | 137 | `package.json` path |
| `getDocsPath()` | 147 | Docs directory |
| `getChangelogPath()` | 157 | `CHANGELOG.md` path |

### User Configuration (`~/.pi/agent/`)

| Function | Line | Returns |
|---|---|---|
| `getAgentDir()` | 187 | `~/.pi/agent` (or `$PI_CODING_AGENT_DIR`) |
| `getModelsPath()` | 204 | `~/.pi/agent/models.json` |
| `getAuthPath()` | 209 | `~/.pi/agent/auth.json` |
| `getSettingsPath()` | 214 | `~/.pi/agent/settings.json` |
| `getCustomThemesDir()` | 199 | `~/.pi/agent/themes/` |
| `getToolsDir()` | 219 | `~/.pi/agent/tools/` |
| `getBinDir()` | 224 | `~/.pi/agent/bin/` (managed binaries: fd, rg) |
| `getPromptsDir()` | 229 | `~/.pi/agent/prompts/` |
| `getSessionsDir()` | 234 | `~/.pi/agent/sessions/` |

## `detectInstallMethod()` (line 29)

```typescript
export function detectInstallMethod(): InstallMethod
// Returns: "bun-binary" | "npm" | "pnpm" | "yarn" | "bun" | "unknown"
```

Detects package manager for update instructions.
