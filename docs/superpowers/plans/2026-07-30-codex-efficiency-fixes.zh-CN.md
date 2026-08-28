# Codex 效率修复实施计划

> **给 agent 工作者的提示：** 必需的子技能：使用 superpowers:subagent-driven-development（推荐）或 superpowers:executing-plans 来逐任务实施本计划。步骤使用复选框（`- [ ]`）语法进行跟踪。

**目标：** 将 codex-efficiency eval 活动中的五项证据充分处理（T1–T5）作为技能/文档更改发布到 `codex-efficiency-fixes` 上，用活动的评分器对照每个预先注册的标准逐项评分，并为每个通过的处理对照 `dev` 各切一个 PR。

**架构：** 技能文本编辑落在本 worktree（`.worktrees/codex-efficiency-fixes`）；eval 装置工作、假设日志条目与评分器位于 `superpowers-autoresearch`（main，已授权推送）；场景更改位于 `superpowers-autoresearch/campaigns/codex-efficiency/scenarios/`。批次测试通过现有 quorum 容器通道针对 `/tmp/sp-arm-fix` worktree arm 运行。

**技术栈：** Markdown 技能文本；Python 3 评分器（pytest）；bash quorum 运行器；evals 容器中的 Codex/Claude/Gemini CLI。

**规格：** `docs/superpowers/specs/2026-07-30-codex-efficiency-fixes-design.md`（已批准）。规格的基线与标准为准；本计划按任务重复它们。

## 全局约束

- 假设日志（`superpowers-autoresearch/logs/2026-07-30-codex-efficiency-fixes.md`）只追加；修正以新的带日期条目进行，绝不编辑。每个批次测试在运行前都要有预注册条目。
- 原始 rollout 与会话内容绝不进入任何仓库。只进入聚合、评分器输出与提炼后的场景。
- 每次 autoresearch/evals 提交前：对暂存文本与提交消息文件做子串感知 grep，对照活动名称集（客户端名称、主机名、工单 ID）。只允许间接引用。
- 场景后检查（`checks.sh`）绝不断言评分器测量的行为选择。测量在评分器一侧。
- 任何场景/arm/harness 组合在批次测试前先冒烟测试一个重复。
- 评分器输出文件使用重复范围名称；未经 `FORCE=1` 绝不覆盖现有聚合。
- 每个评分器判定都需要人工检查匹配（非循环：绝不用评分器自己的 helper 验证）。
- Brainstorming 的 frontmatter `description:` 不得更改（触发依赖它）。
- 技能编辑不得更改命名目标以外的文件；无纯空白的搅动。
- PR 仅在处理的标准通过后按其处理切出；合并需要 Jesse 对每个 PR 的批准。codex-tools.md 的 PR 声明合并顺序 T3 → T2 → T5。
- 运行批次测试的 subagent 在会话内轮询，使用长时间超时；不设监控。

---

### 任务 1：T1 — SDD worker 审查禁令

**文件：**
- 修改：`skills/subagent-driven-development/implementer-prompt.md`（在 `## Code Organization` 之前插入新章节）
- 修改：`skills/subagent-driven-development/SKILL.md`（调度条目 + Red Flags 行）

**接口：**
- 消费：其他任务无输入。
- 产出：提交 `fix(sdd): implementers never dispatch subagents` — T1 PR 精确 cherry-pick 此提交。

- [ ] **步骤 1：插入 implementer-prompt 章节**

在 `skills/subagent-driven-development/implementer-prompt.md` 中，紧接以 `run the full suite once before committing, not after every edit.` 结尾的段落之后、`    ## Code Organization` 之前插入（保持模板的 4 空格正文缩进）：

```
    ## You Do Not Dispatch Subagents

    Do all of this task's work yourself. Never spawn a subagent to
    implement part of the task, and above all never spawn a reviewer to
    check your work. Self-review (below) means reading your own diff.
    Review is the controller's job: after you report, it dispatches a
    fresh reviewer against your diff. A reviewer you spawn duplicates
    that review at full cost, and its approval counts for nothing in
    the process. If you catch yourself thinking "an independent review
    would strengthen my report" — that review is already scheduled.
    Report instead.
```

- [ ] **步骤 2：向 SKILL.md 添加调度契约条目**

在 `skills/subagent-driven-development/SKILL.md` 的 `### 1. Dispatch the implementer` 部分，紧接以 `- A dispatch prompt describes one task, not the session's history.` 开头的条目之后添加：

```
- The dispatch carries the no-subagents contract (it is in the
  implementer template): the implementer never dispatches subagents —
  not helpers, and never a reviewer. Review arrives from you, after the
  report. In real sessions, every reviewer a worker spawned duplicated
  the task review the controller dispatched anyway — a full extra
  review seat per task.
```

- [ ] **步骤 3：添加 Red Flags 行**

在同一文件的 Red Flags 表（`## Red Flags`）中，在 `| "Ledger bookkeeping is overhead" | ... |` 行之后添加：

```
| "The implementer spawned its own reviewer — free extra assurance" | It's a duplicate seat reviewing the same diff; the task review is the gate. A worker-spawned reviewer is a defect to flag, not rigor. |
```

- [ ] **步骤 4：验证**

运行：`grep -c "You Do Not Dispatch Subagents" skills/subagent-driven-development/implementer-prompt.md` → 预期 `1`。运行：`grep -c "no-subagents contract\|spawned its own reviewer" skills/subagent-driven-development/SKILL.md` → 预期 `2`。

- [ ] **步骤 5：提交**

```bash
git add skills/subagent-driven-development/implementer-prompt.md skills/subagent-driven-development/SKILL.md
git commit -m "fix(sdd): implementers never dispatch subagents

Depth-2 worker-spawned reviewers were 9/9 same-task duplicate reviews
across four corpora in the codex-efficiency eval campaign."
```

---

### 任务 2：T3 — codex-tools.md 版本如实的多 agent 重写

**文件：**
- 修改：`skills/using-superpowers/references/codex-tools.md`（替换配置块之后的段落）

**接口：**
- 消费：无。
- 产出：任务 3 与 4 在其后追加的重写基础章节；提交 `fix(codex): correct multi-agent guidance against Codex source`（T3 PR）。

- [ ] **步骤 1：替换多 agent 段落**

在 `skills/using-superpowers/references/codex-tools.md` 中，将以 `This enables \`spawn_agent\`, \`wait_agent\`, and \`close_agent\`` 开头（并以 `...carrying the brief, the report file, and the findings.` 结尾）的整个单段落替换为：

