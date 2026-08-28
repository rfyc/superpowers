# Claude Code 的跨平台多语言钩子

Claude Code 插件需要能在 Windows、macOS 和 Linux 上工作的钩子。本文档描述了 `hooks/run-hook.cmd` 中使用的单一通用派发器模式。

> **权威来源：** `hooks/run-hook.cmd` 是规范实现。当本文档与代码不一致时，以代码为准。

## 问题

Claude Code 通过 shell 运行钩子命令：
- **macOS/Linux**：bash 或 sh
- **安装了 Git Bash 的 Windows**：Git Bash
- **未安装 Git Bash 的 Windows**：PowerShell（旧版本使用 CMD.exe）

两种 Windows 回退 shell 都无法解析我们的命令字符串：PowerShell 会把开头带引号的路径当作字符串表达式，并在下一个裸词（bareword）处报错；而 CMD.exe 的 `/c` 引号规则会在路径包含元字符（如 `(`）时剥掉外层引号。因此，我们的钩子声明了 `"shell": "bash"`（自 Claude Code 2.1.81 起受支持；旧版本会忽略此键），这会强制走 Git Bash 路线，并且在缺少 Git Bash 时产生一条可操作的"为 Windows 安装 Git"错误，而不是 shell 解析失败。

这产生了几个挑战：

1. **脚本执行**：Windows CMD 无法直接执行 `.sh` 文件
2. **路径格式**：Windows 使用反斜杠（`C:\path`），Unix 使用正斜杠（`/path`）
3. **环境变量**：`$VAR` 语法在 CMD 中不起作用
4. **`.sh` 自动前置**：Claude Code 在 Windows 上会自动为任何路径中包含 `.sh` 的命令前置 `bash` —— 如果脚本带扩展名，这会干扰派发器

## 解决方案：无扩展名脚本 + 单一通用派发器

本仓库对所有钩子使用一个通用的 `run-hook.cmd` 派发器。钩子脚本是**无扩展名**的（`session-start`，而不是 `session-start.sh`）。这是刻意为之：它可以防止 Claude Code 的 Windows 自动检测为派发器命令前置 `bash` 而破坏它。

### 文件结构

```
hooks/
├── hooks.json          # 用无扩展名的脚本名指向 run-hook.cmd
├── run-hook.cmd        # 跨平台派发器（多语言包装器）
└── session-start       # 实际的钩子逻辑 —— 无扩展名的 bash 脚本
```

### hooks.json

```json
{
  "hooks": {
    "SessionStart": [
      {
        "matcher": "startup|clear|compact",
        "hooks": [
          {
            "type": "command",
            "command": "\"${CLAUDE_PLUGIN_ROOT}/hooks/run-hook.cmd\" session-start",
            "shell": "bash",
            "async": false
          }
        ]
      }
    ]
  }
}
```

路径被加引号是因为 `${CLAUDE_PLUGIN_ROOT}` 可能包含空格。

## `run-hook.cmd` 在高层级上如何工作

`run-hook.cmd` 是一个多语言脚本：Windows 把第一个块当作批处理命令，而 Unix shell 把这个块当作无操作 heredoc，并在其后继续执行。

不要从本文档复制实现。在修改派发器时直接阅读 `hooks/run-hook.cmd`，并在此后运行 `tests/hooks/test-session-start.sh`。

### 它在 Windows（CMD.exe）上如何工作

1. 批处理部分验证脚本名，并从派发器自身的位置解析钩子目录。
2. 它会在三个位置尝试 bash：
   - `C:\Program Files\Git\bin\bash.exe`
   - `C:\Program Files (x86)\Git\bin\bash.exe`
   - `PATH` 上的 `bash`（MSYS2、Cygwin 或非默认位置的 Git 安装）
3. 如果找到 bash，它就运行钩子目录中指定的无扩展名钩子脚本。
4. 如果找不到 bash，派发器会静默地以 `0` 退出 —— 插件继续工作，只是跳过钩子。
5. `exit /b` 会在到达 Unix 部分之前停止 CMD。

### 它在 Unix（bash/sh）上如何工作

1. `: << 'CMDBLOCK'` 在一个无操作命令上打开 heredoc。
2. 整个 CMD 批处理块被 heredoc 消费并忽略。
3. 在 `CMDBLOCK` 之后，bash 解析脚本目录，并直接 `exec` 指定的无扩展名脚本。

### 关键设计决策

| 决策 | 原因 |
|----------|-----|
| 无扩展名脚本 | 防止 Claude Code 在 Windows 上的 `.sh` 自动前置干扰派发器命令 |
| 不使用 `-l`（登录 shell） | 不需要；钩子脚本应该是自包含的，不依赖登录 shell 的 PATH 设置 |
| 不使用 `cygpath` | bash 直接接收 Windows 路径并正确处理它；`cygpath` 是老式 `-c "..."` 调用模式所需的，直接 exec 不需要 |
| 无 bash 时静默退出 | 避免为没有 Git for Windows 的用户破坏插件；钩子上下文注入被优雅地跳过 |

## 编写跨平台钩子脚本

你的钩子逻辑放在无扩展名的脚本文件中。一些可移植的模式：

### 要做
- 尽可能使用纯 bash 内建命令
- 使用 `$(command)` 而不是反引号
- 对所有变量展开加引号：`"$VAR"`

### 要避免
- 依赖 PATH 相关工具而没有回退（钩子不带 `-l` 运行，所以登录 shell 的 PATH 不会设置）
- 给脚本加 `.sh` 扩展名 —— 这会触发 Claude Code 在 Windows 上的自动前置

### 示例：不使用外部工具进行 JSON 转义

```bash
escape_for_json() {
    local input="$1"
    local output=""
    local i char
    for (( i=0; i<${#input}; i++ )); do
        char="${input:$i:1}"
        case "$char" in
            $'\\') output+='\\' ;;
            '"') output+='\"' ;;
            $'\n') output+='\n' ;;
            $'\r') output+='\r' ;;
            $'\t') output+='\t' ;;
            *) output+="$char" ;;
        esac
    done
    printf '%s' "$output"
}
```

## 故障排查

### "bash is not recognized"

CMD 无法在派发器尝试的三个位置中的任何一个找到 bash。派发器会静默退出（0）而不是报错，所以钩子被跳过。请在标准路径安装 Git for Windows，或确保 `bash` 在 `PATH` 上。

### 钩子在 Unix 上运行但在 Windows 上什么也不做

检查脚本文件名在 `hooks.json` 中是否是**无扩展名**的。像 `run-hook.cmd session-start.sh` 这样的命令可能触发 Claude Code 的 `.sh` 自动检测并绕过预期的 CMD 派发器路径，或者只是试图运行一个不存在的 `session-start.sh` 脚本。

### 钩子完全不触发

验证 `hooks.json` 中的 `matcher` 与你宿主环境发出的事件类型匹配。Claude Code 使用 `startup|clear|compact`；Cursor 使用 `sessionStart`。Cursor 变体请检查 `hooks-cursor.json`。

## 相关问题

- [anthropics/claude-code#9758](https://github.com/anthropics/claude-code/issues/9758) —— Windows 上 `.sh` 脚本在编辑器中打开
- [anthropics/claude-code#3417](https://github.com/anthropics/claude-code/issues/3417) —— 钩子在 Windows 上不起作用
