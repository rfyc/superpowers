# SDD 任务范围评审派发实施计划

> **适用于智能体工作者：** 必读子技能：使用 superpowers:subagent-driven-development（推荐）或 superpowers:executing-plans 逐步实施本计划。步骤使用复选框（`- [ ]`）语法进行跟踪。

**目标：** 将 SDD 的逐任务评审限定在任务范围内（diff 优先的阅读、合理的扩展、不重复运行测试），同时最终分支评审保持宽泛。

**架构：** 对 subagent-driven-development 技能做四处文案编辑（逐任务质量提示词改为自包含，不再委托给 merge-readiness 模板；规格提示词新增第三个判定通道与有根据的怀疑；实施者提示词新增"修复后重新运行"规则；SKILL.md 新增控制器指导），并在 `evals/` 子模块中新增一个 eval 场景。`skills/requesting-code-review/` 刻意不动。

**技术栈：** Markdown 技能文件；用于 quorum eval 的 Python setup 助手 + bash 检查 + story.md。

**规格：** `docs/superpowers/specs/2026-06-09-sdd-task-scoped-review-dispatch-design.md` — 开始前先阅读。其中已定下的决策：完整重审保留；两个评审阶段保持分离；协调器保留模型判断；`requesting-code-review/` 保持宽泛。

**这些是塑造行为的文案文件，不是代码。** 它们没有单元测试。每个任务的验证步骤都是精确的 `grep` 检查，确认编辑已落地；行为验证是任务 6（静态）和任务 7（实时 evals，维护者把关）。

---

### 任务 1：将逐任务质量评审者提示词改写为自包含

当前文件委托给 `../requesting-code-review/code-reviewer.md`，后者是 merge-readiness 评审（架构、安全、生产就绪、"Ready to merge?"）。将整个文件替换为自包含、任务范围的模板。

**文件：**
- 重写：`skills/subagent-driven-development/code-quality-reviewer-prompt.md`

- [ ] **步骤 1：将完整文件内容替换为：**

````markdown
# Code Quality Reviewer Prompt Template

Use this template when dispatching a code quality reviewer subagent.

**Purpose:** Verify one task's implementation is well-built (clean, tested, maintainable)

**Only dispatch after spec compliance review passes.**

```
Subagent (general-purpose):
  description: "Review code quality for Task N"
  prompt: |
    You are reviewing one task's implementation for code quality. This is a
    task-scoped gate, not a merge review — a broad whole-branch review happens
    separately after all tasks are complete.

    ## What Was Implemented

    [DESCRIPTION]

    ## Task Requirements (context only)

    [TASK_TEXT]

    ## Git Range to Review

    **Base:** [BASE_SHA]
    **Head:** [HEAD_SHA]

    ```bash
    git diff --stat [BASE_SHA]..[HEAD_SHA]
    git diff [BASE_SHA]..[HEAD_SHA]
    ```

    ## Read-Only Review

    Your review is read-only on this checkout. Do not mutate the working tree,
    the index, HEAD, or branch state in any way. Use tools like `git show`,
    `git diff`, and `git log` to inspect history.

    ## Scope

    Spec compliance was already verified by a separate reviewer. Do not
    re-check whether the code matches the requirements or the plan.

    Start from the diff. Read the changed files first. Inspect code outside
    the diff only to evaluate a concrete risk you can name — and name it in
    your report. Cross-cutting changes are legitimate named risks: if the
    diff changes lock ordering, a function or API contract, or shared mutable
    state, checking the call sites is the right method. Do not crawl the
    codebase by default.

    ## Tests

    The implementer already ran the tests and reported results with TDD
    evidence for exactly this code. Do not re-run the suite to confirm their
    report. Run a test only when reading the code raises a specific doubt
    that no existing run answers — and then a focused test, never a
    package-wide suite, race detector run, or repeated/high-count loop. If
    heavy validation seems warranted, recommend it in your report instead of
    running it. If you cannot run commands in this environment, name the
    test you would run.

    ## What to Check

    **Code quality:**
    - Clean separation of concerns?
    - Proper error handling?
    - DRY without premature abstraction?
    - Edge cases handled?

    **Tests:**
    - Do the new and changed tests verify real behavior, not mocks?
    - Are the task's edge cases covered?

    **Structure:**
    - Does each file have one clear responsibility with a well-defined interface?
    - Are units decomposed so they can be understood and tested independently?
    - Is the implementation following the file structure from the plan?
    - Did this change create new files that are already large, or
      significantly grow existing files? (Don't flag pre-existing file
      sizes — focus on what this change contributed.)

    ## Calibration

    Categorize issues by actual severity. Not everything is Critical.
    Acknowledge what was done well before listing issues — accurate praise
    helps the implementer trust the rest of the feedback.

    ## Output Format

    ### Strengths
    [What's well done? Be specific.]

    ### Issues

    #### Critical (Must Fix)
    [Bugs, data loss risks, broken functionality]

    #### Important (Should Fix)
    [Poor error handling, test gaps, structural problems]

    #### Minor (Nice to Have)
    [Code style, optimization opportunities]

    For each issue:
    - File:line reference
    - What's wrong
    - Why it matters
    - How to fix (if not obvious)

    ### Assessment

    **Task quality:** [Approved | Needs fixes]

    **Reasoning:** [1-2 sentence technical assessment]
```

