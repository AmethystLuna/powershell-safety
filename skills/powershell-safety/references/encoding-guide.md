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
| `Start-Process -RedirectStandardOutput` | UTF-16 LE | Yes | Same as `Out-File` — file output, not console |
| `Start-Process -RedirectStandardError` | UTF-16 LE | Yes | Same behavior |
| `ConvertTo-Json` output | Depends on consumer | — | Escapes non-ASCII as `\uXXXX` in PS 5.1; raw UTF-8 in PS 7+ |

### PowerShell 7+ (Core)

| Cmdlet | Default Encoding | BOM |
|--------|:---------------:|:---:|
| `Out-File` | UTF-8 | No |
| `Set-Content` | UTF-8 | No |
| `Add-Content` | UTF-8 | No |
| `>` / `>>` | UTF-8 | No |
| `Start-Process -RedirectStandardOutput` | UTF-8 | No |

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

### CRITICAL: `[Text.Encoding]::UTF8` vs `[Text.UTF8Encoding]::new($false)`

On .NET Framework (PowerShell 5.1):

- `[System.Text.Encoding]::UTF8` — includes BOM (`EF BB BF`). **Do NOT use for source files.**
- `[System.Text.UTF8Encoding]::new($false)` — no BOM. **Use this.**
- `[System.Text.UTF8Encoding]::new($true)` — with BOM. Same as `Encoding.UTF8`.

On .NET Core (PowerShell 7+), `Encoding.UTF8` does NOT include BOM by default — but the `UTF8Encoding::new($false)` pattern works on both and is safer.

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

## `$OutputEncoding` — The Pipeline Gateway

`$OutputEncoding` controls how PowerShell encodes text when sending it to a native executable's stdin via the pipeline:

```powershell
# BAD: PS 5.1 default $OutputEncoding is ASCII — non-ASCII chars get mangled
$OutputEncoding  # → System.Text.ASCIIEncoding

# GOOD: set to UTF-8 without BOM before piping to native tools
$OutputEncoding = [System.Text.UTF8Encoding]::new($false)
```

## `[Console]::OutputEncoding` — The Display Decoder

`[Console]::OutputEncoding` controls how the console decodes bytes from a native executable's **stdout** for display. This is distinct from `$OutputEncoding` (which controls stdin encoding to the native exe).

```powershell
# Check current console output encoding
[Console]::OutputEncoding  # PS 5.1 default: system code page (CP936 on Chinese Windows)

# If native exe outputs UTF-8 but console displays garbled CJK:
[Console]::OutputEncoding = [System.Text.UTF8Encoding]::new($false)
```

### The Two-Encoding Pipeline Pattern

When both sending to and receiving from a native executable:

```powershell
# Step 1: Set stdin encoding (data TO native exe)
$OutputEncoding = [System.Text.UTF8Encoding]::new($false)

# Step 2: Set stdout decoding (data FROM native exe)
[Console]::OutputEncoding = [System.Text.UTF8Encoding]::new($false)

# Step 3: Run the pipeline
"中文输入" | native-tool.exe
```

## `Start-Process -RedirectStandardOutput` Encoding

On PS 5.1, `Start-Process -RedirectStandardOutput` writes the output file as **UTF-16 LE** (same default as `Out-File`). This is especially dangerous for build tools (UV4.exe, GCC, make) whose output is consumed by parsers or log analyzers.

```powershell
# BAD: build.log is UTF-16 LE — most log parsers choke
Start-Process -FilePath "UV4.exe" -ArgumentList "-b project.uvprojx -o build.log" `
    -Wait -NoNewWindow -RedirectStandardOutput build.log

# GOOD: capture in variable, write as UTF-8
$output = & "UV4.exe" -b project.uvprojx -o build.log 2>&1
[System.IO.File]::WriteAllText("build.log", $output, [System.Text.UTF8Encoding]::new($false))
```

## `ConvertFrom-Json` / `ConvertTo-Json` Encoding

### `ConvertFrom-Json` — BOM and Encoding Issues

```powershell
# BAD: fails if data.json has UTF-8 BOM
$obj = Get-Content data.json -Raw | ConvertFrom-Json  # BOM → parser error

# GOOD: strip BOM before parsing
$c = Get-Content data.json -Encoding utf8 -Raw
$c = $c -replace '^\xEF\xBB\xBF', ''
$obj = $c | ConvertFrom-Json

# ALSO BAD: data.json is UTF-16 LE from Out-File
$obj = Get-Content data.json | ConvertFrom-Json  # content reads as garbled UTF-16 text
# Fix: re-read with correct encoding first
$c = Get-Content data.json -Encoding Unicode -Raw
[System.IO.File]::WriteAllText("data.json", $c, [System.Text.UTF8Encoding]::new($false))
```

### `ConvertTo-Json` — CJK Escaping

```powershell
# PS 5.1: CJK chars escaped as \uXXXX (valid JSON, but unreadable by humans)
$obj = @{ name = "中文" }
$obj | ConvertTo-Json
# → { "name": "中文" }

# PS 5.1 workaround: build JSON string manually for readability
$json = "{ `"name`": `"中文`" }"
[System.IO.File]::WriteAllText("data.json", $json, [System.Text.UTF8Encoding]::new($false))

