# How to Run/Debug in VS Code
- .vscode/launch.json
- .vscode/tasks.json

Run watchers: Ctrl+Shift+P → Tasks: Run Task → pick dev:coding-agent (or another).
Debug: Ctrl+Shift+D → choose a config like Debug coding-agent CLI (src) → F5.
Set breakpoints: Place them in .ts files under the package src/.

# Agent package

## Agent
- For user input, when do we choose `followup` or `prompt` function call?

## Agent-loop
- To support steering mode, loop call tools sequentially, HOw do we approach this in parallel?