**Placeholders:**
- `[DESCRIPTION]` — task summary, from implementer's report
- `[TASK_TEXT]` — the task's requirements text or plan reference, for context
- `[BASE_SHA]` — commit before this task
- `[HEAD_SHA]` — current commit

**Reviewer returns:** Strengths, Issues (Critical/Important/Minor), Task quality verdict
````

- [ ] **步骤 2：验证改写已落地**

运行：`grep -c "requesting-code-review" skills/subagent-driven-development/code-quality-reviewer-prompt.md || echo ABSENT`
预期：`ABSENT`（不再有委托）

运行：`grep -n "Task quality:" skills/subagent-driven-development/code-quality-reviewer-prompt.md | head -2`
预期：一处匹配（Output Format 判定行；"Reviewer returns" 页脚写的是 "Task quality verdict"，不带冒号）

运行：`grep -n "worktree add\|Ready to merge" skills/subagent-driven-development/code-quality-reviewer-prompt.md || echo CLEAN`
预期：`CLEAN`

- [ ] **步骤 3：提交**

```bash
git add skills/subagent-driven-development/code-quality-reviewer-prompt.md
git commit -m "Make per-task quality reviewer prompt self-contained and task-scoped"
```

---

### 任务 2：规格评审者提示词清理

对 `skills/subagent-driven-development/spec-reviewer-prompt.md` 做四处精确编辑。当前行号针对提交 f55642e 时的文件。

**文件：**
- 修改：`skills/subagent-driven-development/spec-reviewer-prompt.md`

- [ ] **步骤 1：添加"从 diff 判定"条款。** 在下面这行之后（当前第 31 行）：

```
    Only read files in this diff. Do not crawl the broader codebase.
```

插入一个空行和：

```
    Spec compliance is judged by reading the diff against the requirements.
    The implementer already ran the tests and reported TDD evidence — do not
    re-run them. If a requirement cannot be verified from this diff alone
    (it lives in unchanged code or spans tasks), report it as a ⚠️ item
    instead of broadening your search.
```

- [ ] **步骤 2：精简只读部分。** 将（当前第 35 行）：

```
    Your review is read-only on this checkout. Do not mutate the working tree, the index, HEAD, or branch state in any way. Use tools like `git show`, `git diff`, and `git log` to inspect history. If you need a working copy of a different revision, check it out into a separate temporary directory (e.g. `git worktree add /tmp/review-[SHA] [SHA]`) — never move HEAD on this checkout.
```

替换为：

```
    Your review is read-only on this checkout. Do not mutate the working tree, the index, HEAD, or branch state in any way. Use tools like `git show`, `git diff`, and `git log` to inspect history.
```

- [ ] **步骤 3：让怀疑有根据。** 将（当前第 39-40 行）：

```
    The implementer finished suspiciously quickly. Their report may be incomplete,
    inaccurate, or optimistic. You MUST verify everything independently.
```

替换为：

