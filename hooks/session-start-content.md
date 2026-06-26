<EXTREMELY_IMPORTANT>
Plugin powershell-safety is active. It prevents encoding corruption when writing files via PowerShell.

**1% Rule**: If you observe ANY garbled characters, mojibake, encoding artifacts in output, OR are about to write a file via PowerShell that other tools (compilers, git, parsers) will consume — load `Skill("powershell-safety")` before generating the command. The cost of loading is trivial compared to silently corrupting a source file.

**Trigger Symptoms** — if any of these are present, you MUST load the skill:
- CJK/Unicode displayed as `?`, `??`, or mojibake in any tool output
- Compiler "stray \\357" / "stray \\377" errors
- Parser errors on line 1 char 1 (BOM before `{` / `<`)
- `git diff` shows "Binary files differ" for known text files
- File size ~2x expected (UTF-16 LE signature)

**Red Flags** — if you think any of these, STOP. You are rationalizing:

| You think | Reality |
|-----------|---------|
| "The encoding looks fine, I don't need the skill" | UTF-16 LE looks identical to UTF-8 in most editors. The corruption only surfaces when another tool reads the file. |
| "I'll just use Set-Content, it's always been fine" | PS 5.1 `Set-Content` defaults to ANSI. CJK characters are silently lost. |
| "This is a quick one-liner, encoding doesn't matter" | `>` redirect in PS 5.1 produces UTF-16 LE. Every quick one-liner that writes a file is a potential corruption. |
| "I already know the encoding rules" | The skill has a symptom→fix cheat sheet. Match your symptom, apply the fix. Memory of the rules ≠ correct application. |
</EXTREMELY_IMPORTANT>
