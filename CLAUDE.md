# CLAUDE.md

Contributor guidelines for the PowerShell Safety plugin.

## Scope

This plugin covers PowerShell command safety on Windows — encoding pitfalls, quoting, native executable interaction, and destructive command patterns. It does NOT cover PowerShell scripting best practices, module authoring, or DSC configuration.

## PR Requirements

- All PRs must pass `markdownlint` with the project's `.markdownlint.json` config.
- The skill must follow the established frontmatter format: `name` (kebab-case), `description` ("Use when..." format with explicit exclusions).
- Reference material goes in `references/` — keep the skill file focused on immediately actionable rules.
- Encoding tables must be verified against the [official PowerShell documentation](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/out-file) for PS 5.1 and PS 7+.

## Before Submitting

- Run `markdownlint` on all changed files.
- Verify `plugin.json` passes validation for each platform (`claude plugin validate`, etc.).
- Test the plugin locally by installing to `~/.claude/plugins/dev/`.
