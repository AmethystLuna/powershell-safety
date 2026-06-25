# PowerShell Safety

> AI 编程助手的 PowerShell 安全规则 — 乱码检测、文件编码陷阱、BOM 处理、引号规范、破坏性命令防护。

**跨平台** — 支持 Claude Code、Codex CLI、Cursor、Kimi CLI、OpenCode、ZCode。基于 [Agent Skills](https://agentskills.io) 开放标准构建。

## 为什么需要

AI 编程助手每天都在 Windows 上生成 PowerShell 命令。最常见的失败不是逻辑错误——而是编码不匹配导致源文件损坏、管道到原生 exe 时的字符集问题、以及破坏性通配符命令。这个插件在这些命令执行前就防止这些失误。

## 内容

### 技能（1 个）

| 技能 | 用途 |
|------|------|
| `powershell-safety` | 乱码检测、文件编码铁律、BOM 处理、引号规范、原生 exe 管道、破坏性命令安全 |

### 深度参考

该技能包含一份深度参考文档：

| 参考 | 主题 |
|------|------|
| `encoding-guide.md` | 完整编码矩阵（PS 5.1 vs 7+）、BOM 诊断、`$OutputEncoding`、CJK 专项 |

## 快速示例

```powershell
# 错误：文件被写为 UTF-16 LE — GCC 报错，git 识别为二进制
gcc -E config.h > config.i

# 正确：显式 UTF-8 不带 BOM
[System.IO.File]::WriteAllText("config.i", $content, [System.Text.UTF8Encoding]::new($false))
```

```powershell
# 错误：原生 exe 返回值被忽略
& "$keil\UV4\UV4.exe" -b project.uvprojx -o build.log

# 正确：捕获返回值
$p = Start-Process -FilePath "$keil\UV4\UV4.exe" -ArgumentList "-b project.uvprojx -o build.log" -Wait -PassThru -NoNewWindow
if ($p.ExitCode -ne 0) { throw "编译失败" }
```

## 技能加载方式

插件在会话启动时自动注入能力通知。技能按需加载：

- `Skill("powershell-safety")` — 加载核心安全规则
- 技能在 shell 输出或文件内容中出现乱码、mojibake 或编码异常时自动激活
- `encoding-guide.md` 参考文档在写源文件或诊断编码问题时加载

## 安装

### Claude Code

```bash
claude plugin install powershell-safety@AmethystLuna/powershell-safety
```

或手动：克隆到 `~/.claude/plugins/dev/powershell-safety/`。

### Codex CLI

在 `codex.json` 中添加：

```json
{
  "plugins": ["powershell-safety@git+https://github.com/AmethystLuna/powershell-safety.git"]
}
```

### Cursor

将插件目录复制到 Cursor 的插件路径。技能遵循 Agent Skills 标准，从标准路径自动发现。

### Kimi CLI

Kimi CLI 自动从 `.claude/skills/` 等标准路径发现技能。`.kimi-plugin/plugin.json` 为 Kimi 插件管理器注册插件。

### OpenCode

在 `opencode.json` 中添加：

```json
{
  "plugin": ["powershell-safety@git+https://github.com/AmethystLuna/powershell-safety.git"]
}
```

详见 `.opencode/INSTALL.md`。

### ZCode

手动安装 — 复制技能到 `.zcode/skills/`。详见 `.zcode/INSTALL.md`。

## 其他插件推荐

| 插件 | 简介 |
|------|------|
| [embedded-workbench](https://github.com/AmethystLuna/embedded-workbench) | 嵌入式 C/C++ 开发工具箱——FreeRTOS、Keil MDK、ARMCLANG、HardFault、状态机、LVGL |

## 许可证

MIT © 2026 Amethyst Luna
