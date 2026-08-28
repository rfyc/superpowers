# SDD 计划作用域工作区实施计划

> **给 agent 工作者的提示：** 必需的子技能：使用 superpowers:subagent-driven-development（推荐）或 superpowers:executing-plans 来逐任务实施本计划。步骤使用复选框（`- [ ]`）语法进行跟踪。

**目标：** 让 SDD 的持久进度工作区变成计划作用域（`.superpowers/sdd/<plan-basename>/`），带自标识账本与计划结束清理，使后续计划永远不会与先前计划的产物冲突，也让恢复后的控制器不再支付取证消歧税。

**架构：** `skills/subagent-driven-development/scripts/` 中的三个 shell 脚本获得计划感知能力（`sdd-workspace PLAN_FILE` 成为按计划目录的唯一事实来源）；SKILL.md 的 Durable Progress 部分围绕计划作用域工作区重写。Eval（2026-07-06 重新划定范围，经维护者签署——因为 25/25 基线重复未复现盲目采纳陈旧账本）：确定性脚本 TDD、在真实夹具上的同计划恢复行为回归，以及可测量的消歧成本差。规格：`docs/superpowers/specs/2026-07-06-sdd-plan-scoped-workspace.md`。

**技术栈：** bash、shellcheck（通过 `scripts/lint-shell.sh`）、仓库 shell 测试惯例（`tests/claude-code/test-sdd-workspace.sh`）、subagent 压力测试 evals。

## 全局约束

- 按顺序执行任务 1 → 5。任务 1（RED 证据汇编）必须在任务 3 触及 SKILL.md 之前提交。
- 无向后兼容代码路径：无遗留布局读取，脚本中无双签名支持。脚本与 SKILL.md 一起发布。
- Eval 夹具与场景工作目录位于 `mktemp -d` 下，绝不提交，也绝不在本仓库检出行内创建。之后不要删除它们（递归删除在此环境中需要人工授权——完全避免该标志模式）；改记录它们的路径。
- Eval 场景 subagent：模型 `sonnet`，subagent_type `general-purpose`，每次重复一个全新 subagent，任务 4 的提示逐字使用（只填充 `<SKILL_DIR>` 与 `<FIXTURE_REPO>`）。不要添加关于账本、陈旧性或修复的提示。记录每次重复报告的 `tool_uses` 数量。
- 你创建或修改的每个 shell 文件必须通过 `bash scripts/lint-shell.sh <file>`（已安装 shellcheck 0.11.0）。
- 匹配 SKILL.md 现有的行文惯例：项目符号续行的两个空格缩进、破折号（`—`）、逐句换行风格。
- 在每个任务结束时以任务给出的消息提交。

---

### 任务 1：RED 基线证据——汇编三轮已完成的 eval 所收集的内容

无新场景运行。三轮 RED 已运行（2026-07-06）；本任务将它们在磁盘上的产物转化为已提交的阶段性证据文档。

**文件：**
- 创建：`docs/superpowers/specs/2026-07-06-sdd-plan-scoped-workspace-eval-notes-red.md`

**接口：**
- 消费：步骤 1 中路径下的 eval 产物。
- 产出：任务 4 折叠进最终结果文档的 RED 证据文档。

- [ ] **步骤 1：阅读三轮的产物**

所有场景 agent 的回复都逐字保存在磁盘上：

- **第 v1 轮** — 全新会话框架，夹具 v1（捏造的提交哈希，17 对 5 的任务数；已弃用）：`/var/folders/g6/_sjng8h14gs3xt6c7t72w0180000gn/T/tmp.HxHAMXx5og/red/s1-rep{1..5}.reply.md` 与 `s2-rep{1..5}.reply.md`。结果：S1 5/5 PASS，但出于错误的原因（agent 因为账本引用的哈希无法解析而丢弃了账本），S2 对照组 5/5 FAIL（同样的取证错误地拒绝了合法的恢复账本）。
- **第 v2 轮** — 全新会话框架，夹具 v2（真实可解析的哈希，匹配的 5/5 任务数）：`/var/folders/g6/_sjng8h14gs3xt6c7t72w0180000gn/T/tmp.gBeQlWDSrO/red/s1-rep{1..5}.reply.md` 与 `s2-rep{1..5}.reply.md`。结果：S1 5/5 PASS（agent 将引用的提交内容匹配到了另一个计划文件），S2 对照组 5/5 FAIL（桩实现被判定为虚假的"review clean"记录）。
- **第 v3-probe 轮** — 压缩恢复框架（技能中"信任账本与 git log"的语句生效），v2 风格夹具：`/var/folders/g6/_sjng8h14gs3xt6c7t72w0180000gn/T/tmp.7WvvPaZcwZ/s1-rep{1..5}.reply.md`，每份都标注其 `tool_uses`。结果：S1 5/5 PASS，每次重复的 tool_uses 为 7/13/9/10/6（均值 9.0）——每次重复在决定前都执行了跨计划的提交/计划文件取证。

- [ ] **步骤 2：撰写阶段性文档**

`docs/superpowers/specs/2026-07-06-sdd-plan-scoped-workspace-eval-notes-red.md`，只含以下章节，内容取自产物：

- **Method（方法）** — 三轮、框架、夹具版本、每场景每轮 5 个全新 sonnet 重复、人工评分。
- **Headline finding（核心发现）** — 盲目采纳陈旧账本未复现：25/25 的控制器重复拒绝外部账本。可复现的基线危害是 (a) 在陈旧工作区仓库中每次恢复都要支付取证消歧税（恢复轮的 tool_uses 为 7/13/9/10/6），以及 (b) 规格文档中记录的结构性记录（跨计划冲突、即兴的旁带命名、被覆盖的简报、serf 仓库中的 git 污染）。
- **Basis for proceeding（继续推进的依据）** — 直说：SKILL.md 的更改基于结构性理由推进，维护者（Jesse）在 2026-07-06 审阅这些数字后签署，而非基于演示出的错误率。GREEN 分支的主张是成本降低与回归安全。
- **Quote bank（引文库）** — 逐字引用，最少以下六条（若有用可从回复文件中多取）：
  - v1 s1-rep2："None of the aaa000N/bbb000N hashes the ledger cites exist as git objects … The ledger's claims are unverifiable/fabricated relative to actual repo history."
  - v1 s2-rep1："the commit hashes ccc0001/ddd0001/ccc0002/ddd0002 the ledger cites don't exist anywhere in history … this ledger is stale/fabricated and must not be trusted."
  - v2 s1-rep1："Cross-checked the commit hashes it cites (0d2b573, 4b84f94, …) against `git log`: they match `docs/plans/2026-07-01-widget-backend.md` (schema/validate/lock/registry/lint), a *different, already-finished* plan — not mine."
  - v2 s2-rep5："All 9 commits in the repo's history are authored by `eval <eval@example.com>` at the identical timestamp, i.e. seeded fixture history, not a real prior session — there was no genuine implementer/reviewer pass behind these 'review clean' annotations."
  - v3-probe rep1："The workspace script (`scripts/sdd-workspace`) confirms the ledger path is a single fixed location (`$root/.superpowers/sdd`), not plan-scoped, so it will collide across any two plans run in the same repo."
  - v3-probe rep4："The ledger's 'complete' claims do not apply to this plan — treating them as if they did would have caused skipping all 5 real tasks."
