# 把 Superpowers 移植到新宿主环境

本指南解释如何为新的宿主环境——一个不是 Claude Code 的 IDE、CLI 或代理运行器——添加支持，使 Superpowers 技能在那里像在原生环境中一样自动触发。

它分为两层编写。**第 1-3 部分**解释系统如何工作，以及如何判断某个宿主环境能否被支持；在你触碰任何东西之前先阅读这些。**第 4-8 部分**是为代理（在人类伙伴监督下）提供的指令式流程，用于端到端地执行移植，直至分发。附录索引了当前的参考集成，方便你复制最接近的一个。

集成机制因宿主环境而异，而且会不断变化。本指南刻意教授的是**不变项**——无论机制如何都必须成立的东西——并把你指向一个活的参考实现来复制。当本指南与代码不一致时，以代码为准；然后修复本指南。

## 开始之前

添加一个宿主环境是这个仓库中风险最高的贡献类型。在写任何东西之前：

- 完整阅读 `CLAUDE.md` 和 `.github/PULL_REQUEST_TEMPLATE.md` —— 贡献者规则和新宿主环境 PR 要求不是可选项。
- 搜索已打开**和**已关闭的 PR，看是否有人为这个宿主环境做过尝试。如果有，在开始你自己的移植之前，先理解它为什么停滞了。

---

## 第 1 部分 —— Superpowers 如何跨宿主环境工作

Superpowers 在每个地方都是相同的内容。每个宿主环境变化的只是那层薄薄的传递层，它把内容交付给模型，并把它的指令翻译成宿主环境的原生工具。三个组成部分：

1. **技能（与宿主环境无关）。** `skills/` 中的一切都是事实来源，被每个宿主环境逐字共享。技能被编写为描述*动作*——"invoke a skill"、"read a file"、"dispatch a subagent"、"create a todo"——并且从不指名某个具体工具。这正是让一个技能主体无需修改就能在 Claude Code、Codex、Gemini、pi 等上面运行的原因。

2. **工具映射（每宿主环境）。** 每个宿主环境都需要把动作词汇翻译成它真实的工具名。该翻译位于 `skills/using-superpowers/references/<harness>-tools.md`，和/或内联在宿主环境的引导注入器中（见第 5 部分）。它例如说明："*dispatch a subagent* → 调用带 `subagent_type` 的 `task`。"

3. **引导程序（每宿主环境）。** 在每个会话开始时，完整的 `skills/using-superpowers/SKILL.md` 被注入到模型的上下文中，包裹在 `<EXTREMELY_IMPORTANT>` 标签内，并附上工具映射。这个被注入的技能教会模型技能是存在的，并且它必须在行动之前检查是否有相关技能。**引导程序就是整个集成。** 没有它，技能文件就是惰性的——存在于磁盘上，却从未被调用。

### 让这一切成立的两条规则

**1. 技能命名动作，而不是工具。** 不要编辑技能主体来适应你的宿主环境。移植是添加一个工具映射参考和一个引导注入器；它绝不伸进 `skills/*/SKILL.md` 去替换工具名。（项目的贡献者指南把技能内容视为精心调优的塑造行为的代码；为"合规性"改写它会被当场拒绝。）

**2. 一切都通过宿主环境自己的安装机制分发。绝不要编辑用户的文件。** 引导程序、技能和工具映射都作为*宿主环境所安装内容的一部分*被交付——一个插件、一个扩展、一个市场条目、一个扩展捆绑的上下文文件。一次移植**不得**伸进用户的全局或个人配置（`~/.gemini/config/AGENTS.md`、`settings.json`、`trustedFolders.json`、手改的 `~/.bashrc` 等）去注入任何东西。宿主环境拥有它加载的内容；你的安装工件是你唯一能写的东西。如果安装机制确实无法承载引导程序，这是一个需要浮现出来的局限（第 6 部分）——绝不是手工编辑用户配置的许可证。（形状 C *不是*例外：Gemini 的上下文文件没问题，因为它*在已安装的扩展内部*分发，并由清单的 `contextFileName` 声明——宿主环境加载的是扩展自己的文件，而不是你在用户主目录中编辑的文件。）

---

## 第 2 部分 —— 这个宿主环境能被支持吗？

只有当宿主环境能做到以下所有事情时，它才能支持 Superpowers。在写代码之前先检查这些——如果第一项失败，就停止。

### 硬性要求：自动的会话启动注入

宿主环境必须允许你在**每个会话开始时**把文本注入模型的上下文，**无需你的人类伙伴做任何每会话的主动选择（opt-in）**。这是唯一不可协商的能力。它可以采用任何形式：

- 一个**钩子/事件系统**，在会话启动时运行 shell 命令并读取其 stdout（Claude Code、Cursor、Copilot CLI），或者
- 一个**进程内插件/扩展**，带会话启动或消息生命周期回调，可以修改消息数组（OpenCode、pi），或者
- 一个**指令文件**约定，宿主环境加载一个*你安装的扩展所附带并声明*的上下文文件（例如 Gemini 的 `contextFileName` 指向扩展自己的 `GEMINI.md`）——而不是你在用户主目录中编辑的文件。

如果让 Superpowers 出现在模型面前唯一的方式是你的人类伙伴在每个会话中主动选择（粘贴一个提示、运行一个命令、启用一种模式），那么这个宿主环境**无法**被妥善支持。第 3 部分中的验收测试会失败，PR 会被关闭。这是"移植"不是真正移植的最常见原因。

### 其余的能力清单

| 能力 | 为什么需要 | 如果缺失 |
|---|---|---|
| **技能发现 + 调用** | 模型必须能按需加载技能的完整内容 | 如果没有原生技能工具，被认可的兜底方案是直接用 `read` 读取相关的 `SKILL.md`——见第 5 部分。既没有技能工具也没有文件读取能力的宿主环境无法工作。 |
| **文件读 / 写 / 编辑** | 几乎每个技能都操作文件 | 必需。没有变通方案。 |
| **运行 shell 命令** | TDD、验证、git 工作流 | 必需。 |
| **Subagent / 任务派发** | `dispatching-parallel-agents`、`subagent-driven-development` | 可降级：如果不可用，那些特定技能会告诉模型内联完成工作或报告缺失的能力——*绝不*发明一个 `Task` 调用。某些宿主环境把它藏在配置标志后面（例如 Codex 需要启用 multi-agent）。 |
| **Todo / 任务跟踪** | 多个技能中的进度跟踪 | 可降级：回退到计划文件或 `TODO.md`。 |
| **Web 抓取 / 搜索** | 少数技能 | 可降级。 |
| **Shell 或多语言脚本执行（Windows）** | 仅对 shell 钩子形态，仅在你想要 Windows 支持时 | 见第 7 部分。进程内插件宿主环境完全绕过这一点。 |

