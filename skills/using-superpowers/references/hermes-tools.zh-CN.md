# Hermes Agent 工具映射

技能以动作来表述（"派发子代理"、"创建待办"、"读取文件"）。在 Hermes Agent 上，这些动作对应为下列工具。

## 工具

| 技能请求的动作 | Hermes 工具 |
|---|---|
| 读取文件 | `read_file` |
| 创建新文件 | `write_file` |
| 编辑文件（定向补丁） | `patch` |
| 运行 shell 命令 | `terminal` |
| 搜索文件内容 | `search_files` |
| 按名称查找文件 | 配合 `find` 使用 `terminal` |
| 抓取 URL / 读取网页 | `web_extract(urls=[...])` |
| 搜索网络 | `web_search(query=...)` |
| 派发子代理 | `delegate_task(goal=..., context=..., toolsets=[...], role="leaf")` |
| 任务跟踪 | `todo` 工具 |
| 调用技能 | `skill_view("skill-name")` |

## 指令文件

当技能提到"你的指令文件"时，在 Hermes Agent 上它就是项目目录中的 **`AGENTS.md`**，或全局位于 `~/.hermes/SOUL.md` 的 **`SOUL.md`**。

## 调用技能

Hermes Agent 有一个 `skills` 工具集，包含 `skill_view` 和 `skills_list` 工具。
要调用 superpowers 技能，请使用：

```
skill_view("brainstorming")
skill_view("test-driven-development")
```

如果 `skill_view` 找不到 superpowers 技能（在插件完全注册它之前，它可能不会出现在目录
中），请回退为直接读取 SKILL.md：

```
read_file(path="~/.hermes/plugins/superpowers/skills/<skill-name>/SKILL.md")
```

这种回退机制与其他没有原生技能加载能力的运行环境所使用的机制相同。

## 子代理派发

使用 `delegate_task` 生成隔离的子代理，用于并行或顺序的工作流：

```
delegate_task(goal="...", context="...", toolsets=[...], role="leaf")
```

如果 `delegate_task` 不可用，请以内联方式完成工作，而不是凭空捏造工具调用。

## 任务跟踪

在会话内使用 `todo` 工具进行任务跟踪。对于多代理任务看板，如果可用，请使用 `hermes kanban` CLI。将较旧的 `TodoWrite` 引用视为任务跟踪动作。