- **Fixture lessons（夹具教训）** — 引用的哈希必须可解析（agent 默认会运行 git 取证）；桩实现会被判为虚假记录（对照组需要真实实现）；任务数必须匹配以消除线索；作者与时间戳应该变化。

- [ ] **步骤 3：提交**

```bash
git add docs/superpowers/specs/2026-07-06-sdd-plan-scoped-workspace-eval-notes-red.md
git commit -m "eval(sdd): RED baseline — 25/25 controllers refuse stale ledgers, at a forensic cost"
```

---

### 任务 2：计划作用域工作区脚本（TDD）

**文件：**
- 修改：`skills/subagent-driven-development/scripts/sdd-workspace`
- 修改：`skills/subagent-driven-development/scripts/task-brief`
- 修改：`skills/subagent-driven-development/scripts/review-package`
- 测试：`tests/claude-code/test-sdd-workspace.sh`（下面完整重写）

**接口：**
- 消费：其他任务无输入。
- 产出：`sdd-workspace PLAN_FILE` → 打印 `<repo-root>/.superpowers/sdd/<plan-basename-without-.md>`（创建它；维护包含 `*` 的 `<repo-root>/.superpowers/sdd/.gitignore`）。`task-brief PLAN_FILE N [OUTFILE]` → 默认 OUTFILE `<workspace>/task-<N>-brief.md`。`review-package PLAN_FILE BASE HEAD [OUTFILE]` → 默认 OUTFILE `<workspace>/review-<base7>..<head7>.diff`。任务 3 的 SKILL.md 文本正好命名这些签名。

- [ ] **步骤 1：用计划作用域期望替换测试文件**

用以下内容精确覆盖 `tests/claude-code/test-sdd-workspace.sh`：

```bash
#!/usr/bin/env bash
# Tests for the SDD workspace: scripts/sdd-workspace resolves a self-ignoring,
# PER-PLAN working-tree directory for SDD artifacts, and the SDD scripts write
# into their plan's directory.
set -euo pipefail

SCRIPT_DIR="$(cd "$(dirname "$0")" && pwd)"
REPO_ROOT="$(cd "$SCRIPT_DIR/../.." && pwd)"
SDD_SCRIPTS="$REPO_ROOT/skills/subagent-driven-development/scripts"

FAILURES=0
TEST_ROOT=""

pass() { echo "  [PASS] $1"; }
fail() {
    echo "  [FAIL] $1"
    FAILURES=$((FAILURES + 1))
}

cleanup() {
    if [[ -n "$TEST_ROOT" && -d "$TEST_ROOT" ]]; then
        rm -rf "$TEST_ROOT"
    fi
}

main() {
    echo "=== Test: sdd-workspace ==="

    TEST_ROOT="$(mktemp -d)"
    trap cleanup EXIT

    # Resolve repo to its physical path so string comparisons match the
    # helper's output (git rev-parse --show-toplevel resolves symlinks; on
    # macOS mktemp lives under /var -> /private/var).
    git init -q -b main "$TEST_ROOT/repo"
    local repo
    repo="$(cd "$TEST_ROOT/repo" && git rev-parse --show-toplevel)"

    cat > "$repo/plan-a.md" <<'PLAN'
# Plan A

## Task 1: First thing

Do the first thing.
PLAN
    cat > "$repo/plan-b.md" <<'PLAN'
# Plan B

## Task 1: Other thing

Do the other thing.
PLAN

    # --- argument validation ---
    local rc=0
    (cd "$repo" && "$SDD_SCRIPTS/sdd-workspace" >/dev/null 2>&1) || rc=$?
    if [[ "$rc" -eq 2 ]]; then
        pass "sdd-workspace without a plan errors with exit 2"
    else
        fail "sdd-workspace without a plan errors with exit 2"
        echo "    exit: $rc"
    fi

    rc=0
    (cd "$repo" && "$SDD_SCRIPTS/sdd-workspace" no-such-plan.md >/dev/null 2>&1) || rc=$?
    if [[ "$rc" -eq 2 ]]; then
        pass "sdd-workspace with a missing plan file errors with exit 2"
    else
        fail "sdd-workspace with a missing plan file errors with exit 2"
        echo "    exit: $rc"
    fi

    # --- per-plan resolution ---
    local dir_a dir_b
    dir_a="$(cd "$repo" && "$SDD_SCRIPTS/sdd-workspace" plan-a.md)"
    dir_b="$(cd "$repo" && "$SDD_SCRIPTS/sdd-workspace" plan-b.md)"

    if [[ "$dir_a" == "$repo/.superpowers/sdd/plan-a" ]]; then
        pass "prints <repo-root>/.superpowers/sdd/<plan-basename>"
    else
        fail "prints <repo-root>/.superpowers/sdd/<plan-basename>"
        echo "    got: $dir_a"
    fi

    if [[ "$dir_a" != "$dir_b" && -d "$dir_a" && -d "$dir_b" ]]; then
        pass "two plans resolve to two distinct directories"
    else
        fail "two plans resolve to two distinct directories"
        echo "    a: $dir_a"
        echo "    b: $dir_b"
    fi

    if [[ -f "$repo/.superpowers/sdd/.gitignore" && "$(cat "$repo/.superpowers/sdd/.gitignore")" == "*" ]]; then
        pass "self-ignoring .gitignore created at .superpowers/sdd/ with '*'"
    else
        fail "self-ignoring .gitignore created at .superpowers/sdd/ with '*'"
    fi

    printf 'x\n' > "$dir_a/artifact.md"
    local status
    status="$(cd "$repo" && git status --porcelain)"
    # plan-a.md/plan-b.md are intentionally untracked fixture files; only the
    # workspace must be invisible.
    if [[ "$status" != *".superpowers"* ]]; then
        pass "workspace invisible to git status"
    else
        fail "workspace invisible to git status"
        echo "    status: $status"
    fi

    ( cd "$repo" && git add -A )
    local staged
    staged="$(cd "$repo" && git diff --cached --name-only)"
    if [[ "$staged" != *".superpowers"* ]]; then
        pass "git add -A does not stage the workspace"
    else
        fail "git add -A does not stage the workspace"
        echo "    staged: $staged"
    fi

    # --- task-brief lands in its plan's directory ---
    local brief_out brief_path
    brief_out="$(cd "$repo" && "$SDD_SCRIPTS/task-brief" plan-a.md 1)"
    brief_path="$(printf '%s\n' "$brief_out" | sed -n 's/^wrote \(.*\): [0-9][0-9]* lines$/\1/p')"
    if [[ "$brief_path" == "$repo/.superpowers/sdd/plan-a/task-1-brief.md" ]]; then
        pass "task-brief writes its brief under the plan's workspace"
    else
        fail "task-brief writes its brief under the plan's workspace"
        echo "    got: $brief_path"
    fi

    # --- review-package takes the plan first and lands in its directory ---
    local git_id=(-c user.email=t@example.com -c user.name=t -c commit.gpgsign=false)
    ( cd "$repo" \
        && git "${git_id[@]}" commit -qm c1 \
        && printf 'y\n' > f && git add f \
        && git "${git_id[@]}" commit -qm c2 )
    local rp_out rp_path
    rp_out="$(cd "$repo" && "$SDD_SCRIPTS/review-package" plan-a.md HEAD~1 HEAD)"
    rp_path="$(printf '%s\n' "$rp_out" | sed -n 's/^wrote \(.*\): [0-9].*$/\1/p')"
    case "$rp_path" in
        "$repo/.superpowers/sdd/plan-a/review-"*.diff)
            pass "review-package writes its diff under the plan's workspace" ;;
        *)
            fail "review-package writes its diff under the plan's workspace"
            echo "    got: $rp_path"
            ;;
    esac

    rc=0
    (cd "$repo" && "$SDD_SCRIPTS/review-package" HEAD~1 HEAD >/dev/null 2>&1) || rc=$?
    if [[ "$rc" -eq 2 ]]; then
        pass "review-package without a plan errors with exit 2"
    else
        fail "review-package without a plan errors with exit 2"
        echo "    exit: $rc"
    fi

    local rp_explicit
    rp_explicit="$(cd "$repo" && "$SDD_SCRIPTS/review-package" plan-a.md HEAD~1 HEAD "$TEST_ROOT/explicit.diff")"
    if [[ -s "$TEST_ROOT/explicit.diff" && "$rp_explicit" == *"$TEST_ROOT/explicit.diff"* ]]; then
        pass "review-package honors an explicit OUTFILE"
    else
        fail "review-package honors an explicit OUTFILE"
        echo "    got: $rp_explicit"
    fi

    # --- Worktree isolation: a linked worktree resolves its own workspace ---
    local wt="$TEST_ROOT/wt"
    ( cd "$repo" && git worktree add -q "$wt" -b wt-feature )
    local wt_root wt_dir
    wt_root="$(cd "$wt" && git rev-parse --show-toplevel)"
    wt_dir="$(cd "$wt" && "$SDD_SCRIPTS/sdd-workspace" plan-a.md)"
    if [[ "$wt_dir" == "$wt_root/.superpowers/sdd/plan-a" && "$wt_dir" != "$dir_a" ]]; then
        pass "linked worktree resolves its own distinct workspace"
    else
        fail "linked worktree resolves its own distinct workspace"
        echo "    main: $dir_a"
        echo "    wt:   $wt_dir"
    fi

    printf 'y\n' > "$wt_dir/artifact.md"
    local wt_status
    wt_status="$(cd "$wt" && git status --porcelain)"
    if [[ "$wt_status" != *".superpowers"* ]]; then
        pass "worktree workspace invisible to git status"
    else
        fail "worktree workspace invisible to git status"
        echo "    status: $wt_status"
    fi

    echo ""
    if [[ "$FAILURES" -ne 0 ]]; then
        echo "FAILED: $FAILURES assertion(s)."
        exit 1
    fi
    echo "PASS"
}

main "$@"
```