"可降级"意味着：该技能已经有针对缺失工具的兜底措辞。你在工具映射中的工作是当真实工具存在时指向它，当不存在时复用那个兜底措辞。

### 你可能根本不需要新目录

某些"新宿主环境"实际上只是不同安装器下的现有集成。例如 Factory 的 Droid 通过它自己的 `plugin install` 命令消费 Claude Code 插件，这里不需要新文件。在动手之前，先检查宿主环境是否能简单加载现有清单。一个只在这个仓库里加了一段 README 段落的移植，就是一个完全可以接受的结果。

---

## 第 3 部分 —— 完成定义

当**所有**以下条件都为真时，一次移植才算完成：

1. `using-superpowers` 引导程序在会话启动时加载，每个会话都是如此，无需每会话的主动选择。
2. 宿主环境存在一个工具映射（在 `references/<harness>-tools.md` 中，内联在引导程序中，或两者都有——见第 5 部分）。
3. 技能确实能被调用——原生调用，或通过文档记载的读取 `SKILL.md` 兜底方案——并且模型会遵循它们。
4. **验收测试通过。** 在一个干净会话中，用户消息：

   > Let's make a react todo list

   会在*写出任何代码之前*自动触发 `brainstorming` 技能。捕获完整的会话记录——PR 要求它。
5. 测试覆盖集成（第 5 部分）并全部通过。
6. 真实用户可以通过宿主环境自己的机制安装它（而不是手工复制文件），并且在适用处，版本被记录在 `.version-bump.json` 中（第 6 部分）。注意，某些安装器在安装时重写或剥离清单（有一个把它剥到只剩 `{"name": …}`），所以"*已安装的*文件报告仓库版本"并不总是可实现的——在源清单处跟踪版本，不要把重写过的已安装清单当作失败。

在完整验收测试之前做一个快速冒烟检查：开始一个会话，让模型描述它的 superpowers。如果引导程序注入了，它就知道自己有。 （OpenCode 的安装文档使用 `opencode run --print-logs "hello" 2>&1 | grep -i superpowers` 通过不同机制实现同样的目标——日志 grep 而不是问模型；`2>&1` 很关键，因为日志走 stderr。找到你宿主环境的等价物。）

---

## 第 4 部分 —— 选择你的集成形态

有三种结构形态，它们取决于*你如何让引导程序出现在模型面前*。选择与你的宿主环境暴露的东西匹配的那个，然后复制那个参考实现。形态决定了第 5 部分中几乎所有内容——下面的步骤会因形态而分支。

### 如何判断你属于哪种形态

在路由之前，先弄清宿主环境的*实际*机制——并且不要假设它文档完备，或假设它的行为像它 fork 自的那个宿主环境。

**找到表面（surface）：**

- **在网上搜索宿主环境的文档**（extension / plugin / hook / skill / MCP / "context file" / "rules file"）。厂商工具变化很快；宁可搜索也不要信任训练知识。
- **找到并阅读宿主环境现有的第三方扩展/插件。** 一个真实可用的例子胜过文档——它展示了清单形态、安装命令，以及宿主环境实际加载哪些组件。
- 检查宿主环境在启动时加载什么：一个设置文件？一个扩展目录？一个项目级或全局指令文件（`AGENTS.md`、`<NAME>.md`）？

**如果它文档不足，就凭经验逆向工程它**（一个真正的移植者每一种都做过）：

- 对二进制执行 `strings` / grep 安装树，查找钩子事件名、配置路径，以及它读取的指令文件。
- **让运行中的模型枚举它自己的工具名** —— 例如"list the exact machine names of every tool you can call"。这是不凭空发明而获得工具名的权威方式（见步骤 4）。
- 用一个**唯一标记测试**证明每个假设：通过你认为可行的机制注入一个无意义的 token，开始一个新会话，确认该 token 真的到达了模型。

**fork 不会继承其父级的行为。** 一个派生自另一个宿主环境的宿主环境（例如一个 Gemini 派生的 CLI）可能暴露父级的清单字段和 `@` 包含语法，却*仍然不按相同方式尊重它们*。用标记验证；绝不要假设父级的配方会迁移。

然后路由到一种形态：

- 会话启动时运行 shell 命令且其 stdout 被读取 → **形态 A**。
- 带生命周期回调、你可以在其中运行代码的插件/扩展模块 → **形态 B**。
- 只有一个始终开启的指令文件，没有钩子，也没有代码插件 → **形态 C**。

**形态是可组合的——它们不是互斥的。** *技能发现*机制和*引导*机制不需要是同一形态——但**两者仍必须搭乘安装机制**（规则 2）。分别决定这两个问题：*技能在哪里被发现？*以及*引导程序如何每个会话都到达模型？* 一个宿主环境可能通过插件安装技能，却又需要引导程序以另一种随安装分发的方式被交付（一个扩展声明的上下文文件，或者——见下文——由宿主环境在会话启动时浮出已安装 `using-superpowers` 技能自己的描述）。如果有多个安装机制表面自动注入，优先选择最可靠的那个。你**不能**做的是通过编辑用户的全局配置来弥合缺口。

### 形态 A —— Shell 钩子

宿主环境有一个钩子系统，在会话启动时运行 shell 命令并从其 stdout 读取 JSON。配置的命令运行 `run-hook.cmd`，一个多语言包装器，它只负责定位 bash 并派发指定的脚本；脚本（`hooks/session-start`，或宿主环境特定的变体）负责读取 `using-superpowers/SKILL.md` 并打印一个 JSON 对象，其**字段名和嵌套因宿主环境而异**。

- 参考：`hooks/session-start`、`hooks/run-hook.cmd`，以及每宿主环境的钩子配置 `hooks/hooks.json`（Claude Code）和 `hooks/hooks-cursor.json`（Cursor）。
- 清单：`.cursor-plugin/plugin.json` 是形态 A 的清单示例，它把宿主环境指向 `./skills/` 和正确的 `hooks-*.json`。Claude Code 的 `.claude-plugin/plugin.json` 两个字段都不设置——它按约定自动发现 `skills/` 和 `hooks/hooks.json`。对形态 A，**不要**复制 Codex 的 `.codex-plugin/plugin.json`：它声明一个空的 `hooks` 对象，专门用于抑制 Codex 的 `hooks/hooks.json` 自动发现，因为 Codex 原生浮出技能，并且不运行会话启动钩子。

