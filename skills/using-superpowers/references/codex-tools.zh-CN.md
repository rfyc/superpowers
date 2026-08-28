## 子代理派发需要多代理（multi-agent）支持

添加到你的 Codex 配置（`~/.codex/config.toml`）：

```toml
[features]
multi_agent = true
```

这会启用诸如 `dispatching-parallel-agents` 和 `subagent-driven-development`
这类技能所使用的多代理工具。你获得的工具取决于你的模型预设所选定的多代理
版本（当前预设运行 V2；较旧的运行 V1）。当实际工具列表与本表——包括本表——
不一致时，以实际工具列表为准。

- **生成（Spawning）：** 使用 `spawn_agent {fork_turns: "none"}` 为子代理提供
  干净上下文；默认的 `"all"` 会将你的完整对话记录复制进子代理。在 Codex 0.145+
  上，`~/.codex/agents/` 下的角色文件通过 `agent_type` 附加到隔离分支
  （isolated forks）。完整历史分支接受 `model` 和 `reasoning_effort` 覆盖
  （此处仅拒绝 `agent_type`）——隔离分支是 SDD 出于上下文卫生的默认选择，
  而不是因为覆盖需要它们。
- **修复轮次（Fix rounds）：** 使用 `followup_task` 恢复实现者（implementer）
  ——它会送达你的消息、触发一轮对话，并透明地重新加载被运行环境逐出的子代理。
  切勿基于"已生成的代理无法再次被发送消息"的理论派发全新的实现者；在 V2 上
  它始终可以。
- **生命周期（Lifecycle）：** V2 没有 `close_agent`。当需要槽位时，已完成的
  子代理会被自动逐出；不关闭它们不会产生任何代价。只有 V1 会话才有
  `close_agent`——在 V1 中，当审查者（reviewer）返回审查结果后关闭它们，
  并在每个实现者的任务审查通过后关闭每个实现者。
- **模型名称（Model names）：** 切勿从技能、表格或旧会话中复制模型名称到
  `spawn_agent`，除非先对照你当前的生成允许列表进行检查——V2 仅接受具备 V2
  能力的预设，对其余预设直接硬性报错。

## 等待子代理

`wait_agent` 是一种事件订阅，而非轮询：一次长等待会在子代理产生邮箱活动的
那一刻被唤醒，延迟与短等待相同。短超时轮询毫无收益，却每次轮询都消耗一次
工具调用——以及一次上下文重新计费。在实测会话中，约三分之二的等待调用都是
超时的短轮询。

- 当你仍有本地工作要做时，完全不要等待。已完成的子代理的最终答案会被推送
  到你的邮箱，并在你的下一轮对话时到达。
- 当你确实空闲而仍有未完成的子代理时，按有界的时间段等待：`wait_agent` 的
  `timeout_ms` 设为 300000-600000（5-10 分钟）。每段等待之后——无论唤醒还是
  超时——发布一行状态、运行 `list_agents`，并跟进任何已结束但未汇报的子代理。
  切勿堆叠短于五分钟的轮询；事件订阅唤醒一个有界等待段的速度与短等待相同。
- 完成邮件无法唤醒空闲的控制器（它在不触发新一轮对话的情况下送达）；覆盖
  那段空闲窗口正是 `wait_agent` 的唯一职责。一段无任何活动的等待超时，是你的
  提示去核对状态，而不是去缩短下一段等待。

## 生成时的模型路由

你发出的每一个 `spawn_agent`——包括当你自己是一个执行扇出（fan-out）的被生成
子代理时——都会根据你正在执行的技能的模型选择（Model Selection）规则，显式地
设置 `model` 和 `reasoning_effort`。仅设置 `model` 是一个陷阱：子代理的
reasoning_effort 会静默重置为该模型的默认值，而不是你的。

请你的人类搭档在 `~/.codex/config.toml` 中添加一个机器级兜底，以便任何漏网的
生成仍路由到一个经过考量的层级，而不是静默继承会话中最昂贵的模型：

```toml
[agents]
default_subagent_model = "<a mid-tier model from your spawn allowlist>"
default_subagent_reasoning_effort = "medium"
```

## 环境检测

创建 worktree 或完成分支的技能在继续之前，应使用只读的 git 命令检测其环境：

```bash
GIT_DIR=$(cd "$(git rev-parse --git-dir)" 2>/dev/null && pwd -P)
GIT_COMMON=$(cd "$(git rev-parse --git-common-dir)" 2>/dev/null && pwd -P)
BRANCH=$(git branch --show-current)
```

- `GIT_DIR != GIT_COMMON` → 已在关联的 worktree 中（跳过创建）
- `BRANCH` 为空 → 处于游离的 HEAD（detached HEAD）（无法从沙箱创建分支/推送/发起 PR）

参见 `using-git-worktrees` 第 0 步和 `finishing-a-development-branch`
第 1 步，了解各技能如何使用这些信号。

## Codex App 收尾

当沙箱阻止分支/推送操作（在外部管理的 worktree 中处于游离 HEAD）时，代理会
提交所有工作，并告知用户使用 App 的原生控件：

- **"Create branch"（创建分支）** —— 命名分支，然后通过 App 界面提交/推送/发起 PR
- **"Hand off to local"（移交给本地）** —— 将工作转移到用户的本地检出（checkout）

代理仍然可以运行测试、暂存文件，并输出建议的分支名、提交信息和 PR 描述
供用户复制。