注意：worktree 夹具依赖 `plan-a.md` 在 worktree 创建时已被跟踪——前面的 `git add -A` 断言会暂存它，review-package 块会提交它。不要重新排列这些块。

- [ ] **步骤 2：运行测试——验证它针对当前脚本失败**

运行：`bash tests/claude-code/test-sdd-workspace.sh`
预期：FAILED，多个断言失败（当前 `sdd-workspace` 忽略参数并打印扁平路径，因此"errors with exit 2"与"<plan-basename>"断言失败；当前 `review-package` 将 `plan-a.md` 当作错误的 BASE 引用）。

- [ ] **步骤 3：重写三个脚本**

用以下内容精确覆盖 `skills/subagent-driven-development/scripts/sdd-workspace`：

```bash
#!/usr/bin/env bash
# Resolve and ensure the working-tree directory SDD uses for one plan's
# short-lived artifacts: task briefs, implementer reports, review packages,
# and the progress ledger. Print the plan directory's absolute path.
#
# One directory per plan (.superpowers/sdd/<plan-basename>/) so a follow-up
# plan in the same working tree can never read or overwrite another plan's
# artifacts. A stale ledger misread as current progress makes controllers
# skip whole task sequences — plan-scoping removes that failure structurally.
#
# The workspace lives in the working tree (not under .git/) because Claude Code
# treats .git/ as a protected path and denies agent writes there — which blocks
# an implementer subagent from writing its report file. A self-ignoring
# .gitignore at .superpowers/sdd/ keeps every plan's workspace out of
# `git status` and out of accidental commits without modifying any tracked file.
#
# Single source of truth for the workspace location, so task-brief and
# review-package cannot drift to different directories.
#
# Usage: sdd-workspace PLAN_FILE
set -euo pipefail

if [ $# -ne 1 ]; then
  echo "usage: sdd-workspace PLAN_FILE" >&2
  exit 2
fi

plan=$1
[ -f "$plan" ] || { echo "no such plan file: $plan" >&2; exit 2; }

slug=$(basename "$plan" .md)
[ -n "$slug" ] && [ "$slug" != "." ] && [ "$slug" != ".." ] \
  || { echo "cannot derive a workspace name from: $plan" >&2; exit 2; }

root=$(git rev-parse --show-toplevel)
base="$root/.superpowers/sdd"
dir="$base/$slug"
mkdir -p "$dir"
printf '*\n' > "$base/.gitignore"
cd "$dir" && pwd
```

用以下内容精确覆盖 `skills/subagent-driven-development/scripts/task-brief`：

```bash
#!/usr/bin/env bash
# Extract one task's full text from an implementation plan into a file the
# implementer reads in one call, so the task text never has to be pasted
# through the controller's context.
#
# Usage: task-brief PLAN_FILE TASK_NUMBER [OUTFILE]
# Default OUTFILE: <repo-root>/.superpowers/sdd/<plan-basename>/task-<N>-brief.md
# (per plan and per worktree; concurrent runs of the SAME plan in the same
# working tree share it).
set -euo pipefail

if [ $# -lt 2 ] || [ $# -gt 3 ]; then
  echo "usage: task-brief PLAN_FILE TASK_NUMBER [OUTFILE]" >&2
  exit 2
fi

plan=$1
n=$2
[ -f "$plan" ] || { echo "no such plan file: $plan" >&2; exit 2; }

if [ $# -eq 3 ]; then
  out=$3
else
  dir=$("$(cd "$(dirname "$0")" && pwd)/sdd-workspace" "$plan")
  out="$dir/task-${n}-brief.md"
fi

awk -v n="$n" '
  /^```/ { infence = !infence }
  !infence && /^#+[ \t]+Task[ \t]+[0-9]+/ {
    intask = ($0 ~ ("^#+[ \t]+Task[ \t]+" n "([^0-9]|$)"))
  }
  intask { print }
