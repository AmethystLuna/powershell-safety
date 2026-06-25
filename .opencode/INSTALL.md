# Installing PowerShell Safety for OpenCode

## Installation

Add to the `plugin` array in your `opencode.json` (global or project-level):

```json
{
  "plugin": ["powershell-safety@git+https://github.com/AmethystLuna/powershell-safety.git"]
}
```

Or pin a specific version:

```json
{
  "plugin": ["powershell-safety@git+https://github.com/AmethystLuna/powershell-safety.git#v0.1.0"]
}
```

Restart OpenCode.

Verify: ask "What PowerShell safety rules do you have available?"

## Manual Install

```bash
git clone https://github.com/AmethystLuna/powershell-safety.git ~/.config/opencode/plugins/powershell-safety
```

Skills are auto-discovered from the standard `.claude/skills/` and `.codex/skills/` paths within the plugin directory.

## Getting Help

- Issues: <https://github.com/AmethystLuna/powershell-safety/issues>
- Docs: <https://github.com/AmethystLuna/powershell-safety>
