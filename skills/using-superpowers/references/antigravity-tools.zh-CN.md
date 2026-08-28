# Antigravity CLI（`agy`）工具映射

技能以动作来表述（"派发子代理"、"创建待办"、"读取文件"）。在 Antigravity CLI（`agy`）上，这些动作对应为下列工具。

| 技能请求的动作 | Antigravity CLI 对应工具 |
|----------------------|----------------------|
| 派发子代理（`Subagent (general-purpose):` 模板） | 使用内置的 `TypeName` 调用 `invoke_subagent`——`self` 用于全能力工作，`research` 用于只读 |
| 任务跟踪（"创建待办"、"标记完成"） | 一个**任务工件（task artifact）**——以 `IsArtifact: true` 和 `ArtifactType: "task"` 调用 `write_to_file`（参见[任务跟踪](#task-tracking)）。**不是** `manage_task`，后者管理的是后台进程。 |

## 任务跟踪

Antigravity **没有待办工具**（`manage_task` 管理的是后台进程——`list`/`kill`/`status`/`send_input`——它*不是*清单）。当技能说要创建待办列表或跟踪任务时，请维护一个**任务工件（task artifact）**：一个用 `write_to_file`（`IsArtifact: true`、`ArtifactMetadata.ArtifactType: "task"`）保存的 markdown 清单，并随进度用 `replace_file_content` / `multi_replace_file_content` 编辑。

在任何多步骤任务开始时，创建列出你计划每个步骤的任务工件。每完成一步，就编辑工件将其标记为已完成（`- [x]`）。如果计划发生变化，就更新清单。保持它的时效性——它是你剩余工作的真实来源；一旦对话变长，在开始每一步之前重新读取它。