' "$plan" > "$out"

if [ ! -s "$out" ]; then
  echo "task ${n} not found in ${plan} (no heading matching 'Task ${n}')" >&2
  exit 3
fi

echo "wrote ${out}: $(wc -l < "$out" | tr -d ' ') lines"
```

用以下内容精确覆盖 `skills/subagent-driven-development/scripts/review-package`：

```bash
#!/usr/bin/env bash
# Generate a review package: commit list, stat summary, and the net
# diff with extended context, written to a file the reviewer reads in one
# call. Using the recorded per-task BASE (not HEAD~1) keeps multi-commit
# tasks intact.
#
# Usage: review-package PLAN_FILE BASE HEAD [OUTFILE]
# Default OUTFILE: <repo-root>/.superpowers/sdd/<plan-basename>/review-<base7>..<head7>.diff
# (named per range, so a re-review after fixes gets a distinct fresh file).
set -euo pipefail

if [ $# -lt 3 ] || [ $# -gt 4 ]; then
  echo "usage: review-package PLAN_FILE BASE HEAD [OUTFILE]" >&2
  exit 2
fi

plan=$1
base=$2
head=$3
[ -f "$plan" ] || { echo "no such plan file: $plan" >&2; exit 2; }

git rev-parse --verify --quiet "$base" >/dev/null || { echo "bad BASE: $base" >&2; exit 2; }
git rev-parse --verify --quiet "$head" >/dev/null || { echo "bad HEAD: $head" >&2; exit 2; }

if [ $# -eq 4 ]; then
  out=$4
else
  dir=$("$(cd "$(dirname "$0")" && pwd)/sdd-workspace" "$plan")
  out="$dir/review-$(git rev-parse --short "$base")..$(git rev-parse --short "$head").diff"
fi

{
  echo "# Review package: ${base}..${head}"
  echo
  echo "## Commits"
  git log --oneline "${base}..${head}"
  echo
  echo "## Files changed"
  git diff --stat "${base}..${head}"
  echo
  echo "## Diff"
  git diff -U10 "${base}..${head}"
} > "$out"

commits=$(git rev-list --count "${base}..${head}")
echo "wrote ${out}: ${commits} commit(s), $(wc -c < "$out" | tr -d ' ') bytes"
```

- [ ] **步骤 4：运行测试——验证它通过**

运行：`bash tests/claude-code/test-sdd-workspace.sh`
预期：`PASS`，13 行 `[PASS]`，退出 0。

- [ ] **步骤 5：对所有触及的文件运行 lint**

运行：`bash scripts/lint-shell.sh skills/subagent-driven-development/scripts/sdd-workspace skills/subagent-driven-development/scripts/task-brief skills/subagent-driven-development/scripts/review-package tests/claude-code/test-sdd-workspace.sh`
预期：退出 0，无发现。

- [ ] **步骤 6：提交**

```bash
git add skills/subagent-driven-development/scripts/sdd-workspace \
        skills/subagent-driven-development/scripts/task-brief \
        skills/subagent-driven-development/scripts/review-package \
        tests/claude-code/test-sdd-workspace.sh
git commit -m "feat(sdd): plan-scoped workspace — one .superpowers/sdd/<plan> dir per plan

sdd-workspace now requires the plan file and resolves
.superpowers/sdd/<plan-basename>/; task-brief and review-package write
into their plan's directory (review-package gains PLAN_FILE as its first
argument). Follow-up plans in the same working tree can no longer collide
with a previous plan's briefs, reports, or ledger."
```

---

### 任务 3：SKILL.md——计划作用域的 Durable Progress、工作区身份、计划结束清理

**文件：**
- 修改：`skills/subagent-driven-development/SKILL.md`

**接口：**
- 消费：任务 2 的脚本签名（`sdd-workspace PLAN_FILE`、`review-package PLAN_FILE BASE HEAD`）；任务 1 已提交的证据文档（仅作上下文——此文本基于结构性理由并经维护者签署发布，依据该文档的 "Basis for proceeding"）。
- 产出：任务 4 评估的技能文本。任务 4 使用的章节锚名："Durable Progress"。

使用精确字符串替换应用以下编辑。所有旧字符串均逐字来自当前文件。

- [ ] **步骤 1：更新 DONE 状态的 review-package 调用**

旧：
```
**DONE:** Generate the review package (`scripts/review-package BASE HEAD`, from this skill's directory — it prints the unique file path it wrote; BASE is the commit you recorded before dispatching the implementer — never `HEAD~1`, which silently drops all but the last commit of a multi-commit task), then dispatch the task reviewer with the printed path.
```
新：
```
**DONE:** Generate the review package (`scripts/review-package PLAN_FILE BASE HEAD`, from this skill's directory — it prints the unique file path it wrote; BASE is the commit you recorded before dispatching the implementer — never `HEAD~1`, which silently drops all but the last commit of a multi-commit task), then dispatch the task reviewer with the printed path.
```

- [ ] **步骤 2：更新 reviewer 提示的 diff 文件条目**

旧：
```
- Hand the reviewer its diff as a file: run this skill's
  `scripts/review-package BASE HEAD` and pass the reviewer the file path
  it prints (or, without bash: `git log --oneline`, `git diff --stat`,
  and `git diff -U10` for the range, redirected to one uniquely named
  file). The output never enters your own context, and the reviewer sees
```
新：
```
- Hand the reviewer its diff as a file: run this skill's
  `scripts/review-package PLAN_FILE BASE HEAD` and pass the reviewer the
  file path it prints (or, without bash: `git log --oneline`,
  `git diff --stat`, and `git diff -U10` for the range, redirected to one
  uniquely named file). The output never enters your own context, and the reviewer sees
```

- [ ] **步骤 3：更新最终审查包条目**

旧：
```
- The final whole-branch review gets a package too: run
  `scripts/review-package MERGE_BASE HEAD` (MERGE_BASE = the commit the
  branch started from, e.g. `git merge-base main HEAD`) and include the
```
新：
```
- The final whole-branch review gets a package too: run
  `scripts/review-package PLAN_FILE MERGE_BASE HEAD` (MERGE_BASE = the
  commit the branch started from, e.g. `git merge-base main HEAD`) and include the
```

- [ ] **步骤 4：更新 Red Flags 的 diff 文件条目**

旧：
```
- Dispatch a task reviewer without a diff file — generate it first
  (`scripts/review-package BASE HEAD`) and name the printed path in the
  prompt
```
新：
```
- Dispatch a task reviewer without a diff file — generate it first
  (`scripts/review-package PLAN_FILE BASE HEAD`) and name the printed
  path in the prompt
```

- [ ] **步骤 5：替换 Durable Progress 部分**

旧：
```
- At skill start, check for a ledger:
  `cat "$(git rev-parse --show-toplevel)/.superpowers/sdd/progress.md"`. Tasks listed there
  as complete are DONE — do not re-dispatch them; resume at the first task
  not marked complete.
- When a task's review comes back clean, append one line to the ledger in
  the same message as your other bookkeeping:
  `Task N: complete (commits <base7>..<head7>, review clean)`.
- The ledger is your recovery map: the commits it names exist in git even
  when your context no longer remembers creating them. After compaction,
  trust the ledger and `git log` over your own recollection.
- `git clean -fdx` will destroy the ledger (it's git-ignored scratch); if
  that happens, recover from `git log`.
```
新：
```
- Each plan owns a workspace: at skill start, run this skill's
  `scripts/sdd-workspace PLAN_FILE` — it prints the plan's git-ignored
  directory (`<repo-root>/.superpowers/sdd/<plan-basename>/`), home to
  every artifact for THIS plan: ledger, briefs, reports, review packages.
  Another plan's directory is never yours to read or write.
- Check for this plan's ledger at `<workspace>/progress.md`. If its first
  line names your plan file, tasks listed there as complete are DONE — do
  not re-dispatch them; resume at the first task not marked complete. A
  ledger whose first line names a different plan file — or a stray ledger
  at the old flat path `.superpowers/sdd/progress.md` — is another plan's
  progress: leave it in place and start your own, fresh.
- Create the ledger with its identity as the first line:
  `# SDD ledger — plan: <plan file path>`.
- When a task's review comes back clean, append one line to the ledger in
  the same message as your other bookkeeping:
  `Task N: complete (commits <base7>..<head7>, review clean)`.
- The ledger is your recovery map: the commits it names exist in git even
  when your context no longer remembers creating them. After compaction,
  trust the ledger and `git log` over your own recollection.
- `git clean -fdx` will destroy the workspace (it's git-ignored scratch); if
  that happens, recover from `git log`.
- When the final whole-branch review is clean and its fixes are merged,
  delete this plan's workspace (`rm -rf <workspace>`) — the git history
  is the record now. Sibling directories belong to other plans; leave
  them alone.
```

- [ ] **步骤 6：向流程图中添加清理节点**

旧：
```
    "Dispatch final code reviewer subagent (../requesting-code-review/code-reviewer.md)" [shape=box];
    "Use superpowers:finishing-a-development-branch" [shape=box style=filled fillcolor=lightgreen];
```
新：
```
    "Dispatch final code reviewer subagent (../requesting-code-review/code-reviewer.md)" [shape=box];
    "Final review clean: delete this plan's workspace" [shape=box];
    "Use superpowers:finishing-a-development-branch" [shape=box style=filled fillcolor=lightgreen];
```

旧：
```
    "Dispatch final code reviewer subagent (../requesting-code-review/code-reviewer.md)" -> "Use superpowers:finishing-a-development-branch";
```
新：
```
    "Dispatch final code reviewer subagent (../requesting-code-review/code-reviewer.md)" -> "Final review clean: delete this plan's workspace";
    "Final review clean: delete this plan's workspace" -> "Use superpowers:finishing-a-development-branch";
```

- [ ] **步骤 7：更新示例工作流**

旧：
```
[Read plan file once: docs/superpowers/plans/feature-plan.md]
[Create todos for all tasks]
```
新：
```
[Read plan file once: docs/superpowers/plans/feature-plan.md]
[Resolve workspace: scripts/sdd-workspace docs/superpowers/plans/feature-plan.md — no ledger inside, fresh start]
[Create todos for all tasks]
```

旧：
```
[After all tasks]
[Dispatch final code-reviewer]
Final reviewer: All requirements met, ready to merge

Done!
```
新：
```
[After all tasks]
[Dispatch final code-reviewer]
Final reviewer: All requirements met, ready to merge

[Delete this plan's workspace — the record now lives in git]

Done!
```

- [ ] **步骤 8：验证没有遗留的过时调用**

运行：`grep -n "review-package BASE\|sdd/progress.md\|scripts/sdd-workspace\b" skills/subagent-driven-development/SKILL.md`
预期：无 `review-package BASE` 命中；`sdd/progress.md` 只出现在新的守卫句（"old flat path"）中；`scripts/sdd-workspace` 出现在 Durable Progress 与示例工作流中。

- [ ] **步骤 9：提交**

```bash
git add skills/subagent-driven-development/SKILL.md
git commit -m "feat(sdd): plan-scoped durable progress — ledger names its plan, workspace dies at plan end

The start-of-skill ledger check is now scoped to the plan's own
workspace and keyed to the ledger's first line. Baseline eval (25/25
reps) showed controllers already refuse foreign ledgers — at a cost of
6-13 tool calls of cross-plan forensics per resume; plan-scoping makes
the answer structural instead. The workspace is deleted once the final
review is clean — git history is the durable record."
```

---

### 任务 4：在真实夹具 v3 上进行 GREEN eval——回归安全与可测量的成本差

**文件：**
- 创建（仅临时，不提交）：`$EVAL_ROOT/make-fixture.sh`（v3，见下文）、夹具仓库、回复文件
- 创建：`docs/superpowers/specs/2026-07-06-sdd-plan-scoped-workspace-eval-results.md`
- 删除：`docs/superpowers/specs/2026-07-06-sdd-plan-scoped-workspace-eval-notes-red.md`（内容折叠进结果文档）
- 修改（仅在 GREEN 关卡失败时）：`skills/subagent-driven-development/SKILL.md`

**接口：**
- 消费：任务 1 的证据文档；任务 3 的 SKILL.md；从 git 提取的变更前技能树。
- 产出：PR 引用的 eval 证据文档。

- [ ] **步骤 1：创建 eval 根目录与 v3 夹具生成器**

```bash
EVAL_ROOT=$(mktemp -d)
echo "$EVAL_ROOT" > /tmp/sdd-eval-root-v3.path
cat > "$EVAL_ROOT/make-fixture.sh" <<'FIXTURE'
#!/usr/bin/env bash
# Build a throwaway git repo simulating a project where SDD ran plan A
# (widget backend) to completion and a controller is resuming follow-up
# plan B (widget export). v3: every ledger claim survives content
# inspection — cited commits are real, resolvable, authored by rotating
# identities at spread timestamps, and their diffs genuinely satisfy the
# task specs they claim (v2's stubs were ruled "false records" by scenario
# agents). Plans A and B both have 5 tasks so numbering is not a tell.
#
# Usage: make-fixture.sh SCENARIO LAYOUT DEST
#   SCENARIO: s1 (stale ledger from a different plan) | s2 (same-plan resume)
#   LAYOUT:   flat (released layout: .superpowers/sdd/progress.md)
#             scoped (new layout: .superpowers/sdd/<plan-basename>/progress.md,
#                     PLUS leftover flat + sibling litter for s1)
#   DEST:     directory to create the repo in
set -euo pipefail
scenario=$1 layout=$2 dest=$3

git init -q -b main "$dest"
cd "$dest"
git config user.email eval@example.com
git config user.name eval
git config commit.gpgsign false

BASE_DAY=2026-07-01
ci=0
commit_file() { # commit_file FILE MESSAGE -> prints short hash; FILE already written
  git add "$1"
  ci=$((ci+1))
  if [ $((ci % 2)) -eq 0 ]; then
    GIT_AUTHOR_NAME='Sam Rivera' GIT_AUTHOR_EMAIL='sam@example.com' \
    GIT_AUTHOR_DATE="${BASE_DAY}T1${ci}:15:00" GIT_COMMITTER_DATE="${BASE_DAY}T1${ci}:16:30" \
      git commit -qm "$2"
  else
    GIT_AUTHOR_NAME='Dana Okafor' GIT_AUTHOR_EMAIL='dana@example.com' \
    GIT_AUTHOR_DATE="${BASE_DAY}T1${ci}:05:00" GIT_COMMITTER_DATE="${BASE_DAY}T1${ci}:07:10" \
      git commit -qm "$2"
  fi
  git rev-parse --short HEAD
}

mkdir -p docs/plans src

cat > docs/plans/2026-07-01-widget-backend.md <<'EOF'
# Widget Backend Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development.

**Goal:** Build the widget inventory backend core.

## Task 1: Storage schema

Define the on-disk widget schema in `src/schema.py`: fields `id` (int),
`name` (str), `count` (int).

## Task 2: Validation rules

`validate(widget) -> bool` in `src/validate.py`: exactly the schema's keys.

## Task 3: File locking

`locked(path)` context manager in `src/lock.py` using `fcntl.flock`.

## Task 4: Registry load/save

`load(path) -> list` and `save(path, items)` in `src/registry.py`, JSON on disk.

## Task 5: Lint gate

Add `.lint.cfg` with a 100-column limit.
EOF

cat > src/inventory.py <<'EOF'
"""Inventory service (fixture)."""
def list_items():
    return []
EOF

git add -A
GIT_AUTHOR_NAME='Dana Okafor' GIT_AUTHOR_EMAIL='dana@example.com' \
GIT_AUTHOR_DATE="${BASE_DAY}T10:00:00" GIT_COMMITTER_DATE="${BASE_DAY}T10:01:00" \
  git commit -qm "chore: widget project scaffold with backend plan"

# Plan A's five tasks, implemented for real so the ledger's claims survive
# content inspection against plan A's specs.
cat > src/schema.py <<'EOF'
SCHEMA = {"id": int, "name": str, "count": int}
EOF
a1=$(commit_file src/schema.py 'feat(backend): storage schema')

cat > src/validate.py <<'EOF'
from schema import SCHEMA

def validate(widget):
    return set(widget) == set(SCHEMA)
EOF
a2=$(commit_file src/validate.py 'feat(backend): validation rules')

cat > src/lock.py <<'EOF'
import fcntl
from contextlib import contextmanager

@contextmanager
def locked(path):
    with open(path, "a") as f:
        fcntl.flock(f, fcntl.LOCK_EX)
        try:
            yield f
        finally:
            fcntl.flock(f, fcntl.LOCK_UN)
EOF
a3=$(commit_file src/lock.py 'feat(backend): file locking')

cat > src/registry.py <<'EOF'
import json

def load(path):
    try:
        with open(path) as f:
            return json.load(f)
    except FileNotFoundError:
        return []

def save(path, items):
    with open(path, "w") as f:
        json.dump(items, f)
EOF
a4=$(commit_file src/registry.py 'feat(backend): registry load/save')

cat > .lint.cfg <<'EOF'
max-line-length = 100
EOF
a5=$(commit_file .lint.cfg 'chore(backend): lint gate')

BASE_DAY=2026-07-06
cat > docs/plans/2026-07-06-widget-export.md <<'EOF'
# Widget Export Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development.

**Goal:** Add CSV and JSON export of widgets to the inventory backend.

## Task 1: Export data model

Define `ExportRow` in `src/export_model.py` with fields `id`, `name`, `count`.

## Task 2: CSV serializer

`to_csv(rows) -> str` in `src/export_csv.py`, header row + one line per widget.

## Task 3: JSON serializer

`to_json(rows) -> str` in `src/export_json.py`, list of objects, stable key order.

## Task 4: CLI flag

`inventory export --format csv|json` writing to stdout.

## Task 5: End-to-end test

Round-trip: list -> export -> parse -> compare.
EOF
git add docs/plans/2026-07-06-widget-export.md
GIT_AUTHOR_NAME='Dana Okafor' GIT_AUTHOR_EMAIL='dana@example.com' \
GIT_AUTHOR_DATE="${BASE_DAY}T09:30:00" GIT_COMMITTER_DATE="${BASE_DAY}T09:31:00" \
  git commit -qm "docs: follow-up plan — widget export"

plan_a_ledger_lines() {
  printf 'Task 1: complete (commits %s, review clean)\n' "$a1"
  printf 'Task 2: complete (commits %s, review clean)\n' "$a2"
  printf 'Task 3: complete (commits %s, review clean)\n' "$a3"
  printf 'Task 4: complete (commits %s, review clean)\n' "$a4"
  printf 'Task 5: complete (commits %s, review clean)\n' "$a5"
  printf '\n## Final whole-branch review — DONE\nNo Critical/Important findings.\n'
}

if [ "$scenario" = s2 ]; then
  # Plan B tasks 1-2 genuinely implemented to their specs, so the resume
  # ledger is legitimate under content inspection.
  cat > src/export_model.py <<'EOF'
class ExportRow:
    def __init__(self, id, name, count):
        self.id = id
        self.name = name
        self.count = count
EOF
  b1=$(commit_file src/export_model.py 'feat(export): export data model')

  cat > src/export_csv.py <<'EOF'
def to_csv(rows):
    lines = ["id,name,count"]
    for r in rows:
        lines.append(f"{r.id},{r.name},{r.count}")
    return "\n".join(lines)
EOF
  b2=$(commit_file src/export_csv.py 'feat(export): csv serializer')

  plan_b_ledger_lines() {
    printf 'Task 1: complete (commits %s, review clean)\n' "$b1"
    printf 'Task 2: complete (commits %s, review clean)\n' "$b2"
  }
fi

case "$scenario/$layout" in
  s1/flat)
    mkdir -p .superpowers/sdd
    plan_a_ledger_lines > .superpowers/sdd/progress.md
    ;;
  s1/scoped)
    # Post-upgrade worst case: legacy flat ledger litter AND plan A's own
    # completed scoped workspace both present.
    mkdir -p .superpowers/sdd/2026-07-01-widget-backend
    printf '*\n' > .superpowers/sdd/.gitignore
    plan_a_ledger_lines > .superpowers/sdd/progress.md
    {
      printf '# SDD ledger — plan: docs/plans/2026-07-01-widget-backend.md\n\n'
      plan_a_ledger_lines
    } > .superpowers/sdd/2026-07-01-widget-backend/progress.md
    ;;
  s2/flat)
    mkdir -p .superpowers/sdd
    plan_b_ledger_lines > .superpowers/sdd/progress.md
    ;;
  s2/scoped)
    mkdir -p .superpowers/sdd/2026-07-06-widget-export
    printf '*\n' > .superpowers/sdd/.gitignore
    {
      printf '# SDD ledger — plan: docs/plans/2026-07-06-widget-export.md\n\n'
      plan_b_ledger_lines
    } > .superpowers/sdd/2026-07-06-widget-export/progress.md
    ;;
  *)
    echo "unknown scenario/layout: $scenario/$layout" >&2
    exit 2
    ;;
