# Installing PowerShell Safety for ZCode (Z.AI)

ZCode 3.0+ follows the Agent Skills open standard. Skills are auto-discovered from `.zcode/skills/`.

## Install

```bash
# Clone the repo
git clone https://github.com/AmethystLuna/powershell-safety.git

# Symlink skills into ZCode discovery path (project-level)
mkdir -p .zcode/skills
cp -r powershell-safety/skills/* .zcode/skills/
```

Verify: `$powershell-safety` in ZCode chat.

## Notes

- ZCode has no plugin marketplace — manual install only
- ZCode does not support custom agents — only skills are usable
- This plugin contains 1 skill (no agents)
- ZCode auto-discovers skills from `.zcode/skills/`, `.claude/skills/`, and `.codex/skills/`
