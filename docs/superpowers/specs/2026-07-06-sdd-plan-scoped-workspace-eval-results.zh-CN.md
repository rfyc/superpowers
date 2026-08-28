# SDD 计划作用域工作区——eval 结果

- **日期：** 2026-07-06
- **方法：** writing-skills RED→GREEN 压力测试，2026-07-06 重新定域，在 RED 基线未能复现盲目采用陈旧 ledger 之后获得维护者签收。每个分支 5 个全新的 sonnet subagent，压缩恢复框架，每条回复都由人工阅读并评分。
- **规范：** 2026-07-06-sdd-plan-scoped-workspace.md

## 场景

**S1——来自不同计划的陈旧 ledger。** fixture 仓库模拟一个项目，其中 SDD 将计划 A（`docs/plans/2026-07-01-widget-backend.md`，5 个任务）执行到完成，而受测控制器在上下文压缩后正在恢复后续计划 B（`docs/plans/2026-07-06-widget-export.md`，同样 5 个任务）。计划 B 没有任何部分被实现。GREEN 分支使用 `scoped` 布局——升级后的最坏情况：位于 `.superpowers/sdd/progress.md` 的旧扁平布局 ledger 携带计划 A 的五条"complete (review clean)"记录且没有身份头，再加上计划 A 自己已完成的计划定域工作区 `.superpowers/sdd/2026-07-01-widget-backend/progress.md`（第一行身份标明计划 A），而计划 B 没有工作区。一个正确的控制器应从任务 1 开始执行计划 B，且不采用任何一个陈旧产物。（RED S1 分支在下面的早期轮次中运行，针对 fixture v1/v2 的扁平布局。）

**S2——同计划恢复。** 同一项目，但计划 B 的任务 1–2 被真正实现并提交（`feat(export): export data model`、`feat(export): csv serializer`——满足各任务规范的真实代码），并在 ledger 中记录为完成。一个正确的控制器应识别任务 1–2 已完成并派发任务 3。RED 对照分支（发布版文本）使用 `flat` 布局——ledger 位于 `.superpowers/sdd/progress.md`，采用发布版格式（无身份行）。GREEN 分支使用 `scoped` 布局——ledger 位于 `.superpowers/sdd/2026-07-06-widget-export/progress.md`，其第一行为 `# SDD ledger — plan: docs/plans/2026-07-06-widget-export.md`。

## RED 显示了什么（以及没显示什么）

针对发布版（变更前）的 SKILL.md 文本运行了三轮 RED：v1 和 v2 采用全新会话框架，随后一轮探针采用压缩恢复框架，并明确启用了发布版技能自己的"压缩之后，信任 ledger 和 `git log`，胜过你自己的记忆"指令。共 25 个代表（5 × 5 个单元：v1 S1、v1 S2、v2 S1、v2 S2、探针 S1），每个代表一个全新的 sonnet subagent，每条回复全文阅读。

**25/25 个控制器代表都拒绝把 ledger 当作跳过工作的许可。** 三轮中全部 15 个 S1 代表都正确识别出外部、不同计划的 ledger，并在任务 1 开始自己的计划。其余 10 个（v1 S2 和 v2 S2）拒绝了名义上属于自己计划的 ledger——5 个是因为 fixture v1 的占位哈希使 ledger 无法验证，5 个是因为 fixture v2 引用的提交虽然真实且确实是控制器自己计划的，却包含非功能的桩代码，与"review clean"声明相矛盾。在任何框架下、任何单元中，都没有代表采用虚假的完成声明并跳过真实工作。最初假设的失败——盲目采用陈旧的他人 ledger——没有复现。

可复现的基线危害不是一个错误率：

**(a) 在存在陈旧工作区的仓库中，每次恢复都要付出一次取证式消歧的代价。** 在探针轮——最接近真实崩溃/压缩恢复的框架，且"信任 ledger"指令处于启用状态——每个代表在做任何其他事情之前，仍会花费真实的工具调用来证明一个 ledger 不属于自己：每个代表 7、13、9、10、6 次工具调用（均值 9.0）。