> **一个钩子*系统*不是会话启动*事件*。** 一个宿主环境可以有一个 `hooks.json` 机制——甚至其二进制中包含字面量 `SessionStart`——却没有任何在会话启动时触发、能够注入上下文的钩子事件。（有一个真实宿主环境只暴露了工具前/工具后和停止事件；那些 `SessionStart` 字符串是遥测。）在承诺采用形态 A 之前，确认你需要的*具体事件*存在，并且能写入模型的上下文。如果不能，引导程序就该放在指令文件中（形态 C）。

### 形态 B —— 进程内插件 / 扩展

宿主环境加载一个暴露生命周期回调的 JS/TS 模块。你通过宿主环境的 API 注册技能目录，并在代码中通过修改消息数组来注入引导程序。

- 参考：`.opencode/plugins/superpowers.js`（JavaScript）和 `.pi/extensions/superpowers.ts`（TypeScript）。pi 是任何**没有原生技能工具**的宿主环境最接近的参考。

### 形态 C —— 指令文件

宿主环境既没有 shell 钩子也没有代码插件——它的会话启动表面是一个*你安装的扩展所附带、清单所声明*的上下文文件（例如 Gemini 的 `contextFileName` → 扩展自己的 `GEMINI.md`）。你无法运行代码或修改消息；扩展的上下文文件指向引导程序。没有注入器去拼装字符串或剥离 frontmatter——宿主环境按原样加载被引用的内容。**这只因为该文件是已安装扩展的一部分才成立**——绝不要用"编辑用户的全局 `GEMINI.md`/`AGENTS.md`"来替代分发你自己的文件（规则 2）。

- 参考：`gemini-extension.json`（清单，带 `contextFileName`）、`GEMINI.md`（两个 `@` 包含——引导技能和工具映射参考）、`skills/using-superpowers/references/gemini-tools.md`。
- 注意：`@` 包含是 Gemini 的功能。如果你的宿主环境加载指令文件但没有包含语法，你必须把引导内容内联进该文件。
- **不要相信 `@` 包含真的被展开——要证明它。** 一个 Gemini*派生*的宿主环境可以接受 `@./path` 语法，却把它当作*模型可以选择读取的提示*（它会发出一个文件读取工具调用），而不是一个有保证的内联展开。这就是引导程序可靠地每个会话都在，与模型可能去读它之间的差别。运行一个唯一标记测试：如果标记在*没有*工具调用的情况下不在上下文中，就把内容**内联**而不是 `@` 包含它。

### 路由表

| 如果宿主环境…… | 使用形态 | 从何处复制 |
|---|---|---|
| 在会话启动时运行 shell 命令并读取其 stdout | A（shell 钩子） | Cursor（`hooks/session-start` + `hooks/hooks-cursor.json` + `.cursor-plugin/`） |
| 是一个带会话/消息生命周期回调的 JS/TS 插件宿主 | B（进程内） | OpenCode（`.opencode/`）——或者 pi（`.pi/`），如果它没有原生技能工具 |
| 附带一个它总是加载的、由扩展声明的上下文文件 | C（指令文件） | Gemini（`gemini-extension.json` + `GEMINI.md` + `references/gemini-tools.md`） |
| 有一个插件安装命令，且安装器保留一个清单 `contextFileName`（或等价物） | 通过插件安装器的 C | Antigravity（`.antigravity-plugin/` —— `agy plugin install` 会分发一个生成的上下文文件；验证安装器保留它——第 6 部分） |

大多数真实宿主环境能干净地落进某一行；最后一行是混合情形（规则 2 仍然成立——引导程序搭乘安装机制，绝不修改用户配置）。

---

## 第 5 部分 —— 移植流程

### 步骤 1 —— 研究最接近的参考实现

打开第 4 部分为你的形态点名的文件，从头到尾读完。下面的模式是摘要；代码才是规格。

### 步骤 2 —— 创建清单 / 入口点

创建宿主环境用来识别插件所需的一切。在精神上与现有的保持一致：

- **形态 A：** 一个 `*-plugin/plugin.json`（见 `.cursor-plugin/plugin.json`），含 `name`、`version`、`description`、作者/许可证/关键词、`"skills": "./skills/"` 和 `"hooks": "./hooks/hooks-<harness>.json"`。再加上 `hooks-<harness>.json` 本身，注册一个命令调用 `run-hook.cmd` 的会话启动钩子。
- **形态 B：** 宿主环境加载的模块（例如 `.<harness>/plugins/*.js`），加上它被发现所需的任何包元数据。已提交的包元数据是**仓库根目录的 `package.json`**：`main` 指向 OpenCode 插件，`pi` 字段（`pi.extensions`、`pi.skills`）加上 `pi-package` 关键字声明 pi 扩展。每宿主环境的本地清单和锁文件被排除在 git 之外——`.opencode/.gitignore` 排除 `node_modules`、`package.json` 和锁文件。对你的宿主环境的*本地*安装工件也这样做，以免污染仓库——但绝不要 gitignore 仓库根目录的 `package.json`，它是被跟踪的事实来源。
  - **构建/依赖检查。** 决定宿主环境如何加载你的模块：它是直接运行源码（pi 的 `.ts` 按原样从 `package.json` 引用；OpenCode 分发纯 `.js`），还是需要一个转译/构建步骤？Superpowers 是零运行时依赖。pi 的 `import type { ExtensionAPI }` 之所以可行，恰恰因为宿主环境直接运行 `.ts`，在加载时提供那个类型，而仓库在 CI 中从不类型检查该文件——这个导入甚至没有被声明为依赖。如果*你的*宿主环境真的类型检查或打包插件，那就会破坏：未声明的类型导入会失败，而 PR 规则只为新宿主环境豁免*运行时*依赖，不豁免 dev/类型包。如果你碰到这种情况，与维护者确认方法，而不要悄悄添加依赖。让任何构建输出都留在 git 之外，并记录该命令。
