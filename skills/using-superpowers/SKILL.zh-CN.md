---
name: using-superpowers
description: 当开始任何对话时使用——确立如何查找与使用技能，要求在任何回复（包括澄清性问题）之前先调用相关技能
---

<SUBAGENT-STOP>
如果你被作为 subagent 分派去执行特定任务，请忽略这个技能。
</SUBAGENT-STOP>

<EXTREMELY-IMPORTANT>
如果你认为某个技能哪怕有 1% 的可能性适用于你正在做的事情，你绝对必须调用该技能。

如果某个技能适用于你的任务，你就没有选择。你必须使用它。

这一点不容商量。你不能为自己找理由绕开它。
</EXTREMELY-IMPORTANT>

## 规则

**在任何回复或行动之前调用相关或被请求的技能** —— 包括澄清问题、探索代码库或检查文件。如果它最终不适合当前情况，你不必使用它。

**在进入 plan mode 之前：** 如果你还没有进行 brainstorm，请先调用 brainstorming 技能。

然后宣布 "使用 [skill] 进行 [purpose]" 并严格遵循该技能。如果它有 checklist，为每个条目创建一个 todo。

## 技能优先级

当多个技能适用时，流程类技能优先——它们设定方法，然后实现类技能（frontend-design 等）负责执行。Brainstorming 和 systematic-debugging 是 Superpowers 最常见的流程类技能，但这条规则适用于它们中的任何一个。

- "让我们构建 X" → 先使用 superpowers:brainstorming，然后使用实现类技能。
- "修复这个 bug" → 先使用 superpowers:systematic-debugging，然后使用领域技能。

## 警示信号

这些想法意味着停止——你在自我合理化：

| 想法 | 现实 |
|---------|---------|
| "这只是一个简单的问题" | 问题就是任务。检查技能。 |
| "我首先需要更多上下文" | 技能检查先于澄清问题。 |
| "让我先探索代码库" | 技能告诉你如何探索。先检查。 |
| "我可以快速检查 git/文件" | 文件缺乏对话上下文。检查技能。 |
| "让我先收集信息" | 技能告诉你如何收集信息。 |
| "这不需要正式的技能" | 如果技能存在，就使用它。 |
| "我记得这个技能" | 技能会演进。阅读当前版本。 |
| "这不算任务" | 行动 = 任务。检查技能。 |
| "这个技能有点小题大做" | 简单的事情会变得复杂。使用它。 |
| "我只先做这一件事" | 在做任何事之前先检查。 |
| "这感觉很有成效" | 无纪律的行动浪费时间。技能能防止这一点。 |
| "我知道那是什么意思" | 知道概念 ≠ 使用技能。调用它。 |

## 平台适配

如果你的 harness 出现在这里，请阅读其参考文件以获取特别说明：

- Codex: `references/codex-tools.md`
- Pi: `references/pi-tools.md`
- Antigravity: `references/antigravity-tools.md`
- Hermes Agent: `references/hermes-tools.md`

## 用户指令

用户指令（CLAUDE.md、AGENTS.md、GEMINI.md 等，以及直接请求）优先于技能，技能又优先于默认行为。只有在你的人类伙伴明确告知时，才可以跳过技能工作流或指令。
