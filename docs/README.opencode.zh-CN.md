# Superpowers for OpenCode

在 [OpenCode.ai](https://opencode.ai) 中使用 Superpowers 的完整指南。

## 安装

将 superpowers 添加到你的 `opencode.json`（全局或项目级）中的 `plugin` 数组：

```json
{
  "plugin": ["superpowers@git+https://github.com/obra/superpowers.git"]
}
```

重启 OpenCode。插件会通过 OpenCode 的插件管理器安装，并注册所有技能。

通过提问来验证："Tell me about your superpowers"

OpenCode 使用自己的插件安装。如果你同时使用 Claude Code、Codex 或其他宿主环境，请为每个宿主环境分别安装 Superpowers。

### 从旧的基于符号链接的安装迁移

如果你之前使用 `git clone` 和符号链接安装了 superpowers，请移除旧的设置：

```bash
# 移除旧的符号链接
rm -f ~/.config/opencode/plugins/superpowers.js
rm -rf ~/.config/opencode/skills/superpowers

# 可选：移除克隆的仓库
rm -rf ~/.config/opencode/superpowers

# 如果你为 superpowers 添加过 skills.paths，请将其从 opencode.json 中移除
```

然后按照上面的安装步骤操作。

## 用法

### 查找技能

使用 OpenCode 的原生 `skill` 工具列出所有可用的技能：

```
use skill tool to list skills
```

### 加载技能

```
use skill tool to load brainstorming
```

### 个人技能

在 `~/.config/opencode/skills/` 中创建你自己的技能：

```bash
mkdir -p ~/.config/opencode/skills/my-skill
```

创建 `~/.config/opencode/skills/my-skill/SKILL.md`：

```markdown
---
name: my-skill
description: Use when [condition] - [what it does]
---

# My Skill

[Your skill content here]
```

### 项目技能

在你的项目内的 `.opencode/skills/` 中创建项目特定技能。

**技能优先级：** 项目技能 > 个人技能 > Superpowers 技能

## 更新

OpenCode 通过基于 git 的包规范安装 Superpowers。某些 OpenCode 和 Bun 版本会在锁文件或缓存中固定已解析的 git 依赖，因此重启可能不会拾取最新的 Superpowers 提交。如果更新没有出现，请清除 OpenCode 的包缓存或重新安装插件。

要固定特定版本，请使用分支或标签：

```json
{
  "plugin": ["superpowers@git+https://github.com/obra/superpowers.git#v5.0.3"]
}
```

## 它的工作原理

插件做两件事：

1. 通过 `experimental.chat.messages.transform` 钩子**注入引导上下文**，为每个对话添加 superpowers 感知。
2. 通过 `config` 钩子**注册技能目录**，使 OpenCode 无需符号链接或手动配置即可发现所有 superpowers 技能。

### 工具映射

技能以动作来表述，而不是命名任何一个运行时的工具。在 OpenCode 上，它们解析为：

- "Create a todo" / "mark complete in todo list" → `todowrite`
- `Subagent (general-purpose):` 模板 → OpenCode 的 `task` 工具，带 `subagent_type: "general"`（对于代码库探索则用 `"explore"`）
- "Invoke a skill" → OpenCode 的原生 `skill` 工具
- "Read a file" → `read`
- "Create a file" / "edit a file" / "delete a file" → `apply_patch`
- "Run a shell command" → `bash`
- "Search file contents" / "find files by name" → `grep`、`glob`
- "Fetch a URL" → `webfetch`

（已对照所安装的 OpenCode CLI 的工具清单进行验证。）

## 故障排查

### 插件未加载

1. 检查 OpenCode 日志：`opencode run --print-logs "hello" 2>&1 | grep -i superpowers`
2. 验证你的 `opencode.json` 中的插件行是否正确
3. 确保你运行的是较新版本的 OpenCode

### Windows 安装问题

某些 Windows 版 OpenCode 构建在 git 支持的插件规范上存在上游安装程序问题，包括 `git+https` URL 的缓存路径，以及即使 Bun 在普通终端中正常工作也无法找到 `git.exe`。如果 OpenCode 无法安装该插件，请尝试使用系统 npm 安装，并将 OpenCode 指向本地包：

```powershell
npm install superpowers@git+https://github.com/obra/superpowers.git --prefix "$HOME\.config\opencode"
```

然后在 `opencode.json` 中使用已安装的包路径：

```json
{
  "plugin": ["~/.config/opencode/node_modules/superpowers"]
}
```

### 找不到技能

1. 使用 OpenCode 的 `skill` 工具列出可用技能
2. 检查插件是否在加载（见上文）
3. 每个技能都需要一个带有有效 YAML frontmatter 的 `SKILL.md` 文件

### 引导程序未出现

1. 检查 OpenCode 版本是否支持 `experimental.chat.messages.transform` 钩子
2. 配置更改后重启 OpenCode

## 获取帮助

- 报告问题：https://github.com/obra/superpowers/issues
- 主文档：https://github.com/obra/superpowers
- OpenCode 文档：https://opencode.ai/docs/