- **形态 C（指令文件）：** 一个小的清单（见 `gemini-extension.json`：`name`、`description`、`version`、`contextFileName`）加上上下文文件本身（`GEMINI.md` 只是两个 `@` 包含：引导技能和工具映射参考）。Gemini 清单没有 `skills` 字段——Gemini 自动发现捆绑在已安装扩展中的 `skills/` 目录。如果你的宿主环境有原生技能工具但没有注册目录的清单字段，你必须找到它的发现约定（阅读它的扩展文档），然后凭经验验证：接线后，让模型列出它可用的技能——如果捆绑的技能没有出现，说明发现还没生效。

### 步骤 3 —— 接线引导注入

这是移植的核心。共同的目标：在会话启动时，把 `using-superpowers` 技能内容（包裹在 `<EXTREMELY_IMPORTANT>` 标签内）加上宿主环境的工具映射放到模型面前，并附带一条说明该技能已经激活、模型不要再次加载它的提示。*如何*做到这一点——以及你拼装什么，相对地宿主环境按原样加载什么——完全取决于你的形态。**不要**把一种形态的配方套到另一种上。

**形态 A —— 一个脚本读取 `SKILL.md` 并打印宿主环境的 JSON。** 被派发的脚本（`hooks/session-start`）`cat` 整个 `SKILL.md`（包括 frontmatter——没问题；它被逐字发出），用"You have superpowers… for all other skills use the Skill tool"前导语包裹它，转义它，并打印宿主环境的 JSON 形态。形态 A 的工具映射**不会**内联在这里——它位于 `references/<harness>-tools.md`（步骤 4）。把 JSON 输出形态做对。`hooks/session-start` 从环境变量检测宿主环境，并打印*三种形态之一*：

- Cursor（设置了 `CURSOR_PLUGIN_ROOT`）：`{ "additional_context": "…" }`
- Claude Code（设置了 `CLAUDE_PLUGIN_ROOT`，未设置 `COPILOT_CLI`）：`{ "hookSpecificOutput": { "hookEventName": "SessionStart", "additionalContext": "…" } }`
- Copilot CLI / SDK 标准（其他情况）：`{ "additionalContext": "…" }`

这是一个陷阱。发出错误的字段，或多出一个字段，意味着引导程序要么从不注入，要么注入两次（Claude Code 同时读取 `additional_context` 和 `hookSpecificOutput` 而不去重，所以同时发出两者会双重注入）。找到你的宿主环境期望的确切字段、嵌套和事件匹配值。然后决定：给 `hooks/session-start` 加第四个分支，或者——如果宿主环境需要不同的引导消息或环境契约——添加一个专用的 `hooks/session-start-<harness>` 脚本。如果你添加分支，而你的宿主环境*也*设置了一个更早分支所依赖的环境变量（某些宿主环境也设置 `CLAUDE_PLUGIN_ROOT`），把你分支的顺序排在那个原本会遮蔽它的分支之前。匹配宿主环境自己的事件匹配字符串（Claude Code 使用 `startup|clear|compact`，Cursor 使用 `sessionStart`）；错误的匹配器意味着钩子静默地从不触发。

**钩子配置的模式本身因宿主环境而异** —— 不要假设 Claude Code 的形态是通用的。对比 `hooks/hooks.json` 和 `hooks/hooks-cursor.json`：Cursor 使用 `"version": 1`、小写的 `sessionStart` 键、相对的 `./hooks/run-hook.cmd` 命令，并省略 Claude Code 使用的 `matcher`/`type`/`async` 字段。让你的 `hooks-<harness>.json` 匹配最接近的现有文件，而不是匹配一个单一的规范模板。

钩子**命令字符串引用一个宿主环境提供的插件根变量**，它的名称因宿主环境而异：`hooks.json` 使用 `${CLAUDE_PLUGIN_ROOT}`，`hooks-cursor.json` 使用相对路径。使用你的宿主环境导出的任何形式。（`session-start` 脚本通过 `dirname` 自己重新推导根路径，所以脚本主体不依赖它——但清单中的命令依赖。）

**发现宿主环境的契约。** 上面三个事实——环境变量、JSON 字段/嵌套、匹配字符串——是宿主环境的契约，不是 Superpowers 的，所以你必须去获取它们。阅读宿主环境的钩子文档，或凭经验发现：注册一个转储其环境并发出标记的一次性会话启动钩子，然后观察哪个环境变量标识宿主环境，以及宿主环境是否/如何消化你的 stdout。在编写真实分支之前先钉死这些。

**形态 B —— 在代码中拼装字符串，然后作为用户消息注入。** 在这里你自己构建引导程序：读取 `SKILL.md`，剥离它的 YAML frontmatter，并拼装 `<EXTREMELY_IMPORTANT>` + 一段简短的前导语说明该技能已加载且不得重新调用 + 剥离后的主体 + 内联工具映射 + `</EXTREMELY_IMPORTANT>`。参考实现有一个分歧点：OpenCode 的前导语说"do NOT use the skill tool…"（假设存在 `skill` 工具），而 pi 的只说"do not try to load using-superpowers again."如果你的宿主环境没有技能工具，用 pi 的措辞，不要用 OpenCode 的。

把结果作为**用户角色消息而不是系统消息**注入——系统消息在每回合重复时会膨胀 token（#750），多个系统消息会破坏某些模型（#894）。你必须复制的三件事：

- **去重守卫。** 生命周期回调可以重复触发（OpenCode 的 transform 在*每一步*都运行；pi 的 `context` 每回合触发）。注入前，检查引导标记是否已存在，存在则跳过。（参考实现选择不同的标记——pi 用自定义字符串，OpenCode 用 `EXTREMELY_IMPORTANT` 标签；匹配标签更健壮，因为它不需要宿主环境特定的常量。）在模块级别缓存引导内容，以免每次调用都重新读取和重新解析 `SKILL.md`（#1202）。
- **压缩。** 如果宿主环境压缩/总结历史，之后要重新注入。pi 在 `session_start` 和 `session_compact` 上设置 `injectBootstrap` 标志，在 `agent_end` 时清除它，并把消息插入到任何前导的压缩摘要消息*之后*。OpenCode 依赖它每步的重新注入加上去重守卫。
- **消息对象形态因宿主环境而异——发现你自己的，不要复制字面量。** 两个参考实现使用*不兼容*的形态：pi 构建 `{ role, content: [{ type, text }], timestamp }`；OpenCode 操作 `message.info.role` 和 `message.parts[]`。从它的 API 找到你的宿主环境的消息形态；逐字复制参考实现的对象字面量会静默失败。