```
This enables the multi-agent tools that skills like
`dispatching-parallel-agents` and `subagent-driven-development` use.
Which tools you get depends on the multi-agent version your model
preset selects (current presets run V2; older ones run V1). Trust your
actual tool list over any table — including this one — when they
disagree.

- **Spawning:** give children a clean context with
  `spawn_agent {fork_turns: "none"}`; the default `"all"` copies your
  entire transcript into the child. On Codex 0.145+, role files under
  `~/.codex/agents/` attach to isolated forks via `agent_type`.
  Full-history forks accept `model` and `reasoning_effort` overrides
  (only `agent_type` is refused there) — isolated forks are the SDD
  default for context hygiene, not because overrides require them.
- **Fix rounds:** resume the implementer with `followup_task` — it
  delivers your message, triggers a turn, and transparently reloads a
  child the harness evicted. Never dispatch a fresh implementer on the
  theory that a spawned agent cannot be messaged again; on V2 it
  always can.
- **Lifecycle:** V2 has no `close_agent`. Finished children are
  evicted automatically when slots are needed; leaving them unclosed
  costs nothing. Only V1 sessions have `close_agent` — there, close
  reviewers when their review returns, and close each implementer
  after its task's review passes.
- **Model names:** never copy a model name from a skill, table, or old
  session into `spawn_agent` without checking it against your current
  spawn allowlist — V2 accepts only V2-capable presets and hard-errors
  on the rest.
```

- [ ] **步骤 2：验证**

运行：`grep -c "close_agent" skills/using-superpowers/references/codex-tools.md` → 预期 `2`（都在 Lifecycle 条目内）。运行：`grep -c "cannot be messaged again" skills/using-superpowers/references/codex-tools.md` → 预期 `1`。

- [ ] **步骤 3：提交**

```bash
git add skills/using-superpowers/references/codex-tools.md
git commit -m "fix(codex): correct multi-agent guidance against Codex source

Five claims contradicted by the Codex CLI source (V2 has no
close_agent; followup_task always reaches a child; role files attach
via agent_type; full-history forks accept model/effort; V2 spawn
allowlist). Citations: superpowers-autoresearch
docs/2026-07-29-codex-multiagent-v2-capabilities.md."
```

---

### 任务 3：T2 — codex-tools.md 事件驱动等待章节

**文件：**
- 修改：`skills/using-superpowers/references/codex-tools.md`（任务 2 的块之后、`## Environment Detection` 之前的新章节）

**接口：**
- 消费：任务 2 重写的基础章节（直接追加在它之后）。
- 产出：提交 `fix(codex): event-driven waiting instead of short polls`（T2 PR；声明依赖 T3 的 PR）。

- [ ] **步骤 1：插入章节**

紧接 `## Environment Detection` 之前插入：

```
## Waiting on children

`wait_agent` is an event subscription, not a poll: a long wait wakes
the moment a child produces mailbox activity, with the same latency as
a short one. Short-timeout polling buys nothing and costs a tool call —
and a context rebill — per poll. In measured sessions, roughly
two-thirds of all wait calls were short polls that timed out.

- While you still have local work, do not wait at all. A completed
  child's final answer is pushed into your mailbox and arrives with
  your next turn.
- When you are genuinely idle with children outstanding, issue ONE
  `wait_agent` with a long `timeout_ms` — 900000 (15 minutes) or more —
  and let the event wake you.
- Completion mail cannot wake an idle controller (it is delivered
  without triggering a turn); covering that idle window is
  `wait_agent`'s only job. If a long wait times out, check
  `list_agents` for stuck children — do not fall back to short polls.
```

- [ ] **步骤 2：验证**

运行：`grep -c "Waiting on children" skills/using-superpowers/references/codex-tools.md` → 预期 `1`。运行：`grep -n "900000" skills/using-superpowers/references/codex-tools.md` → 预期一处命中。

- [ ] **步骤 3：提交**

```bash
git add skills/using-superpowers/references/codex-tools.md
git commit -m "fix(codex): event-driven waiting instead of short polls

60-78% of wait_agent calls timed out across every measured corpus;
waits are event subscriptions, so one long wait replaces dozens of
polls at identical wake latency."
```

---

### 任务 4：T5 — codex-tools.md 的 spawn 模型路由

**文件：**
- 修改：`skills/using-superpowers/references/codex-tools.md`（任务 3 之后、`## Environment Detection` 之前的新章节）

**接口：**
- 消费：任务 2–3 已就位。
- 产出：提交 `fix(codex): explicit model+effort on every spawn, config backstop`（T5 PR；声明依赖 T3 与 T2 的 PR）。

- [ ] **步骤 1：插入章节**

紧接 `## Environment Detection` 之前插入：

````
## Model routing on spawns

Every `spawn_agent` you issue — including when you are yourself a
spawned child running a fan-out — sets `model` AND `reasoning_effort`
explicitly, per the Model Selection rules of the skill you are
executing. Setting `model` alone is a trap: the child's effort
silently resets to that model's default, not to yours.

Ask your human partner to add a machine-level backstop to
`~/.codex/config.toml` so any spawn that slips through still routes to
a deliberate tier instead of silently inheriting the session's most
expensive model:

```toml
[agents]
default_subagent_model = "<a mid-tier model from your spawn allowlist>"
default_subagent_reasoning_effort = "medium"
```
````

- [ ] **步骤 2：验证**

运行：`grep -c "Model routing on spawns" skills/using-superpowers/references/codex-tools.md` → 预期 `1`。运行：`grep -c "default_subagent_model" skills/using-superpowers/references/codex-tools.md` → 预期 `1`。

- [ ] **步骤 3：提交**

```bash
git add skills/using-superpowers/references/codex-tools.md
git commit -m "fix(codex): explicit model+effort on every spawn, config backstop

Depth-2 child-issued spawns omitted model 2/2 at CLI 0.146; model
without reasoning_effort resets effort to the model default."
```

---

### 任务 5：T4 — brainstorming 三路径路由器（变体 C）

**文件：**
- 修改：`skills/brainstorming/SKILL.md`

frontmatter `description:` 保持不变。以下编辑是对指定区域的完整替换；未命名的一切保持逐字节不变。

- [ ] **步骤 1：替换开篇句、HARD-GATE 与反模式章节**

将从 `Start by understanding the current project context, ...` 行到 `## Anti-Pattern: "This Is Too Simple To Need A Design"` 章节结束（即直到但不包括 `## Checklist`）的内容替换为：