```
    Treat the implementer's report as unverified claims about the code. It may
    be incomplete, inaccurate, or optimistic. Verify the claims against the diff.
```

- [ ] **步骤 4：添加第三个判定通道。** 将（当前第 74-76 行）：

```
    Report:
    - ✅ Spec compliant (if everything matches after code inspection)
    - ❌ Issues found: [list specifically what's missing or extra, with file:line references]
```

替换为：

```
    Report:
    - ✅ Spec compliant (if everything matches after code inspection)
    - ❌ Issues found: [list specifically what's missing or extra, with file:line references]
    - ⚠️ Cannot verify from diff: [requirements you could not verify from the
      diff alone, and what the controller should check — report alongside the
      ✅/❌ verdict for everything you could verify]
```

- [ ] **步骤 5：验证**

运行：`grep -n "suspiciously\|worktree add" skills/subagent-driven-development/spec-reviewer-prompt.md || echo CLEAN`
预期：`CLEAN`

运行：`grep -c "⚠️" skills/subagent-driven-development/spec-reviewer-prompt.md`
预期：`2`（从 diff 判定条款 + 判定通道）

- [ ] **步骤 6：提交**

```bash
git add skills/subagent-driven-development/spec-reviewer-prompt.md
git commit -m "Spec reviewer: judge from the diff, grounded skepticism, ⚠️ verdict channel"
```

---

### 任务 3：实施者提示词 — 修复评审发现后重新运行测试

评审者的"不要重新运行实施者的测试"规则假设实施者在每次修复后都会重新运行测试。让这一点成真。

**文件：**
- 修改：`skills/subagent-driven-development/implementer-prompt.md`

- [ ] **步骤 1：插入一个新章节。** 紧接在下面这行之前（当前第 100 行）：

```
    ## Report Format
```

插入：

```
    ## After Review Findings

    If a reviewer finds issues and you fix them, re-run the tests that cover
    the amended code and include the results in your fix report. Reviewers
    will not re-run tests for you — your report is the test evidence.

```

- [ ] **步骤 2：验证**

运行：`grep -n "After Review Findings" skills/subagent-driven-development/implementer-prompt.md`
预期：一处匹配，位于 `## Report Format` 之前的一行

- [ ] **步骤 3：提交**

```bash
git add skills/subagent-driven-development/implementer-prompt.md
git commit -m "Implementer prompt: re-run covering tests after fixing review findings"
```

---

### 任务 4：SKILL.md 控制器变更

对 `skills/subagent-driven-development/SKILL.md` 做六处精确编辑。当前行号针对提交 f55642e。

**文件：**
- 修改：`skills/subagent-driven-development/SKILL.md`

- [ ] **步骤 1：将最终评审流程图节点指向宽泛模板。** 标签为 `Dispatch final code reviewer subagent for entire implementation` 的节点出现 3 次（当前第 65、84、85 行）。在这 3 处出现中，将标签字符串替换为：

```
Dispatch final code reviewer subagent (../requesting-code-review/code-reviewer.md)
```

（Graphviz 节点按标签文本匹配 — 三者必须逐字节一致，否则图会多出一个幽灵节点。）

- [ ] **步骤 2：按判断选择模型。** 将（当前第 97-99 行）：

```
**Architecture, design, and review tasks**: use the most capable available model.

**Task complexity signals:**
```

替换为：

```
**Architecture and design tasks**: use the most capable available model.

**Review tasks**: choose the model with the same judgment, scaled to the
diff's size, complexity, and risk. A small mechanical diff does not need the
most capable model; a subtle concurrency change does.

**Task complexity signals (implementation tasks):**
```

- [ ] **步骤 3：添加控制器指导章节。** 紧接在下面这行之前（当前第 122 行）：

```
## Prompt Templates
```

插入：

```
## Handling Spec Reviewer ⚠️ Items

The spec reviewer may report "⚠️ Cannot verify from diff" items — requirements
that live in unchanged code or span tasks. These do not block dispatching the
code quality reviewer, but you must resolve each one yourself before marking
the task complete: you hold the plan and cross-task context the reviewer
lacks. If you confirm an item is a real gap, treat it as a failed spec
review — send it back to the implementer and re-review.

## Constructing Reviewer Prompts

Per-task reviews are task-scoped gates. The broad review happens once, at the
final whole-branch review. When you fill a reviewer template:

- Do not add open-ended directives like "check all uses" or "run race tests
  if useful" without a concrete, task-specific reason
- Do not ask a reviewer to re-run tests the implementer already ran on the
  same code — the implementer's report carries the test evidence

```