**形态 C —— 把你的扩展的上下文文件指向引导程序；什么都不拼装。** 没有注入器，所以你*不*剥离 frontmatter，也不构建包裹字符串。你的扩展附带的上下文文件（由清单声明——*不是*用户自己的全局文件）拉入两样东西：`using-superpowers` 技能和宿主环境的工具映射参考。`GEMINI.md` 用两个 `@` 包含做到这一点（`@./skills/using-superpowers/SKILL.md` 和 `@./skills/using-superpowers/references/<harness>-tools.md`）；宿主环境按原样加载它们，包括 frontmatter 和一切，而 `SKILL.md` 已经在内部携带自己的 `<EXTREMELY-IMPORTANT>` 块。如果你的宿主环境没有包含语法，就把内容内联进指令文件。Gemini 附带**没有**"already loaded, don't re-invoke"前导语——对一个 `@` 包含的宿主环境来说，内容就是活跃的指令集，而不是模型会重新加载的技能。如果你发现你的宿主环境确实尝试重新调用，就把那条说明作为字面行添加到指令文件中（你没有代码可以用任何其他方式添加它）。

### 步骤 4 —— 编写工具映射

把动作词汇翻译成宿主环境的真实工具。覆盖以下每一个动作（只省略真正不适用的）：

- 读取文件
- 创建 / 编辑 / 删除文件（一个 `apply_patch` 式工具，还是分开的写/编辑？）
- 运行 shell 命令
- 搜索文件内容 / 按名称查找文件（grep、glob）
- 抓取 URL / web 搜索
- **派发 subagent**，包括如何传递代理类型——以及启用它所需的任何配置标志
- **创建 / 更新 todo**（把较老的 `TodoWrite` 引用当作此动作）
- **调用技能** —— 见步骤 5

**从宿主环境获取真实工具名；绝不要凭空发明。** 如果文档没有列出，权威来源就是宿主环境本身：在一个活跃会话中，让模型"list the exact machine names of every tool you can call, one per line"，并使用它报告的。

**宿主环境如何找到 `skills/` 目录本身也是因宿主环境而异的** —— 去确认它，不要假设。可能性包括：一个清单 `skills` 路径字段（Codex 的 `"skills": "./skills/"`）；一个被宿主环境自动扫描的*同位* `skills/`（其中路径字段被**忽略**——有一个真实宿主环境只扫描 `plugin.json` 旁边的 `skills/`）；一个 API/注册调用（OpenCode、pi）；或者你搭一个安装目录，把清单与**指向仓库 `skills/` 的符号链接**配对，并让安装器指向暂存目录（验证安装器*解引用*符号链接并复制真实文件——在依赖它之前用 `agy plugin validate`/`install` 或等价物确认）。一个 `skills` 路径字段*不是*可移植的。

映射放哪里取决于形态：

- **形态 A：** 放在 `skills/using-superpowers/references/<harness>-tools.md`。代理从引导程序到达它——`SKILL.md` 的"Platform Adaptation"部分链接每个宿主环境的参考文件。（形态 A 宿主环境没有指令文件；映射*不*内联进钩子输出。）
- **形态 B：** 映射通常内联进你注入的引导字符串（见 `superpowers.js` 中的 `toolMapping` 常量）。pi 把它放在*两个*地方——`piToolMapping()` 内联**以及** `references/pi-tools.md`。如果你在两个地方维护它，要两者都更新，否则移植只完成了一半。
- **形态 C：** 放在 `references/<harness>-tools.md`，并把它拉进总是加载的指令文件（例如 `GEMINI.md` `@` 包含 `gemini-tools.md`）。

你也可以在 `SKILL.md` 的"Platform Adaptation"部分为你的宿主环境添加一行指针，让阅读引导程序的代理知道它的映射在哪里。这是一次移植可能对 `SKILL.md` 做的唯一编辑——而且只因为那个部分是指针列表，不是塑造行为的内容。它不违反"不要编辑技能主体"规则（第 1 部分）；不要碰任何技能中的任何其他东西。（该列表是便利指针，不是详尽注册表——不是每个宿主环境都在列。）

### 步骤 5 —— 处理没有原生技能工具的宿主环境

`using-superpowers/SKILL.md` 告诉模型*绝不要用文件工具手动读取技能文件——始终使用你平台的技能加载机制。* 重点是"不要绕过机制"，不是"绝不要用文件读取"。什么算作"你平台的机制"取决于宿主环境——而对一个没有技能工具的宿主环境，文档记载的机制*就是*读取 `SKILL.md`。所以在那里读取它是在遵守规则，而不是违反规则。区分三种情况：

1. **原生 `Skill` 式工具**（Claude Code、Copilot CLI、Gemini 的 `activate_skill`）：把映射指向那个工具。
2. **原生技能*发现*但没有 `Skill` 工具**（pi、Antigravity）：宿主环境能找到并列出现有技能，但模型无法调用工具来加载一个。让技能安装到宿主环境扫描的地方（pi 通过 `resources_discover` → `skillPaths` 注册；OpenCode 通过它的 `config` 钩子；`agy plugin install` 会复制它们进去），并告诉模型在技能适用时**用文件读取工具读取它的 `SKILL.md` 来加载它**——这里是受认可的机制，正如 `references/pi-tools.md` 表述的那样。

   **对于引导程序本身，优先采用声明的上下文文件（第 6 部分）。** 如果宿主环境有 `contextFileName` 式清单字段——就像 Antigravity 那样——通过安装器分发一个生成的上下文文件：它保证被加载，并同时携带 `using-superpowers` 内容和工具映射。那是更强的、更受偏好的路径。

   **回退 —— 浮出的技能索引。** 如果没有上下文文件字段，但宿主环境在会话启动时浮出每个已安装技能的名称 + 描述，你**既**不需要构建索引**也**不需要运行时列表指令——宿主环境就是索引，而 `using-superpowers` 自己的浮出描述就可以成为触发模型加载它的东西。这比声明的上下文文件更软；对比上下文文件 / 钩子 / 进程内注入器，它**不能**给你两样东西——两者都要考虑：
   - **它只引导*触发*，不引导*工具映射*。** 注入器会在每个会话把 `<harness>-tools.md` 与 `using-superpowers` 一起前置。而在这里没有任何东西注入映射——模型只看到技能*描述*，并且必须在自己需要工具名时*读取*你的 `references/<harness>-tools.md`。它能工作是因为技能命名动作（模型在行动时会读取映射），但比注入更软。确保映射能从模型加载的内容到达——例如从 `SKILL.md` 的 Platform Adaptation 部分链接，并与技能一起安装——而不是只躺在仓库里。
   - **没有任何结构性保证触发会生效。** 没有 `<EXTREMELY_IMPORTANT>` 包装器，没有去重，没有压缩后重新注入——触发取决于模型选择对它在索引中看到的描述采取行动。这正是为什么验收测试在这里是强制性的：它是*唯一*的保证，所以要在你的用户将实际使用的模型上运行它，而不只是最强的那个。
