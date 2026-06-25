# PowerShell File Encoding Guide

Deep reference for file encoding in Windows PowerShell 5.1 and PowerShell 7+. Load this when writing files that will be read by compilers, git, JSON/XML parsers, or cross-platform tools.

## Encoding Matrix by Cmdlet

### Windows PowerShell 5.1

| Cmdlet | Default Encoding | BOM | Notes |
|--------|:---------------:|:---:|-------|
| `Out-File` | UTF-16 LE | Yes | `-Encoding utf8` adds UTF-8 BOM |
| `Out-File -Encoding utf8` | UTF-8 | **Yes** | The `utf8` alias means "UTF-8 with BOM" in PS 5.1 |
| `Set-Content` | ANSI (system code page) | No | Chinese Windows: GBK/CP936 |
| `Set-Content -Encoding utf8` | UTF-8 | **Yes** | Same BOM issue |
| `Add-Content` | ANSI (system code page) | No | Inherits from existing file if BOM present |
| `Get-Content` | BOM-detect → ANSI fallback | — | Can misdetect UTF-8 without BOM as ANSI |
| `>` redirect | UTF-16 LE | Yes | Identical to `Out-File` with no `-Encoding` |
| `>>` redirect | UTF-16 LE | Yes | Appends in UTF-16 LE regardless of existing encoding |

### PowerShell 7+ (Core)

| Cmdlet | Default Encoding | BOM |
|--------|:---------------:|:---:|
| `Out-File` | UTF-8 | No |
| `Set-Content` | UTF-8 | No |
| `Add-Content` | UTF-8 | No |
| `>` / `>>` | UTF-8 | No |

### The `utf8` vs `utf8NoBOM` Trap

In PS 5.1:

- `-Encoding utf8` → UTF-8 **with** BOM (bytes `EF BB BF`)
- `-Encoding utf8NoBOM` → does NOT exist in PS 5.1
- The only way to get UTF-8 without BOM is the .NET API

In PS 7+:

- `-Encoding utf8` → UTF-8 **without** BOM
- `-Encoding utf8BOM` → UTF-8 with BOM (new in PS 7+)

## BOM-Free UTF-8: The Reliable Pattern

```powershell
# Works on PS 5.1 and PS 7+: always produces UTF-8 without BOM
[System.IO.File]::WriteAllText(
    $path,
    $content,
    [System.Text.UTF8Encoding]::new($false)   # $false = no BOM
)

# For appending:
[System.IO.File]::AppendAllText(
    $path,
    $content,
    [System.Text.UTF8Encoding]::new($false)
)
```

**When to use BOM-free UTF-8**:

- `.c`, `.h` — GCC, Clang, ARMCLANG choke on BOM
- `.json` — many parsers reject BOM before `{`
- `.yml`, `.yaml` — YAML spec says BOM is allowed but many tools reject it
- `.md` — GitHub and most renderers handle BOM, but some linters flag it
- `.py` — Python 3 accepts UTF-8 BOM but linters may warn

**When BOM is acceptable or expected**:

- `.ps1` — PowerShell itself handles BOM correctly
- `.cs` — Visual Studio and Roslyn accept BOM
- `.xml` — BOM is valid in XML per spec

## $OutputEncoding — The Pipeline Gateway

`$OutputEncoding` controls how PowerShell encodes text when sending it to a native executable's stdin via the pipeline:

```powershell
# BAD: PS 5.1 default $OutputEncoding is ASCII — non-ASCII chars get mangled
$OutputEncoding  # → System.Text.ASCIIEncoding

# GOOD: set to UTF-8 without BOM before piping to native tools
$OutputEncoding = [System.Text.UTF8Encoding]::new($false)
```

This is distinct from `[Console]::OutputEncoding` which controls the console display encoding.

## Symptom Diagnosis Reference

| Symptom | Typical Cause | Fix |
|---------|--------------|-----|
| Chinese/Japanese/Korean text as `?`, `??`, mojibake | File written as ANSI (`Set-Content` without `-Encoding`) | Use `-Encoding utf8` or .NET UTF8Encoding |
| GCC reports "stray \\357 in program" | UTF-8 BOM (`EF BB BF`) in `.c`/`.h` file | Write with `UTF8Encoding::new($false)` |
| GCC reports "stray \\377 in program" | UTF-16 LE BOM (`FF FE`) in source | Never use `>` redirect or bare `Out-File` for source |
| `git diff` shows "Binary files differ" for text | File is UTF-16 LE (git sees 0x00 bytes and treats as binary) | Re-write as UTF-8; PS 5.1 `>` produces UTF-16 |
| JSON parser error on line 1 char 1 | BOM before `{` or `[` | Strip BOM or write BOM-free |
| `ConvertFrom-Json` fails on valid JSON | UTF-8 with BOM in PS 5.1 | Use `Get-Content -Encoding utf8` then `-replace '^\xEF\xBB\xBF'` |
| File size ~2x larger than expected | UTF-16 LE uses 2 bytes per ASCII char | Use UTF-8 for ASCII-heavy files |
| `Out-File` output has extra empty line at end | `Out-File` appends trailing newline | Use `Set-Content` or `[System.IO.File]::WriteAllText` for exact content |

## CJK Character Encoding in PS 5.1

Chinese (Simplified/Traditional), Japanese, and Korean characters are a special concern on PS 5.1 because:

1. **ANSI default means system code page**: On Chinese Windows, `Set-Content` without `-Encoding` writes GBK/CP936 — not UTF-8. Files are not portable to other systems.
2. **`>` redirect produces UTF-16 LE**: CJK characters are preserved but the file is twice the size and incompatible with POSIX tools.
3. **`Get-Content` without `-Encoding` misdetects**: UTF-8 files without BOM may be read as ANSI, mangling CJK characters that decode differently in GBK vs UTF-8.

**Rule for CJK content on PS 5.1**: Always explicit encoding on both read and write. Never rely on defaults.

## Cross-Platform Considerations

- **Windows → WSL/Linux**: Write as UTF-8 without BOM. Avoid UTF-16 entirely.
- **Windows → macOS**: Same as above. macOS tools expect UTF-8.
- **PS 5.1 → PS 7+**: PS 7+ defaults to UTF-8 without BOM. Code written for PS 5.1 may need BOM stripping when run on PS 7+.
- **git on Windows**: `git` can be configured to handle encoding conversion (`core.autocrlf`, `i18n.commitEncoding`). But the safest approach is to commit UTF-8 without BOM and let git handle line endings, not encoding.
