# Codex 效率修复——设计

日期：2026-07-30
状态：由 Jesse 批准（会话内）
分支：自 `dev` 分出的 `codex-efficiency-fixes`

## 来源

- Eval 活动收尾：`superpowers-autoresearch/reports/2026-07-codex-efficiency-campaign.md`（处理表 §4；下方每项处理都有计分者和测得的 `dev` 基线）。
- Codex 源码侦察：`superpowers-autoresearch/docs/2026-07-29-codex-multiagent-v2-capabilities.md`（对 Codex CLI 源码的 file:line 引用；为 T2、T3、T5 提供依据）。
- 已发表的实验报告：`superpowers-evals/docs/experiments/`。
- Drew 的拆出栈（PR #2036、#2035）是**证据，而非采纳的文本**：Jesse 想在采纳其中任何一项之前更深入地研究那些修复；它们只用于形成问题陈述。

## 目标

把 codex-efficiency eval 活动中证据最强的五项处理作为 superpowers 技能/文档变更发布，每项在裁剪其 PR 之前都按其预先登记的标准由活动计分者评分。第二阶段（收尾处理表中的其余所有项）随后进行，每个条目首先以新的基线工作为门。

## 范围决策（与 Jesse 已敲定）

- **第一阶段 = 证据最强的五项**（下面的 T1–T5）。第二阶段条目每一项都需要一个失败的基线，任何修复才能发布（判别规则：零结果不明确即为停止）。
- **一个分支，每个处理一个 PR。** 开发与电池测试在 `codex-efficiency-fixes` 上进行；当一项处理胜过其标准时，它连同其 eval 证据被裁剪成针对 `dev` 的独立 PR。未经 Jesse 逐 PR 批准不得合并。
- **T4 以全局回归电池测试跨 harness 发布**（Claude Code、Codex、Gemini），采用变体 C 形态：仪式可扩展，批准永不扩展。

## 五项处理

### T1. SDD worker 审查禁令

**证据：** 4 个语料库中的 9/9 个深度-2 派生全部是 implementer 发出的审查者；全部 9 个都是控制器无论如何都会派发的审查的同任务重复。派发契约从未说明审查不是 worker 的职责；在子进程可以派生的 harness（Codex）上，implementer 提示词中的"self-review"被具体化为一个审查者 subagent。

**变更：**
- `skills/subagent-driven-development/implementer-prompt.md`：一条显式的"You do not dispatch subagents"条款——self-review 意味着阅读你自己的 diff；控制器拥有所有审查派发；你派生的审查者复制了流程已提供的审查。
- `skills/subagent-driven-development/SKILL.md`：任务循环中的一条派发契约行，外加一条 Red Flags 行："An independent review would strengthen my report" → 审查是控制器的下一步；你的审查者是一个重复座位。
- 与 harness 无关的措辞（在子进程无法派生的地方为空操作）。

**评分者：** `score_e6.py`（按派生者角色的深度-2 派生、重复审查家族）；同范围变体用 `score_e5.py`。
**基线：** 9/9 由 worker 发出，0 个反例。
**标准：** 0 次 worker 发出的深度-2 派生，且审查覆盖保持（每个任务仍恰好得到一次控制器派发的任务审查）。

### T2. 事件驱动等待

**证据：** 每个语料库中 60–78% 的 `wait_agent` 调用超时（dev 67.1%，拆出 60.2%）。源码侦察：V2 等待是事件订阅，而非轮询——一次长等待的唤醒延迟与 10 秒轮询相同，而调用量约为其 1/90；一个已完成子进程的 FINAL_ANSWER 被推入父进程的 mailbox，并在下一次模型请求中零等待地排空。

**变更**（`skills/using-superpowers/references/codex-tools.md`）：
- 绝不短超时轮询。
- 当仍有本地工作可做时，不要等待——子进程结果随你的下一轮通过 mailbox 到达。
- 真正空闲时，发出 ONE 次 `wait_agent`，带长 `timeout_ms`（900000+；harness 上限 3600000）。
- 说明 V2 注意事项：完成邮件携带 `trigger_turn=false`，不会唤醒空闲的控制器——这正是 `wait_agent` 唯一的职责。

**评分者：** `score_e7.py`（超时率、轮询间隔节奏、缓存回扣估算——回扣数字保持标记为估算值）。
**基线：** dev 超时率 67.1%。
**标准：** 超时率 < 25% 且不损失任务完成率。

