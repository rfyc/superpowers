# 平台中立的行文（prose）——Phase A 设计

## 背景

Superpowers 面向多个 agent 运行时发布（Claude Code、Codex、Cursor、OpenCode、Copilot CLI、Gemini CLI）。技能内容和配套文档最初是为 Claude Code 编写的，在适用任何运行时 agent 的地方使用了 "Claude"。OpenAI 的 vendored fork（openai/plugins#217）尝试过全面重写，但某些地方是有实际错误的——重写了历史归属路径、模型名和平台特定的安装说明——我们希望在移除平台中心化行文的同时避免那个错误，前提是这些行文确实只是附带性的。

整个工作按引用类别划分为多个阶段。**本规格仅覆盖 Phase A：** 在非平台特定语境中提及 "Claude" 的通用第三人称行文。后续阶段（配置文件引用、营销文案、工具名引用）不在本规格范围内，将有各自的规格。

## 范围内

在以下内容中提及 "Claude" 的通用行文：

- 活跃技能目录中的 `skills/*/SKILL.md` 及配套 `.md` 文件
- `skills/writing-skills/anthropic-best-practices.md`
- `README.md`（仅当提及属于通用行文而非平台营销时）

外加一个造词重命名：**Claude Search Optimization (CSO) → Skill Discovery Optimization (SDO)**，位于 `skills/writing-skills/SKILL.md`。

## 范围外

- **平台 / 运行时声明**——"In Claude Code:"、安装说明、工具映射引用。（Phase D 候选。）
- **配置文件引用**——CLAUDE.md、AGENTS.md、GEMINI.md 优先级列表和"项目约定放在哪里"的标注。（Phase B。）
- **工具名引用**——`Skill`、`Bash`、`Read`、`Task`、`TodoWrite`。技能是用 Claude Code 的工具词汇编写的；现有的 `references/{codex,copilot,gemini}-tools.md` 文件负责映射它们。（在编写本规格时，计划是推迟或跳过这些。Phase E 最终做了它们——用动作语言替换活跃技能中的工具名，并围绕同一词汇统一平台工具引用。）
- README 中的**营销文案**——"Superpowers for Claude Code"、以平台命名的安装部分。（Phase C。）
- **历史产物**——`docs/plans/*.md`、`docs/superpowers/specs/*.md`、`CREATION-LOG.md`。这些是带日期的、时点性的文档；重写它们就是重写历史。
- **模型标识符**——Claude Haiku / Sonnet / Opus。这些是真实产品名。
- **文件名 / URL 引用**——`CLAUDE.md`、`claude.com`、`claude-plugin/`、`~/.claude/` 下的路径。
- **`anthropic-best-practices.md` 文件名**——尽管我们重写了其中的行文，该文件仍以其来源命名。

## 替换风格

使用在英语中读起来自然的组合：

- **第二人称——"your agent"**，当对技能作者谈论*他们自己的*运行时
  - "your agent reads the description"
- **第三人称——"the agent" / "agents" / "an agent"**，当通用地描述系统行为时
  - "Future agents find your skills"
  - "Use words an agent would search for"
  - "Agents read SKILL.md only when the skill becomes relevant"

选择最适合周围句子的形式；不要以牺牲拗口为代价强制一致性。在自然时使用复数（"future agents"、"agents read"），而不是总是说 "the agent"。

### 保留为 "Claude" 的例外项

- 模型名：Claude Haiku、Claude Sonnet、Claude Opus
- 文件名和 URL：`CLAUDE.md`、`claude.com`、`~/.claude/`
- 品牌化平台名 "Claude Code"（只要它指代该运行时本身；在后续阶段处理）

### 造词重命名

- **Claude Search Optimization (CSO) → Skill Discovery Optimization (SDO)**
  - 出现在 `skills/writing-skills/SKILL.md` 的章节标题及附近行文中。重命名标题、缩写及文件内任何交叉引用。

## 受影响文件

基于排除例外项后的 `grep` 的近似数量：

| 文件 | 通用行文提及 |
|------|------------------------|
| `skills/writing-skills/SKILL.md` | ~12（含 CSO 标题 + 正文） |
| `skills/writing-skills/anthropic-best-practices.md` | ~30 |
| `skills/writing-skills/examples/CLAUDE_MD_TESTING.md` | ~1——文件名保持不变（它是 CLAUDE.md 测试产物）；"Variant C: Claude.AI Emphatic Style" 标题也保持不变（它是命名特定风格的标签） |
| `README.md` | ~1 |

最终列表在实现期间通过重新运行过滤后的 grep 确认。

## 提交计划

四个原子提交，按顺序：

1. **将 CSO → SDO 重命名**，在 `skills/writing-skills/SKILL.md` 中。机械性、隔离的、如果我们改变关于术语的想法易于回滚。
2. **活跃技能行文**——在 `skills/*/SKILL.md` 及配套 `.md` 中，将通用的 "Claude" → "agent" 形式，排除 `anthropic-best-practices.md`。
3. **`anthropic-best-practices.md` 行文**——同样的替换规则。单独提交，因为该文件是外部文档的 vendored 改编；隔离该变更使未来与上游的协调更容易阅读。
4. **README.md 行文**（*仅当过滤后仍剩通用行文提及时*）。如果为空则跳过。

每个提交消息都点名阶段（"Phase A"）和分片（"rename CSO to SDO"、"agent prose in active skills" 等），使整个序列自文档化。

## 验证

每次提交后：

- `grep -rn "Claude" <touched-paths>`——每个剩余命中都必须落入已记录的例外项（模型名、文件名、URL、"Claude Code" 平台名、历史产物）。
- 从头到尾通读被改动的文件——替换不应破坏句子流畅性、代词一致性或列表平行结构。
- 无需运行测试；这只是行文变更。

最终提交后：

- 在活跃会话中略读每个被修改的技能，确认没有读起来拗口的地方。

## 非目标

- 不要改变行为、结构、标题（CSO→SDO 除外）、示例、代码块或 YAML frontmatter。
- 不要引入新章节、标注或兼容性说明。
- 编辑时不要在替换之外"改进"行文。