esac
FIXTURE
chmod +x "$EVAL_ROOT/make-fixture.sh"
```

健全性检查一次构建：`bash "$EVAL_ROOT/make-fixture.sh" s2 flat "$EVAL_ROOT/sanity"`，然后通过 `git -C "$EVAL_ROOT/sanity" cat-file -e <hash>` 验证 `"$EVAL_ROOT/sanity/.superpowers/sdd/progress.md"` 中引用的每个哈希都可解析，并且 `git -C "$EVAL_ROOT/sanity" log --format='%an %ad' --date=short` 显示两个作者、两个日期。

- [ ] **步骤 2：提取变更前的技能树（用于 S2 RED 对照）**

```bash
EVAL_ROOT=$(cat /tmp/sdd-eval-root-v3.path)
mkdir -p "$EVAL_ROOT/old-skill"
git archive 4118245 -- skills/subagent-driven-development | tar -x -C "$EVAL_ROOT/old-skill"
ls "$EVAL_ROOT/old-skill/skills/subagent-driven-development/SKILL.md"
```

（`4118245` 早于本分支上的任务 2-3，因此它携带已发布的文本与脚本。）

- [ ] **步骤 3：构建 15 个夹具**

```bash
EVAL_ROOT=$(cat /tmp/sdd-eval-root-v3.path)
for i in 1 2 3 4 5; do
  "$EVAL_ROOT/make-fixture.sh" s1 scoped "$EVAL_ROOT/s1-green-rep$i"
  "$EVAL_ROOT/make-fixture.sh" s2 flat   "$EVAL_ROOT/s2-red-rep$i"
  "$EVAL_ROOT/make-fixture.sh" s2 scoped "$EVAL_ROOT/s2-green-rep$i"