### T3. codex-tools.md 更正

**证据：** 当前指导中的五条主张与 Codex 源码相矛盾（全部在能力文档中 file:line 引用）：
1. 多智能体 V2 中不存在 `close_agent`（仅 V1 有）。V2 会自动 LRU 驱逐已完成的子进程；不关闭零成本；`followup_task` 透明地重新加载一个被驱逐的子进程。
2. 修复轮次总能通过 `followup_task` 恢复 implementer——dev 的"如果你的 harness 无法向已派生的 agent 发送另一条消息，就把每个修复轮次作为新的 implementer 派发"分支在 V2 上已死。
3. 角色文件（`~/.codex/agents/**.toml`）确实会通过隔离 fork（0.145+）上的 `agent_type` 附加到派生。
4. 全历史 fork 接受 `model`/`reasoning_effort` 覆盖；只有 `agent_type` 被拒绝。（出于上下文卫生原因，隔离 fork 仍是 SDD 指导，此说明是准确的。）
5. 派发指导绝不能点名非 V2 的模型预设——V2 派生允许列表只有 v2 预设；其他的会硬报错。

**变更：** 重写 `skills/using-superpowers/references/codex-tools.md` 的多智能体段落，使其在版本上诚实（在 V1 与 V2 行为有差异处标注）。

**评分者：** 源码引用（已验证）；共享电池上无计分者回归。`score_e8.py` 保留为 V1/V2 schema 检测器，而非卫生评分器——不发布 `close_agent` 检查清单。

### T4. Brainstorming 三路径路由器（变体 C：始终批准）

**证据：** 微测——当前的 HARD-GATE 文本把有界任务推到 FULL 仪式 5/5，而 Z-null（无指导）和三路径路由器都能 5/5 区分：绝对化措辞压制了模型本能的判别力。FULL 电池——仪式量适度扩展（16.7 对 24.0 次工具调用，有界对架构），但双文档仪式（规范文件 → 计划文件）在每个代表中都无条件运行。测得的浪费是无条件产物仪式，而非批准门。

**设计（变体 C）：** 三条路径扩展的是产物；每条路径在实现之前都保留人工批准：
- **Spike**（可行性问题，明确可弃）：用 2–3 句话呈现问题和预期的探针，得到点头，开工。无文档。发现作为建议返回；任何构建出来的东西都保持标记为可弃。
- **Bounded**（对现有、已理解流程的范围良好变更）：在聊天中呈现一个简短设计，获得批准，实现。无规范文件，无 writing-plans 调用。
- **Architectural**（重构组件、新子系统、公开接口变更）：完整的当前流程——规范文档、审查、writing-plans。

**守卫（全部随路由器一起发布）：**
- 分类被大声说出（"这看起来是有界的，所以我在这里呈现一个简短设计，而不是写规范"），以便人类可以覆盖。
- 两个路径之间拿不准时，走更重的那个。
- 单向棘轮：路径中途发现的隐藏复杂性升级路径；绝不在任务中途降级。
- 新增 Red Flags 行，针对把分类当作逃生舱的做法（"我把它叫有界以跳过文档"）。

**变更**（`skills/brainstorming/SKILL.md`）：HARD-GATE 保留"批准之前不得实现"，去掉作为仪式驱动力的"regardless of perceived simplicity"；反模式部分重新表述（罪过是跳过批准，而非跳过文档）；检查清单步骤 6–9 变为架构路径；流程流转图新增路由器；添加 Red Flags 行。这是精心调优的内容——编辑遵循 writing-skills 方法论，且只在附带下方完整 eval 证据时发布。

**评分者（三层）：**
1. **微测**（`ceremony-path-micro.py`，适配）：变体 C 字面文本，外加活动从未测过的对抗性含糊 brief（一个模式匹配有界、却隐藏公开接口变更的任务）。标准：spike/bounded/arch 能区分（每单元 ≥4/5）；含糊 brief 升级到 FULL（≥4/5）；arch 绝不降级（5/5）。
2. **Codex 仪式电池：** 修复臂上的 `cx-ceremony-{spike,bounded,arch}`，每项 3 个代表，`score_e4.py` 普查。标准：有界代表显示一个批准轮，但零个已提交的规范文件且零 writing-plans 仪式；arch 代表保留完整的双文档流程；spike 代表保持最小。
3. **全局回归电池：** 相同的三个仪式场景在 Claude Code 和 Gemini 上运行（rig 工作：那些场景目前受 codex 门控），每项 3 个代表；外加在全部三个 harness 上的触发接受度检查（"Let's make a react todo list" 自动触发 brainstorming 进入完整/架构路径）。