```
Start by classifying how much process the request needs, then work
through your path: understand the context, refine the idea, present a
design, and get your human partner's approval.

<HARD-GATE>
Do NOT invoke any implementation skill, write any code, scaffold any
project, or take any implementation action until you have told your
human partner what you intend and they have approved it. This applies
to EVERY task on EVERY path below — the ceremony scales with the task;
the approval gate never does.
</HARD-GATE>

## Three Paths

Before your first question, classify the request and say the
classification out loud — "this looks bounded, so I'll present a short
design here rather than write a spec" — so your human partner can
override it:

- **Spike** — a feasibility question ("can we...", "is it possible...",
  "quick and dirty is fine") whose output is an answer, not code you
  keep. Present the question and what you'll try in 2-3 sentences, get
  a nod, then find out as cheaply as correctness allows. No design
  doc, no spec file. Report findings as a recommendation; anything you
  built stays labeled throwaway.
- **Bounded** — a well-scoped change to an existing, understood flow: a
  new flag, a small endpoint, a one-file fix. Ask the clarifying
  questions that matter, present a short design IN CHAT (a few
  sentences to a few short paragraphs), and get approval. No spec
  file, no implementation plan document.
- **Architectural** — new projects, new subsystems, changes that
  restructure how components fit together or alter interfaces others
  depend on. Follow the full process: questions, approaches, sectioned
  design, written spec, then the writing-plans skill.

When in doubt between two paths, take the heavier one. The ratchet is
one-way: hidden complexity discovered mid-task upgrades the path —
stop, say so, and step up. Nothing downgrades mid-task.

## Anti-Pattern: "Too Simple To Need Approval"

Every path ends with your human partner approving your intent before
implementation. A todo list, a single-function utility, a config
change — the design may be two sentences in chat, but you MUST present
it and get approval. "Simple" tasks are where unexamined assumptions
cause the most wasted work. What scales with simplicity is the
artifact, never the approval.

## Red Flags

| Thought | Reality |
|---------|---------|
| "This is too simple to need a design" | Simple means a short design, not no design. Two sentences in chat, then approval. |
| "I'll call it bounded and skip the spec" | Reaching for a label to skip work IS the doubt — take the heavier path. |
| "The spike works, so I'll keep the code" | A spike's output is an answer. Keeping the code is a new request — classify it. |
| "It grew, but I'm almost done — no need to re-classify" | Hidden complexity upgrades the path mid-task. Stop and say so. |
| "They approved the spike, so the follow-up change is approved too" | Each task gets its own classification and its own approval. |
```

- [ ] **步骤 2：替换 Checklist 章节**

将 `## Checklist` 章节正文（从 `You MUST create a task...` 到第 9 项）替换为：

```
Classify first, announce the path, then create a task for each item on
your path and complete them in order.

**Spike:**
1. **Explore project context** — enough to frame the probe
2. **Present question + probe plan** — 2-3 sentences
3. **Get approval** — a nod is enough
4. **Investigate** — as cheaply as correctness allows
5. **Report findings** — a recommendation; label anything built as throwaway

**Bounded:**
1. **Explore project context** — check files, docs, recent commits
2. **Ask clarifying questions** — one at a time, the ones that matter
3. **Present short design in chat** — approach, files touched, testing
4. **Get approval** — explicit, before any implementation
5. **Implement** — proceed with the normal development workflow (TDD applies); no plan document

**Architectural:**
1. **Explore project context** — check files, docs, recent commits
2. **Offer the visual companion just-in-time** — NOT upfront. The first time a question would genuinely be clearer shown than described, offer it then (its own message); on approval its browser tab opens for you. If no visual question ever arises, never offer it. See the Visual Companion section below.
3. **Ask clarifying questions** — one at a time, understand purpose/constraints/success criteria
4. **Propose 2-3 approaches** — with trade-offs and your recommendation
5. **Present design** — in sections scaled to their complexity, get user approval after each section
6. **Write design doc** — save to `docs/superpowers/specs/YYYY-MM-DD-<topic>-design.md` and commit
7. **Spec self-review** — quick inline check for placeholders, contradictions, ambiguity, scope (see below)
8. **User reviews written spec** — ask user to review the spec file before proceeding
9. **Transition to implementation** — invoke writing-plans skill to create implementation plan
```

- [ ] **步骤 3：替换 Process Flow 图**

将 `## Process Flow` 下整个 ```dot ... ``` 块替换为：

```
digraph brainstorming {
    "Classify: spike / bounded / architectural" [shape=diamond];
    "Present question + probe (2-3 sentences)" [shape=box];
    "Ask clarifying questions (bounded)" [shape=box];
    "Present short design in chat" [shape=box];
    "Human approves?" [shape=diamond];
    "Investigate; report recommendation" [shape=doublecircle];
    "Implement via normal workflow (no plan doc)" [shape=doublecircle];
    "Explore project context" [shape=box];
    "Ask clarifying questions" [shape=box];
    "Propose 2-3 approaches" [shape=box];
    "Present design sections" [shape=box];
    "User approves design?" [shape=diamond];
    "Write design doc" [shape=box];
    "Spec self-review\n(fix inline)" [shape=box];
    "User reviews spec?" [shape=diamond];
    "Invoke writing-plans skill" [shape=doublecircle];
    "Hidden complexity? Upgrade path" [shape=box];

    "Classify: spike / bounded / architectural" -> "Present question + probe (2-3 sentences)" [label="spike"];
    "Classify: spike / bounded / architectural" -> "Ask clarifying questions (bounded)" [label="bounded"];
    "Classify: spike / bounded / architectural" -> "Explore project context" [label="architectural"];
    "Present question + probe (2-3 sentences)" -> "Human approves?";
    "Ask clarifying questions (bounded)" -> "Present short design in chat";
    "Present short design in chat" -> "Human approves?";
    "Human approves?" -> "Investigate; report recommendation" [label="spike: yes"];
    "Human approves?" -> "Implement via normal workflow (no plan doc)" [label="bounded: yes"];
    "Hidden complexity? Upgrade path" -> "Classify: spike / bounded / architectural";
    "Explore project context" -> "Ask clarifying questions";
    "Ask clarifying questions" -> "Propose 2-3 approaches";
    "Propose 2-3 approaches" -> "Present design sections";
    "Present design sections" -> "User approves design?";
    "User approves design?" -> "Present design sections" [label="no, revise"];
    "User approves design?" -> "Write design doc" [label="yes"];
    "Write design doc" -> "Spec self-review\n(fix inline)";
    "Spec self-review\n(fix inline)" -> "User reviews spec?";
    "User reviews spec?" -> "Write design doc" [label="changes requested"];
    "User reviews spec?" -> "Invoke writing-plans skill" [label="approved"];
}
```

- [ ] **步骤 4：替换终态段落**

将段落 `**The terminal state is invoking writing-plans.** Do NOT invoke frontend-design, mcp-builder, or any other implementation skill. The ONLY skill you invoke after brainstorming is writing-plans.` 替换为：

