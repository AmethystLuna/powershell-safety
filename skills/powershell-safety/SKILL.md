---
name: powershell-safety
description: "Use when writing files via PowerShell consumed by other tools (compilers, git, parsers), when debugging garbled text or encoding issues, or when generating destructive/wide-scope PowerShell commands. NOT needed for read-only commands (grep, git status, dotnet build, markdownlint)."
---

# PowerShell Safety

## Encoding — Iron Rules

These are non-negotiable. Breaking any of them causes bugs that are hard to diagnose. For the full encoding matrix, BOM diagnostics, and cmdlet-specific defaults, load `references/encoding-guide.md`.

1. **Always `-Encoding utf8`** on `Out-File`, `Set-Content`, `Add-Content`. On PS 5.1 this produces UTF-8 **with** BOM — source files (`.c`, `.h`, `.py`) need BOM-free. Use `[System.IO.File]::WriteAllText($path, $content, [System.Text.UTF8Encoding]::new($false))` instead.

2. **Never use `>` or `>>` redirect on PS 5.1** for files read by non-PowerShell tools. These produce UTF-16 LE. Use `Out-File -Encoding utf8` or `Set-Content -Encoding utf8` instead.

3. **Set `$OutputEncoding`** when piping text to native executables:

   ```powershell
   $OutputEncoding = [System.Text.UTF8Encoding]::new($false)
   ```

4. **Specify encoding on read**: `Get-Content -Encoding utf8` for UTF-8, `Get-Content -Encoding Unicode` for UTF-16 LE. Don't rely on auto-detection in PS 5.1.

5. **For UTF-8 without BOM on PS 5.1**, use the .NET API directly:

   ```powershell
   [System.IO.File]::WriteAllText($path, $content, [System.Text.UTF8Encoding]::new($false))
   ```

### Encoding Defaults by Cmdlet (PS 5.1)

| Cmdlet | Default | BOM? |
|--------|:-------:|:----:|
| `Out-File` | UTF-16 LE | Yes |
| `Set-Content` | ANSI (system code page) | No |
| `Add-Content` | ANSI (system code page) | No |
| `>` / `>>` redirect | UTF-16 LE | Yes |

### Quick Symptom Diagnosis

| Symptom | Likely Cause |
|---------|-------------|
| CJK text as `?` or `??` | File written as ANSI (`Set-Content` without `-Encoding`) |
| GCC: "stray \\357" | UTF-8 BOM in source file |
| GCC: "stray \\377" | UTF-16 LE BOM in source file |
| JSON/XML parser fails on first char | BOM before `{` or `<` |
| `git diff` shows "Binary files differ" | UTF-16 detected as binary (2 bytes/char) |

---

## Command Safety

- Prefer standard cmdlet names: `Get-ChildItem` not `ls`, `Select-String` not `sls`. Spell out intent.
- Single-quote literal strings. Double-quote only when interpolation is needed.
- Quote all paths that contain spaces: `"$env:ProgramFiles\App\app.exe"`.
- One operation per line. Use `;` for sequencing, not bash-style `&&` or `||`.
- Verify target path before side effects: `Test-Path`, `Resolve-Path`, `Get-Location`.
- Destructive commands (`Remove-Item -Recurse`, `Stop-Process`) require explicit paths. Never generate broad wildcard deletes unless explicitly asked.
- Read-only commands stay read-only. Don't mix inspection and mutation in one step.

---

## Native Executables

- Use `Start-Process -Wait -PassThru` for tools that produce no stdout (UV4.exe, custom builders):

  ```powershell
  $p = Start-Process -FilePath "$tool" -ArgumentList "$args" -Wait -PassThru -NoNewWindow
  if ($p.ExitCode -ne 0) { throw "Failed (exit $($p.ExitCode))" }
  ```

- Check `$LASTEXITCODE` for native tools. `$?` alone is not enough.
- Avoid `2>&1` on native exe in PS 5.1 — it wraps stderr lines in ErrorRecord and sets `$?` to `$false` even on exit code 0. Use `Start-Process` to capture stderr cleanly.

---

## Deep Reference

This skill's `references/` directory contains:

| Reference | Topic | Load When |
|-----------|-------|-----------|
| `encoding-guide.md` | Full encoding matrix, BOM handling, `$OutputEncoding`, PS 5.1 vs 7+ differences, CJK specifics | Writing source files, debugging garbled output, or choosing the right write API |