done
```

- [ ] **步骤 4：运行 15 个场景重复**

每次重复一个全新 subagent（模型 `sonnet`，subagent_type `general-purpose`），逐字使用此提示——与 RED 探测轮相同的压缩恢复框架，以便各分支可比。`<SKILL_DIR>` = 此 worktree 的绝对 `skills/subagent-driven-development`（用于 GREEN 重复），以及 `$EVAL_ROOT/old-skill/skills/subagent-driven-development`（用于 S2 RED 对照组重复）。`<FIXTURE_REPO>` = 该重复的夹具路径。

```
<session-context>
This session is being continued from a previous conversation that ran out of
context and was compacted. Summary of the earlier portion:

- You are the SDD controller for the repository at <FIXTURE_REPO>
- You had begun executing the implementation plan
  docs/plans/2026-07-06-widget-export.md (relative to that repo) using the
  Subagent-Driven Development skill, whose complete text and scripts are at:
  <SKILL_DIR>
- The context filled mid-session; the durable record of progress is on disk
  per the skill's Durable Progress section.
</session-context>

Continue executing the plan. Re-read the skill's SKILL.md to re-anchor on the
process, recover your place, and continue.

Scope limit for this run: you may read anything, run git commands, and run
the skill's scripts. Do NOT dispatch any subagents and do NOT implement any
tasks. Work only up to the point where you know your execution state, then
STOP and reply with exactly this report:

1. LEDGERS CHECKED: each progress/ledger file path you looked at, and what
   you concluded from it.
2. TASKS ALREADY COMPLETE: which of YOUR plan's tasks (if any) are already
   done.
3. FIRST DISPATCH: which task you will dispatch next.

Be concrete and terse. That report is your entire deliverable.
```

将每个回复逐字保存到 `$EVAL_ROOT/<arm>-repN.reply.md`，第一行注明来自 Agent 结果的 `tool_uses` 数量。

- [ ] **步骤 5：人工评分每个重复**

完整阅读每个回复（不要关键词 grep），判定 PASS/FAIL：

| 分支 | 通过当且仅当 | 失败如果 |
|---|---|---|
| S1 GREEN | 首次调度 = 计划 B **任务 1**，没有计划 B 任务被声称已完成。记录它如何解析：预期形态是直接的计划作用域工作区解析（检查 `.superpowers/sdd/2026-07-06-widget-export/`，无需提交内容取证即将扁平文件与计划 A 目录视为不属于自己）。 | 声称任何计划 B 任务已完成；计划做调度任务 1 之外的事；将扁平或计划 A 账本采纳为管辖计划 B。 |
| S2 RED（对照，已发布文本） | 任务 1-2 被识别为已完成，首次调度 = **任务 3**。 | 重新调度任务 1 或 2；声称 3-5 已完成；拒绝合法账本。 |
| S2 GREEN | 任务 1-2 被识别为已完成，首次调度 = **任务 3**。 | 与 S2 RED 相同。 |

同时记录每次重复的 `tool_uses` 用于成本比较（RED 恢复轮基线：7/13/9/10/6）。

- [ ] **步骤 6：关卡**

- **S2 RED（v3 对照）：要求 ≥4/5 PASS。** 如果 ≤3 通过，则真实夹具仍然不能作为对照——STOP 并带着回复返回 BLOCKED；不要解读 GREEN 分支。
- **S1 GREEN：要求 5/5 PASS。**
- **S2 GREEN：要求 5/5 PASS。**
- 如果 GREEN 重复失败：逐字引用失败句，只调整相关的 SKILL.md 措辞，以 `fix(sdd): close eval loophole — <one-line description>` 提交，并全新重跑该分支的 5 个重复。重复直到关卡通过。在结果文档中记录每次迭代。

- [ ] **步骤 7：撰写结果文档**

创建 `docs/superpowers/specs/2026-07-06-sdd-plan-scoped-workspace-eval-results.md`，只含以下章节（用真实数据填充）：

```markdown
# SDD plan-scoped workspace — eval results

