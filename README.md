# PowerShell Safety

> PowerShell safety rules for AI coding agents — garbled text detection, encoding pitfalls, BOM handling, quoting, and destructive command patterns.

**Cross-platform** — works with Claude Code, Codex CLI, Cursor, Kimi CLI, OpenCode, and ZCode. Built on the [Agent Skills](https://agentskills.io) open standard.

## Why

AI coding agents generate PowerShell commands every day on Windows. The most common failures aren't logic errors — they're encoding mismatches that corrupt source files, broken pipes to native executables, and destructive wildcard commands. This plugin prevents those failures before the command runs.

## What's Inside

### Skills (1)

| Skill | Purpose |
|-------|---------|
| `powershell-safety` | Garbled text detection, file encoding iron rules, BOM handling, quoting, native executable piping, destructive command safety |

### Deep Reference

The skill includes one deep reference document:

| Reference | Topic |
|-----------|-------|
| `encoding-guide.md` | Full encoding matrix (PS 5.1 vs 7+), BOM diagnostics, `$OutputEncoding`, CJK specifics |

## Quick Examples

```powershell
# BAD: file written as UTF-16 LE — GCC chokes, git treats as binary
gcc -E config.h > config.i

# GOOD: explicit UTF-8 without BOM
[System.IO.File]::WriteAllText("config.i", $content, [System.Text.UTF8Encoding]::new($false))
```

```powershell
# BAD: native exe exit code ignored
& "$keil\UV4\UV4.exe" -b project.uvprojx -o build.log

# GOOD: capture exit code
$p = Start-Process -FilePath "$keil\UV4\UV4.exe" -ArgumentList "-b project.uvprojx -o build.log" -Wait -PassThru -NoNewWindow
if ($p.ExitCode -ne 0) { throw "Build failed" }
```

## How Skills Load

The plugin uses a **SessionStart hook** to inject encoding trigger symptoms and the **1% Rule** into every session. This means the model is proactively reminded — not waiting to passively remember the skill exists.

- At session start, a `<EXTREMELY_IMPORTANT>` block is injected with garbled-text detection symptoms, the 1% Rule ("if any encoding symptom is present, you MUST load the skill"), and Red Flags that intercept common encoding rationalizations ("it looks fine", "I'll just use Set-Content", "this is a quick one-liner")
- `Skill("powershell-safety")` — load the core safety rules and symptom→fix cheat sheet
- The `encoding-guide.md` reference loads when writing source files or diagnosing encoding issues

## Installation

### Claude Code

```bash
claude plugin install powershell-safety@AmethystLuna/powershell-safety
```

Or manually: clone to `~/.claude/plugins/dev/powershell-safety/`.

### Codex CLI

Add to `codex.json`:

```json
{
  "plugins": ["powershell-safety@git+https://github.com/AmethystLuna/powershell-safety.git"]
}
```

### Cursor

Copy the plugin directory to Cursor's plugins path. Skills follow the Agent Skills standard and auto-discover from standard paths.

### Kimi CLI

Kimi CLI discovers skills from `.claude/skills/` paths automatically. The `.kimi-plugin/plugin.json` manifest registers the plugin for Kimi's plugin manager.

### OpenCode

Add to `opencode.json`:

```json
{
  "plugin": ["powershell-safety@git+https://github.com/AmethystLuna/powershell-safety.git"]
}
```

See `.opencode/INSTALL.md` for details.

### ZCode

Manual install — copy skills to `.zcode/skills/`. See `.zcode/INSTALL.md` for details.

## Other Plugins Recommended

| Plugin | Description |
|--------|-------------|
| [embedded-workbench](https://github.com/AmethystLuna/embedded-workbench) | Embedded C/C++ development toolbox — FreeRTOS, Keil MDK, ARMCLANG, HardFault, state machines, LVGL |

## License

MIT © 2026 Amethyst Luna