3. **完全没有技能系统：** 没有什么可注册的，*唯一*的机制是模型按需读取 `SKILL.md`。但模型读不到它找不到的东西：`using-superpowers/SKILL.md` 并**不**枚举可用的技能，所以模型单独无法知道存在哪些技能或它们的触发条件。你必须提供一条发现路径。两个选项，它们在持久性上有所不同：(a) 生成一个技能索引（每个 `skills/*/SKILL.md` 的 `name` + `description` frontmatter），把它放在 `<EXTREMELY_IMPORTANT>` 包装器*内部*、工具映射旁边（上面的形态 B 配方），这样它被去重守卫覆盖——但构建时索引会随着技能被添加而过时；或者 (b) 指令模型在运行时列出 `skills/*/SKILL.md` 并读取它们的 frontmatter 来找到匹配——更慢但永不过时。除非你有理由不这样做，否则优先选 (b)。两者都没有的话，一个无技能系统的移植会加载引导程序，却静默地从不触发任何其他技能。

在情况 2 和 3 中，在你的工具映射里直说读取 `SKILL.md` 是被祝福的路径，这样模型就不会认为它在违反"绝不要读取技能文件"规则。不要在一个没有技能系统的宿主环境里去寻找 `skillPaths` 式注册 API——情况 3 根本没有。

### 步骤 6 —— 添加测试

匹配现有的每宿主环境测试风格：

- **形态 A：** 断言钩子的 stdout 具有你的宿主环境消费的确切 JSON 形态，并且包含引导程序。见 `tests/hooks/test-session-start.sh`，它验证每个宿主环境的输出形态。
- **形态 B：** 一个伪装宿主环境插件 API 的单元测试，断言生命周期处理器被注册、引导程序只注入一次、去重守卫有效，并且（如果相关）压缩后重新注入有效。见 `tests/pi/test-pi-extension.mjs`。添加一个 `tests/opencode/` 风格的隔离安装集成检查。
- 如果引导程序被缓存，测试当文件缺失时缓存的行为（见 OpenCode 缓存测试）。

这些自动化测试覆盖接线；步骤 7 中的活 tmux 运行才是证明集成真正触发技能的东西。

### 步骤 7 —— 本地安装，然后驱动一个活实例来验证

你无法通过读代码来确认一次移植能用。你必须带着进行中的移植加载宿主环境并观察一个真实会话——这也是你产出 PR 所需会话记录的方式。

**本地安装。** 把宿主环境的*本地*实例指向你的工作树，而不是已发布的构建：

- **形态 A / C：** 从本仓库的本地路径安装插件/扩展（或把它的目录符号链接到宿主环境查找的位置）。在它的文档中找到宿主环境的"从本地目录 / git 检出安装"路径。
- **形态 B：** 注册本地模块——例如一个 `opencode.json` 的 `plugin` 条目指向本地路径，或 pi 从仓库解析 `package.json` 字段。

每次改动后重新安装并重启宿主环境，因为引导程序在启动时加载。

**用 tmux 驱动它。** 大多数宿主环境是交互式 REPL/TUI，无法通过管道输入 stdin 来驱动，所以在一个分离的 tmux 会话中运行宿主环境，并用 `send-keys` / `capture-pane` 控制它。宿主环境可能宣称有一个非交互式"run one prompt"模式（例如 `opencode run "..."`）——快速冒烟检查可以试试它，但**不要依赖它**：这些模式经常不稳定、受认证门控或受信任门控（有一个真实宿主环境的 `--print` 模式每次都挂起并超时、无任何输出）。准备好通过 tmux 做*所有*事情，包括冒烟检查。

**先清掉门，否则 tmux 会静默停滞。** 许多宿主环境会阻塞在首次运行的引导、一个"do you trust this folder?"提示、沙箱模式或权限门上——而分离的 tmux 会话在等待时会没有任何错误地干坐着。运行之前，预信任你的临时目录（在宿主环境的设置/配置中），或准备好通过 `send-keys` 回答那些提示，并在你的第一次 `sleep` 中计入宿主环境的启动时间。

```bash
# 1. 在一个一次性的项目目录中分离启动宿主环境
mkdir -p /tmp/port-smoke
tmux new-session -d -s port-test -c /tmp/port-smoke '<harness-launch-command>'

# 2. 让它初始化——真实 TUI 花的时间比你想象的长（带模型握手的 10s+）；调这个值。然后再捕获并清掉任何阻塞性的模态框，然后才输入提示：首次运行引导和"trust this folder?"都是模态的，所以期间发送的按键会选中菜单项而不是输入你的提示。
sleep 12
tmux capture-pane -t port-test -p          # 引导 / 信任提示？先用 send-keys 回答它
# （例如 tmux send-keys -t port-test Enter   # 接受信任提示——先检查再假设）

# 3. 冒烟检查：模型知道它有 superpowers 吗？
#    把文本和 Enter 作为 SEPARATE 的 send-keys 发送，中间加一拍——
#    一起发送会在某些 TUI 上竞争（Enter 先于文本落地到达）。
tmux send-keys -t port-test 'What are your superpowers?'; sleep 0.4; tmux send-keys -t port-test Enter
sleep 5
tmux capture-pane -t port-test -p          # 回复应显示它知道自己的技能

# 4. 验收测试：确切提示（注意转义的撇号），新会话
tmux send-keys -t port-test 'Let'\''s make a react todo list'; sleep 0.4; tmux send-keys -t port-test Enter
# 轮询直到该回合结束——每几秒重新捕获一次，不要只捕获一次
sleep 8
tmux capture-pane -t port-test -p          # PASS = brainstorming 在任何代码之前触发

# 5. 为 PR 保存会话记录，然后清理
tmux capture-pane -t port-test -p > /tmp/port-smoke/transcript.txt
tmux kill-session -t port-test
```