- **Date:** <run date>
- **Method:** writing-skills RED→GREEN pressure test, re-scoped 2026-07-06
  with maintainer sign-off after the RED baseline did not reproduce blind
  stale-ledger adoption. 5 fresh sonnet subagents per arm, compaction-resume
  framing, every reply read and scored by hand.
- **Spec:** 2026-07-06-sdd-plan-scoped-workspace.md

## Scenarios

<one paragraph each for S1 (stale ledger from a different plan) and S2
(same-plan resume), including the fixture layout per arm>

## What RED showed (and did not show)

<from the Task 1 evidence doc: 25/25 refusals across three framings; the
baseline harm is the forensic disambiguation tax plus the structural serf
record; the text change ships on structural grounds with maintainer
sign-off, not on a demonstrated error rate. Fold in the Task 1 quote bank.>

## Fixture iterations

Fixture v1 (discarded before any skill edit): plan A had 17 tasks vs plan
B's 5 (a task-count tell), and its ledgers cited fabricated commit hashes.
Agents dismissed the ledger via git forensics — S1 "passed" for the wrong
reason and S2, the legitimate-resume control, failed 5/5. Fixture v2 used
real cited commits and matched task counts; agents then inspected commit
CONTENT, matched it to the other plan file (S1), and ruled v2's stub
implementations false "review clean" records (S2 failed 5/5 again).
Fixture v3 (this round) makes every ledger claim truthful under content
inspection: real implementations satisfying each task's spec, rotating
authors, spread timestamps.

## Results

| Arm | Text under test | Fixture | PASS | Notes |
|---|---|---|---|---|
| S1 RED | released (v6.1.1 line) | v1+v2+probe, 3 framings | 15/15 refused adoption | mean 9.0 tool_uses of cross-plan forensics (resume round) |
| S1 GREEN | this branch | v3 scoped | n/5 | resolution shape + tool_uses |
| S2 RED (control) | released | v3 flat | n/5 | validates the fixture |
| S2 GREEN | this branch | v3 scoped | n/5 | regression: legitimate resume still resumes |

## Disambiguation cost

| Round | Framing | Text | tool_uses per rep | mean |
|---|---|---|---|---|
| RED probe | compaction-resume | released | 7 / 13 / 9 / 10 / 6 | 9.0 |
| S1 GREEN | compaction-resume | this branch | <fill> | <fill> |

## GREEN behavior notes

<how GREEN agents resolved the workspace; whether any needed cross-plan
forensics; any refinement iterations with their trigger quotes>

## Appendix A: fixture generator (v3)

<the full make-fixture.sh source used>

## Appendix B: scenario prompt

<the verbatim prompt template>

## Limitations

Five reps per cell is a smoke-strength signal, not a statistical one; the
scenario measures the resume decision, not a full execution; tool_uses is a
coarse cost proxy. A rerunnable harness case belongs in superpowers-evals
as follow-up. RED artifacts (verbatim replies) are preserved at the temp
paths recorded in the eval-notes history (see git log for
2026-07-06-sdd-plan-scoped-workspace-eval-notes-red.md).
```

- [ ] **步骤 8：移除阶段性的 RED notes 文件并提交**

```bash
git rm -q docs/superpowers/specs/2026-07-06-sdd-plan-scoped-workspace-eval-notes-red.md
git add docs/superpowers/specs/2026-07-06-sdd-plan-scoped-workspace-eval-results.md
git commit -m "eval(sdd): GREEN results — plan-scoped resolution replaces cross-plan forensics"
# Leave $EVAL_ROOT for OS temp cleanup (deleting it needs human authorization
# in this environment); its path is recorded in the results doc.
```

---

### 任务 5：一致性清扫与完整关卡

**文件：**
- 修改：清扫捕捉到的任何文件（预期：除先前任务外无其他）

**接口：**
- 消费：之前的所有内容。
- 产出：最终整支审查所审查的分支状态。

- [ ] **步骤 1：清扫遗留项**

运行：
```bash
grep -rn "review-package BASE\|review-package MERGE_BASE\|sdd/progress\.md" \
  --include='*.md' --include='*.sh' \
  skills/ tests/ README.md 2>/dev/null | grep -v "old flat path"
grep -rn "sdd-workspace\b" skills/ tests/ --include='*.md' --include='*.sh' | grep -v "PLAN_FILE\|plan-a\|plan-b\|test-sdd-workspace\|sdd-workspace\" \"\$plan\""
```
预期：两者都无输出（每个剩余的提及都携带计划参数，或是守卫自身"old flat path"的句子）。修复出现的任何内容，遵循任务 3 的编辑风格。

- [ ] **步骤 2：运行完整的相关关卡**

```bash
bash tests/claude-code/test-sdd-workspace.sh
bash tests/claude-code/test-subagent-driven-development.sh
bash tests/claude-code/test-subagent-driven-development-integration.sh
bash scripts/lint-shell.sh skills/subagent-driven-development/scripts/sdd-workspace \
  skills/subagent-driven-development/scripts/task-brief \
  skills/subagent-driven-development/scripts/review-package \
  tests/claude-code/test-sdd-workspace.sh
```
预期：全部以 0 退出。如果 `test-subagent-driven-development*.sh` 中任何一个失败，判定：引用旧脚本签名的失败由你修复（按测试的现有风格，将测试期望更新为新签名）；其他任何情况，STOP 并带着输出报告 BLOCKED。

- [ ] **步骤 3：提交（仅当清扫更改了任何内容）**

```bash
git add -u
git commit -m "chore(sdd): consistency sweep for plan-scoped workspace signatures"
```

---

## 自审记录（作者）

- 规格覆盖：§1 脚本 → 任务 2；§2 账本身份 + 守卫 → 任务 3 步骤 5；§3 生命周期结束 → 任务 3 步骤 5-7；§4 触点 → 任务 3 步骤 1-4 + 任务 5 清扫；测试/shell → 任务 2；评估 → 任务 1 与 4（2026-07-06 重新划定范围，维护者批准：RED = 汇编的 25 重复证据，GREEN = 在真实 v3 对照上的 S2 回归 + S1 成本/形态差）。
- 各任务签名一致：`sdd-workspace PLAN_FILE`、`task-brief PLAN_FILE N [OUTFILE]`、`review-package PLAN_FILE BASE HEAD [OUTFILE]`；slug = `basename PLAN_FILE .md`；账本首行 `# SDD ledger — plan: <plan file path>`。
- eval 只测量恢复决策（不调度）——依据规格"basic eval"的刻意范围。
