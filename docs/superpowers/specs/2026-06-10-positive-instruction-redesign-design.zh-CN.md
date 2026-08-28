# 技能指导的正向指令重设计——设计规格

**状态：** 提议（作为 2026-06-09 SDD 审查派发工作的后续；按一问题一 PR 规则单独 PR）
**动因：** 有测量证据（2026-06-10）表明技能行文中的一些负面指令会适得其反，而另一些有效——且差异是可预测的。

## 本规格所泛化的实测发现

2026-06-10 的微测试（opus，每种措辞 5 次重复，程序化评分；harness 如下所述）测量了指导措辞如何改变控制器所编写的内容：

| 用例 | 措辞 | 结果 |
|---|---|---|
| 派发组合（"不要复述简报"） | 禁止 | **4.4** 个规格值被重新输入——*比没有指导还糟*（3.6） |
| 派发组合 | 正向配方（"你的派发应包含：（1）…（5）"） | **3.0，零方差**——被采纳 |
| 派发组合 | 配方 + 细微差别条款（"只引用片段…"） | 3.8，嘈杂——细微差别稀释配方 |
| 测试重跑指令（"不要要求审查者重新运行测试"） | 禁止 | **0/5 违规**——很好用（对照：3/5） |
| 测试重跑指令 | 正向配方 | 0/5——相同，但更长 |

**信条**（用它来分类任何负面指令）：

1. **绊线（tripwire）有效。** 针对具体 token 的短语级自检（"如果你正在编写的提示词包含 'do not flag' … 停止"）可靠触发。
2. **识别表有效。** Red-Flags/合理化表在决策时读取，而不是组合时。
3. **离散指令禁止有效。** "不要让 X 做 Y" 在模型对做 Y 没有竞争性激励时成立。
4. **组合禁止会适得其反**，当模型对输出有它自己的议程时（例如，复述规格感觉像是有用的整理）。只有正向组合配方能推动这些——而且给制胜配方添加细微差别条款会使其更糟，而非更好。
5. **平局时选更短措辞。** Codex 在长会话中重新读取 SKILL.md 约 500 次（2026-06-10 测量）；行文长度是真实成本。

## 审计结果（2026-06-10，全部约 30 个技能 + 提示词模板）

计数：3 条绊线（保留）、14 个识别表（保留）、约 20 个策略门控（保留——"未经许可绝不推送"是策略，不是组合塑造）、5 个组合禁止：

| # | 位置 | 处置 |
|---|---|---|
| 1 | `subagent-driven-development/task-reviewer-prompt.md`——"Cite, don't narrate" | **已排队到 PR #1717 批次**：以正向半句开头（"你的报告应指向证据：每个发现的 file:line…"），删除禁止半句（死重——正向半句已存在并承担负载） |
| 2 | `subagent-driven-development/SKILL.md`——"不要添加开放式指令" | **原样保留**：微测试在 15 个样本中无法引出失败；两种方式都没有证据；更短者胜 |
| 3 | `subagent-driven-development/SKILL.md`——"不要要求审查者重新运行测试" | **原样保留**：实测 0/5 违规；禁止还有效地把自身传播到派发中 |
| 4 | `subagent-driven-development/SKILL.md`——"不要在其之上重新审查" | **已排队到 PR #1717 批次**：替换为三元素检查清单（"重新派发审查者前，确认修复报告包含：覆盖的测试、运行的命令和输出"） |
| 5 | `writing-plans/SKILL.md`——"No Placeholders" 禁止模式列表 | **本规格的主要主题**——见下文 |

边界情况，与 #5 一起推迟：`task-reviewer-prompt.md` 中的 "Don't flag pre-existing file sizes — focus on what this change contributed"（正向半句存在且承重；影响低；如方便可与 #5 一起测试）。

## writing-plans 变更（推迟项 #5）

### 当前状态

`skills/writing-plans/SKILL.md`，"No Placeholders"：一句正向陈述（"Every step must contain the actual content an engineer needs"），后跟一个六条禁止模式列表（"never write them: 'TBD', 'TODO', 'Add appropriate error handling', 'Write tests for the above', 'Similar to Task N', …"）。

### 为什么它重要，为什么它确实不确定

- 计划是工作流中**最大的生成产物**，模型有真实的竞争性激励去发出占位符（在长度压力下它们是最省力的路径）——这是禁止被实测为适得其反的那个用例的激励结构。
- 但被禁止的项是**离散、可识别的 token**——这是禁止被实测成立的用例的形状。
- **该列表在其他地方承重：** 技能的 Self-Review 部分引用它（"Placeholder scan: search your plan for red flags — any of the patterns from the 'No Placeholders' section above"）。这些 token 兼作审查时扫描的库存，而审查时识别是有效的那个类别。天真地换成正向检查清单会破坏该引用，并丢弃好的绊线 token。

### 要测试的变体