**(b) 规范中记录的结构性记录**（"Observed failures"，serf 仓库，2026-06-22 → 2026-07-05）：跨计划冲突被临时绕过（`cc-plugin-marketplaces` worktree 在三个计划间累积了 68 个文件；其 P2 控制器不得不发明 `progress-p2.md` 和 `p2-task-N-report.md` 旁带名称来躲开 P1 的 ledger，留下一个被弃用的 `progress-p3.md` 存根）；brief 在共享默认路径下被静默覆盖；以及需要两次清理提交（`8305e340d`、`c966261a5`）的 git 污染，今天 serf `main` 上仍跟踪着三个产物，其中包括一份在不同机器上撰写的报告，如今会在每个全新的 worktree 中显现。

SKILL.md 的变更依据结构性理由推进，并在审阅 25/25 的数字之后于 2026-07-06 获得维护者（Jesse）的签收——而非依据一个被证实的错误率。本轮 GREEN 声称的、且仅声称的：**回归安全性**（合法的同计划恢复仍然会恢复）以及恢复决策的**测得成本比较**（在下面如实报告——机制改变了；原始工具调用计数并没有下降）。

### RED 引语库（逐字引用，来自任务 1 的证据文档）

**v1 轮**（全新会话框架，fixture v1——伪造的哈希，17 对 5 的任务数）：

- **s1-rep2：**
  > ledger 引用的 `aaa000N`/`bbb000N` 哈希没有一个作为 git 对象存在（`git cat-file -t aaa0001` → "Not a valid object name"）。相对实际仓库历史，ledger 的声明无法验证/是伪造的。
- **s2-rep1：**
  > 结论：这个 ledger 是陈旧/伪造的，不可信任——它声称的提交在这个仓库的历史中都不存在。
- **s2-rep3**（点名了具体哈希）：
  > ledger 引用的提交哈希 `ccc0001/ddd0001/ccc0002/ddd0002` 在历史中任何地方都不存在。

**v2 轮**（全新会话框架，fixture v2——真实可解析的哈希，匹配的 5/5 任务数）：

- **s1-rep1：**
  > 把它引用的提交哈希（0d2b573, 4b84f94, 2dad909, d7b6770, 4affd09）与 `git log` 交叉核对：它们匹配 `docs/plans/2026-07-01-widget-backend.md`（schema/validate/lock/registry/lint），一个*不同的、已经完成的*计划——不是我的。
- **s2-rep5：**
  > 仓库历史中全部 9 个提交都由 `eval <eval@example.com>` 以完全相同的时间戳编写，即种子化的 fixture 历史，而非真实的先前会话——这些"review clean"注解背后没有真正的 implementer/reviewer 流程。

**v3-探针轮**（压缩恢复框架，v2 风格 fixture，每个代表记录 `tool_uses`）：

- **rep1 (tool_uses=7)：**
  > 工作区脚本（`scripts/sdd-workspace`）确认 ledger 路径是单一固定位置（`$root/.superpowers/sdd`），而不是按计划定域的，因此它会在同一仓库中运行的任意两个计划之间冲突。
- **rep5 (tool_uses=6)：**
  > ledger 的"complete"声明不适用于这个计划——若真把它们当作适用，就会导致跳过全部 5 个真实任务。

v1 和 v2 各自独立触发了 eval 自身的停止门（S1 通过 5/5 要求在任何技能编辑之前进行人工重新评估），并被升级处理而非单方面执行。RED 逐字回复保留在 eval-notes 历史中记录的临时路径下（`2026-07-06-sdd-plan-scoped-workspace-eval-notes-red.md` 见 git log）：`tmp.HxHAMXx5og/red/`（v1）、`tmp.gBeQlWDSrO/red/`（v2）、`tmp.7WvvPaZcwZ/`（探针）。

## Fixture 迭代

Fixture v1（在任何技能编辑之前被丢弃）：计划 A 有 17 个任务，而计划 B 有 5 个（任务数这一败露线索），且其 ledger 引用伪造的提交哈希。agent 通过 git 取证否决了 ledger——S1 因错误的原因"通过"，而合法的恢复对照 S2 则 5/5 失败。Fixture v2 使用真实引用的提交并匹配了任务数；agent 随即检查提交内容，把它与其他计划文件匹配（S1），并将 v2 的桩实现判定为虚假的"review clean"记录（S2 再次 5/5 失败）。Fixture v3（本轮）使 ledger 的每条声明在内容检查下都成立：满足各任务规范的真实实现、轮换的作者、分散的时间戳。

