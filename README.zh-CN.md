# Superpowers

Superpowers 是一套为你的编码代理打造的完整软件开发方法论，构建在一组可组合的技能之上，外加一些确保你的代理会使用它们的初始指令。

## 目录

- [它的工作原理](#how-it-works)
- [商业服务](#commercial-services)
- [开始使用](#installation)
  - [Claude Code](#claude-code)
  - [Antigravity](#antigravity)
  - [Codex App](#codex-app)
  - [Codex CLI](#codex-cli)
  - [Cursor](#cursor)
  - [Devin CLI](#devin-cli)
  - [Factory Droid](#factory-droid)
  - [Gemini CLI](#gemini-cli)
  - [GitHub Copilot CLI](#github-copilot-cli)
  - [Grok Build CLI](#grok-build-cli)
  - [Kimi Code](#kimi-code)
  - [OpenCode](#opencode)
  - [Pi](#pi)
  - [Hermes Agent](#hermes-agent)
- [基本工作流](#the-basic-workflow)
- [社区](#community)
- [里面有什么](#whats-inside)
- [理念](#philosophy)
- [贡献](#contributing)
- [更新](#updating)
- [许可证](#license)
- [可视化伴侣遥测](#visual-companion-telemetry)

## 它的工作原理

它从你启动编码代理的那一刻开始。一旦它看到你在构建某个东西，它就*不会*直接跳进尝试写代码。相反，它会退后一步，问你到底想做什么。

当它从对话中提炼出一份规格说明后，会以足够简短、能让你真正阅读和消化的分块形式展示给你。

在你签字认可设计之后，你的代理会整理出一份实现计划，其清晰程度足以让一位热情洋溢、但品味不佳、毫无判断力、没有项目背景、又讨厌测试的初级工程师都能照做。它强调真正的红/绿 TDD、YAGNI（你不会需要它）和 DRY。

接下来，一旦你说"开始"，它就会启动一个 *subagent-driven-development*（子代理驱动开发）流程，让代理逐项处理每个工程任务，检查并审查他们的工作，然后继续推进。你的代理不偏离你制定的计划、自主工作一两个小时，这种情况并不少见。

还有很多其他内容，但那就是这个系统的核心。而且因为技能会自动触发，你不需要做任何特殊的事情。你的编码代理只是拥有了 Superpowers。

## 商业服务

如果你在企业中使用 Superpowers，并希望获得商业支持、额外工具或托管式支出管理，请随时联系我们：sales@primeradiant.com。

## 安装

安装方式因宿主环境而异。如果你使用多个宿主环境，请为每个宿主环境分别安装 Superpowers。

### Claude Code

Superpowers 可通过 [官方 Claude 插件市场](https://claude.com/plugins/superpowers) 获得

#### 官方市场

- 从 Anthropic 的官方市场安装插件：

  ```bash
  /plugin install superpowers@claude-plugins-official
  ```

#### Superpowers 市场

Superpowers 市场为 Claude Code 提供 Superpowers 及其他一些相关插件。

- 注册市场：

  ```bash
  /plugin marketplace add obra/superpowers-marketplace
  ```

- 从此市场安装插件：

  ```bash
  /plugin install superpowers@superpowers-marketplace
  ```

### Antigravity

从此仓库将 Superpowers 作为插件安装：

```bash
agy plugin install https://github.com/obra/superpowers
```

Antigravity 会运行插件的会话启动钩子，因此 Superpowers 从第一条消息起就处于激活状态。用相同命令重新安装即可更新。

### Codex App

Superpowers 可通过 [官方 Codex 插件市场](https://github.com/openai/plugins) 获得。

- 在 Codex 应用中，点击侧边栏中的 Plugins。
- 你应该会在 Coding 部分看到 `Superpowers`。
- 点击 Superpowers 旁边的 `+` 并按照提示操作。

### Codex CLI

Superpowers 可通过 [官方 Codex 插件市场](https://github.com/openai/plugins) 获得。

- 打开插件搜索界面：

  ```bash
  /plugins
  ```

- 搜索 Superpowers：

  ```bash
  superpowers
  ```

- 选择 `Install Plugin`。

### Cursor

- 在 Cursor Agent 聊天中，从市场安装：

  ```text
  /add-plugin superpowers
  ```

- 或者直接在插件市场中搜索"superpowers"。

### Devin CLI

- 从此仓库安装插件：

  ```bash
  devin plugins install obra/superpowers
  ```

- 用以下命令更新到最新版本：

  ```bash
  devin plugins update superpowers
  ```

### Factory Droid

- 注册市场：

  ```bash
  droid plugin marketplace add https://github.com/obra/superpowers
  ```

- 安装插件：

  ```bash
  droid plugin install superpowers@superpowers
  ```

### Gemini CLI

- 安装扩展：

  ```bash
  gemini extensions install https://github.com/obra/superpowers
  ```

- 稍后更新：

  ```bash
  gemini extensions update superpowers
  ```

### GitHub Copilot CLI

- 注册市场：

  ```bash
  copilot plugin marketplace add obra/superpowers-marketplace
  ```

- 安装插件：

  ```bash
  copilot plugin install superpowers@superpowers-marketplace
  ```

### Grok Build CLI

Superpowers 可通过 [官方 Grok 插件市场](https://github.com/xai-org/plugin-marketplace) 获得。

- 从 xAI 的官方市场安装插件：

  ```bash
  grok plugin install superpowers@xai-official --trust
  ```

- 或者在 TUI 中打开市场，搜索 Superpowers 并安装它：

  ```text
  /marketplace
  ```

### Kimi Code

Superpowers 在 Kimi Code 的插件市场中可用。

- 打开 Kimi Code 的插件管理器：

  ```text
  /plugins
  ```

- 转到 `Marketplace` > `Superpowers` 并安装它。

- 或者直接从本仓库安装：

  ```text
  /plugins install https://github.com/obra/superpowers
  ```

- 详细文档：[docs/README.kimi.md](docs/README.kimi.md)

### OpenCode

OpenCode 使用自己的插件安装方式；即使你已在另一个宿主环境中使用 Superpowers，也要单独安装它。

- 告诉 OpenCode：

  ```
  Fetch and follow instructions from https://raw.githubusercontent.com/obra/superpowers/refs/heads/main/.opencode/INSTALL.md
  ```

- 详细文档：[docs/README.opencode.md](docs/README.opencode.md)

### Pi

从此仓库将 Superpowers 作为 Pi 包安装：

```bash
pi install git:github.com/obra/superpowers
```

对于本地开发，使用此检出作为临时包来运行 Pi：

```bash
pi -e /path/to/superpowers
```

Pi 包会加载 Superpowers 技能，以及一个在会话启动时以及压缩之后注入 `using-superpowers` 引导程序的小型扩展。Pi 拥有原生技能，因此不需要兼容性的 `Skill` 工具。Subagent 和任务列表工具仍然是可选的 Pi 配套包。

### Hermes Agent

从此仓库将 Superpowers 作为 Hermes 插件安装：

```bash
hermes plugins install obra/superpowers --enable
```

安装后，请重启任何处于活动状态的 Hermes 会话。注意：Hermes 没有压缩后钩子，因此一个在首个回合就发生压缩的极长会话会丢失引导程序——如果技能停止触发，请开始一个新会话。

## 基本工作流

1. **brainstorming** - 在编写代码之前激活。通过提问来打磨粗略的想法，探索备选方案，分节呈现设计以供验证。保存设计文档。

2. **using-git-worktrees** - 在设计批准后激活。在新分支上创建隔离的工作区，运行项目设置，验证干净的测试基线。

3. **writing-plans** - 在设计批准后激活。将工作拆分成小任务（每个 2-5 分钟）。每个任务都有确切的文件路径、完整代码和验证步骤。

4. **subagent-driven-development** 或 **executing-plans** - 有计划时激活。为每个任务派发全新的 subagent，并进行两阶段审查（先规格合规，再代码质量），或以批处理方式执行并设置人类检查点。

5. **test-driven-development** - 在实现期间激活。强制遵循 RED-GREEN-REFACTOR：先写失败的测试，看着它失败，再写最少的代码，看着它通过，然后提交。会删除在测试之前写的代码。

6. **requesting-code-review** - 在任务之间激活。对照计划进行审查，按严重程度报告问题。严重问题会阻止进度。

7. **finishing-a-development-branch** - 当任务完成时激活。验证测试，呈现选项（合并/PR/保留/丢弃），清理 worktree。

**代理会在任何任务之前检查相关技能。** 这些是强制性工作流，而非建议。

## 社区

Superpowers 由 [Jesse Vincent](https://blog.fsck.com) 和 [Prime Radiant](https://primeradiant.com) 的其他成员共同构建。

- **Discord**：[加入我们](https://discord.gg/35wsABTejz) 获取社区支持、提问，并分享你用 Superpowers 构建的内容
- **Issues**：https://github.com/obra/superpowers/issues
- **发布公告**：[注册](https://primeradiant.com/superpowers/) 以获取新版本通知

## 里面有什么

### 技能库

**测试**
- **test-driven-development** - RED-GREEN-REFACTOR 循环（包含测试反模式参考）

**调试**
- **systematic-debugging** - 4 阶段根因流程（包含 root-cause-tracing、defense-in-depth、condition-based-waiting 技术）
- **verification-before-completion** - 确保它确实被修复了

**协作**
- **brainstorming** - 苏格拉底式的设计打磨
- **writing-plans** - 详细的实现计划
- **executing-plans** - 带检查点的批量执行
- **dispatching-parallel-agents** - 并发 subagent 工作流
- **requesting-code-review** - 预审查清单
- **receiving-code-review** - 回应反馈
- **using-git-worktrees** - 并行开发分支
- **finishing-a-development-branch** - 合并/PR 决策工作流
- **subagent-driven-development** - 带两阶段审查（先规格合规，再代码质量）的快速迭代

**元**
- **writing-skills** - 遵循最佳实践创建新技能（包含测试方法论）
- **using-superpowers** - 技能系统入门

## 理念

- **测试驱动开发** - 永远先写测试
- **系统性优于临时性** - 流程优于猜测
- **降低复杂性** - 简单是首要目标
- **证据优于声明** - 在宣布成功之前先验证

阅读[最初的发布公告](https://blog.fsck.com/2025/10/09/superpowers/)。

## 贡献

Superpowers 的通用贡献流程如下。请记住，我们一般不接受新技能形式的贡献，而且任何对技能的更新都必须在我们所支持的所有编码代理上正常工作。

1. Fork 本仓库
2. 切换到 'dev' 分支
3. 为你的工作创建一个分支
4. 遵循 `writing-skills` 技能来创建和测试新的或修改过的技能
5. 提交 PR，务必填好 pull request 模板。

技能行为测试使用来自 [superpowers-evals](https://github.com/prime-radiant-inc/superpowers-evals/) 的 drill 评估宿主环境，克隆到 `evals/` —— 设置方法见 `evals/README.md`。插件基础设施测试位于 `tests/`，通过相关的 `run-*.sh` 或 `npm test` 运行。

完整指南请参阅 `skills/writing-skills/SKILL.md`。

## 更新

Superpowers 的更新在一定程度上依赖于编码代理，但通常是自动的。

## 许可证

MIT License - 详情见 LICENSE 文件

## 可视化伴侣遥测

因为技能和插件不会向创作者提供任何反馈，所以我们完全不知道有多少人在使用 Superpowers。默认情况下，brainstorming 可选可视化伴侣功能上的 Prime Radiant 徽标是从我们的网站加载的。它包含正在使用的 Superpowers 版本。它不包含关于你的项目、提示或编码代理的任何细节。我们看不到你的点击，也看不到你在构建什么。这帮助我们大致了解有多少人在使用 Superpowers，以及他们使用的是哪个版本。它 100% 可选。要禁用它，请将环境变量 `SUPERPOWERS_DISABLE_TELEMETRY` 设置为任何真值。Superpowers 也尊重 Claude Code 的 `DISABLE_TELEMETRY` 和 `CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC` 退出选择。
