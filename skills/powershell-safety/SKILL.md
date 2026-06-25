---
name: powershell-safety
description: "Use when you observe garbled characters, mojibake, or encoding artifacts in any shell output, file content, or tool result — BEFORE generating any PowerShell file-write or pipe, load this skill. Also use when writing files via PowerShell consumed by other tools (compilers, git, parsers), or when generating destructive/wide-scope PowerShell commands. NOT needed for read-only commands (grep, git status, dotnet build, markdownlint)."
---

# PowerShell Safety

## When This Skill Applies — Garbled Text Detection

If you observe ANY of these in shell output, file content, or tool results, this skill is relevant and **must be loaded BEFORE generating any file-write or pipe command**:

- CJK/Unicode characters displayed as `?`, `??`, `□□`, or mojibake (e.g. `ç»´æŠ¤`, `æ›´æ–°` — UTF-8 bytes decoded as ANSI)
- Compiler errors: "stray `\357`", "stray `\377`", "stray `\376`" in source files
- Parser errors on line 1 char 1 of JSON/XML (BOM before `{` / `<`)
- `git diff` shows "Binary files differ" for known text files
- File content with `NUL` bytes between every ASCII character (UTF-16 LE signature)
- File size ~2× larger than expected for the content length
- `Get-Content` returns garbled CJK when the file is known to contain Chinese/Japanese/Korean (PS 5.1 misdetected UTF-8 no-BOM as ANSI)
- Native executable receives `?` instead of CJK characters via pipe (`$OutputEncoding` is ASCII)
- `ConvertFrom-Json` throws on valid-looking JSON (BOM before `{`)
- `ConvertTo-Json` produces `\uXXXX` escape sequences for CJK characters (PS 5.1 default)
- Console displays `□` (white squares) for CJK text — distinguish: if only SOME characters show `□` it's a font issue (use MS Gothic / Windows Terminal); if ALL show `□` or `?` it's encoding
- `[System.IO.File]::WriteAllText` with `[System.Text.Encoding]::UTF8` still produces BOM in output (`.NET Framework` includes BOM by default — must use `new UTF8Encoding($false)`)

## Encoding — Iron Rules

These are non-negotiable. Breaking any of them causes bugs that are hard to diagnose. For the full encoding matrix, BOM diagnostics, and cmdlet-specific defaults, load `references/encoding-guide.md`.

1. **Always `-Encoding utf8`** on `Out-File`, `Set-Content`, `Add-Content`. On PS 5.1 this produces UTF-8 **with** BOM — source files (`.c`, `.h`, `.py`) need BOM-free. Use `[System.IO.File]::WriteAllText($path, $content, [System.Text.UTF8Encoding]::new($false))` instead.

2. **Never use `>` or `>>` redirect on PS 5.1** for files read by non-PowerShell tools. These produce UTF-16 LE. Use `Out-File -Encoding utf8` or `Set-Content -Encoding utf8` instead.

3. **Set `$OutputEncoding`** when piping text to native executables:

   ```powershell
   $OutputEncoding = [System.Text.UTF8Encoding]::new($false)
   ```

   `$OutputEncoding` controls stdin encoding to native exe. Separate from `[Console]::OutputEncoding` (controls how native exe **stdout** bytes are decoded for console display — if stdout looks garbled even though the file is UTF-8, check this).

4. **Specify encoding on read**: `Get-Content -Encoding utf8` for UTF-8, `Get-Content -Encoding Unicode` for UTF-16 LE. Don't rely on auto-detection in PS 5.1. Using the wrong `-Encoding` (e.g. `-Encoding utf8` on a UTF-16 LE file) produces garbled output.

5. **For UTF-8 without BOM on PS 5.1**, use the .NET API directly. `[System.Text.Encoding]::UTF8` **is NOT enough** — it includes BOM on .NET Framework. Must use:

   ```powershell
   [System.IO.File]::WriteAllText($path, $content, [System.Text.UTF8Encoding]::new($false))
   ```

6. **`ConvertFrom-Json` / `ConvertTo-Json` encoding traps**:
   - `ConvertFrom-Json` fails if content has BOM. Strip it first: `$c = Get-Content $p -Encoding utf8 -Raw; $c = $c -replace '^\xEF\xBB\xBF'; $c | ConvertFrom-Json`
   - `ConvertTo-Json` in PS 5.1 escapes non-ASCII chars as `\uXXXX` (valid JSON, but unreadable). PS 7+ outputs raw UTF-8. To preserve CJK on PS 5.1, write the JSON manually with .NET APIs.
   - `ConvertTo-Json | Out-File` → double trap: `\uXXXX` escaping + UTF-16 LE output.

### Encoding Defaults by Cmdlet (PS 5.1)

| Cmdlet | Default | BOM? |
|--------|:-------:|:----:|
| `Out-File` | UTF-16 LE | Yes |
| `Set-Content` | ANSI (system code page) | No |
| `Add-Content` | ANSI (system code page) | No |
| `>` / `>>` redirect | UTF-16 LE | Yes |
| `Start-Process -RedirectStandardOutput` | UTF-16 LE | Yes |
| `Start-Process -RedirectStandardError` | UTF-16 LE | Yes |

### Quick Symptom → Fix

Match your symptom in the left column, apply the fix directly:

| Symptom | Root Cause | Quick Fix |
|---------|-----------|-----------|
| CJK text as `?` or `??` in file | `Set-Content` without `-Encoding` wrote as ANSI | `Set-Content -Path $p -Value $c -Encoding utf8` |
| CJK text as mojibake (`ç»´æŠ¤`, `æ›´æ–°`) when reading | `Get-Content` misdetected UTF-8 no-BOM as ANSI | `Get-Content -Encoding utf8 $path` |
| GCC: "stray \\357 in program" | UTF-8 BOM in source file | `[System.IO.File]::WriteAllText($p, $c, [System.Text.UTF8Encoding]::new($false))` |
| GCC: "stray \\377 in program" | UTF-16 LE BOM in source | `$c = Get-Content $p -Encoding Unicode -Raw; [System.IO.File]::WriteAllText($p, $c, [System.Text.UTF8Encoding]::new($false))` |
| JSON/XML parser fails line 1 char 1 | BOM before `{`/`<` | `$c = Get-Content $p -Encoding utf8 -Raw; $c = $c -replace '^\xEF\xBB\xBF'; [System.IO.File]::WriteAllText($p, $c, [System.Text.UTF8Encoding]::new($false))` |
| `git diff` shows "Binary files differ" | File is UTF-16 LE | Same as "stray \\377" row above — re-write as UTF-8 |
| File size ~2× expected | UTF-16 LE uses 2 bytes/char | Same as above — re-write as UTF-8 |
| Native exe receives `?` for CJK via pipe | `$OutputEncoding` is ASCII (PS 5.1 default) | `$OutputEncoding = [System.Text.UTF8Encoding]::new($false)` before the pipe |
| Native exe stdout shows garbled CJK | `[Console]::OutputEncoding` mismatch | `[Console]::OutputEncoding = [System.Text.UTF8Encoding]::new($false)` |
| `ConvertFrom-Json` throws on valid JSON | BOM before `{` | `$c = Get-Content $p -Encoding utf8 -Raw; $c = $c -replace '^\xEF\xBB\xBF'; $obj = $c \| ConvertFrom-Json` |
| `ConvertTo-Json` escapes CJK as `\uXXXX` | PS 5.1 default — valid JSON, but unreadable | PS 7+: use `-EscapeHandling EscapeNonAscii`. PS 5.1: accept `\uXXXX` or write JSON manually with .NET |
| `Start-Process -RedirectStandardOutput` writes garbled file | `-RedirectStandard*` produces UTF-16 LE on PS 5.1 (same as `Out-File`) | Use `-NoNewWindow` + capture stdout in variable, then write with .NET API |
| `[System.IO.File]::WriteAllText` with `[Text.Encoding]::UTF8` still has BOM | `Encoding.UTF8` includes BOM in .NET Framework | Use `[System.Text.UTF8Encoding]::new($false)` — `$false` = no BOM |
| Console displays `□` (white squares) for CJK | Console font lacks CJK glyphs (NOT an encoding issue) | Switch to Windows Terminal, or set conhost font to "NSimSun" / "MS Gothic" |

### Heuristic: "Is it encoding or is it font?"

- All CJK chars are `?` or `□□` → **encoding** (wrong code page or missing `-Encoding`)
- Only some CJK chars are `□`, others display correctly → **font** (glyph missing in console font)
- ASCII chars display fine but CJK is mojibake → **encoding** (UTF-8 bytes interpreted as ANSI)
- File looks fine in VS Code but garbled in console → **console rendering** (font or `chcp`)

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

- **Avoid `-RedirectStandardOutput` / `-RedirectStandardError` on PS 5.1** — these write output files as UTF-16 LE (same as `Out-File`). If you need to capture output, use `-NoNewWindow` and assign to a variable, or redirect inside the `-ArgumentList` string using the target executable's own redirection.
- Check `$LASTEXITCODE` for native tools. `$?` alone is not enough.
- Avoid `2>&1` on native exe in PS 5.1 — it wraps stderr lines in ErrorRecord and sets `$?` to `$false` even on exit code 0. Use `Start-Process` to capture stderr cleanly.

---

## Resolution Cheat Sheet

Top 5 most common scenarios. Match and run:

### ➊ Write a C/C++ source file (`.c`, `.h`) — MUST be BOM-free

```powershell
[System.IO.File]::WriteAllText($path, $content, [System.Text.UTF8Encoding]::new($false))
```

### ➋ Write a JSON/XML/YAML config file — MUST be BOM-free

```powershell
$c = Get-Content $path -Encoding utf8 -Raw                          # read
$c = $c -replace '^\xEF\xBB\xBF'                                    # strip BOM if present
[System.IO.File]::WriteAllText($path, $c, [System.Text.UTF8Encoding]::new($false))
```

### ➌ Convert a UTF-16 LE file to UTF-8

```powershell
$c = Get-Content $path -Encoding Unicode -Raw
[System.IO.File]::WriteAllText($path, $c, [System.Text.UTF8Encoding]::new($false))
```

### ➍ Pipe CJK text to a native executable

```powershell
$OutputEncoding = [System.Text.UTF8Encoding]::new($false)
"中文内容" | native-tool.exe
[Console]::OutputEncoding = [System.Text.UTF8Encoding]::new($false)  # also fix stdout display
```

### ➎ Write general text (`.md`, `.txt`, `.ps1`) — BOM acceptable

```powershell
Set-Content -Path $path -Value $content -Encoding utf8
```

---

## Deep Reference

This skill's `references/` directory contains:

| Reference | Topic | Load When |
|-----------|-------|-----------|
| `encoding-guide.md` | Full encoding matrix, BOM handling, `$OutputEncoding`, `[Console]::OutputEncoding`, PS 5.1 vs 7+ differences, CJK specifics, `chcp` code page interaction | Writing source files, debugging garbled output, or choosing the right write API |