### T5. 子进程发出的派发上显式模型

**证据：** CLI 0.146 上根派生 100% 显式模型（dev 14/14）；活的缺口在深度-2——2/2 个由子进程发出的派生省略了 `model`。源码侦察：不带 `reasoning_effort` 的 `model` 会把 effort 重置为 MODEL 的默认值，而非父进程的。

**变更**（`skills/using-superpowers/references/codex-tools.md`）：
- 你发出的每个派生——包括作为子进程时——都设置 `model` 和 `reasoning_effort`；点名 effort 重置陷阱。
- 建议把 `~/.codex/config.toml` 中的 `[agents].default_subagent_model` 和 `[agents].default_subagent_reasoning_effort` 作为机器级兜底，为任何漏网的派生兜底。

**评分者：** 共享电池上的 `score_e1.py`（按深度的每派生显式模型率）。
**基线：** 深度-2：0/2 显式。
**标准：** 每个深度的每次派生都携带显式 model + effort。预先登记的注意事项：如果 T1 完全消除深度-2 派生，则 T5 按根派生回归（保持 100%）加文档正确性评分，并在深度-2 记为零结果不明确——此时配置兜底是实际起作用的机制。

## 评分计划

- **共享 SDD 电池**承载 T1、T2、T5：`cx-sdd-small`，修复分支臂（`/tmp/sp-arm-fix`），两个容器通道各 8 个代表。dev 基线已测得；不重跑基线。
- **T4 电池**如上所列（微测 + codex 仪式 + 全局回归）。
- **预先登记：** 每个电池在运行之前都在 `superpowers-autoresearch/logs/2026-07-30-codex-efficiency-fixes.md` 中有一条假设日志条目（预测、计分者、标准）。常设规则延续：只追加的日志、修复臂运行上的计分者匹配人工检查（非循环验证）、不提交原始 rollout、每个裁决中正确性与成本并驾齐驱。
- **归因：** 一个合并分支上的正交计分者；意外回归按处理提交二分。
- **预算：** 共享电池约 $40，codex 仪式约 $40，全局回归约 $40–80，微测约 $5 → 第一阶段约 $150–200，出自活动 $1000 中剩余的约 $850。

## 流程

- 工作在 `codex-efficiency-fixes` worktree 中进行（自 `dev` 分叉）；通过 subagent-driven-development 从一份书面计划执行。
- 技能文本变更遵循 writing-skills 方法论。
- 场景/rig 变更（为 Claude Code/Gemini 取消仪式场景的门控、对抗性微 brief）按授权落在 `superpowers-evals` main 上。
- 每处理一个针对 `dev` 的 PR，各自带其 eval 证据和标准身份块；仅在 Jesse 逐 PR 批准后合并。

## 第二阶段队列（基线优先；不在本计划的任务内）

每个条目在任何修复发布之前都需要一个失败的基线：
1. **派发路由 / 长会话漂移**——需要一个长会话引导 rig（全新会话在 CLI 0.146 上无法复现该病理）。Drew 的栈为处理形态提供信息。
2. **验证租约 / 证据收据**——需要先给 `score_e3.py` 添加子串感知的重复计数器（当前 1/23 个精确字符串对的基础太弱）。
3. **补救上限**——小 n 基线（2/3 个代表）需要更多代表。
4. **跨任务竞争探针重新设计**——`score_e5.py` 的探针按设计权衡为零结果不明确；需要一个更强的探针。
5. **E5 D4 shell 命令解析器**——修复审查范围分类器无法解析复合命令；这是计分者工作，而非技能工作。

## 范围之外

- 采纳 Drew 的拆出栈（#2036/#2035）或其文本。
- RoboRev、Codex token 遥测（独立代码库）。
- `close_agent` 卫生检查清单（V2 没有这样的工具——在活动中以 do-not-ship 关闭）。
- 超出 T4 回归电池范围的 Claude Code/Gemini 特定效率处理。