- **V0（当前）：** 组合时的正向陈述 + 禁止列表；Self-Review 引用该列表。
- **V1（审计者检查清单）：** 仅组合时正向配方——"Before finalizing a step, confirm it has: the literal code to write, a runnable command with expected output, types and method names defined within this plan, error handling shown explicitly. A step is complete when an engineer could implement it without asking any follow-up questions." Self-Review 保留通用占位符扫描。
- **V2（按机制重构——预测的赢家）：** 组合时只得到 V1 的正向配方；具名模式整体移入 Self-Review 的占位符扫描步骤，重新框定为识别（"when you scan, look for: 'TBD', 'TODO', 'Similar to Task N', …"）。相同的 token，从启动（prime）的类别搬迁到检测的类别。
- **V3（对照）：** 仅正向陈述，任何地方都没有列表。

### 微测试设计

- **任务：** opus 从刻意规格不足的 spec 编写 2-3 个任务的实现计划（规格不足正是引诱占位符的东西）。使用具有以下内容的 fixture spec：一个规格良好的任务、一个 spec 对其错误处理含糊其辞的任务、一个与第一个相似的任务（引诱 "Similar to Task 1"）。
- **采样：** 每个变体 5+ 次重复，默认 temperature，模型 `claude-opus-4-8`（实践中编写计划的模型）。
- **程序化评分**（除非注明，越低越好）：
  - 禁止 token 计数：`TBD|TODO|implement later|fill in details|appropriate error handling|handle edge cases|Similar to Task|Write tests for the above`
  - 在步骤更改代码的地方缺少围栏代码块的步骤
  - 引用计划输出中任何地方都未定义的类型/函数
  - （越高越好）每个任务带有预期输出的可运行命令
- **V2 的两阶段评分：** 还测试 Self-Review 半部分——用变体的 Self-Review 部分把每个生成的计划喂回，并测量扫描是否实际捕获植入的占位符（向 fixture 计划插入 2 个已知占位符；检测率就是指标）。
- **验收：** 只有当变体在禁止 token 计数上击败 V0 且不失去代码块覆盖或自我审查检测率时才采纳。预期成本：总计约 $6-10。

### PR 范围

单独 PR（writing-plans 是另一个技能；其 "No Placeholders" 列表是贡献者指南要求 eval 证据的调优内容）。PR 必须包含：微测试 harness + 结果表、前后文本，以及 V2 搬迁的理由。

## 微测试 harness（方法，免得丢失）

`/tmp/sdd-exp/micro/run-micro.py` 和 `/tmp/sdd-exp/micro2/run-micro2.py`（2026-06-10；要作为 `docs/superpowers/skills/micro-testing-prompt-guidance.md` + 脚本提交到 superpowers-evals）：

- 每个样本一次 API 调用：系统提示词 = 处于现实周围上下文中的技能指导变体；用户 = 一个现实的工作流中场景；输出 = 被组合的产物（派发提示词、计划、报告）。
- 用 grep 进行程序化评分以寻找明确标记；**在信任裁决前手动检查每个匹配**——今晚的一个"违规"其实是控制器正确地引用了禁止，而自动化否定检测又误标了另一个。
- 每个样本约 $0.15-0.30，每次迭代数秒，对比 $12/50 分钟的完整 eval 运行。在这里迭代措辞；仅在变更是结构性时才在完整运行中确认赢家。
- 始终包含无指导对照——今晚它同时揭示了一个适得其反（复述：禁止比没有更糟）和一个有效的禁止（测试重跑：对照 3/5 失败 vs 任一种措辞 0/5）。

## 结果：writing-plans 微测试（2026-06-10 运行，在本规格编写之后）

**已解决——无需变更。** 阶段 1（3 任务 spec，无压力）：所有四种变体（包括无指导对照）的全部 20 个计划中 0 个占位符。阶段 1b（10 任务 spec，五个近乎相同的命令引诱 "Similar to Task N"，显式约 2,500 词的经济目标）：40/40 干净——唯一的 regex 命中是 V2 自我审查*作证* "no TBD/TODO ✓"。当前世代的 opus 即使在刻意压力下也不会产生计划占位符，无论有没有禁止模式列表。处置：让 No Placeholders 部分保持原样（它几乎不花钱，且反事实不可测量）；不要打开后续 PR。V2 搬迁设计在此存档，以备未来某代模型退化。

## 还明确未放弃的（已测试并被否决，带数据）

记录下来以免有人没有新证据就重新提议——完整数字在 2026-06-09 SDD 设计规格的 Cost-iterations 部分：

- **控制器轮次批处理 / 一条消息中的并行工具调用：** 控制器每条消息恰好发出一次工具调用（每次被测量的运行中 0 条多工具消息，无论有无指导）。控制器 46% 的轮次是无工具调用的思考/叙述——提示词免疫的底线。
- **通过并行调用的流水线审查：** 因同样的原因而死亡。
- **通过 `run_in_background` 的流水线审查：** 机制在提供时被采纳（7/28 次派发），但收益低于 45 分钟场景上的运行间噪声底线（每次审查只有约 30-60 秒）；增加双结果流协调。仅对审查个别较长的计划值得重新审视。
- **附加在制胜配方上的细微差别条款：** 可测量地使其退化（C2：3.8 嘈杂 vs C：3.0 一致）。通过重新推导配方来迭代，而不是附加注意事项。