```
**Terminal states are path-bound.** Architectural: the ONLY skill you
invoke after brainstorming is writing-plans — never frontend-design,
mcp-builder, or any other implementation skill. Bounded: after
approval, implementation proceeds directly through the normal
development workflow; no plan document. Spike: the terminal state is a
reported recommendation.
```

- [ ] **步骤 5：将长章节限定到各自路径**

(a) 紧接 `## The Process` 之下，作为第一行插入：

```
The subsections below serve the bounded and architectural paths (a
spike stops at "present the probe, get a nod"). Sections from
**Exploring approaches** onward are architectural-path depth — for
bounded work, context plus a few questions plus a short in-chat design
is the whole process.
```

(b) 将标题 `## After the Design` 重命名为 `## After the Design (architectural path)`。

- [ ] **步骤 6：验证**

运行：`grep -c "Three Paths" skills/brainstorming/SKILL.md` → `1`；`grep -c "regardless of perceived simplicity" skills/brainstorming/SKILL.md` → `0`；`grep -c "^description:" skills/brainstorming/SKILL.md` 与 `git show origin/dev:skills/brainstorming/SKILL.md | grep -c "^description:"` 相比不变；确认 `git diff origin/dev -- skills/brainstorming/SKILL.md` 无 frontmatter hunk。如果安装了 `dot`：`awk '/^```dot$/,/^```$/' skills/brainstorming/SKILL.md | sed '1d;$d' | dot -Tcanon >/dev/null` → 退出 0。

- [ ] **步骤 7：提交**

```bash
git add skills/brainstorming/SKILL.md
git commit -m "feat(brainstorming): three-path router — ceremony scales, approval never does

Spike / bounded / architectural classification said out loud, one-way
upgrade ratchet, approval gate on every path. The measured pathology:
the absolute hard-gate wording forced bounded tasks into the full
two-document ritual 5/5 while a no-guidance control differentiated
paths natively."
```

---

### 任务 6：修复 arm + 假设日志

**文件：**
- 创建：`superpowers-autoresearch/logs/2026-07-30-codex-efficiency-fixes.md`
- 创建（文件系统，不提交）：`/tmp/sp-arm-fix` worktree

**接口：**
- 消费：任务 1–5 已提交到 `codex-efficiency-fixes`。
- 产出：每个批次测试所针对的 arm；之后每个任务追加到的日志。

- [ ] **步骤 1：创建 arm**

```bash
git -C /Users/jesse/git/superpowers/superpowers worktree add --detach /tmp/sp-arm-fix codex-efficiency-fixes
git -C /tmp/sp-arm-fix log --oneline -1   # must show Task 5's commit
```

（在任何之后的批次测试前，用 `git -C /tmp/sp-arm-fix checkout --detach codex-efficiency-fixes` 刷新。）

- [ ] **步骤 2：创建日志**

编写 `superpowers-autoresearch/logs/2026-07-30-codex-efficiency-fixes.md`：标题 `# Codex efficiency fix cycle — hypothesis log`；一个头部块声明其只追加、指明规格路径（`superpowers/docs/superpowers/specs/2026-07-30-codex-efficiency-fixes-design.md`）、分支、arm（`/tmp/sp-arm-fix`）与沿用常驻规则（批次测试前预注册、人工匹配检查、无原始 rollout、区分规则）；以及一个 `## Pre-registered criteria (from the approved spec)` 章节，逐字复述规格各处理章节的五行标准（T1：0 个 worker 发出的深度 2 spawn 且审查覆盖保留；T2：超时率 < 25%，无完成丢失；T3：来源引用的修正，无评分器回归；T4：三层标准；T5：每个深度都显式 model+effort，带预注册的"因零而不确定"保留条款）。

- [ ] **步骤 3：隐私清扫并提交（autoresearch）**

```bash
cd /Users/jesse/git/superpowers/superpowers-autoresearch
git add logs/2026-07-30-codex-efficiency-fixes.md
git diff --cached | grep -iE 'paradise|magic-kingdom|flower|pallas|web1|teststrip|PRI-[0-9]' && echo LEAK || echo clean   # must print clean
git commit -m "docs: open the codex-efficiency fix-cycle hypothesis log"
```

---

### 任务 7：Micro — 变体 C + 对抗简报

**文件：**
- 修改：`superpowers-autoresearch/campaigns/codex-efficiency/ceremony-path-micro.py`
- 创建：`superpowers-autoresearch/campaigns/codex-efficiency/out/` micro 输出（不提交；聚合进日志）

**接口：**
- 消费：任务 5 发布的路由器文本（变体必须引用它）、任务 6 的日志。
- 产出：T4 第一层的 micro 判定。

- [ ] **步骤 1：扩展脚本**

先阅读当前脚本。然后：(a) 添加变体 `C-approval`，其文本为任务 5 步骤 1 的 Three Paths 块（三个条目加上怀疑/棘轮段落，逐字）；(b) 中和共享的 SYSTEM 答案定义，使分类测量的是 ARTIFACT 层级而非审批步骤——用以下三条替换三个定义条目：SPIKE = "dive straight into a minimal throwaway investigation — no design document"；BOUNDED = "make the change after at most brief clarification and a short in-chat design — no design document, no implementation plan"；FULL = "run the complete design process — written design document and implementation plan before touching code"；(c) 在现有三条之后添加两条简报：

`ambig-interface`: "Add a --json flag to our export CLI command so output can be piped to jq. The current text output of export is parsed line-by-line by three downstream scripts in tools/ that other teams run in their pipelines."

`ambig-crosscut`: "Fix the timezone bug in report_generator.py where daily rollups are off by one day for users west of UTC. Rollup boundaries are also computed independently in the billing exporter and the retention job, which must stay consistent with reports."

矩阵：3 变体（Z-null、A-current、C-approval）× 5 简报 × 5 重复 = 75 次调用，模型 `claude-opus-4-8`，与活动运行相同的单词正则评分与答案文件验证。

- [ ] **步骤 2：在日志中预注册**

在运行前追加一个条目：分类法变更（新的 SYSTEM 定义——仅限轮内比较；不可与活动的 micro 逐格比较）、矩阵与预测：C-approval — spike→SPIKE ≥4/5，bounded→BOUNDED ≥4/5，arch→FULL 5/5，两条 ambig→FULL ≥4/5；A-current — bounded→FULL 持续存在；Z-null 在 ambig 简报上的结果记录为观察（无引导的判断会升级吗？）。提交该条目。

- [ ] **步骤 3：运行并验证**

运行扫描。用独立命令（例如 `awk 'NF!=1' out/micro-c/*.txt`）而非评分器自己的解析器，独立验证每个答案文件恰好是一个单词。

- [ ] **步骤 4：判定条目 + 提交**