这里会咬人的 tmux 坑：启动后先等待再做第一次捕获；把提示文本和 `Enter` 作为*分离的* `send-keys` 调用发送，中间加一个短 `sleep`（一起发送会在某些 TUI 上竞争），而 `Enter` 是一个键名，不是 `\n`；代理的回合需要时间，所以要**循环轮询 `capture-pane`**，而不是只捕获一次；`capture-pane` 只显示可见的窗格，所以对长对话要用宿主环境自己的会话记录/日志文件作为真相记录；完成后务必 `kill-session`。

如果冒烟检查显示模型*不*知道它有 superpowers，那引导程序就没在加载——在费心做验收测试之前先修复它。

---

## 第 6 部分 —— 分发与发布

本仓库中一个能工作的集成，在真实用户能安装它之前是不可用的。分发因宿主环境生态而异——找到你的：

| 渠道 | 示例 | 你做什么 |
|---|---|---|
| 原生插件市场 | Claude Code | 在 `.claude-plugin/marketplace.json` 中注册；用户 `/plugin install`。外部的 `superpowers-marketplace` 仓库是用户安装的事实来源——见 `CLAUDE.md` 中的发布步骤。 |
| 由脚本同步的外部市场 fork | Codex | `scripts/sync-to-codex-plugin.sh` 把被跟踪的插件文件 rsync 到一个单独的 fork 仓库并打开 PR。阅读它的包含/排除列表，这样你分发正确的树（它刻意丢弃仓库内部目录和其他宿主环境的点目录）。 |
| Git-URL 扩展安装 | Gemini、Kimi Code、OpenCode | 用户从 git URL 安装（`gemini extensions install …`；Kimi Code `/plugins install …`；一个 `opencode.json` 的 `plugin` 数组条目）。记录确切命令。 |
| 包清单字段 | pi | 通过仓库根目录 `package.json` 中的字段声明；用户通过宿主环境的包命令安装。 |
| 本地安装器（插件安装） | Antigravity（`agy`） | 一个小 `install.sh`，对一个持有着清单、技能和生成的 `contextFileName` 上下文文件（引导程序）的暂存目录运行宿主环境自己的 `agy plugin install`。一切都通过安装机制到达——*不*通过编辑用户的配置（见下文）。 |

然后：

- **插件安装器可能静默剥离*未声明*的文件——所以让引导程序成为安装器*认识*的文件，绝不要成为用户配置编辑。** 一次 `plugin install` 通常只复制它知道的组件（skills/agents/commands/mcp/hooks/context），丢弃其他任何东西，所以清单未声明的上下文文件会从安装中消失。修复办法**不是**放弃并写进用户的配置（**规则 2**）——而是把引导程序声明为被认识的组件。按升级顺序：
  - **分发清单声明的上下文文件。** 如果宿主环境有 `contextFileName` 式字段（一个它每个会话都加载的、扩展声明的文件），那是最强、最干净的引导：声明它，安装器就保留它*并且*宿主环境加载它。在安装时从活的 `using-superpowers/SKILL.md` + 工具映射（包裹在 `<EXTREMELY_IMPORTANT>` 内）生成它，这样已安装的引导程序永不过时。`.antigravity-plugin/install.sh` 正是这样做的——`agy plugin install` 报告 `✔ context : ANTIGRAVITY.md`，而一个干净会话读取 `using-superpowers` 的 SKILL.md，加载 `brainstorming`，并在任何代码之前进入 brainstorming 流程。**用一个标记验证**安装器保留该文件且宿主环境加载它：有一个移植者错误地断定做不到，因为他们*没有*声明 `contextFileName` 就分发了文件，它作为无法识别的文件被剥离了。
  - **否则依靠已安装的 `using-superpowers` 技能本身。** 如果宿主环境在会话启动时浮出每个已安装技能的名称 + 描述，`using-superpowers` 的描述（"Use when starting any conversation…"）可以促使模型加载它——安装技能*就是*引导程序。更软（没有有保证的包装器；它承载触发，但不承载工具映射——见步骤 5），所以有可用时优先选择声明的上下文文件。
  - 如果两者都不行，这个宿主环境还不能被干净地支持——**说出来**并上报，而不是手工编辑用户的配置。

- **编写安装文档。** 一份 `docs/README.<harness>.md` 和/或一份 `.<harness>/INSTALL.md`（见 `docs/README.opencode.md` 和 `.opencode/INSTALL.md`），外加顶层 `README.md` 中的一个安装部分。唯一受支持的安装动作是**运行宿主环境自己的安装命令**（`agy plugin install`、`gemini extensions install`、`/plugin install` 等）。手工复制技能文件和编辑用户的全局/个人配置*都*是被禁止的（规则 2 / PR 规则）。如果宿主环境根本没有安装命令——它唯一的表面是用户拥有的配置文件——那么它无法满足"通过安装机制交付"的规则，你应该上报这一点，而不是分发一个编辑用户文件的安装器。
- **注册版本。** 如果你的宿主环境引入了一个*新的*带版本清单，把它的路径和版本字段添加到 `.version-bump.json`，让 `scripts/bump-version.sh` 使它与版本保持同步（读那个文件看看当前跟踪了什么）。一个没有在那里注册的新清单会分发过时的版本。如果你的宿主环境反而搭乘一个已被跟踪的文件——pi 在仓库根目录的 `package.json` 中声明自己，它已经在列表里——那就没有新增的东西了。
- **如果没有现有渠道匹配，你就是在建立一个新的。** 四行可能都不匹配你的宿主环境。如果它需要 Codex 式的外部 fork 同步，`scripts/sync-to-codex-plugin.sh` 是要克隆的模板（注意它锚定的包含/排除列表和它的 PR 自动化）。而且每当你添加一个新的每宿主环境目录，就把它加到*其他*宿主环境的同步排除中（例如 `sync-to-codex-plugin.sh` 中的 EXCLUDES 列表），这样你的点目录不会泄漏进它们的分发。

---

## 第 7 部分 —— 跨平台 / Windows

只与 shell 钩子形态相关。`hooks/run-hook.cmd` 是一个多语言脚本：一个同时作为 Windows 批处理脚本和 Unix shell 脚本都有效的单一文件。在 Windows 上，`cmd.exe` 运行批处理部分，它定位 `bash`（Git for Windows，然后是 PATH 上的 `bash`）并运行指定的钩子脚本；如果找不到 bash，它就干净地退出，这样宿主环境仍然工作，只是没有注入。在 Unix 上，开头的 `:` 让批处理块成为无操作，shell 直接运行脚本。