# PS 7+: use -EscapeHandling
$obj | ConvertTo-Json -EscapeHandling EscapeNonAscii  # → { "name": "中文" }
$obj | ConvertTo-Json -EscapeHandling Default          # → { "name": "中文" }
```

## Code Page (`chcp`) and System Locale

The system ANSI code page determines what `Set-Content` without `-Encoding` produces:

```powershell
# Check current console code page
chcp  # e.g. 936 = GBK (Simplified Chinese), 437 = US, 65001 = UTF-8

# PS 5.1 default ANSI encoding
[System.Text.Encoding]::Default  # → GBK (CP936) on Chinese Windows
```

On Chinese Windows:

- `Set-Content $p "中文"` → writes as GBK/CP936, NOT UTF-8
- File is readable on Chinese Windows but garbled on US/Japanese Windows or Linux
- `Get-Content` on a UTF-8 no-BOM file → may misdetect as GBK → decodes incorrectly

**Rule**: Never rely on `[System.Text.Encoding]::Default` or ANSI defaults. Always specify encoding explicitly.

## Symptom Diagnosis Reference

| Symptom | Typical Cause | Fix |
|---------|--------------|-----|
| Chinese/Japanese/Korean text as `?`, `??`, mojibake | File written as ANSI (`Set-Content` without `-Encoding`) | Use `-Encoding utf8` or .NET UTF8Encoding |
| CJK as mojibake (`ç»´æŠ¤`, `æ›´æ–°`) when reading | `Get-Content` misdetected UTF-8 no-BOM as ANSI | `Get-Content -Encoding utf8 $path` |
| GCC reports "stray \\357 in program" | UTF-8 BOM (`EF BB BF`) in `.c`/`.h` file | Write with `UTF8Encoding::new($false)` |
| GCC reports "stray \\377 in program" | UTF-16 LE BOM (`FF FE`) in source | Never use `>` redirect or bare `Out-File` for source |
| `git diff` shows "Binary files differ" for text | File is UTF-16 LE (git sees 0x00 bytes and treats as binary) | Re-write as UTF-8; PS 5.1 `>` produces UTF-16 |
| JSON parser error on line 1 char 1 | BOM before `{` or `[` | Strip BOM or write BOM-free |
| `ConvertFrom-Json` fails on valid JSON | UTF-8 with BOM in PS 5.1 | Use `Get-Content -Encoding utf8` then `-replace '^\xEF\xBB\xBF'` |
| `ConvertTo-Json` produces `\uXXXX` for CJK | PS 5.1 default | Accept as valid JSON or use PS 7+ / manual JSON building |
| File size ~2x larger than expected | UTF-16 LE uses 2 bytes per ASCII char | Use UTF-8 for ASCII-heavy files |
| `Out-File` output has extra empty line at end | `Out-File` appends trailing newline | Use `Set-Content` or `[System.IO.File]::WriteAllText` for exact content |
| Native exe stdout displays garbled CJK in console | `[Console]::OutputEncoding` mismatch | `[Console]::OutputEncoding = [System.Text.UTF8Encoding]::new($false)` |
| Native exe receives `?` for CJK via pipe | `$OutputEncoding` is ASCII (PS 5.1 default) | `$OutputEncoding = [System.Text.UTF8Encoding]::new($false)` |
| `Start-Process -RedirectStandardOutput` file is UTF-16 LE | Default behavior on PS 5.1 | Use `-NoNewWindow` + variable capture + .NET write |
| `[Text.Encoding]::UTF8` still writes BOM | .NET Framework default | Use `[System.Text.UTF8Encoding]::new($false)` |
| `chcp 65001` but console still shows garbled CJK | Console font lacks CJK glyphs | Use Windows Terminal; or set conhost font to NSimSun/MS Gothic |

## CJK Character Encoding in PS 5.1

Chinese (Simplified/Traditional), Japanese, and Korean characters are a special concern on PS 5.1 because:

1. **ANSI default means system code page**: On Chinese Windows, `Set-Content` without `-Encoding` writes GBK/CP936 — not UTF-8. Files are not portable to other systems.
2. **`>` redirect produces UTF-16 LE**: CJK characters are preserved but the file is twice the size and incompatible with POSIX tools.
3. **`Get-Content` without `-Encoding` misdetects**: UTF-8 files without BOM may be read as ANSI, mangling CJK characters that decode differently in GBK vs UTF-8.
4. **`ConvertTo-Json` escapes CJK as `\uXXXX`**: In PS 5.1, `{ "name": "中文" }` becomes `{ "name": "中文" }`. Valid JSON but not human-readable.
5. **`$OutputEncoding` is ASCII by default**: Piping CJK to a native executable sends `?` for every non-ASCII character.

**Rule for CJK content on PS 5.1**: Always explicit encoding on both read and write. Never rely on defaults.

## Cross-Platform Considerations

- **Windows → WSL/Linux**: Write as UTF-8 without BOM. Avoid UTF-16 entirely.
- **Windows → macOS**: Same as above. macOS tools expect UTF-8.
- **PS 5.1 → PS 7+**: PS 7+ defaults to UTF-8 without BOM. Code written for PS 5.1 may need BOM stripping when run on PS 7+.
- **git on Windows**: `git` can be configured to handle encoding conversion (`core.autocrlf`, `i18n.commitEncoding`). But the safest approach is to commit UTF-8 without BOM and let git handle line endings, not encoding.