将结果表与判定追加到日志（标准来自步骤 2）。隐私清扫，提交脚本 + 日志条目。如果 C-approval 在某格失败：STOP，向控制器报告——路由器文本需要修订，任何批次测试都不应在它上面花钱。

---

### 任务 8：共享 SDD 批次测试（T1、T2、T5）

**文件：**
- 创建：`superpowers-autoresearch/campaigns/codex-efficiency/out/` 修复 arm 聚合（重复范围文件名）
- 修改：`superpowers-autoresearch/logs/2026-07-30-codex-efficiency-fixes.md`（预注册 + 判定）
- 可能修改：`superpowers-autoresearch/campaigns/codex-efficiency/run-quorum.sh`（仅当 `fix` arm 名称需要接线时——先读它；活动惯例把 ARM 映射到 `/tmp/sp-arm-$ARM`）

**接口：**
- 消费：任务 1–6。
- 产出：T1/T2/T5 判定；任务 11 的回归比较可能重用的运行。

- [ ] **步骤 1：预注册** — 追加批次测试条目（arm=fix @ arm 的 SHA，场景 `cx-sdd-small`，通道 A 与 B 共 8 个重复，JOBS=2，评分器 e6/e7/e1，标准逐字来自日志的标准章节，预算估计约 $40）。提交。

- [ ] **步骤 2：冒烟测试** — 1 个重复：`EVALS_ROOT=<lane A> JOBS=1 bash run-quorum.sh fix cx-sdd-small 1`。人工检查判定与一个 rollout：会话已运行、spawn 存在、无设置失败。任何基础设施异常都会停止批次测试。

- [ ] **步骤 3：运行批次测试** — 其余 7 个重复跨通道分配（按运行器的惯例 `REP_START`，例如重复 2–4 在通道 A，5–8 在通道 B）。在会话内轮询，使用长超时；不设监控。

- [ ] **步骤 4：评分** — 对修复 arm 的运行运行 `score_e6.py`、`score_e7.py`、`score_e1.py`，输出使用重复范围名称。人工检查：每个深度 2 spawn（应无 worker 发出的）、2+ 次运行的每个等待调用分类、2+ 次运行的每个 spawn 元组——对照原始 rollout，而非评分器 helper。

- [ ] **步骤 5：判定条目** — 按处理对照预注册标准；如果深度 2 spawn 消失，诚实地记录 T5 的"因零而不确定"分支。账本行带测量成本。隐私清扫；提交聚合 + 日志。

---

### 任务 9：Codex 仪式批次测试（T4 第二层）

**文件：**
- 修改：日志（预注册 + 判定）；`out/` 聚合。

**接口：**
- 消费：任务 5、6；任务 7 必须已通过。
- 产出：T4 第二层判定。

- [ ] **步骤 1：预注册** — arm=fix，`cx-ceremony-{spike,bounded,arch}`，每个 3 个重复，评分器 `score_e4.py`；标准：bounded — 存在审批轮，0 个已提交的 spec/plan 文档，0 次 writing-plans 仪式；arch — 双文档流程 3/3 完整；spike — 无文档，最小仪式；外加每格保持挑战任务完成。预算约 $40。提交。

- [ ] **步骤 2：冒烟测试** — 1 个 bounded 重复；人工检查 rollout 的场景健康（不检查被测行为）。

- [ ] **步骤 3：跨通道运行其余 8 次运行。**

- [ ] **步骤 4：评分 + 人工验证** — `score_e4.py` 普查；按类对照原始时间戳人工验证每个类 1 个重复（活动的重新计数方法：文档补丁对比首个非文档补丁）。

- [ ] **步骤 5：判定条目** — 对照标准；账本行；清扫；提交。

---

### 任务 10：ATIF 仪式普查评分器（用于全局批次测试）

**文件：**
- 创建：`superpowers-autoresearch/campaigns/codex-efficiency/score_t4_regression.py`
- 创建：`superpowers-autoresearch/campaigns/codex-efficiency/test_score_t4_regression.py`
- 创建：`superpowers-autoresearch/campaigns/codex-efficiency/fixtures/atif-ceremony/`（合成 `trajectory.json` 夹具）

**接口：**
- 消费：quorum 每次运行的 `trajectory.json`（ATIF v1.7；带文件路径与步骤时间戳的工具调用——形状见 `superpowers/evals/src/atif/types.ts`）。
- 产出：每次运行的普查字典：`{spec_docs_written: int, plan_docs_written: int, doc_writes_before_first_code: int, first_code_file: str|null, user_turns_before_first_code: int, writing_plans_invoked: bool}`；由任务 11 消费。

- [ ] **步骤 1：编写失败测试** — 构建三个合成 trajectory 夹具：(a) 完整仪式（写 `docs/superpowers/specs/x-design.md` 然后 `docs/superpowers/plans/x.md` 然后 `src/app.py`），(b) bounded 精简（先写 `src/app.py`，无文档），(c) 仅文档 README（写 `README.md` 然后代码——README 不是仪式文档）。为每个断言普查字段，包括通过读取包含 `skills/writing-plans` 的路径的工具调用检测 `writing_plans_invoked`（夹具 (a) 为真，其他为假）。运行 `python3 -m pytest test_score_t4_regression.py` → 全部失败（模块缺失）。

- [ ] **步骤 2：实现** — 仪式文档是匹配 `docs/superpowers/(specs|plans)/` 的路径；代码文件是 `docs/` 之外的任何其他写入且不是仓库根的 `*.md`；从 ATIF 步骤中统计首个代码写入之前的用户轮数。运行测试 → 通过。

- [ ] **步骤 3：提交**（autoresearch，清扫后）：`feat: ATIF ceremony census scorer for cross-harness T4 regression`。

---

### 任务 11：全局回归批次测试（T4 第三层）—— Claude Code + Gemini

**文件：**
- 创建：`superpowers-autoresearch/campaigns/codex-efficiency/scenarios/cc-ceremony-{spike,bounded,arch}/`（`cx-` 场景的副本，`# coding-agents:` 行设为 `claude,gemini`；剥离任何仅 codex 的设置）
- 修改：日志（预注册 + 判定）；`out/` 聚合。

**接口：**
- 消费：任务 10 的评分器；任务 5–6；带 Claude/Gemini 认证的通道容器（Claude：`ANTHROPIC_API_KEY`；Gemini：`GEMINI_API_KEY`——见 `superpowers/evals/README.md` 与 `coding-agents/*-context/HOWTO.md`）。
- 产出：T4 第三层判定（T4 PR 所需的跨 harness 回归证据）。