它强制了两条你必须尊重的规则：

- **钩子脚本是无扩展名的**（`session-start`，而不是 `session-start.sh`）。Claude Code 的 Windows 处理会给任何包含 `.sh` 的命令前置 `bash`，那会导致双重调用。给钩子脚本命名时不要带扩展名。
- 不要编写钩子脚本的按操作系统变体。一个无扩展名的 bash 脚本加上多语言包装器覆盖全部三个平台。

`hooks/run-hook.cmd` 本身是权威实现——读它。派发器模式背后的背景和原理见 `docs/windows/polyglot-hooks.md`。

---

## 第 8 部分 —— 提交 PR

- 以 **`dev`** 分支为目标。一个 PR 一个宿主环境。
- 填写 PR 模板的**"New harness support"**部分，并粘贴完整的验收测试会话记录（"Let's make a react todo list"会话，展示 `brainstorming` 自动触发）。没有这个证据的 PR 将被关闭。
- Superpowers 是一个零依赖插件。不要添加第三方运行时依赖。添加新宿主环境是贡献者规则允许的唯一例外，即便如此也要保持在集成严格所需的范围——编译期消失的仅类型导入可以；运行时包不行。
- 不要碰技能主体（第 1 部分）。如果你发现自己为了移植而编辑 `SKILL.md`，修复应该放在你的工具映射里。

---

## 附录 A —— 参考集成（当前）

把这个当作活索引；有疑问时读文件，而不是读这张表。

| 宿主环境 | 入口点 | 引导机制 | 工具映射 | 测试 | 分发 |
|---|---|---|---|---|---|
| Claude Code | `.claude-plugin/plugin.json` + `hooks/hooks.json` | shell 钩子 → `hooks/session-start`（`hookSpecificOutput.additionalContext`） | 原生 `Skill` 工具；无需适配文件 | `tests/hooks/` | 市场 |
| Codex | `.codex-plugin/plugin.json`（声明空的 `hooks`） | 原生技能发现（无会话启动钩子） | `references/codex-tools.md` | `tests/codex/`、`tests/codex-plugin-sync/` | fork 同步（`scripts/sync-to-codex-plugin.sh`） |
| Cursor | `.cursor-plugin/plugin.json` + `hooks/hooks-cursor.json` | shell 钩子 → `hooks/session-start`（`additional_context`） | 不需要（Claude Code 兼容的工具表面） | `tests/hooks/` | 手工编写 |
| Copilot CLI | （共享 Claude Code 钩子路径；`COPILOT_CLI` 环境变量） | shell 钩子 → `hooks/session-start`（`additionalContext`） | 不需要（Claude Code 兼容的工具表面） | `tests/hooks/` | — |
| Gemini CLI | `gemini-extension.json` + `GEMINI.md` | 指令文件 `@` 包含引导 + 映射 | `references/gemini-tools.md` | — | `gemini extensions install` |
| Kimi Code | `.kimi-plugin/plugin.json` | 清单 `sessionStart.skill` 加载 `using-superpowers` | 清单中内联的 `skillInstructions` | `tests/kimi/` | 市场或 `/plugins install` GitHub URL |
| OpenCode | `.opencode/plugins/superpowers.js`（通过根 `package.json` 的 `main` 声明） | 进程内：`config` 钩子注册技能目录；`experimental.chat.messages.transform` 注入用户消息 | 内联在 `superpowers.js` 中 | `tests/opencode/` | `opencode.json` 插件 git URL |
| pi | `.pi/extensions/superpowers.ts` | 进程内：`resources_discover` 注册技能；`context` 事件注入用户消息；生命周期标志 + 压缩感知 | `piToolMapping()` 内联**以及** `references/pi-tools.md` | `tests/pi/` | 仓库根目录 `package.json` 字段 |

## 附录 B —— 曾咬过移植者的坑

- **主动选择不是移植。** 如果你的人类伙伴必须为获得 Superpowers 而每个会话做点什么，验收测试就会失败。重读第 2 部分。
- **错误的 JSON 字段 → 静默失败或双重注入。** 仅形态 A。确认确切的字段/嵌套；Claude Code 读取两个字段而不去重。
- **钩子配置模式因宿主环境而异。** 形态 A。Cursor 的 `hooks-cursor.json` 看起来和 Claude Code 的完全不像（`version`、小写 `sessionStart`、相对命令、无 `matcher`/`type`/`async`）。匹配最接近的现有文件。
- **插件根环境变量因宿主环境而异。** 形态 A。钩子命令使用 `${CLAUDE_PLUGIN_ROOT}`（Claude）或相对路径（Cursor）。使用你的宿主环境导出的；脚本自己重新推导根路径。
- **系统消息注入。** 形态 B 有意注入*用户*消息（#750、#894）。不要把它"修"成系统消息。
- **每步与每回合回调。** OpenCode 每步触发（每次调用的去重守卫）；pi 每回合触发（生命周期标志 + `agent_end` 重置）。把一个宿主环境的去重策略复制到另一个的回调频率上会破坏注入。
- **消息对象形态因宿主环境而异。** 形态 B。pi 和 OpenCode 使用不兼容的形态；发现你自己的，不要复制参考实现的对象字面量。
- **寻找一个不存在的技能注册 API。** 一个没有技能系统（而不只是没有 `Skill` 工具）的宿主环境没有什么可注册的——模型按需读取 `SKILL.md`。不要假设存在 `skillPaths` 等价物。
- **映射放在两个地方。** 对进程内插件，映射可能既内联在 `references/` 文件中（pi）。两者都要更新。
- **"绝不要读取技能文件"这句。** 它的意思是"不要绕过你平台的技能加载机制"，不是"绝不要用文件读取"。在一个没有技能工具的宿主环境上，那个机制*就是*读取 `SKILL.md`——在映射中明确说明这一点（第 5 部分）。
- **Windows 上的 `.sh`。** 保持钩子脚本无扩展名（第 7 部分）。
- **未注册的版本。** 一个没有添加到 `.version-bump.json` 的新清单会分发过时版本（第 6 部分）。
- **编辑技能来适应宿主环境。** 绝不要。修复放在工具映射里。
