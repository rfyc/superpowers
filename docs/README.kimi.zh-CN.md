# Superpowers for Kimi Code

在 [Kimi Code](https://github.com/MoonshotAI/kimi-code) 中使用 Superpowers 的完整指南。

## 安装

Superpowers 在 Kimi Code 的插件市场中可用。

打开插件管理器：

```text
/plugins
```

转到 `Marketplace` > `Superpowers` 并安装它。

你也可以从本仓库安装：

```text
/plugins install https://github.com/obra/superpowers
```

要针对 `dev` 进行未发布的验证，请显式固定分支：

```text
/plugins install https://github.com/obra/superpowers/tree/dev
```

Kimi Code 会将插件改动应用到新会话。在安装、更新、启用、禁用或重新加载插件之后，使用 `/new` 开始一个新会话。

## 它的工作原理

Kimi 插件清单位于 `.kimi-plugin/plugin.json`。

该清单做三件事：

1. 将 Kimi Code 指向现有的 `skills/` 目录。
2. 通过 `sessionStart.skill` 在会话开始时加载 `using-superpowers`。
3. 通过 `skillInstructions` 提供 Kimi 特定的工具映射。

Kimi Code 从本仓库读取 Superpowers 技能。这里没有复制的技能、符号链接、钩子或额外的运行时依赖。

## 工具映射

技能描述的是动作，而不是硬编码某个运行时（runtime）的工具名。在 Kimi Code 上，它们解析为：

- "Ask the user" / "ask clarifying questions" -> `AskUserQuestion`
- "Create a todo" / "mark complete in todo list" -> `TodoList`
- "Dispatch a subagent" -> `Agent`
- "Invoke a skill" -> Kimi Code 的原生 `Skill` 工具
- "Read a file" / "write a file" / "edit a file" -> `Read`、`Write`、`Edit`
- "Run a shell command" -> `Bash`
- "Search file contents" -> `Grep`
- "Find files by path or pattern" -> `Glob`
- "Fetch a URL" -> `FetchURL`
- "Search the web" -> `WebSearch`

## 更新

使用 Kimi Code 的插件管理器：

```text
/plugins
```

选择 Superpowers 并从那里更新它。更新后使用 `/new` 开始一个新会话。

## 故障排查

### 插件未加载

1. 运行 `/plugins info superpowers` 并检查诊断信息。
2. 确保插件已启用。
3. 在安装或更新后使用 `/new` 开始一个新会话。

### 直接 GitHub 安装使用了旧版本

当存在发布版本时，Kimi Code 会对裸仓库 URL 安装最新的 GitHub 发布版。要在下一个 Superpowers 发布之前测试未发布的改动，请显式安装分支：

```text
/plugins install https://github.com/obra/superpowers/tree/dev
```

### 技能未触发

1. 确认 `/plugins info superpowers` 显示插件已启用。
2. 使用 `/new` 开始一个新会话。
3. 尝试验收提示词：`Let's make a react todo list`。一个可用的安装应该在写代码之前加载 `brainstorming`。