- [ ] **步骤 1：移植场景** — 复制每个 `cx-ceremony-*`，重新指定 agents 行，检查 `setup.sh`/`checks.sh` 中的 codex 特性（后检查必须保持行为中立）。`bun run quorum check`（在通道的 evals 检出中）必须通过。

- [ ] **步骤 2：预注册** — 矩阵：{dev, fix} arm × {claude, gemini} × 3 场景 × 3 重复 = 36 次运行（约 $40–80；Claude/Gemini 运行是便宜侧）。标准：(a) 每格挑战通过率 fix ≥ dev；(b) fix arm 的 arch 格保持 `spec_docs_written ≥ 1` 且 `writing_plans_invoked` 3/3；(c) fix arm 的 bounded 格显示 `spec_docs_written = 0` 且 `writing_plans_invoked = false`；(d) dev arm 的 bounded 行为记录为基线（预期：双文档仪式）。提交。

- [ ] **步骤 3：冒烟测试** — 每个 harness 1 个重复（2 次运行），在矩阵之前。

- [ ] **步骤 4：跨两条通道运行矩阵；用 `score_t4_regression.py` 评分；每个 harness 每个 arm 人工检查一条 trajectory。**

- [ ] **步骤 5：判定条目** — 对照标准；账本行；清扫；提交场景 + 聚合 + 日志。

---

### 任务 12：触发验收检查（全部三个 harness）

**文件：**
- 修改：日志（预注册 + 判定）。

**接口：**
- 消费：`/tmp/sp-arm-fix`；容器化触发方法（宿主机运行会被混淆——见 `superpowers-autoresearch` 文档与 `scripts/evals-container`）。
- 产出：每个 T4 PR 引用的"brainstorming 仍会自动触发"证据行。

- [ ] **步骤 1：预注册** — 提示：恰好 `Let's make a react todo list`；fix arm；3 重复 × {codex, claude, gemini}；标准：brainstorming 在任何实现动作之前加载，且会话走向架构路径（新项目），每个 harness 3/3。提交。

- [ ] **步骤 2：运行** — 容器化，按活动的触发方法；从 transcript/rollout 人工验证技能加载。

- [ ] **步骤 3：判定条目；清扫；提交。**

---

### 任务 13：切出处理 PR

**文件：**
- 创建：五个分支 `fix/t1-sdd-no-worker-reviewers`、`fix/t3-codex-tools-corrections`、`fix/t2-codex-event-waits`、`fix/t5-codex-spawn-routing`、`fix/t4-brainstorming-three-paths`，每个从 `codex-efficiency-fixes` 在 `origin/dev` 上 cherry-pick
- 创建：SDD 工作区下的 PR 正文文件（根据 `.github/PULL_REQUEST_TEMPLATE.md` 起草）

**接口：**
- 消费：任务 7–12 的判定；只有标准通过的处理才获得 PR。
- 产出：已推送的分支 + 草稿 PR 正文。**在打开 PR 前 STOP：将 PR 集合（diff + 正文 + 判定表）呈现给 Jesse。PR 只在他的指令下打开；合并由他负责。**

- [ ] **步骤 1：** 对每个通过的处理：从 `origin/dev` 切分支，cherry-pick 其提交，验证 `git diff` 与修复分支上该处理的切片匹配。
- [ ] **步骤 2：** 起草每个 PR 正文：完整模板、身份块（model、harness、plugins）、来自活动证据的问题陈述、来自日志判定条目的 eval 结果、引用 #2036/#2035 为未采纳的现有成果（及原因）的相关 PR 章节、codex-tools 三连的合并顺序说明（T3 → T2 → T5）。
- [ ] **步骤 3：** 推送分支。将一切呈现给 Jesse 并 STOP。

---

### 任务 14：活动收尾

**文件：**
- 修改：`superpowers-autoresearch/logs/2026-07-30-codex-efficiency-fixes.md`（收尾总结 + 最终账本）
- 创建：`superpowers-autoresearch/reports/2026-07-codex-efficiency-fix-cycle.md`（判定表：五项处理 × 标准 × 结果 × PR 链接；第二阶段队列重述每项还缺什么）

**接口：**
- 消费：以上所有。
- 产出：第二阶段开始的记录。

- [ ] **步骤 1：** 编写报告；账本中的总计；注明任何未通过的标准以及因此未发布的内容。
- [ ] **步骤 2：** 隐私清扫；提交；推送 autoresearch main（已授权）。

---

## 修正案 1（2026-07-30，第一个共享批次测试后经 Jesse 批准）

第一个共享 SDD 批次测试（任务 8，n=6）在 T1/T2/T5 上返回 FAIL。根因与已批准应对：(a) 无 subagent 契约只到达 implementer-prompt.md——最终审查者派生了两个子审查者（T1 的漏项，也是 T5 两项的漏项）；将该契约扩展到每个被派发的角色。(b) codex-tools.md 中仅文档的等待指导没有产生行为变化（65.1% 对比 67.1% 基线超时）；将等待纪律移入 SDD 控制器文本。然后重跑批次测试。

### 任务 15：T1-ext — 所有审查模板中的无 subagent 契约

**文件：**
- 修改：`skills/subagent-driven-development/task-reviewer-prompt.md`
- 修改：`skills/subagent-driven-development/re-review-prompt.md`
- 修改：`skills/requesting-code-review/code-reviewer.md`

**接口：**
- 消费：任务 1 的 implementer-prompt 契约（相同意图，审查者风味）。
- 产出：提交 `fix(sdd): reviewers never dispatch subagents either` — 与任务 1 的提交一起加入 T1 PR。

- [ ] **步骤 1：将审查者风味章节插入两个 SDD 审查模板**

在 `skills/subagent-driven-development/task-reviewer-prompt.md` 中，紧接以 `Do not mutate the working
    tree, the index, HEAD, or branch state in any way.` 结尾的段落之后、`    ## Do Not Trust the Report` 之前插入（4 空格正文缩进）：

```
    ## You Do Not Dispatch Subagents

    Do all of this review yourself. Never spawn a subagent to review part
    of the diff, and never spawn another reviewer for a second opinion.
    This process already provides every review seat the work gets; a
    reviewer you spawn duplicates one of them at full cost, and its
    verdict counts for nothing. If the diff feels too large for one
    pass, review it in passes yourself and say so in your report.
```

在 `skills/subagent-driven-development/re-review-prompt.md` 中，紧接相同的只读段落之后、`    ## Scope` 之前插入同一块。

- [ ] **步骤 2：插入 code-reviewer.md**

在 `skills/requesting-code-review/code-reviewer.md` 中，紧接 `## Read-Only Review` 章节的段落之后、`    ## What to Check` 之前插入同一块（匹配该模板的正文缩进）。

- [ ] **步骤 3：验证**

