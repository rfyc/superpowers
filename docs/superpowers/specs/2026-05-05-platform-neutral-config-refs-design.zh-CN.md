# 平台中立的配置文件引用——Phase B 设计

## 背景

Phase A（见 `2026-05-05-platform-neutral-prose-design.md`）将通用的第三人称"Claude"表述替换为 agent 中立的表达。本阶段处理下一类别：技能中引用的各平台指令文件（CLAUDE.md、AGENTS.md、GEMINI.md）。

该插件运行在多个 harness 上，每个 harness 读取各自的指令文件。当技能把 CLAUDE.md 当成唯一文件来点名时，这是一种以 Claude Code 为中心的假设，在 Codex / Gemini CLI / OpenCode 上并不成立。

## 范围内

现有技能中的两行特定内容：

1. **`skills/writing-skills/SKILL.md:58`**——`Project-specific conventions (put in CLAUDE.md)`
2. **`skills/receiving-code-review/SKILL.md:30`**——`"You're absolutely right!" (explicit CLAUDE.md violation)`

## 范围外

- **`skills/using-superpowers/SKILL.md:22, 26`**——指令优先级列表。该列表已经以包容方式点名全部三个文件（CLAUDE.md、GEMINI.md、AGENTS.md），这是正确的：该部分在多个平台插件上对*什么算用户指令*做出了真实声明。无需更改。
- **历史 / 示例产物**：
  - `skills/systematic-debugging/CREATION-LOG.md`——归属路径（`~/.claude/CLAUDE.md`）是历史事实。
  - `skills/writing-skills/examples/CLAUDE_MD_TESTING.md`——整个文件是测试 CLAUDE.md 内容变体的完整工作示例。文件名、正文以及来自 `testing-skills-with-subagents.md` 的引用都保持不变；将它们规范化反而破坏了示例。
- **平台工具引用**——Phase D 候选：
  - `skills/using-superpowers/SKILL.md:40`（关于 GEMINI.md 的 Gemini CLI 工具映射说明）
  - `skills/using-superpowers/references/gemini-tools.md`（`save_memory` 持久化到 GEMINI.md）

## 替换规则

每行范围内内容各一个替换，共两个不同的调用。

### 规则 1："项目特定约定放在哪里"

`writing-skills/SKILL.md:58`：

- **之前：** `Project-specific conventions (put in CLAUDE.md)`
- **之后：** `Project-specific conventions (put in your instructions file)`

使用通用措辞而非挑选一个文件名。不同 harness 读取不同的文件（CLAUDE.md、AGENTS.md、GEMINI.md 等），技能不应假定某个文件。平台工具参考文档（`references/{codex,copilot,gemini}-tools.md`）才是点名各平台首选文件的正确位置。

### 规则 2："(explicit CLAUDE.md violation)" 括号

`receiving-code-review/SKILL.md:30`：

- **之前：** `"You're absolutely right!" (explicit CLAUDE.md violation)`
- **之后：** `"You're absolutely right!" (explicit instruction-file violation)`

该括号承担了实际工作——它表明这个短语不仅风格不佳，而且主动违反了用户常放在指令文件中的规则。"Instruction file" 是自然的跨平台术语，涵盖 AGENTS.md / CLAUDE.md / GEMINI.md 的集合，同时保留原始信号，无需挑选某个文件名或弱化为"common"。

## 提交计划

原子提交，按顺序：

1. **`writing-skills/SKILL.md`**——在"项目约定放在哪里"一行中，CLAUDE.md → "your instructions file"
2. **`receiving-code-review/SKILL.md`**——在违规括号中，CLAUDE.md → instruction-file
3. **平台工具参考文档**——为每个 `references/{codex,copilot,gemini}-tools.md` 添加各平台首选指令文件名（CLAUDE.md、AGENTS.md、GEMINI.md 等），以便读者能将 "your instructions file" 解析为真实的文件名。

每个提交消息都点名 "Phase B" 和所处理的分片。

## 验证

每次提交后：

- 阅读周围段落，确认语法和含义仍然成立。
- `grep -n "CLAUDE\.md" <touched-file>`——活跃正文中无剩余命中（例外项已记录在案）。

两次提交后：

- `grep -rn "CLAUDE\.md" skills/` 应只返回已记录的例外项（CREATION-LOG、CLAUDE_MD_TESTING 及其入站引用、using-superpowers 中的优先级列表）。

## 非目标

- 不要改动 `using-superpowers/SKILL.md` 中优先级列表的顺序。重新排列 CLAUDE.md / GEMINI.md / AGENTS.md 是审美上的改动，不是替换，超出本规格范围。
- 不要重命名 `examples/CLAUDE_MD_TESTING.md` 或更改其内容。
- 不要修改 Gemini CLI 特定的工具引用（Phase D 候选）。

## 实现说明

此处所写的 Phase B 覆盖了三个提交和三个非 Claude Code 平台工具引用。实现又往前走了一步：第四个引用 `references/claude-code-tools.md` 在提交 `8505703` 中被添加以求对称，使 Claude Code 的指令文件约定和工具名列表与其余文件并列，而不是隐含在周围技能正文中。该添加在本规格中未被预见到，但与其意图一致。