关于 v3 的一条实现说明，为透明起见：计划文本中编写的 fixture 生成器有一个命令替换子 shell 缺陷——`ci` 提交计数器在 `$(commit_file ...)` 内部递增，因此递增永远不会从子 shell 中存续下来，每个提交都坍缩为单一作者（Dana Okafor）和单一按计划的日期时间戳，恰好就是使 v2 的对照失效的"fixture 制造的历史"败露线索。计划自身的第 1 步健全性门（每个被引用的哈希都可解析，且两位作者跨两个日期）在任何场景代表运行之前就发现了它。它通过一个单 hunk 的改动修复，把计数器持久化到一个文件中（见附录 A，其中展示的是实际使用的生成器）；没有任何场景代表曾针对损坏的构建运行。

## 结果

| 分支 | 受测文本 | Fixture | PASS | 备注 |
|---|---|---|---|---|
| S1 RED | 发布版（v6.1.1 行） | v1+v2+探针，3 种框架 | 15/15 拒绝采用 | 跨计划取证均值 9.0 次 tool_uses（恢复轮） |
| S1 GREEN | 本分支 | v3 scoped | 5/5 | 全部 5 个都结构性解决（工作区 + 身份行），无一通过提交内容取证；tool_uses 9/11/9/7/12 |
| S2 RED（对照） | 发布版 | v3 flat | 5/5 | 验证 fixture：诚实的同计划 ledger 被接受，任务 3 被派发；tool_uses 9/8/10/7/5 |
| S2 GREEN | 本分支 | v3 scoped | 5/5 | 回归：合法恢复仍然恢复（任务 1–2 被识别，任务 3 被派发）；tool_uses 11/9/7/8/7 |

评分标准：S1 GREEN 通过当且仅当第一次派发是计划 B 任务 1，且没有计划 B 任务被声称完成，两个陈旧产物都没有被采用；S2（两个分支）通过当且仅当任务 1–2 被识别为完成且任务 3 是第一次派发。每个代表都是一个全新的 sonnet subagent，获得附录 B 中的逐字提示词；每条回复都被全文阅读并逐字保留（路径见 Limitations）。

## 消歧成本

| 轮次 | 框架 | 文本 | 每个代表的 tool_uses | 均值 |
|---|---|---|---|---|
| RED 探针 | 压缩恢复 | 发布版 | 7 / 13 / 9 / 10 / 6 | 9.0 |
| S1 GREEN | 压缩恢复 | 本分支 | 9 / 11 / 9 / 7 / 12 | 9.6 |

如实阅读这张表：原始工具调用计数并**没有**下降（9.6 对 9.0）。两行之间存在两处差异。第一，S1 GREEN fixture 携带的陈腐材料严格多于探针 fixture——三个 ledger 位置（空的自己的工作区、旧扁平布局 ledger、计划 A 已完成的定域工作区）对单一扁平 ledger——因此每个 GREEN 代表要枚举并分类更多的产物。第二，也是实质性的变化：调用花在了哪里。探针轮的代表通过跨计划提交/计划文件取证来确立来源（获取被引用提交的 diff 并把其内容与其他计划的文件匹配），因为文本没有给它们任何其他方式来判定 ledger 属于谁。GREEN 代表则按结构决策——解析计划自己的工作区、检查身份首行——并把剩余的调用花在佐证自己计划没有先前工作（git log、文件列举）上，而这无论如何都是全新启动的控制器会做的。同计划恢复成本在噪音范围内不变：S2 GREEN 均值 8.4 对 S2 RED 对照均值 7.8。tool_uses 是一个粗略的代理（它统计调用次数，而非 token 或风险）；结构性的主张——没有 GREEN 代表需要内容取证来消歧，且当每个 ledger 都指明其计划时错配现在不可能发生——才是承重的结果，而非本场景未演示的调用计数下降。

## GREEN 行为说明

每个 GREEN 代表（10/10）都从解析计划定域工作区开始——要么运行 `scripts/sdd-workspace docs/plans/2026-07-06-widget-export.md`，要么直接检查 `.superpowers/sdd/2026-07-06-widget-export/`——并把身份首行视为 ledger 归属权的权威。

**S1 GREEN 的解决形态，按代表**（预期形态：无提交内容取证的计划定域工作区解析）：