- [ ] **步骤 4：Prompt Templates 列表 — 添加最终评审指针。** 将（当前第 126 行）：

```
- [code-quality-reviewer-prompt.md](code-quality-reviewer-prompt.md) - Dispatch code quality reviewer subagent
```

替换为：

```
- [code-quality-reviewer-prompt.md](code-quality-reviewer-prompt.md) - Dispatch code quality reviewer subagent
- Final whole-branch review: use superpowers:requesting-code-review's [code-reviewer.md](../requesting-code-review/code-reviewer.md)
```

- [ ] **步骤 5：示例工作流的判定词汇。** 两处替换：

将（当前第 157 行）：
```
Code reviewer: Strengths: Good test coverage, clean. Issues: None. Approved.
```
替换为：
```
Code reviewer: Strengths: Good test coverage, clean. Issues: None. Task quality: Approved.
```

将（当前第 191 行）：
```
Code reviewer: ✅ Approved
```
替换为：
```
Code reviewer: ✅ Task quality: Approved
```

（最终评审者的 "ready to merge" 行，当前第 199 行，保持不变。）

- [ ] **步骤 6：Integration 部分。** 将（当前第 272 行）：

```
- **superpowers:requesting-code-review** - Code review template for reviewer subagents
```

替换为：

```
- **superpowers:requesting-code-review** - Code review template for the final whole-branch review
```

- [ ] **步骤 7：验证**

运行：`grep -c "Dispatch final code reviewer subagent (../requesting-code-review/code-reviewer.md)" skills/subagent-driven-development/SKILL.md`
预期：`3`

运行：`grep -n "most capable available model" skills/subagent-driven-development/SKILL.md`
预期：恰好一处匹配（架构/设计条目）

运行：`grep -n "Handling Spec Reviewer\|Constructing Reviewer Prompts" skills/subagent-driven-development/SKILL.md`
预期：两个章节标题，均位于 `## Prompt Templates` 之前

运行：`grep -c "Task quality: Approved" skills/subagent-driven-development/SKILL.md`
预期：`2`

- [ ] **步骤 8：提交**

```bash
git add skills/subagent-driven-development/SKILL.md
git commit -m "SDD controller: reviewer prompt budgets, ⚠️ handling, final-review pointer, model judgment"
```

---

### 任务 5：新增 eval 场景 — 逐任务质量评审者捕获植入的缺陷

位于 `evals/` **子模块**中（独立仓库，`superpowers-evals`）。在那里开分支工作；父级子模块指针的推进在收尾时按 `evals/CLAUDE.md` 进行。

fixture 计划的 Task 2 实现片段逐字重复了 Task 1 的格式化逻辑。该重复符合规格，因此规格评审者应放行它 — 被测试的把关者是逐任务质量评审者（DRY 违规）。

**文件：**
- 创建：`evals/setup_helpers/sdd_quality_defect_plan.py`
- 修改：`evals/setup_helpers/__init__.py`
- 创建：`evals/scenarios/sdd-quality-reviewer-catches-planted-defect/story.md`
- 创建：`evals/scenarios/sdd-quality-reviewer-catches-planted-defect/setup.sh`
- 创建：`evals/scenarios/sdd-quality-reviewer-catches-planted-defect/checks.sh`

- [ ] **步骤 0：在子模块中开分支**

```bash
cd evals
git checkout -b sdd-quality-defect-scenario
```

- [ ] **步骤 1：创建 `evals/setup_helpers/sdd_quality_defect_plan.py`：**

````python
"""Setup helper for the sdd-quality-reviewer-catches-planted-defect scenario.

Scaffolds a tiny Node project with a 2-task plan whose Task 2
implementation snippet duplicates Task 1's formatting logic verbatim.
The duplication is spec-compliant — the requirements only describe
behavior — so the spec compliance reviewer should pass it. The test
measures whether the per-task code quality reviewer catches the DRY
violation and forces a refactor in the review-fix loop.
"""

