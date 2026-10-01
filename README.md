# windows-shell

**Windows 命令执行规则 · Agent Skill** —— 教 POSIX 系编码代理（ZCode / Claude Code 这类默认 shell 为 Git Bash 的工具）在 Windows 上正确跑 PowerShell，告别双层转义地狱。

## 为什么需要它

Windows 上命令要过两层解析：bash 先解析一遍，PowerShell 再解析一遍。`$`、`` ` ``、`"`、`'` 在两层里都有含义，内联手写嵌套转义几乎必错——比如 `powershell -Command "echo $HOME"` 里的 `$HOME` 会被 bash 先吃掉，变成 POSIX 形式的路径；`reg query /v` 的 `/v` 会被 Git Bash 当成路径转换掉，静默返回空。

本 skill 把实战踩坑固化为一套可复用规则，让代理（而不是你）来消化这些摩擦。

## 内容一览

- **先分流**：纯文件操作 / grep / curl / git 留给 bash；PowerShell 只干它独占的活（注册表、服务、计划任务、WMI/CIM、MSI、UAC 提权）
- **第一选择：写 .ps1 再执行**（消灭 90% 的转义问题），含脚本编码铁律（5.1 把无 BOM UTF-8 按 GBK 解析，含中文必炸）、`$ErrorActionPreference = 'Stop'`、退出码透传
- **内联引号铁律**：bash 单引号外包 + PS 内层规则表 + "bash 单引号放不了单引号"的兜底
- **本机高频坑**：MSYS 路径转换静默吃 `/v` `/x` 开关、UAC 提权套路（`RunAs` + `Start-Transcript` 回读验证）、`> nul` 在 cmd / PS 5.1 / pwsh 7 / bash 的四方行为差异
- **pwsh 7 vs Windows PowerShell 5.1 差异表**：重定向编码、脚本解析编码、原生参数引号（`PSNativeCommandArgumentPassing`）

## 安装

把本仓库放入你的代理技能目录即可（目录名任意，含 `SKILL.md` 就能被索引）：

```bash
git clone https://github.com/whitsats/windows-shell.git
# ZCode / Claude Code 用户级全局技能目录：
cp -r windows-shell ~/.agents/skills/windows-shell
```

新会话自动进入技能索引；会话内可手动 `/windows-shell` 唤起。

## 适用范围

- **适用**：Windows 上以 Git Bash 为默认执行 shell 的 agent（ZCode、Claude Code 等 POSIX 系工具）
- **不需要**：执行层原生 PowerShell 的 agent（如 Codex Windows 原生版、Antigravity）本身没有这层转义坑——不过规则内容仍有参考价值

## 许可证

[MIT](LICENSE)