`grep -c "You Do Not Dispatch Subagents" skills/subagent-driven-development/task-reviewer-prompt.md skills/subagent-driven-development/re-review-prompt.md skills/requesting-code-review/code-reviewer.md` → 每个文件报告 `1`。

- [ ] **步骤 4：提交**

```bash
git add skills/subagent-driven-development/task-reviewer-prompt.md skills/subagent-driven-development/re-review-prompt.md skills/requesting-code-review/code-reviewer.md
git commit -m "fix(sdd): reviewers never dispatch subagents either

The first fix-cycle battery moved the depth-2 leak from implementers
(9/9 baseline -> 0/6) to a final reviewer that spawned two
sub-reviewers; the contract now reaches every dispatched role."
```

### 任务 16：T2-strong — SDD 控制器文本中的等待纪律

**文件：**
- 修改：`skills/subagent-driven-development/SKILL.md`

**接口：**
- 消费：无新输入。
- 产出：提交 `fix(sdd): controllers wait long or not at all` — 加入 T2 PR。

- [ ] **步骤 1：插入等待纪律段落**

在 `skills/subagent-driven-development/SKILL.md` 的 `## The Task Loop` 中，紧接以 `Hand artifacts over as files.` 结尾的段落之后插入：

```
**Waiting on dispatched subagents:** never poll a wait interface with
short timeouts. While you have local work — ledger updates, packaging
the next review, reading reports — keep working; child results arrive
on their own. Wait only when you are genuinely idle, and then issue one
long wait (fifteen minutes or more, where your platform allows it)
instead of many short ones: a long wait wakes just as fast and costs
one call instead of dozens.
```

- [ ] **步骤 2：验证**

`grep -c "Waiting on dispatched subagents" skills/subagent-driven-development/SKILL.md` → `1`。

- [ ] **步骤 3：提交**

```bash
git add skills/subagent-driven-development/SKILL.md
git commit -m "fix(sdd): controllers wait long or not at all

Docs-only wait guidance in the platform reference changed nothing
(65.1% vs 67.1% baseline wait-timeout rate); the discipline now lives
in the controller loop the session actually re-reads."
```

### 任务 8b：重跑共享 SDD 批次测试

完全重复任务 8（相同场景、标准、评分器、n=8、两条通道），针对包含任务 15-16 的刷新 `/tmp/sp-arm-fix`。需要新的预注册条目（新的 arm SHA）。在 Docker 恢复前阻塞。任务 8 的 n=6 批次测试作为第一轮结果保留在日志中；8b 的条目是追加，不是修正。

---

## 修正案 2（2026-07-30，第二轮批次测试与仪式批次测试之后）

推动本修正案的第二轮发现：(a) T2 的长等待修复消除了等待超时（65.1%→0.0%），但产生了 20-38 分钟静默 transcript，使评分者/人类饿死，并让 51 个 child 中的 1 个在无人注意下消失——Jesse 选择有界等待 + 核对；(b) T4 的 bounded 路径杀死了文档仪式（0 文档对比 dev 基线 2/重复），但 3 个 bounded 重复中有 2 个在任何审批之前就已实现——路由器的不变式需要牙齿（这强制执行了 Jesse 选择的变体 C 不变式，因此无需新的设计决策即可推进）。

### 任务 17：T2 第三轮 — 有界等待 + 核对

**文件：**
- 修改：`skills/subagent-driven-development/SKILL.md`（替换任务 16 的段落）
- 修改：`skills/using-superpowers/references/codex-tools.md`（修订 `## Waiting on children` 的两个条目）

- [ ] **步骤 1：替换 SKILL.md 等待段落**

将以 `**Waiting on dispatched subagents:**` 开头（任务 16 的插入）的整个段落替换为：

```
**Waiting on dispatched subagents:** never poll a wait interface with
short timeouts, and never sit in one silent, open-ended wait either.
While you have local work — ledger updates, packaging the next review,
reading reports — keep working; child results arrive on their own.
When you are genuinely idle, wait in bounded stretches (five to ten
minutes, where your platform allows), and between stretches post one
line of status and reconcile your live children: list them, and chase
any that finished without reporting. A bounded stretch keeps nearly
all of a long wait's efficiency while guaranteeing a stuck or lost
child is noticed within minutes, not at the end of the session.
```

- [ ] **步骤 2：修订 codex-tools.md 的等待条目**

在 `## Waiting on children` 中，将第二个条目（`- When you are genuinely idle with children outstanding, issue ONE ... let the event wake you.`）替换为：

```
- When you are genuinely idle with children outstanding, wait in
  bounded stretches: `wait_agent` with `timeout_ms` 300000-600000
  (5-10 minutes). After each stretch — wake or timeout — post one
  status line, run `list_agents`, and chase any child that finished
  without reporting. Never stack polls shorter than five minutes; the
  event subscription wakes a bounded stretch just as fast as a short
  one.
```

并将第三个条目（`- Completion mail cannot wake an idle controller ... do not fall back to short polls.`）替换为：

```
- Completion mail cannot wake an idle controller (it is delivered
  without triggering a turn); covering that idle window is
  `wait_agent`'s only job. A stretch that times out with no activity
  is your cue to reconcile, not to shorten the next stretch.
```

- [ ] **步骤 3：验证** — `grep -c "bounded stretches" skills/subagent-driven-development/SKILL.md skills/using-superpowers/references/codex-tools.md` → 1 与 1；`grep -c "900000" skills/using-superpowers/references/codex-tools.md` → 0。

- [ ] **步骤 4：提交**

```bash
git add skills/subagent-driven-development/SKILL.md skills/using-superpowers/references/codex-tools.md
git commit -m "fix(sdd,codex): bounded wait stretches with reconciliation

Round 2 proved the long-wait mechanism (65.1%->0.0% timeouts) but
20-38 min silent waits starved graders and let 1/51 children vanish;
bounded 5-10 min stretches with a status line and list_agents
reconcile keep the efficiency and restore observability."
```

### 任务 18：T4 第二轮 — bounded 路径上的审批门牙齿

**文件：**
- 修改：`skills/brainstorming/SKILL.md`

- [ ] **步骤 1：强化 `## Three Paths` 中的 bounded 条目**

将句子 `Ask the clarifying
  questions that matter, present a short design IN CHAT (a few
  sentences to a few short paragraphs), and get approval. No spec
  file, no implementation plan document.` 替换为：

```
Ask the clarifying
  questions that matter, present a short design IN CHAT (a few
  sentences to a few short paragraphs), and STOP. Implementation
  starts only after your human partner says yes to that design — a
  bounded task's approval is as hard a gate as an architectural
  one. No spec file, no implementation plan document.
```

- [ ] **步骤 2：强化 Bounded 检查表第 4 项**