from __future__ import annotations

from pathlib import Path

from setup_helpers.base import _git

PACKAGE_JSON = """\
{
  "name": "report-quality",
  "version": "1.0.0",
  "type": "module",
  "scripts": {
    "test": "node --test"
  }
}
"""

PLAN_BODY = """\
# Report Formatter — Implementation Plan

Two report formatting functions. Implement exactly what each task
specifies.

## Task 1: User Report

**File:** `src/report.js`

**Requirements:**
- Function named `formatUserReport`
- Takes one parameter `user`: an object with `name`, `email`, `visits`
- Returns a multi-line string: a banner of 40 `=` characters, then
  `Report for <name> <<email>>`, then the banner again, then
  `Visits: <visits>`, then a closing banner
- Export the function

**Implementation:**
```javascript
export function formatUserReport(user) {
  const banner = "=".repeat(40);
  const lines = [];
  lines.push(banner);
  lines.push(`Report for ${user.name} <${user.email}>`);
  lines.push(banner);
  lines.push(`Visits: ${user.visits}`);
  lines.push(banner);
  return lines.join("\\n");
}
```

**Tests:** Create `test/report.test.js` verifying:
- the result contains `Report for Ada <ada@example.com>` for that user
- the result contains `Visits: 3` when `visits` is `3`
- the result starts and ends with the 40-char banner

**Verification:** `npm test`

## Task 2: Admin Report

**File:** `src/report.js` (add to existing file)

**Requirements:**
- Function named `formatAdminReport`
- Takes one parameter `admin`: an object with `name`, `email`, `lastLogin`
- Same banner layout as the user report; the body line is
  `Last login: <lastLogin>` instead of the visits line
- Export the function; keep `formatUserReport` working

**Implementation:**
```javascript
export function formatAdminReport(admin) {
  const banner = "=".repeat(40);
  const lines = [];
  lines.push(banner);
  lines.push(`Report for ${admin.name} <${admin.email}>`);
  lines.push(banner);
  lines.push(`Last login: ${admin.lastLogin}`);
  lines.push(banner);
  return lines.join("\\n");
}
```

**Tests:** Add to `test/report.test.js`:
- the result contains `Report for Grace <grace@example.com>` for that admin
- the result contains `Last login: 2026-06-01`
- the result starts and ends with the 40-char banner

**Verification:** `npm test`
"""


def scaffold_sdd_quality_defect_plan(workdir: Path) -> None:
    workdir = Path(workdir)
    workdir.mkdir(parents=True, exist_ok=True)
    _git(["git", "init", "-b", "main"], cwd=workdir)
    _git(["git", "config", "user.email", "drill@test.local"], cwd=workdir)
    _git(["git", "config", "user.name", "Drill Test"], cwd=workdir)

    (workdir / "package.json").write_text(PACKAGE_JSON)
    plans_dir = workdir / "docs" / "superpowers" / "plans"
    plans_dir.mkdir(parents=True, exist_ok=True)
    (plans_dir / "report-plan.md").write_text(PLAN_BODY)

    _git(["git", "add", "-A"], cwd=workdir)
    _git(["git", "commit", "-m", "initial: report formatter plan"], cwd=workdir)
````

（注意 PLAN_BODY 内 JS 片段中的 `\\n`：Python 源码必须在 markdown 中产出字面量 `\n`，这样 JS 读到的是 `lines.join("\n")`。）

- [ ] **步骤 2：注册助手。** 在 `evals/setup_helpers/__init__.py` 中：

在下面这行之后：
```python
from setup_helpers.sdd_real_projects import scaffold_sdd_go_fractals, scaffold_sdd_svelte_todo
```
添加：
```python
from setup_helpers.sdd_quality_defect_plan import scaffold_sdd_quality_defect_plan
```

在注册表条目之后：
```python
    "scaffold_sdd_yagni_plan": scaffold_sdd_yagni_plan,
```
添加：
```python
    "scaffold_sdd_quality_defect_plan": scaffold_sdd_quality_defect_plan,
```

