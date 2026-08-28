# SDD 计划作用域工作区——设计

- **日期：** 2026-07-06
- **状态：** 已批准的方向（Jesse，2026-07-06）；本规范记录了调查所得的推荐修复方案
- **问题负责人：** subagent-driven-development 技能（`skills/subagent-driven-development/`）

## 问题

SDD 的持久进度工作区（`.superpowers/sdd/`，于 v6.0.0/v6.0.3 引入）没有计划身份，也没有生命周期终点。每个产物都以裸任务编号为 key（`progress.md`、`task-N-brief.md`、`task-N-report.md`），且 SKILL.md 指示启动时的控制器把它找到的任何 ledger 都当作自己的进度：

> 技能启动时，检查是否有 ledger：
> `cat "$(git rev-parse --show-toplevel)/.superpowers/sdd/progress.md"`。其中被标记为完成的任务就是 DONE——不要重新派发它们；从第一个未标记完成的任务处继续。

一个在同一 worktree 中执行**后续计划**的全新会话会把上一计划的 ledger 当作自己的来读。按字面理解技能文本，它会被告知跳过任务。没有任何东西会删除工作区，因此陈旧状态会无限期地持续并不断累积。

### 观察到的失败（serf 仓库，2026-06-22 → 2026-07-05）

- **跨计划冲突，临时绕过了事：** `cc-plugin-marketplaces` worktree 在三个计划间累积了 68 个文件。P2 控制器不得不发明 `progress-p2.md` 和 `p2-task-N-report.md` 来躲开 P1 的 ledger；P2 的 brief 在默认路径下静默覆盖了 P1 的；一个被弃用的 `progress-p3.md` 存根至今仍在。
- **Git 污染，出现了三次：** SDD 的临时文件被提交，需要两次清理提交（`8305e340d`、`c966261a5`）；今天 serf `main` 上仍跟踪着三个产物，其中包括一份在不同机器上撰写的报告，如今会在每个全新的 worktree 中显现。后续计划的 task-1 报告覆盖了一个无关的已跟踪文件，留下了永久的 `git status` 噪音。
- 自我忽略的 `.gitignore` 只在脚本运行时才会被写入。手动追加 ledger 的控制器（已观察到）永远不会创建它，而一旦文件被跟踪，gitignore 就无能为力了。

### 根本原因

身份没有存在于数据中的任何位置；正确性依赖于一个没有触发器的清理机制。任何仅依赖计划结束清理的修复，恰恰会在 ledger 本应帮助幸存的崩溃/压缩（compaction）情形中失败。身份必须是结构性的。

## 设计

### 1. 每计划工作区目录（结构性身份）

工作区变为 `.superpowers/sdd/<plan-slug>/`，其中 `<plan-slug>` 是计划文件的文件名去掉 `.md` 扩展名后的部分（计划文件名已经是带日期的 kebab-case，例如 `2026-07-04-plugin-marketplaces-p1-backend-core`）。来自不同计划的产物不再可能冲突；一个陈旧的兄弟目录是惰性的，因为没有任何指令会指向它。

脚本接口（全部位于 `skills/subagent-driven-development/scripts/`）：

- `sdd-workspace PLAN_FILE`——解析并创建 `<repo-root>/.superpowers/sdd/<plan-slug>/`，维护位于 `.superpowers/sdd/.gitignore`（父级，内容为 `*`）的自我忽略 `.gitignore`，打印计划目录的绝对路径。缺少参数或计划文件不存在时出错（退出码 2）。去掉扩展名后的 slug 必须非空。
- `task-brief PLAN_FILE N [OUTFILE]`——签名不变；默认 OUTFILE 经由 `sdd-workspace PLAN_FILE` 移动到 `<workspace>/task-N-brief.md`。
- `review-package PLAN_FILE BASE HEAD [OUTFILE]`——新增 PLAN_FILE 作为第一个参数；默认 OUTFILE 移动到 `<workspace>/review-<base7>..<head7>.diff`。

旧的扁平布局没有兼容路径：脚本和 SKILL.md 在同一插件版本中一起发布，且没有别的东西会调用这些脚本。（已明确确认：不做向后兼容处理。）

### 2. Ledger 指明其计划（为手写 ledger 加的保险带）

ledger 仍位于 `<workspace>/progress.md`。创建时，其第一行必须是：

```
# SDD ledger — plan: docs/superpowers/plans/<plan-file>.md
```

SKILL.md 的技能启动检查变为按计划作用域，并携带一个以该可观测行为 key 的条件守卫，用正面措辞表述（是配方，不是禁令）：用 `sdd-workspace PLAN_FILE` 解析你计划的工作区，在那里读取 `progress.md`；其计划行指向不同计划文件的 ledger 是另一计划的进度——让它留在原地，并使用你自己计划的工作区。这覆盖了不运行脚本而手动编写 ledger 的控制器（在 serf 的 ask_user 会话中观察到），也覆盖了升级前留在旧扁平路径的杂物。

守卫的确切措辞从属于 eval 结果（见"评估"）；只有 RED 基线中实际观察到的失败才会添加计数器。

### 3. 工作区生命周期终点（卫生清理，而非正确性）

