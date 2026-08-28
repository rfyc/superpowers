# Gemini CLI 工具映射

技能以动作来表述（"派发子代理"、"创建待办"、"读取文件"）。在 Gemini CLI 上，这些动作对应为下列工具。

| 技能请求的动作 | Gemini CLI 对应工具 |
|----------------------|----------------------|
| 读取文件 | `read_file` |
| 一次读取多个文件 | `read_many_files` |
| 创建新文件 | `write_file` |
| 编辑文件 | `replace` |
| 运行 shell 命令 | `run_shell_command` |
| 搜索文件内容 | `grep_search` |
| 按名称查找文件 | `glob` |
| 列出文件与子目录 | `list_directory` |
| 抓取 URL | `web_fetch` |
| 搜索网络 | `google_web_search` |
| 调用技能 | `activate_skill` |
| 派发子代理（`Subagent (general-purpose):` 模板） | 使用 `agent_name: "generalist"` 调用 `invoke_agent`（可通过 `@generalist` 聊天语法调用——参见[子代理支持](#subagent-support)） |
| 多次并行派发 | 在同一个响应中多次调用 `invoke_agent` |
| 任务跟踪（"创建待办"、"标记完成"） | `write_todos`（状态：pending、in_progress、completed、cancelled、blocked） |

## 指令文件

当技能提到"你的指令文件"时，在 Gemini CLI 上它就是 **`GEMINI.md`**。Gemini CLI 分层加载 `GEMINI.md`：全局位于 `~/.gemini/GEMINI.md`，项目级文件位于工作区目录及其上级目录中，当工具访问某些子目录中的文件时，还会加载这些子目录中的 `GEMINI.md` 文件。

## 个人技能目录

用户级技能位于 **`~/.gemini/skills/`**，而 **`~/.agents/skills/`** 是跨运行时的别名（与 Codex 和 Copilot CLI 共享）。当同一层级同时存在两个目录时，`.agents/skills/` 优先。每个技能都是一个包含 `SKILL.md`（带有 `name` 和 `description` frontmatter）的子目录。

## 子代理支持

Gemini CLI 通过 `invoke_agent` 工具派发子代理，该工具接受 `agent_name` 和 `prompt` 参数。同样的派发也以聊天语法快捷方式呈现：输入 `@generalist <prompt>` 等同于以 `agent_name: "generalist"` 调用 `invoke_agent`。内置代理名称包括 `generalist`、`cli_help`、`codebase_investigator`，以及（启用浏览器工具时）`browser_agent`。

技能以 `Subagent (general-purpose):` 派发，并引用提示模板文件（例如 `superpowers:subagent-driven-development` 的 `./implementer-prompt.md`），或提供内联提示。在 Gemini CLI 上：

| 技能派发形式 | Gemini CLI 对应工具 |
|---------------------|----------------------|
| 引用 `*-prompt.md` 模板（implementer、task-reviewer、code-reviewer 等） | 填充模板，然后以 `agent_name: "generalist"` 和填充后的提示调用 `invoke_agent` |
| 引用 `superpowers:requesting-code-review` 的 `./code-reviewer.md` | 以 `agent_name: "generalist"` 和填充后的审查模板调用 `invoke_agent` |
| 内联提示（未引用模板） | 以 `agent_name: "generalist"` 和你的内联提示调用 `invoke_agent` |

### 提示填充

技能提供带有占位符的提示模板，例如 `{WHAT_WAS_IMPLEMENTED}` 或 `[FULL TEXT of task]`。在将完整提示传给 `invoke_agent` 之前，填好所有占位符。提示模板本身包含代理的角色、审查标准以及预期的输出格式——子代理会遵循它。

### 并行派发

Gemini CLI 支持并行子代理派发。在同一个响应中发出多次 `invoke_agent` 调用（或在一条提示中多次调用 `@generalist`），以并行运行相互独立的子代理工作。相互依赖的任务保持顺序执行，但不要仅仅为了保留更简单的历史记录而将独立的子代理任务串行化。

## Gemini CLI 附加工具

这些工具是 Gemini CLI 独有的：

| 工具 | 用途 |
|------|---------|
| `save_memory`（旧版） | 当 `experimental.memoryV2 = false` 时，跨会话持久化事实 |
| `get_internal_docs` | 查找 Gemini CLI 自带的文档 |
| `ask_user` | 向用户提出结构化问题（文本 / 单选 / 多选） |
| `enter_plan_mode` / `exit_plan_mode` | 进入和退出只读计划模式 |
| `update_topic` | 更新当前对话的主题 / 战略意图元数据 |
| `complete_task` | 标记 Gemini 子代理已完成，并将其结果返回给父代理 |
| `tracker_create_task`、`tracker_update_task`、`tracker_get_task`、`tracker_list_tasks`、`tracker_add_dependency`、`tracker_visualize` | 支持依赖关系与可视化的丰富任务跟踪器 |
| `read_mcp_resource`、`list_mcp_resources` | MCP 资源访问 |