- [ ] **步骤 3：创建 `evals/scenarios/sdd-quality-reviewer-catches-planted-defect/story.md`：**

```markdown
---
id: sdd-quality-reviewer-catches-planted-defect
title: SDD's per-task code quality review catches a planted DRY violation
status: ready
tags: subagent-driven-development
quorum_max_time: 90m
---

You have a small plan at docs/superpowers/plans/report-plan.md — two report
formatting functions. The plan's Task 2 implementation snippet duplicates
Task 1's formatting logic verbatim instead of sharing it. The duplication is
spec-compliant (the requirements only describe behavior), so the spec
compliance reviewer should pass it — the per-task code quality reviewer is
the gate under test. You are spec-aware — name the skill.

When the agent is ready for input, tell it to execute the plan with SDD. Use
phrasing like:

"I have a small plan at docs/superpowers/plans/report-plan.md — two report
formatting functions. Use the superpowers:subagent-driven-development skill
to execute it end-to-end — dispatch fresh subagents per task and run the
two-stage review after each."

Let the agent proceed autonomously. If it asks clarifying questions, give
brief answers. If it asks where the finished work should land — merge to the
main branch, open a PR, etc. — tell it to **merge the work into the main
checkout** (this is a local repo with no remote). If a quality reviewer
flags the duplicated formatting logic and an implementer refactors it, let
the review-fix cycle play out — that cycle is exactly the behavior under
test.

The deliverable must end up in the checkout you launched in (the main
working tree). If the agent did its work on a branch or in a worktree, it
is not done until it has merged/finished that work back into the main
checkout. Once the agent reports the plan is complete (both functions
implemented, tests passing) AND the code is present on the main checkout,
you are done.

## Acceptance Criteria

- A `Skill` invocation naming `superpowers:subagent-driven-development`
  and at least one `Agent` (subagent dispatch) tool call appear in the
  session log.
- The duplicated report-formatting logic did not survive to the end of
  the run. Either (a) the implementer never introduced the duplication
  (wrote or self-reviewed its way to shared logic), or (b) the per-task
  code quality reviewer flagged the duplication as an issue and a
  review-fix loop removed it. A fail looks like the duplicated logic
  shipping with the per-task quality reviewer approving it, or the
  duplication being caught only by the final whole-branch review.
- The per-task quality reviewers stayed task-scoped: no package-wide
  test suites, race detector runs, or repeated/high-count test loops
  appear in reviewer subagent activity, and reviewers did not re-run
  the full test suite merely to confirm the implementer's report.
- `npm test` passes in the main checkout and both `formatUserReport` and
  `formatAdminReport` are exported from src/report.js. The deterministic
  assertions gate this; the criteria above are about whether the
  *per-task quality review* was the mechanism that kept the code clean.
```

- [ ] **步骤 4：创建 `evals/scenarios/sdd-quality-reviewer-catches-planted-defect/setup.sh`：**

```bash
#!/usr/bin/env bash
set -euo pipefail
uv run setup-helpers run scaffold_sdd_quality_defect_plan
```

然后：`chmod +x evals/scenarios/sdd-quality-reviewer-catches-planted-defect/setup.sh`

- [ ] **步骤 5：创建 `evals/scenarios/sdd-quality-reviewer-catches-planted-defect/checks.sh`**（无可执行位）：

```bash
pre() {
    git-repo
    git-branch main
    requires-tool npm
    file-exists 'docs/superpowers/plans/report-plan.md'
    file-contains 'docs/superpowers/plans/report-plan.md' 'formatAdminReport'
    file-contains 'docs/superpowers/plans/report-plan.md' 'repeat\(40\)'
}

post() {
    skill-called superpowers:subagent-driven-development
    tool-called Agent
    command-succeeds 'npm test'
    file-contains 'src/report.js' 'export function formatUserReport'
    file-contains 'src/report.js' 'export function formatAdminReport'
    command-succeeds 'test "$(grep -c "repeat(40)" src/report.js)" -le 1'
}
```

（最后一项检查是确定性的 DRY 门禁：横幅构造 `"=".repeat(40)` 在最终文件中至多出现一次 — 被共享，而非逐函数重复。）