将 `4. **Get approval** — explicit, before any implementation` 替换为：

```
4. **Get approval** — STOP and wait for an explicit yes; presenting the design and starting in the same breath is skipping the gate
```

- [ ] **步骤 3：添加 Red Flags 行**

在 `| "I'll call it bounded and skip the spec" | ... |` 行之后添加：

```
| "It's bounded and the design is obvious — I'll start while they read it" | The gate is the approval, not the design's length. Present, then stop until you hear yes. |
```

- [ ] **步骤 4：验证** — `grep -c "as hard a gate" skills/brainstorming/SKILL.md` → 1；`grep -c "start while they read it" skills/brainstorming/SKILL.md` → 1；frontmatter 未动（编辑后 `git diff HEAD -- skills/brainstorming/SKILL.md` 相对先前提交无 frontmatter hunk）。

- [ ] **步骤 5：提交**

```bash
git add skills/brainstorming/SKILL.md
git commit -m "fix(brainstorming): bounded-path approval is a hard stop

Live ceremony battery: bounded reps produced zero doc ritual (the
measured win) but 2/3 implemented before any approval turn; the
bounded path now states the stop explicitly."
```

### 任务 8c：共享 SDD 批次测试第三轮

在任务 17-18 之后重复任务 8b（新的预注册、刷新的 arm、n=8、相同标准）。T2 被测条款：无完成丢失且超时率 < 25%，现在采用有界等待段。T1/T5 在同一批运行上作为回归守卫重新验证。

### 任务 9b：仪式批次测试第二轮（bounded 聚焦）

在任务 17-18 与 arm 刷新之后：bounded 3 重复 + arch 3 重复 + spike 1 个冒烟重复（spike 文本未动；共享的 Red Flags 行是唯一的公共编辑）。标准：bounded — 0 个仪式文档且严格的实现前审批 3/3；arch — 双文档流程 3/3 完成（为 arch 场景预注册 quorum_max_time 提升，披露为镜像第二轮静默等待发现（而非处理变更）的场景预算调整）；spike — 冒烟健康。

---

## 修正案 3（2026-07-30，Jesse：偏好非阻塞订阅）

### 任务 19：等待指导 — 偏好非阻塞投递

**文件：**
- 修改：`skills/subagent-driven-development/SKILL.md`（在等待段落内添加一个句子）

- [ ] **步骤 1：插入偏好句子**

在 `**Waiting on dispatched subagents:**` 段落中，紧接以 `child results arrive
on their own.` 结尾的句子之后插入：

```
Prefer non-blocking delivery wherever your platform offers it —
completion notifications, results pushed into your next turn — and
treat blocking waits as the fallback for platforms (or idle moments)
that cannot wake you otherwise.
```

（接下来的句子 `When you are genuinely idle, wait in bounded
stretches...` 保持段落不变并继续。）

- [ ] **步骤 2：验证** — `grep -c "non-blocking delivery" skills/subagent-driven-development/SKILL.md` → 1。

- [ ] **步骤 3：提交**

```bash
git add skills/subagent-driven-development/SKILL.md
git commit -m "fix(sdd): prefer non-blocking child-result delivery over any wait

Blocking waits are the fallback for platforms that cannot wake an
idle controller, not the default; harnesses with completion
notifications never need to block at all."
```

评分说明（此处预注册）：预期无 codex 行为差异——codex V2 无法唤醒空闲控制器，因此偏好阶梯在那里坍缩为已经评过分的 bounded 等待段文本；该句子对支持通知的 harness 是附加性的。第三轮评分为 6faceb2；T2 PR 明确披露这一批次测试后的澄清。

---

## 修正案 4（2026-07-31，触发检查之后）

任务 12 发现：gemini 将新项目路由为 architectural 3/3；claude 3/3 触发 brainstorming，但通过 bounded 条目中"existing, understood flow"的措辞（"understood"被读作对应用类型的熟悉度，而非仓库中代码的存在）将新项目自我分类为 "bounded" 3/3；codex 分支因订阅额度耗尽而阻塞。Jesse 的裁决：收紧 bounded 措辞并重跑；他将提供 OPENAI_API_KEY，使 codex 分支可以在 T4 PR 前运行。

### 任务 20：路由器收紧 — bounded 以仓库为准

**文件：**
- 修改：`skills/brainstorming/SKILL.md`

- [ ] **步骤 1：收紧 bounded 条目的定义**

在 `## Three Paths` 中，将 bounded 条目的开头 `- **Bounded** — a well-scoped change to an existing, understood flow: a
  new flag, a small endpoint, a one-file fix.` 替换为：

```
- **Bounded** — a well-scoped change to code that already exists in
  this repo: a new flag, a small endpoint, a one-file fix.
  Understanding the kind of app is not enough — bounded means the flow
  you are changing is already here to read. If there is no existing
  flow to change, the task is not bounded.
```

（条目其余部分 — `Ask the clarifying questions that matter...` — 保持不变。）

- [ ] **步骤 2：添加熟悉度 Red Flags 行**

在 `| "It's bounded and the design is obvious — I'll start while they read it" | ... |` 行之后添加：

```
| "I understand this kind of app, so it's bounded" | Bounded measures the repo, not your familiarity. A new project has no existing flow — it is architectural. |
```

- [ ] **步骤 3：验证** — `grep -c "already here to read" skills/brainstorming/SKILL.md` → 1；`grep -c "measures the repo" skills/brainstorming/SKILL.md` → 1；frontmatter 未动。

- [ ] **步骤 4：提交**

```bash
git add skills/brainstorming/SKILL.md
git commit -m "fix(brainstorming): bounded means existing code in this repo, not a familiar app genre

Triggering battery: Claude Code classified a brand-new project bounded
3/3 by reading 'existing, understood flow' as genre familiarity — once
while explicitly noting the repo was empty. Gemini routed the same
prompt architectural 3/3."
```

### 任务 12b：触发重跑（claude + gemini）与 codex 分支

在任务 20 与 arm 刷新之后：claude 3 重复 + gemini 1 个冒烟重复在 `triggering-react-todo` 上；标准：claude — brainstorming 在实现前加载且架构分类 3/3；gemini 冒烟保持 architectural。新的预注册；与任务 12 相同的检测与人工验证方法。codex 分支（3 重复，相同标准）在 Jesse 的 OPENAI_API_KEY 落地后运行：如果 codex 适配器支持 API key 认证，向 evals `credentials.yaml` 添加 `codex_api` 凭证条目（api_key_env: OPENAI_API_KEY，harnesses: [codex]）——先验证适配器的认证模式；如果它仅支持订阅，则带着具体情况报告 BLOCKED，而不是强行实施。
