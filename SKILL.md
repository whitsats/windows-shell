---
name: windows-shell
description: 在 Windows 上正确运行 PowerShell/cmd 命令，避免 Git Bash/PowerShell 双层转义错误。当需要执行 PowerShell、调用 Windows 专属功能（注册表、服务、计划任务、WMI/CIM、环境变量、MSI）、编写或运行 .ps1 脚本、UAC 提权（RunAs）、遇到 MSYS 路径转换或嵌套引号报错时使用。
---

# Windows 命令执行规则（bash ↔ PowerShell 双层环境）

## 0. 先分流：这件事根本该不该用 PowerShell

纯文件操作（增删改查、移动、打包）、grep/sed 类文本处理、curl 请求、git 操作——全部优先用 bash + POSIX 工具，更快也更少坑。
PowerShell 只留给它独占的能力：**注册表、Windows 服务、计划任务、WMI/CIM、Windows 环境变量、MSI/Windows Update、ACL/提权**。

## 1. 为什么总是转义错误

命令要过两层解析：bash 先解析一遍，PowerShell 再解析一遍。`$`、`` ` ``、`"`、`'` 在两层里都有含义，手工嵌套转义几乎必错。

总原则：**永远不要手工写两层以上的转义**。按下面的优先级选方案。

## 2. 第一选择：写 .ps1 文件再执行（消灭 90% 的转义问题）

用文件编辑工具创建 .ps1（内容零转义、可读、可复用），然后：

```bash
powershell -NoProfile -ExecutionPolicy Bypass -File ./script.ps1
# 本机有 pwsh 7 时优先：
pwsh -NoProfile -File ./script.ps1
```

适用：任何超过一行的逻辑、含 `$` 或引号的任务、需要复用的任务。临时脚本放临时目录，用完删除。

脚本文件本身的坑：

- **编码铁律**：PS 5.1 把无 BOM 的 UTF-8 .ps1 按 ANSI/GBK 解析，脚本里的中文（路径/文件名/字符串）会乱码，导致"文件找不到"或语法错误。对策：**脚本内容全用英文**；必须写中文时存成 UTF-8 with BOM。
- **开头加 `$ErrorActionPreference = 'Stop'`**：`$LASTEXITCODE` 只对原生命令生效，cmdlet 出错不设退出码，不加这句失败会静默继续。
- 结尾透传退出码：跑过原生命令后 `exit $LASTEXITCODE`，否则失败被吞。

## 3. 第二选择：内联命令的引号铁律

外层（bash）用单引号包住整条命令，让 bash 完全不碰内容：

```bash
powershell -NoProfile -Command 'Get-ChildItem "C:\Program Files" | Select-Object -First 3'
```

内层（PowerShell）规则：

| 场景 | 写法 |
|---|---|
| 字符串默认用单引号（字面量） | `'字面 $100 不插值'` |
| 需要插值才用双引号 | `"用户是 $env:USERNAME"` |
| PS 单引号里再要一个单引号 | `''`（写成两个） |
| PS 双引号里的字面 `$` | `` `$ ``（反引号转义） |
| 含空格的路径 | 必须加引号；PS 里 `/` 和 `\` 都能用 |

外层（bash）的限制：

- bash 单引号里**放不了单引号**。内层必须要单引号时用 `'...'\''...'` 拼接，或直接升级为 .ps1。
- Windows 原生命令的斜杠开关（`reg query /v`、`msiexec /x{GUID}`、`schtasks /run`）会被 Git Bash 当路径转换改写，且常是**静默失败**（如 `reg query` 返回空）。内联时前缀 `MSYS_NO_PATHCONV=1`，或走 .ps1。

需要管理员时（服务/计划任务/Program Files/ACL）：写 .ps1 + `Start-Process powershell -Verb RunAs -ArgumentList '-File','xxx.ps1' -Wait`，**先提醒用户点 UAC 弹窗**，脚本里 `Start-Transcript` 写日志，结束后读日志验证结果。

## 4. 版本：pwsh 7 与 5.1 的行为差异

优先 `pwsh`（`command -v pwsh` 检测），不存在再回退 `powershell.exe`（5.1）。**检测不到 pwsh 时，下面这些 5.1 的坑全部生效**：

| 行为 | pwsh 7 | Windows PowerShell 5.1 |
|---|---|---|
| `>` 重定向 / Out-File 默认编码 | UTF-8 无 BOM | **UTF-16 LE** |
| 无 BOM UTF-8 的 .ps1 | 按 UTF-8 解析 | **按 ANSI/GBK 解析**（含中文必炸） |
| 原生命令参数引号 | 7.3+ `PSNativeCommandArgumentPassing` 默认已修好 | 手拼必错，用 `--%` 或数组传参 |

## 5. 强制规则

- **中文输出**先设编码：`[Console]::OutputEncoding = [System.Text.Encoding]::UTF8`；文件读写显式 `-Encoding UTF8`（5.1 的 `>` 重定向写出的是 UTF-16）。
- **丢弃输出**：PS 里用 `> $null` 或 `| Out-Null`。`> nul` 只属于 cmd（设备重定向，安全）；PS 5.1 里 `> nul` 直接报错；**bash 里 `> nul` 会真的创建名为 nul 的文件**。
- **对象管道，不要文本解析**：`Get-Process | Where-Object CPU -gt 100` 而不是 `tasklist | findstr`。需要机器可读结果时输出 JSON：`... | ConvertTo-Json -Depth 3`（默认 Depth=2，嵌套必丢，务必显式给）。
- **给原生命令传复杂参数**：用 `--%` 停止解析符（主要剩 5.1 需要）或 `Start-Process -ArgumentList`（数组，免转义），不要手拼引号。
- 禁止：内联命令里手写两层以上嵌套引号；双引号字符串里出现未检查的 `$`。

## 6. 典型错误对照

```bash
# 错：bash 先把 $HOME 替换成 bash 的 HOME（/c/Users/... 形式）
powershell -Command "echo $HOME"
# 对：单引号包住，PS 自己解析 $HOME（C:\Users\... 形式）
powershell -Command 'echo $HOME'

# 错：内联 JSON——bash 双引号里的 \" 到了 PS 不是转义（PS 用 `\" 或 ""），直接断串
powershell -Command "Invoke-RestMethod -Uri 'https://example.com' -Body '{\"a\":1}'"
# 对：整体走 .ps1，或 body 放文件：-Body (Get-Content body.json -Raw)

# 错：5.1 下 > 重定向写出 UTF-16
powershell -Command 'Get-Content in.txt > out.txt'
# 对：
powershell -Command 'Get-Content in.txt -Encoding UTF8 | Set-Content out.txt -Encoding UTF8'
```