- [ ] **步骤 6：在 evals 仓库中验证与测试**

```bash
cd evals
uv run quorum check
uv run ruff check
uv run pytest -x -q
```

预期：全部通过；`quorum check` 无错误地列出新场景。

- [ ] **步骤 7：提交（在子模块中）**

```bash
cd evals
git add setup_helpers/sdd_quality_defect_plan.py setup_helpers/__init__.py scenarios/sdd-quality-reviewer-catches-planted-defect/
git commit -m "Add sdd-quality-reviewer-catches-planted-defect scenario"
```

---

### 任务 6：静态验证扫描

**文件：** 无修改 — 仅验证。

- [ ] **步骤 1：父仓库中无悬空引用**

运行：`grep -rn "requesting-code-review" skills/subagent-driven-development/`
预期：仅 SKILL.md 中匹配（最终评审流程图节点 ×3、Prompt Templates 指针、Integration 条目）。code-quality-reviewer-prompt.md 中无匹配。

运行：`grep -rn "Ready to merge" skills/subagent-driven-development/ || echo CLEAN`
预期：`CLEAN`

- [ ] **步骤 2：插件基础设施测试**

运行：`bash tests/shell-lint/test-lint-shell.sh`
预期：全部 PASS（我们只在 evals 子模块内添加了 `setup.sh`，该子模块有自己的检查）。

- [ ] **步骤 3：跨平台工具表仍保持一致**

运行：`grep -n "code-quality-reviewer" skills/using-superpowers/references/antigravity-tools.md skills/using-superpowers/references/gemini-tools.md`
预期：两张表仍将 `code-quality-reviewer` 列为评审者模板（新提示词中的"If you cannot run commands in this environment, name the test you would run"一行保持了只读 `research` 映射的有效性 — 无需编辑表格）。

---

### 任务 7：实时前后对比 evals（维护者把关）

实时 quorum 运行会以宽松模式启动 agent CLI — **受信任维护者操作；由 Jesse 启动**，按 `evals/CLAUDE.md`。需要 `ANTHROPIC_API_KEY`。

- [ ] **步骤 1：基线（dev 上发布的技能）** — 从主 checkout（`/Users/jesse/git/superpowers/superpowers`，位于 dev）或任何不含本分支变更的 checkout 运行：

```bash
cd evals
export SUPERPOWERS_ROOT=/Users/jesse/git/superpowers/superpowers
uv run quorum run scenarios/sdd-rejects-extra-features --coding-agent claude
uv run quorum run scenarios/sdd-go-fractals --coding-agent claude
uv run quorum run scenarios/sdd-svelte-todo --coding-agent claude
uv run quorum run scenarios/spec-reviewer-catches-planted-flaws --coding-agent claude
```

- [ ] **步骤 2：之后（本分支的技能）** — 将 `SUPERPOWERS_ROOT` 指向本 worktree：

```bash
cd evals
export SUPERPOWERS_ROOT=/Users/jesse/git/superpowers/superpowers/.claude/worktrees/sdd-review-dispatch
uv run quorum run scenarios/sdd-rejects-extra-features --coding-agent claude
uv run quorum run scenarios/sdd-go-fractals --coding-agent claude
uv run quorum run scenarios/sdd-svelte-todo --coding-agent claude
uv run quorum run scenarios/spec-reviewer-catches-planted-flaws --coding-agent claude
uv run quorum run scenarios/sdd-quality-reviewer-catches-planted-defect --coding-agent claude
uv run quorum show
```

- [ ] **步骤 3：比较**

通过标准：变更后全部四个既有场景仍通过（捕获率无回归）；新植入缺陷场景通过。对于探索成本，比较前后运行记录中评审 subagent 的工具调用次数（尚无自动化检查 — 规格将其列为已知缺口）。

---

## 收尾

所有任务通过后：evals 子模块的提交需要落入 `superpowers-evals`（向其 `main` 提 PR），然后本分支推进 `evals` 子模块指针 — 按 `evals/CLAUDE.md`，父级推进属于传播的一部分，而非可选。然后使用 superpowers:finishing-a-development-branch。针对 superpowers 的 PR 以 `dev` 为目标。