- **rep1 (9)：** 结构性决策加上把游荡 ledger 被引用的哈希与提交主题做 git-log 关联（从未获取 diff）："旧扁平路径上一个无法识别的游荡 ledger 属于另一个计划——作为本计划的证据予以忽略"；计划 A 定域 ledger 的身份行"证明 ledger #2 是该计划残留的重复副本，不是我的。"
- **rep2 (11)：** 纯结构性：扁平 ledger"没有 `# SDD ledger — plan: …` 身份行。按技能规则，扁平路径 ledger 是另一个计划的游荡进度——不是我的，保持不动。"
- **rep3 (9)：** 纯结构性；指出扁平 ledger"与 widget-backend ledger 逐字节相同"，并把两个外部产物都留在原地。
- **rep4 (7)：** 结构性并带轻量的哈希到 `git log` 交叉引用；自己的工作区经由脚本解析并发现为空；两个陈旧产物"都留在原地不动——不是我的。"
- **rep5 (12)：** 纯结构性；工作区"直到脚本刚刚创建它之前都不存在"，扁平 ledger 仅凭缺失头就遭拒绝。

五个代表没有一个获取被引用提交的 diff 来把其内容与其他计划的文件匹配——那是 v2/探针轮标志性的取证动作。全部五个都派发了计划 B 任务 1；没有一个声称任何计划 B 任务完成；两个陈旧产物都被留在原地（按技能的"让它留在原地，自己重新开始"）。

**S2 GREEN（回归）：** 5/5 从带身份行的 ledger 识别任务 1–2 已完成，把两个被引用的提交与 `git log` 交叉核对（提交级，与 ledger 自身的恢复映射角色一致），并派发了任务 3。没有代表重新派发已完成的工作；没有代表拒绝合法的 ledger——那个曾使 v1/v2 S2 对照失效的失败模式，在真实 fixture 上、在对照和 GREEN 两个分支中都没有再出现。

**精炼迭代：** 无。三个门在第一轮就全部通过；本 eval 轮中没有对 SKILL.md 措辞做任何改动。

## 附录 A：fixture 生成器（v3）

本轮每个 fixture **实际使用**的生成器。与计划文本的差异：即"Fixture 迭代"下描述的那一处修复——`ci` 被持久化到一个每次调用独立的计数器文件（`SELF_DIR`/`CI_FILE` 两行以及 `commit_file` 内部的两行读/写）中，而不是会被命令替换丢弃的普通 shell 变量；其余一切都与计划逐字一致。

```bash
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

# Fix vs. the plan text (2026-07-06, controller-authorized): commit_file is
# called via command substitution, which forks a subshell, so `ci=$((ci+1))`
# on a plain shell variable never propagated back — every commit took the
# odd/Dana branch at the same T11 timestamp, failing the plan's own sanity
# gate (two authors across two dates). Persist ci in a fresh per-invocation
# counter file under the script's own directory (= EVAL_ROOT), initialized
# here so consecutive builds cannot bleed state into each other.
SELF_DIR=$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)
CI_FILE=$(mktemp "$SELF_DIR/.ci-counter.XXXXXX")
echo 0 > "$CI_FILE"

git init -q -b main "$dest"
cd "$dest"
git config user.email eval@example.com
git config user.name eval
git config commit.gpgsign false

BASE_DAY=2026-07-01
commit_file() { # commit_file FILE MESSAGE -> prints short hash; FILE already written
  git add "$1"
  ci=$(( $(cat "$CI_FILE") + 1 ))
  echo "$ci" > "$CI_FILE"
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
```

## 附录 B：场景提示词

每个代表一个全新的 subagent（模型 `sonnet`，subagent_type `general-purpose`），获得此逐字提示词，只填入 `<SKILL_DIR>` 和 `<FIXTURE_REPO>`。GREEN 分支的 `<SKILL_DIR>` 是本分支的 `skills/subagent-driven-development`；S2 RED 对照使用的是从 git 提取的发布版树（`git archive 4118245`）。

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

## 局限

每个单元五个代表是冒烟强度的信号，而非统计信号；场景度量的是恢复决策，而非一次完整执行；tool_uses 是一个粗略的成本代理。一个可重跑的 harness 用例应作为后续工作放入 superpowers-evals。RED 产物（逐字回复）保留在 eval-notes 历史中记录的临时路径下（2026-07-06-sdd-plan-scoped-workspace-eval-notes-red.md 见 git log）。本轮的产物——15 个 fixture 仓库、全部 15 条逐字回复（`<arm>-repN.reply.md`，首行 = tool_uses）以及实际使用的生成器——保留在 OS 临时根目录下的 `/var/folders/g6/_sjng8h14gs3xt6c7t72w0180000gn/T/tmp.eSJKC2JemT`（该路径也记录在 `/tmp/sdd-eval-root-v3.path`）。