当最终的整分支审阅干净、其修复波（如有）已合并时——即在移交给 `superpowers:finishing-a-development-branch` 之前——控制器删除其计划的工作区目录（`rm -rf "$WORKSPACE"`）。工作的记录就是 git 历史；ledger 的职责（计划中途的压缩恢复）已经完成。兄弟目录永远不会被触碰：崩溃的或并行的计划各自拥有自己的目录，而刻意停放（park）的跨计划产物（观察到的模式：`WAVE1-HANDOFF.md`）直接位于 `.superpowers/sdd/` 下，不受任何计划的清理影响。

### 4. SKILL.md 触点

- **Durable Progress 部分：** 通过 `sdd-workspace PLAN_FILE` 解析工作区；ledger 检查限定在计划自己的工作区范围内；ledger 创建格式包含计划行；不匹配守卫；完成删除；`git clean -fdx` 风险说明更新为新路径。
- **Handling Implementer Status / Constructing Reviewer Prompts / File Handoffs / Red Flags / Example Workflow：** 将脚本调用更新为新签名（`review-package PLAN_FILE BASE HEAD`）以及任何路径提及。`implementer-prompt.md` 和 `task-reviewer-prompt.md` 不包含工作区路径（已核实），无需更改。
- 仅当 RED 基线显示出一个结构性修复加守卫文本都无法消除的失败时，才添加 Red Flags 条目。

## 范围之外（刻意为之）

- 不改动 `finishing-a-development-branch` 或任何其他技能。
- 在现有父级 `.gitignore` 之外，不为提交 `.superpowers/` 添加 git 级防护。
- 不对 serf 仓库做追溯性清理（单独的后续工作）。
- 不做旧布局迁移或回退读取。

## 测试

### 确定性 shell 测试（`tests/claude-code/test-sdd-workspace.sh`，扩展）

- `sdd-workspace PLAN` 打印 `<root>/.superpowers/sdd/<slug>` 并创建它；没有计划参数时出错；计划文件缺失时出错。
- 两个不同的计划文件解析到两个不同的目录；经由 `task-brief` 写入的产物落在各自计划的目录中。
- `review-package PLAN BASE HEAD` 写入计划目录之下。
- 父级 `.gitignore` 自我忽略：工作区对 `git status` 和 `git add -A` 不可见（现有断言，重新锚定）。
- 链接 worktree 的区分性（现有断言，重新锚定）。
- 审计现有套件 `test-subagent-driven-development.sh` / `-integration.sh` 中的旧路径预期（初步 grep 未发现；审计反正就是一道任务关卡）。

### 评估（writing-skills RED → GREEN，2026-07-06 重新定域）

压力场景作为全新的 sonnet subagent 会话，针对临时目录中的 fixture 仓库运行（绝不在本 worktree 内），采用压缩恢复的框架，每个代表由人工评分；测得的输出是控制器的恢复决策（不进行真实的 implementer 派发）。

**迫使重新定域的 RED 结果（维护者决策，Jesse，2026-07-06）：** 最初假设的失败——控制器盲目地采用一个陈旧的他人 ledger 作为自己的进度——**没有**复现：三种框架（全新会话、可能被恢复、忠实执行压缩后恢复并启用技能的"信任 ledger"文本）下的 25/25 个代表都对 ledger 引用的提交与 git 历史和计划文件做了取证式的交叉核对，拒绝了他人的 ledger，并从任务 1 开始执行计划 B——每次恢复都要为此花费 6–13 次工具调用的跨计划取证。为了诚实地证明这一点烧掉了两个 fixture 迭代（v1：伪造的哈希被一眼识破；v2：桩实现被判定为虚假的"review clean"记录——S2 对照两次都失败）。完整记录在已提交的 eval 文档中。

**重新定域的声明与门限：**

- 该变更依据结构性记录（冲突、临时想出的旁带名称、被覆盖的 brief、git 污染——serf 仓库）以及测得的消歧成本而发布，并以明确的维护者签收代替 writing-skills 对 SKILL.md 文本的失败基线要求。
- **S1 GREEN（要求 5/5）：** 新的定域布局中存在陈旧的计划 A 工作区，外加旧扁平布局的杂物；一个在计划 B 上恢复的控制器直接解析自己按计划定域的工作区并从任务 1 开始；记录每个代表相对 RED 基线（7/13/9/10/6）的 `tool_uses` 作为成本差。
- **S2 RED 对照（要求 ≥4/5）与 S2 GREEN（要求 5/5）**，基于真实的 v3 fixture（引用的提交确实实现了各自任务的规范、作者轮换、时间戳分散）：合法的同计划恢复——任务 1–2 被识别，任务 3 被派发。这保护了 ledger 原本的用途；修复不得破坏它，而对照验证了 fixture。

结果写入 `docs/superpowers/specs/2026-07-06-sdd-plan-scoped-workspace-eval-results.md`，并在 PR 中总结。

## 风险

- **不同目录下 basename 相同的不同计划之间出现 slug 冲突：** 已接受；计划文件名按惯例带日期前缀，且 basename 相同在实践中即同一计划（此时恢复正是期望的行为）。
- **控制器完全跳过脚本**（一切都手动）：ledger 计划行守卫就是缓解措施；eval 的 S1 会度量文本是否真的产生约束力。
- **在计划的工作区历经一次崩溃仍存活后从头重新运行一个已完成计划：** 该 ledger 合法地属于同一计划；恢复而非重启是设计好的行为，而 `git log` 交叉核对（现有技能文本）覆盖了分叉的情形。
