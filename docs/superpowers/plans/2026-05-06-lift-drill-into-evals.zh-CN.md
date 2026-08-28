# 将 drill 提升为 superpowers 的 `evals/` — 实施计划

> **适用于智能体工作者：** 必读子技能：使用 superpowers:subagent-driven-development（推荐）或 superpowers:executing-plans 逐步实施本计划。步骤使用复选框（`- [ ]`）语法进行跟踪。

**目标：** 将独立的 `obra/drill` 技能合规性基准测试提升为 superpowers 的顶级 `evals/` 目录，在逐文件通过 subagent 验证 drill 场景覆盖之后，删除 `superpowers/tests/` 下冗余的 bash 测试，并更新顶层文档，使贡献者能落到新结构上。

**架构：** 在 `dev` 上针对新分支 `f/evals-lift` 提一个单一 PR。drill 源码原样复制，使用显式 rsync 排除项，确保 `.git/`、`.venv/` 等不进新目录。`drill/cli.py` 中的一个小助手将 `SUPERPOWERS_ROOT` 默认设置为 `evals/` 目录的父目录，这样贡献者无需设置该环境变量。每次 bash 测试删除都由一个 subagent 把关，该 subagent 将 bash 测试的断言与其声称对应的 drill 场景的 verify 块进行比较。计划文档和发布说明中的历史引用以注释标注，而非重写。

**技术栈：** Python 3.11 + uv（drill 现有工具链，不变）；rsync；bash；git。

**规格：** `docs/superpowers/specs/2026-05-06-lift-drill-into-evals-design.md` — 请先阅读此文档。

**drill 源码位置：** `/Users/jesse/Documents/GitHub/superpowers/drill/`（与 `superpowers/` 同级）。

---

## 任务 1：从 dev 切出分支

**文件：** 无（仅 git 操作）

- [ ] **步骤 1：验证工作树干净**

```bash
cd /Users/jesse/Documents/GitHub/superpowers/superpowers
git status --short
```

预期：输出为空（或仅存在未跟踪的 `.opencode/package-lock.json`，这没问题）。

- [ ] **步骤 2：拉取最新 dev**

```bash
git fetch origin dev:dev
```

- [ ] **步骤 3：创建分支**

```bash
git checkout -b f/evals-lift dev
```

预期：`Switched to a new branch 'f/evals-lift'`。

- [ ] **步骤 4：健全性检查**

```bash
git log --oneline -1
```

预期：输出以 `origin/dev` 所指向的提交开头（当前为 `b4363df docs: turned the dash in "- Jesse" into an escape sequence (#1474)`）。

---

## 任务 2：在复制时记录 drill SHA

**文件：** 无（为 lift 提交信息记录该值）

- [ ] **步骤 1：获取当前 drill HEAD SHA**

```bash
cd /Users/jesse/Documents/GitHub/superpowers/drill
DRILL_SHA=$(git rev-parse HEAD)
echo "$DRILL_SHA"
```

- [ ] **步骤 2：验证 drill 没有未提交的工作**

```bash
cd /Users/jesse/Documents/GitHub/superpowers/drill
git status --short
```

预期：为空（无未跟踪或已修改文件）。如果输出非空，停止并报告 — 在 lift 之前 drill 工作树必须干净，否则 SHA 锁定毫无意义。

- [ ] **步骤 3：将 SHA 保存到 shell 环境变量供下一任务使用**

```bash
echo "DRILL_SHA=$DRILL_SHA"  # write this down for use in Task 3
```

---

## 任务 3：将 drill rsync 到 evals/

**文件：**
- 创建：`evals/`（来自 drill 的完整目录树，减去排除项）

- [ ] **步骤 1：验证源与目标路径**

```bash
cd /Users/jesse/Documents/GitHub/superpowers/superpowers
test -d /Users/jesse/Documents/GitHub/superpowers/drill && echo "drill source: OK"
test ! -d evals && echo "evals/ does not yet exist: OK"
```

预期：两条 echo 都会打印。

- [ ] **步骤 2：使用显式排除项将 drill rsync 到 evals/**

```bash
cd /Users/jesse/Documents/GitHub/superpowers/superpowers
rsync -a \
  --exclude=.git \
  --exclude=.venv \
  --exclude=results \
  --exclude=.env \
  --exclude=__pycache__ \
  --exclude='*.egg-info' \
  --exclude=.private-journal \
  --exclude='*.pyc' \
  /Users/jesse/Documents/GitHub/superpowers/drill/ \
  evals/
```

- [ ] **步骤 3：验证排除项生效**

```bash
find evals -name '.git' -type d
find evals -name '.venv' -type d
find evals -name 'results' -type d
find evals -name '.env'
find evals -name '__pycache__' -type d
find evals -name '*.egg-info' -type d
```

预期：每条命令都无输出。如果任一条返回了路径，在继续前手动 `rm -rf` 它。

- [ ] **步骤 4：确认为提交信息准备的源 SHA**

```bash
cd /Users/jesse/Documents/GitHub/superpowers/drill
DRILL_SHA=$(git rev-parse HEAD)
echo "$DRILL_SHA"
```

预期：与任务 2 步骤 1 的 SHA 相同。

- [ ] **步骤 5：暂存全部内容**

```bash
git add evals/
git status --short | head -20
```

预期：输出以列出大量新增文件的 `A  evals/...` 行开头。其中很多位于 scenarios/、drill/、backends/、setup_helpers/ 等目录下。

- [ ] **步骤 6：提交**

```bash
: "${DRILL_SHA:?Set DRILL_SHA from Task 2 before committing}"
git commit -m "$(cat <<EOF
Lift drill into evals/ at $DRILL_SHA

rsync of obra/drill@$DRILL_SHA into superpowers/evals/, excluding
.git/, .venv/, results/, .env/, __pycache__/, *.egg-info/,
.private-journal/.

The drill repo is unaffected by this commit; archival is a separate
manual step after this PR merges.

Source SHA recorded in this commit message for provenance.
EOF
)"
```

---

## 任务 4：用校验和验证复制结果

**文件：** 无（仅验证）

- [ ] **步骤 1：获取 drill 中存在但不应出现在 evals 中的文件列表（即排除项）**

```bash
cd /Users/jesse/Documents/GitHub/superpowers/drill
find . \
  \( -name '.git' -prune \
  -o -name '.venv' -prune \
  -o -name 'results' -prune \
  -o -name '__pycache__' -prune \
  -o -name '*.egg-info' -prune \
  -o -name '.private-journal' -prune \
  -o -name '*.pyc' -prune \
  -o -name '.env' -prune \) \
  -o -type f -print | sort > /tmp/drill-files.txt
wc -l /tmp/drill-files.txt
```

- [ ] **步骤 2：获取 evals/ 中的文件列表**

```bash
cd /Users/jesse/Documents/GitHub/superpowers/superpowers
find evals -type f | sed 's|^evals/|./|' | sort > /tmp/evals-files.txt
wc -l /tmp/evals-files.txt
```

- [ ] **步骤 3：比较两个列表**

排除路径移除后，文件列表应完全一致。

```bash
diff /tmp/drill-files.txt /tmp/evals-files.txt
```

预期：无输出。

- [ ] **步骤 4：逐文件校验和验证**

```bash
cd /Users/jesse/Documents/GitHub/superpowers/drill
while read -r f; do
  sha1=$(shasum -a 256 "$f" | cut -d' ' -f1)
  sha2=$(shasum -a 256 "/Users/jesse/Documents/GitHub/superpowers/superpowers/evals/${f#./}" | cut -d' ' -f1)
  if [ "$sha1" != "$sha2" ]; then
    echo "MISMATCH: $f ($sha1 vs $sha2)"
  fi
done < /tmp/drill-files.txt | head -20
```

预期：无输出（每个文件的校验和在 drill 与 evals 之间都一致）。

- [ ] **步骤 5：冒烟检查 — 安装依赖**

```bash
cd /Users/jesse/Documents/GitHub/superpowers/superpowers/evals
uv sync
```

预期：输出 `Installed N packages` 或类似内容。无错误。

- [ ] **步骤 6：冒烟检查 — drill list**

```bash
cd /Users/jesse/Documents/GitHub/superpowers/superpowers/evals
uv run drill list 2>&1 | head -5
```

预期：以场景名开头。（很可能因缺少 SUPERPOWERS_ROOT 而报错或警告 — 没关系，下一任务会修复。）

- [ ] **步骤 7：派发验证 subagent**

派发一个 `general-purpose` subagent，提示词如下：

```
You are verifying a verbatim copy of the drill repo at
/Users/jesse/Documents/GitHub/superpowers/drill into
/Users/jesse/Documents/GitHub/superpowers/superpowers/evals.

Verify:

1. The lift commit message records the SHA reported by:
  cd /Users/jesse/Documents/GitHub/superpowers/drill && git rev-parse HEAD

2. None of these excluded paths exist under evals/: .git/, .venv/,
results/, .env/, __pycache__/, *.egg-info/, .private-journal/.

3. Every non-excluded file in drill has a SHA-256-identical
counterpart in evals/, and there are no extra files in evals/.

4. The pyproject.toml, uv.lock, scenarios/*.yaml, backends/*.yaml,
setup_helpers/*.py, drill/*.py, prompts/*.md, fixtures/, bin/, and
docs/ are all present.

Report each check with PASS/FAIL. If any FAIL, dump enough detail
that the parent can fix.
```

如果 subagent 报告任何 FAIL，在继续前修复根本问题（删除泄漏的文件、重新 rsync 等）。

---

## 任务 5：添加 `SUPERPOWERS_ROOT` 默认助手

**文件：**
- 修改：`evals/drill/cli.py:11-14`

- [ ] **步骤 1：读取当前 cli.py 头部**

```bash
sed -n '1,20p' /Users/jesse/Documents/GitHub/superpowers/superpowers/evals/drill/cli.py
```

预期输出：

```python
"""Drill CLI: run, compare, list."""

from __future__ import annotations

import secrets
from pathlib import Path

import click
from dotenv import load_dotenv

PROJECT_ROOT: Path = Path(__file__).parent.parent

load_dotenv(PROJECT_ROOT / ".env")
```

- [ ] **步骤 2：为该助手编写一个会失败的测试**

打开 `evals/tests/test_cli.py`，在末尾添加以下测试：

```python
def test_set_superpowers_root_default_when_unset(monkeypatch, tmp_path):
    """When SUPERPOWERS_ROOT is unset, helper sets it to PROJECT_ROOT.parent."""
    monkeypatch.delenv("SUPERPOWERS_ROOT", raising=False)
    from drill.cli import _set_superpowers_root_default, PROJECT_ROOT

    _set_superpowers_root_default()

    import os
    assert os.environ["SUPERPOWERS_ROOT"] == str(PROJECT_ROOT.parent)


def test_set_superpowers_root_default_respects_existing(monkeypatch):
    """When SUPERPOWERS_ROOT is already set, helper does not override."""
    monkeypatch.setenv("SUPERPOWERS_ROOT", "/custom/path")
    from drill.cli import _set_superpowers_root_default

    _set_superpowers_root_default()

    import os
    assert os.environ["SUPERPOWERS_ROOT"] == "/custom/path"
```

- [ ] **步骤 3：运行测试并观察其失败**

```bash
cd /Users/jesse/Documents/GitHub/superpowers/superpowers/evals
uv run pytest tests/test_cli.py -k set_superpowers_root_default -v
```

预期：2 个测试失败，报 `AttributeError: module 'drill.cli' has no attribute '_set_superpowers_root_default'`。

- [ ] **步骤 4：将助手添加到 cli.py**

编辑 `/Users/jesse/Documents/GitHub/superpowers/superpowers/evals/drill/cli.py`。将第 1–14 行替换为：

```python
"""Drill CLI: run, compare, list."""

from __future__ import annotations

import os
import secrets
from pathlib import Path

import click
from dotenv import load_dotenv

PROJECT_ROOT: Path = Path(__file__).parent.parent

load_dotenv(PROJECT_ROOT / ".env")


def _set_superpowers_root_default() -> None:
    """Default SUPERPOWERS_ROOT to the parent of evals/ if not already set.

    Drill historically required contributors to export SUPERPOWERS_ROOT
    pointing at the superpowers checkout. After lifting drill into
    superpowers/evals/, the parent of PROJECT_ROOT is always the
    superpowers root, so we can supply this default automatically.

    Existing SUPERPOWERS_ROOT environment values are respected as overrides.
    """
    os.environ.setdefault("SUPERPOWERS_ROOT", str(PROJECT_ROOT.parent))


_set_superpowers_root_default()
```

模块底部的 `_set_superpowers_root_default()` 调用在导入时运行，紧随 `load_dotenv()` 之后。这确保 `engine.py` 和 `setup.py`（直接读取 `os.environ["SUPERPOWERS_ROOT"]`）以及 YAML 插值（在后端 YAML 加载时读取 `os.environ`）都能看到该值。

- [ ] **步骤 5：运行测试并观察其通过**

```bash
cd /Users/jesse/Documents/GitHub/superpowers/superpowers/evals
uv run pytest tests/test_cli.py -k set_superpowers_root_default -v
```

预期：2 个测试通过。

- [ ] **步骤 6：提交**

```bash
cd /Users/jesse/Documents/GitHub/superpowers/superpowers
git add evals/drill/cli.py evals/tests/test_cli.py
git commit -m "evals: default SUPERPOWERS_ROOT to parent of evals/ if unset

Adds _set_superpowers_root_default() to drill/cli.py, called at
module import after load_dotenv(). PROJECT_ROOT resolves to evals/
post-lift; its parent is the superpowers repo root, which is the
correct value for SUPERPOWERS_ROOT.

Existing env values are respected as overrides via os.environ.setdefault.

Tests:
- helper sets default when var is unset
- helper does not override when var is already set"
```

---

## 任务 6：更新后端 YAML 以反映新的环境变量契约

**文件：**
- 修改：`evals/backends/codex.yaml`（从 `required_env` 中移除 `SUPERPOWERS_ROOT`）
- 修改：`evals/backends/gemini.yaml`（从 `required_env` 中移除 `SUPERPOWERS_ROOT`）

五个 `claude*.yaml` 后端配置将 `${SUPERPOWERS_ROOT}` 插值进 `--plugin-dir` 标志的 `args` 中 — 它们保留 `SUPERPOWERS_ROOT` 在 `required_env` 中，因为插值需要它。codex/gemini 配置列出它，仅是为了 engine.py/setup.py 的 `os.environ` 读取，现在助手已满足该需求。

- [ ] **步骤 1：确认当前状态**

```bash
grep -A3 'required_env:' /Users/jesse/Documents/GitHub/superpowers/superpowers/evals/backends/codex.yaml
grep -A2 'required_env:' /Users/jesse/Documents/GitHub/superpowers/superpowers/evals/backends/gemini.yaml
```

预期输出中包含 `- SUPERPOWERS_ROOT` 行。

- [ ] **步骤 2：完整阅读 codex.yaml**

```bash
cat /Users/jesse/Documents/GitHub/superpowers/superpowers/evals/backends/codex.yaml
```

- [ ] **步骤 3：编辑 codex.yaml — 移除 `required_env` 下的 `- SUPERPOWERS_ROOT` 行**

打开 `evals/backends/codex.yaml`，找到：

```yaml
required_env:
  - OPENAI_API_KEY
  - SUPERPOWERS_ROOT
```

替换为：

```yaml
required_env:
  - OPENAI_API_KEY
```

- [ ] **步骤 4：编辑 gemini.yaml — 移除 `required_env` 下的 `- SUPERPOWERS_ROOT` 行**

打开 `evals/backends/gemini.yaml`，找到：

```yaml
required_env:
  - SUPERPOWERS_ROOT
```

替换为：

```yaml
required_env: []
```

（使用空列表而非删除该字段，这样 YAML schema 验证不会报错。）

- [ ] **步骤 5：运行 drill 的 pytest 套件，确保没有破坏任何东西**

```bash
cd /Users/jesse/Documents/GitHub/superpowers/superpowers/evals
uv run pytest -x 2>&1 | tail -20
```

预期：全部测试通过。如果 `tests/test_backend.py` 抱怨 codex/gemini 的 `required_env` 成员关系，参见任务 7。

- [ ] **步骤 6：提交**

```bash
cd /Users/jesse/Documents/GitHub/superpowers/superpowers
git add evals/backends/codex.yaml evals/backends/gemini.yaml
git commit -m "evals: drop SUPERPOWERS_ROOT from codex/gemini required_env

These backends only read SUPERPOWERS_ROOT via engine.py/setup.py's
os.environ access, which the new cli.py default helper supplies
automatically. claude*.yaml keep SUPERPOWERS_ROOT in required_env
because they interpolate \${SUPERPOWERS_ROOT} into --plugin-dir args."
```

---

## 任务 7：为新契约更新 drill 的 pytest 套件

**文件：**
- 修改：`evals/tests/test_backend.py`（若任务 6 步骤 5 暴露了失败，则逐测试更新）

- [ ] **步骤 1：运行测试套件**

```bash
cd /Users/jesse/Documents/GitHub/superpowers/superpowers/evals
uv run pytest tests/test_backend.py -v 2>&1 | tail -30
```

如果全部测试通过，跳到步骤 5（不提交任何内容，进入任务 8）。否则：

- [ ] **步骤 2：阅读失败的测试**

对每个失败，打开 `evals/tests/test_backend.py` 中的测试并阅读其断言。

- [ ] **步骤 3：更新断言**

对断言 `SUPERPOWERS_ROOT` 属于 `codex.yaml` 或 `gemini.yaml` 的 `required_env` 的测试：反转断言以确认其缺失。示例：

```python
# Before:
def test_codex_requires_superpowers_root():
    backend = load_backend("codex")
    assert "SUPERPOWERS_ROOT" in backend.required_env

# After:
def test_codex_does_not_require_superpowers_root():
    """codex.yaml dropped SUPERPOWERS_ROOT from required_env;
    the cli.py helper supplies the default."""
    backend = load_backend("codex")
    assert "SUPERPOWERS_ROOT" not in backend.required_env
```

- [ ] **步骤 4：重新运行测试套件**

```bash
cd /Users/jesse/Documents/GitHub/superpowers/superpowers/evals
uv run pytest -x 2>&1 | tail -10
```

预期：全部测试通过。

- [ ] **步骤 5：提交（仅当步骤 1 有失败时）**

```bash
cd /Users/jesse/Documents/GitHub/superpowers/superpowers
git add evals/tests/test_backend.py
git commit -m "evals: update test_backend.py for relaxed required_env contract"
```

---

## 任务 8：更新 evals/README.md 和 evals/CLAUDE.md

**文件：**
- 修改：`evals/README.md`（移除 SUPERPOWERS_ROOT 设置步骤）
- 修改：`evals/CLAUDE.md`（移除 SUPERPOWERS_ROOT 设置步骤）

- [ ] **步骤 1：编辑 evals/README.md**

找到类似下面的段落：

```markdown
Required environment:
```bash
export SUPERPOWERS_ROOT=/path/to/superpowers
export ANTHROPIC_API_KEY=sk-...
```
```

替换为：

```markdown
Required environment:
```bash
export ANTHROPIC_API_KEY=sk-...
```

`SUPERPOWERS_ROOT` 默认设置为 `evals/` 的父目录（superpowers 仓库根目录），仅当你要针对不同的 superpowers checkout 运行 drill 时才需要设置。
```

- [ ] **步骤 2：编辑 evals/CLAUDE.md**

找到下面的段落：

```markdown
## Required env

```
SUPERPOWERS_ROOT=/path/to/superpowers
ANTHROPIC_API_KEY=sk-...
```
```

替换为：

```markdown
## Required env

```
ANTHROPIC_API_KEY=sk-...
```

`SUPERPOWERS_ROOT` 默认设置为 `evals/` 的父目录（superpowers 仓库根目录）。仅当针对不同的 superpowers checkout 运行 drill 时才覆盖它。
```

- [ ] **步骤 3：提交**

```bash
cd /Users/jesse/Documents/GitHub/superpowers/superpowers
git add evals/README.md evals/CLAUDE.md
git commit -m "evals: drop SUPERPOWERS_ROOT setup step from README/CLAUDE

The cli.py helper now defaults the env var. Mention as override only."
```

---

## 任务 9：从新位置验证

**文件：** 无（仅验证 — 除非需要修复某些问题，否则不提交）

- [ ] **步骤 1：运行 drill 的完整 pytest 套件**

```bash
cd /Users/jesse/Documents/GitHub/superpowers/superpowers/evals
unset SUPERPOWERS_ROOT
uv run pytest 2>&1 | tail -5
```

预期：全部测试通过。`unset` 确保我们测试的是助手，而非继承的环境变量。

- [ ] **步骤 2：运行 drill list**

```bash
cd /Users/jesse/Documents/GitHub/superpowers/superpowers/evals
unset SUPERPOWERS_ROOT
uv run drill list 2>&1 | head -10
```

预期：场景列表，无关于缺失 SUPERPOWERS_ROOT 的错误。

- [ ] **步骤 3：导入环境文件**

```bash
set -a
source /Users/jesse/Documents/GitHub/prime-radiant-inc/sprout/.env
set +a
echo "ANTHROPIC_API_KEY set: ${ANTHROPIC_API_KEY:+yes}"
```

预期：`ANTHROPIC_API_KEY set: yes`。

- [ ] **步骤 4：运行一个低成本的 drill 场景**

```bash
cd /Users/jesse/Documents/GitHub/superpowers/superpowers/evals
unset SUPERPOWERS_ROOT
uv run drill run triggering-test-driven-development -b claude 2>&1 | tail -3
```

预期：`claude: 1 passed, 0 failed, 0 errors`。

如果 FAIL，在继续前调试。路径默认值的变化是最可能的元凶；通过在助手调用之后临时添加 `print(os.environ["SUPERPOWERS_ROOT"])` 来检查助手是否真的被触发。

---

## 任务 10：bash 测试删除阶段 — 逐文件并带 subagent 把关

本任务包含很多子步骤，因为每个候选删除文件都要经过自己的 subagent 验证 + 提交。候选列表来自规格的覆盖映射。对下面的每一项：

1. 读取 bash 测试文件。
2. 读取候选 drill 场景 YAML。
3. 派发一个 subagent，同时提供两者的内容与比较提示词。
4. subagent 报告逐断言的匹配表。
5. 如果每个 bash 断言都有匹配：删除该 bash 测试，提交。
6. 如果有任何不匹配：停止、上报，不删除。

**Subagent 提示词模板（每次删除都使用）：**

```
You are gating a bash test deletion. The bash test is allegedly
covered by a drill scenario; your job is to verify that claim.

BASH TEST: <paste full contents of bash test>

DRILL SCENARIO: <paste full contents of drill scenario YAML>

Output a markdown table with columns: BASH ASSERTION, DRILL CHECK,
STATUS. List EVERY assertion the bash test makes (every grep, every
[ ], every test command, every PASS/FAIL emit). For each, find a
matching drill check (in verify.assertions or verify.criteria) or
mark as UNMATCHED.

After the table, output "VERDICT: SAFE TO DELETE" if every bash
assertion has a match, otherwise "VERDICT: KEEP — N unmatched
assertions". Be conservative: if you are uncertain about a match,
mark as UNMATCHED.
```

### 任务 10a：技能触发提示词（6 个文件）

**文件：**
- 删除：`tests/skill-triggering/prompts/dispatching-parallel-agents.txt`
- 删除：`tests/skill-triggering/prompts/executing-plans.txt`
- 删除：`tests/skill-triggering/prompts/requesting-code-review.txt`
- 删除：`tests/skill-triggering/prompts/systematic-debugging.txt`
- 删除：`tests/skill-triggering/prompts/test-driven-development.txt`
- 删除：`tests/skill-triggering/prompts/writing-plans.txt`
- 保留：`tests/skill-triggering/run-test.sh`、`run-all.sh`

这些提示文件是 bash 运行器的输入 — 它们自己没有断言。断言由运行器脚本完成。将每个提示映射到它的 drill 场景：

| 提示 | Drill 场景 |
|--------|----------------|
| dispatching-parallel-agents.txt | triggering-dispatching-parallel-agents.yaml |
| executing-plans.txt | triggering-executing-plans.yaml |
| requesting-code-review.txt | triggering-requesting-code-review.yaml |
| systematic-debugging.txt | triggering-systematic-debugging.yaml |
| test-driven-development.txt | triggering-test-driven-development.yaml |
| writing-plans.txt | triggering-writing-plans.yaml |

- [ ] **步骤 1：对每个提示文件派发 subagent**

对提示 `tests/skill-triggering/prompts/<name>.txt` 和场景 `evals/scenarios/triggering-<name>.yaml`，将两者的内容粘贴进 subagent 提示词模板运行。subagent 的工作是验证提示内容与 drill 场景的 `turns[].intent` 描述是否一致。

如果全部 6 个都验证为 SAFE TO DELETE，进入步骤 2。如果任何一个验证为 KEEP，保留那一个，其余仍可继续。

- [ ] **步骤 2：验证运行器对无关场景仍有用**

```bash
ls /Users/jesse/Documents/GitHub/superpowers/superpowers/tests/skill-triggering/prompts/
```

如果按计划删除后 prompts/ 目录为空，同时删除 `tests/skill-triggering/run-test.sh` 和 `run-all.sh`（它们没有可运行的内容）。否则保留运行器。

- [ ] **步骤 3：删除并提交**

```bash
cd /Users/jesse/Documents/GitHub/superpowers/superpowers
git rm tests/skill-triggering/prompts/dispatching-parallel-agents.txt
git rm tests/skill-triggering/prompts/executing-plans.txt
git rm tests/skill-triggering/prompts/requesting-code-review.txt
git rm tests/skill-triggering/prompts/systematic-debugging.txt
git rm tests/skill-triggering/prompts/test-driven-development.txt
git rm tests/skill-triggering/prompts/writing-plans.txt
# If runner is now orphaned:
git rm tests/skill-triggering/run-test.sh tests/skill-triggering/run-all.sh
rmdir tests/skill-triggering/prompts/ 2>/dev/null || true
rmdir tests/skill-triggering/ 2>/dev/null || true
git commit -m "tests: remove skill-triggering bash prompts (covered by drill triggering-* scenarios)

Subagent verification confirmed each prompt's intent matches its
corresponding drill scenario's turns[].intent. Drill scenarios are
canonical; bash runner has no remaining prompts to drive."
```

### 任务 10b：显式技能请求（选择性删除）

**文件：**
- 检查：`tests/explicit-skill-requests/` 中的 6 个文件
- 删除：仅删除经验证被 drill 场景 100% 覆盖的那些
- 保留：其余

按规格的更新版覆盖映射，其中大多数没有对应的 drill 场景。可能被删除的：

| Bash 测试 | 候选 drill 场景 | 可能的结果 |
|-----------|--------------------------|----------------|
| `run-test.sh` | 不适用（运行器） | 保留 |
| `run-all.sh` | 不适用（运行器） | 保留 |
| `run-claude-describes-sdd.sh` | `mid-conversation-skill-invocation.yaml` | 可能删除；需验证 |
| `run-haiku-test.sh` | 无（Haiku 专属） | 保留 |
| `run-multiturn-test.sh`、`run-extended-multiturn-test.sh` | 无 | 保留 |
| `prompts/please-use-brainstorming.txt`、`prompts/use-systematic-debugging.txt` | 无 | 保留 |

- [ ] **步骤 1：阅读每个 .sh 文件和提示以确认**

```bash
for f in /Users/jesse/Documents/GitHub/superpowers/superpowers/tests/explicit-skill-requests/*.sh /Users/jesse/Documents/GitHub/superpowers/superpowers/tests/explicit-skill-requests/prompts/*.txt; do
  echo "=== $f ==="
  cat "$f" | head -30
done
```

- [ ] **步骤 2：仅对 `run-claude-describes-sdd.sh` 派发 subagent**

使用上面的 subagent 提示词模板，参数为：
- Bash 测试内容：`tests/explicit-skill-requests/run-claude-describes-sdd.sh`
- Drill 场景：`evals/scenarios/mid-conversation-skill-invocation.yaml`

- [ ] **步骤 3：根据 subagent 裁决行动**

如果 SAFE TO DELETE：

```bash
cd /Users/jesse/Documents/GitHub/superpowers/superpowers
git rm tests/explicit-skill-requests/run-claude-describes-sdd.sh
git commit -m "tests: remove run-claude-describes-sdd.sh (covered by drill mid-conversation-skill-invocation)

Subagent verification: every assertion matches a drill check.
Other tests in tests/explicit-skill-requests/ are preserved
(run-haiku-test.sh, run-*-multiturn-test.sh, please-use-brainstorming
and use-systematic-debugging prompts have no drill coverage)."
```

如果 KEEP：跳过删除，将缺口记录为未来的 drill 场景编写任务。

### 任务 10c：subagent 驱动的真实项目测试

**文件：**
- 检查：`tests/subagent-driven-dev/go-fractals/`、`tests/subagent-driven-dev/svelte-todo/`
- 候选场景：`evals/scenarios/sdd-go-fractals.yaml`、`evals/scenarios/sdd-svelte-todo.yaml`

这些是完整的 fixture 目录，包含 `design.md`、`plan.md`、`scaffold.sh`。每个 fixture 目录都已作为 `evals/fixtures/` 下的 fixture 提升进 drill。

- [ ] **步骤 1：确认 drill 具有 fixture 对等性**

```bash
ls /Users/jesse/Documents/GitHub/superpowers/superpowers/evals/fixtures/sdd-go-fractals/
ls /Users/jesse/Documents/GitHub/superpowers/superpowers/evals/fixtures/sdd-svelte-todo/
```

预期：每个目录包含与 `tests/subagent-driven-dev/` 下源码匹配的 `design.md`、`plan.md`、`scaffold.sh`（或等价内容）。

- [ ] **步骤 2：对每一对派发 subagent**

Subagent 提示词：使用相同模板，其中 bash "测试"是该目录的 `scaffold.sh` 以及（如果存在）任何 `*.sh` 运行器。drill 场景为对应的 `sdd-*.yaml`。

- [ ] **步骤 3：根据裁决行动**

对每个返回 SAFE TO DELETE 的：

```bash
cd /Users/jesse/Documents/GitHub/superpowers/superpowers
git rm -r tests/subagent-driven-dev/go-fractals/   # or svelte-todo
git commit -m "tests: remove subagent-driven-dev/<fixture> (covered by drill sdd-<fixture>)

Subagent verification: drill scenario asserts test suite passes
post-execution. Fixture content lives at evals/fixtures/sdd-<fixture>/."
```

如果两个目录都被删除，且 `tests/subagent-driven-dev/` 变为空目录，也执行 `git rm -r tests/subagent-driven-dev/`。

### 任务 10d：tests/claude-code/test-document-review-system.sh

**候选场景：** `evals/scenarios/spec-reviewer-catches-planted-flaws.yaml`

- [ ] **步骤 1：派发 subagent**

使用 subagent 提示词模板，内容为该 bash 测试和该 drill 场景 YAML。

- [ ] **步骤 2：根据裁决行动**

如果 SAFE TO DELETE：

```bash
cd /Users/jesse/Documents/GitHub/superpowers/superpowers
git rm tests/claude-code/test-document-review-system.sh
git commit -m "tests: remove test-document-review-system.sh (covered by drill spec-reviewer-catches-planted-flaws)

Subagent verification: every assertion matches a drill check."
```

### 任务 10e：tests/claude-code/test-requesting-code-review.sh

**候选场景：** `evals/scenarios/code-review-catches-planted-bugs.yaml`

- [ ] **步骤 1：派发 subagent**

使用 subagent 提示词模板，内容为两者。

- [ ] **步骤 2：根据裁决行动**

如果 SAFE TO DELETE：

```bash
cd /Users/jesse/Documents/GitHub/superpowers/superpowers
git rm tests/claude-code/test-requesting-code-review.sh
git commit -m "tests: remove test-requesting-code-review.sh (covered by drill code-review-catches-planted-bugs)

Subagent verification: every assertion matches a drill check."
```

### 任务 10f：tests/claude-code/test-worktree-native-preference.sh

**候选场景：** `evals/scenarios/worktree-creation-under-pressure.yaml`

- [ ] **步骤 1：派发 subagent**

使用 subagent 提示词模板，内容为两者。

- [ ] **步骤 2：根据裁决行动**

如果 SAFE TO DELETE：

```bash
cd /Users/jesse/Documents/GitHub/superpowers/superpowers
git rm tests/claude-code/test-worktree-native-preference.sh
git commit -m "tests: remove test-worktree-native-preference.sh (covered by drill worktree-creation-under-pressure)

Subagent verification: every assertion matches a drill check."
```

### 任务 10g：tests/claude-code/test-subagent-driven-development-integration.sh

**候选场景：** `evals/scenarios/sdd-rejects-extra-features.yaml`（部分覆盖）

规格将其标记为"几乎肯定保留 + 扩展 drill 场景"。不要删除。而是：

- [ ] **步骤 1：无论如何派发 subagent 做比较**

这显式记录了缺口。

- [ ] **步骤 2：根据 subagent 输出决定**

可能的结果：保留并记录缺口。该 bash 测试断言：`commit_count >= 3`、`npm test` 通过、运行 `analyze-token-usage.py`。drill 场景断言禁止导出 + 以评审者作为把关。这些大多互不相交。

- [ ] **步骤 3：记录缺口**（如果 KEEP）

在 `tests/claude-code/test-subagent-driven-development-integration.sh` 顶部添加注释：

```bash
# Drill coverage: sdd-rejects-extra-features.yaml covers the YAGNI
# enforcement (forbidden exports + reviewer-as-gate). This bash test
# additionally asserts: ≥3 task commits, npm test passes, token
# analysis runs. Keep until those assertions are added to drill or
# explicitly retired.
```

```bash
cd /Users/jesse/Documents/GitHub/superpowers/superpowers
git add tests/claude-code/test-subagent-driven-development-integration.sh
git commit -m "tests: annotate SDD integration test with drill coverage notes

Drill scenario sdd-rejects-extra-features covers the YAGNI subset.
This bash test adds: ≥3 commits, npm test, token analysis. Kept
until drill scenario covers those or they're retired."
```

### 任务 10h：tests/claude-code/test-subagent-driven-development.sh

这是一个元/描述技能测试（按规格）。没有任何 drill 场景覆盖 describe-skill 行为。

- [ ] **步骤 1：通过阅读文件确认**

```bash
cat /Users/jesse/Documents/GitHub/superpowers/superpowers/tests/claude-code/test-subagent-driven-development.sh
```

预期：测试的是让 agent 描述 SDD 技能，而非演练它们。

- [ ] **步骤 2：保留并注释**

在顶部添加：

```bash
# No drill coverage: this test asks the agent to *describe* SDD
# (asserts that asked-about skills can be summarized correctly).
# Drill scenarios test behavior, not description. Kept.
```

```bash
cd /Users/jesse/Documents/GitHub/superpowers/superpowers
git add tests/claude-code/test-subagent-driven-development.sh
git commit -m "tests: annotate SDD describe-skill test with kept-by-design note

Tests agent's ability to *describe* the SDD skill — drill scenarios
test behavior, not description. No drill coverage; kept by design."
```

---

## 任务 11：过时引用清理

**文件：**
- 可能修改：`docs/testing.md`、`README.md`、`CLAUDE.md`、`lefthook.yml`、`.opencode/INSTALL.md`、`.codex-plugin/INSTALL.md`、`.github/*`、`scripts/*`
- 注释（不重写）：`RELEASE-NOTES.md`、`docs/superpowers/plans/*.md`

- [ ] **步骤 1：构建已删除文件的路径列表**

```bash
cd /Users/jesse/Documents/GitHub/superpowers/superpowers
git diff --name-only --diff-filter=D dev..HEAD | sort > /tmp/deleted-paths.txt
cat /tmp/deleted-paths.txt
```

- [ ] **步骤 2：搜索活跃引用**

```bash
cd /Users/jesse/Documents/GitHub/superpowers/superpowers
while read -r path; do
  echo "=== $path ==="
  grep -rln "$path" \
    --include="*.md" \
    --include="*.yml" \
    --include="*.yaml" \
    --include="*.sh" \
    --include="*.json" \
    --exclude-dir=node_modules \
    --exclude-dir=.venv \
    --exclude-dir=evals \
    --exclude-dir=.git \
    .
done < /tmp/deleted-paths.txt
```

这能找到对已删除文件的每个引用。将每个命中分类：

| 命中位置 | 处理方式 |
|--------------|-----------|
| `docs/testing.md` | 更新 — 它在活跃地记录该测试 |
| `README.md`（Contributing 部分） | 如果指向已删除的测试则更新 |
| `CLAUDE.md`、`GEMINI.md`、`AGENTS.md` | 如果引用已删除的测试则更新 |
| `.github/workflows/*.yml` | 更新 — CI 不应尝试运行已删除的测试 |
| `scripts/*` | 如果运行已删除的测试则更新 |
| `.opencode/INSTALL.md`、`.codex-plugin/INSTALL.md` | 如果引用已删除的测试则更新 |
| `lefthook.yml` | 如果钩子调用已删除的测试则更新 |
| `RELEASE-NOTES.md` | 注释，不重写（带日期的产物） |
| `docs/superpowers/plans/*.md` | 注释，不重写（带日期的产物） |

- [ ] **步骤 3：更新活跃引用**

对每个"更新"类命中，编辑文件以：
- 如果该已删除测试是被命名提及的唯一原因，则移除该引用。
- 或替换为指向 drill 场景的指针（例如 "参见 `evals/scenarios/triggering-test-driven-development.yaml`"）。

- [ ] **步骤 4：为带日期的产物添加注释**

对 `RELEASE-NOTES.md` 或 `docs/superpowers/plans/*.md` 的每个命中，在每文件第一个命中处添加内联注释：

```markdown
> Note: this section references `tests/skill-triggering/run-all.sh` and
> related bash tests that were lifted into drill scenarios on 2026-05-06
> (see `evals/scenarios/triggering-*.yaml`). The references are
> preserved as dated artifacts of the work this doc describes.
```

不要修改实际引用 — 它们是历史性的。

- [ ] **步骤 5：派发 subagent 做第二轮清理**

派发一个 `general-purpose` subagent：

```
Working directory: /Users/jesse/Documents/GitHub/superpowers/superpowers

These bash test paths were deleted on the current branch; some are
already addressed, but I want a second pair of eyes:

<paste contents of /tmp/deleted-paths.txt>

Search the entire superpowers tree (excluding evals/, node_modules/,
.venv/, .git/) for any remaining references to those paths. Report
every hit with file:line and one-sentence judgment of whether it
needs an update or is fine as-is. Do not modify files; just report.
```

在继续前处理报告的每个命中。

- [ ] **步骤 6：提交活跃更新**

```bash
cd /Users/jesse/Documents/GitHub/superpowers/superpowers
git add -u  # picks up edits to existing files
git commit -m "docs: update references to lifted-and-deleted bash tests

Active references in docs/testing.md, README.md, CI workflows, etc.
now point at drill scenarios. Historical references in RELEASE-NOTES.md
and docs/superpowers/plans/*.md are annotated as dated artifacts,
not rewritten."
```

---

## 任务 12：顶层文档

**文件：**
- 修改：`docs/testing.md` — 拆分为 "插件测试" + "技能行为评测（evals）"
- 修改：`CLAUDE.md` — 添加 evals 指针
- 修改：`README.md` — 在 Contributing 部分添加指针
- 修改：`.gitignore` — 添加 `evals/results/`、`evals/.venv/`、`evals/.env`

- [ ] **步骤 1：拆分 docs/testing.md**

该文件目前以 Claude Code 为中心。拆分为两个顶层部分。

打开 `/Users/jesse/Documents/GitHub/superpowers/superpowers/docs/testing.md`，将文件内容替换为以下结构（在适用处保留现有的插件测试细节）：

```markdown
# Testing Superpowers

Superpowers has two distinct kinds of tests, each in its own directory:

- **`tests/`** — does the plugin's non-LLM code work? Bash + node + python integration tests for brainstorm-server JS, OpenCode plugin loading, codex-plugin sync, and analysis utilities.
- **`evals/`** — do agents behave correctly on real LLM sessions? Python harness driving real tmux sessions of Claude Code / Codex / Gemini CLI / Copilot CLI, with an LLM actor and verifier judging skill compliance.

## Plugin tests

Live in `tests/`. Currently:

- `tests/brainstorm-server/` — node test suite for the brainstorm server JS code.
- `tests/opencode/` — bash tests for OpenCode plugin loading, bootstrap caching, and tool registration.
- `tests/codex-plugin-sync/` — bash sync verification.
- `tests/claude-code/test-helpers.sh`, `analyze-token-usage.py` — utilities used by remaining bash tests.
- `tests/claude-code/test-subagent-driven-development.sh` — agent-can-describe-SDD test (no drill counterpart).
- `tests/claude-code/test-subagent-driven-development-integration.sh` — extended SDD integration with token analysis (drill covers the YAGNI subset).
- `tests/explicit-skill-requests/` — Haiku-specific, multi-turn, and skill-name-prompted tests not covered by drill.

Run plugin tests via the relevant directory's `run-*.sh` or `npm test`.

## Skill behavior evals

Live in `evals/`. Drill is the harness; scenarios live at `evals/scenarios/*.yaml`. See `evals/README.md` for setup. Quick start:

```bash
cd evals
uv sync
export ANTHROPIC_API_KEY=sk-...
uv run drill run triggering-test-driven-development -b claude
```

Drill scenarios are slow (3-30+ minutes each) and run real LLM sessions. They are not part of CI today; the natural follow-up is a tiered model (fast subset on PR, full sweep nightly + on-demand).
```

- [ ] **步骤 2：更新 CLAUDE.md**

阅读当前 CLAUDE.md，找到项目结构部分附近的位置，添加：

```markdown
## Eval harness

Skill-behavior evals live at `evals/` — see `evals/README.md`. Drill (the harness) drives real tmux sessions of Claude Code / Codex / Gemini CLI / Copilot CLI and judges skill compliance with an LLM verifier. Plugin-infrastructure tests still live at `tests/`.
```

- [ ] **步骤 3：更新 README.md**

找到 Contributing 部分。添加一行：

```markdown
- Skill-behavior tests use the eval harness at `evals/`. See `evals/README.md` for setup. Plugin-infrastructure tests live at `tests/` and run via the relevant `run-*.sh` or `npm test`.
```

- [ ] **步骤 4：更新顶层 .gitignore**

打开 `/Users/jesse/Documents/GitHub/superpowers/superpowers/.gitignore`，在底部添加：

```
# Eval harness — drill ships its own gitignore at evals/.gitignore;
# these are belt-and-suspenders entries for tools that don't recurse.
evals/results/
evals/.venv/
evals/.env
```

- [ ] **步骤 5：提交**

```bash
cd /Users/jesse/Documents/GitHub/superpowers/superpowers
git add docs/testing.md CLAUDE.md README.md .gitignore
git commit -m "docs: introduce evals/ as the canonical skill-behavior eval harness

- docs/testing.md split into Plugin tests + Skill behavior evals
- CLAUDE.md adds Eval harness section pointing at evals/
- README.md Contributing section mentions evals/ alongside tests/
- .gitignore adds evals/{results,.venv,.env} as belt-and-suspenders
  (evals/.gitignore covers these locally; root-level entries help
  tooling that does not recurse into nested ignore files)."
```

---

## 任务 13：重新运行冒烟检查（回归门禁）

**文件：** 无（仅验证）

- [ ] **步骤 1：运行 drill 的 pytest**

```bash
cd /Users/jesse/Documents/GitHub/superpowers/superpowers/evals
unset SUPERPOWERS_ROOT
uv run pytest 2>&1 | tail -5
```

预期：全部测试通过。

- [ ] **步骤 2：运行低成本 drill 场景**

```bash
set -a
source /Users/jesse/Documents/GitHub/prime-radiant-inc/sprout/.env
set +a
cd /Users/jesse/Documents/GitHub/superpowers/superpowers/evals
unset SUPERPOWERS_ROOT
uv run drill run triggering-test-driven-development -b claude 2>&1 | tail -3
```

预期：`claude: 1 passed, 0 failed, 0 errors`。如果 FAIL，说明文档 / 清理 / 删除阶段破坏了一些东西 — 在最近的提交上进行二分排查。

- [ ] **步骤 3：运行幸存的其余插件测试**

```bash
cd /Users/jesse/Documents/GitHub/superpowers/superpowers/tests/brainstorm-server
node server.test.js 2>&1 | tail -3
```

预期：`Results: 25 passed, 0 failed`。

---

## 任务 14：最终对抗性评审

**文件：** 无（仅评审；派发 subagent）

- [ ] **步骤 1：为评审者构建 diff**

```bash
cd /Users/jesse/Documents/GitHub/superpowers/superpowers
git log --oneline dev..HEAD
git diff dev..HEAD --stat
```

捕获两个输出与评审者共享。

- [ ] **步骤 2：并行派发两个 subagent**

使用 `Agent` 工具，两次并行调用。两个提示词相同，采用对抗性框架：

```
Adversarial review competition: 5 points to whoever finds the most
legitimate issues. You're competing against a parallel reviewer
assigned the identical task.

**Branch:** f/evals-lift, in /Users/jesse/Documents/GitHub/superpowers/superpowers
**Base:** dev (currently b4363df)
**Spec:** docs/superpowers/specs/2026-05-06-lift-drill-into-evals-design.md

This branch lifts the obra/drill repo into superpowers/evals/ and
deletes redundant bash tests that drill scenarios cover. Two prior
adversarial reviews caught issues at the spec stage; this is the
post-implementation review.

Run: git log --oneline dev..HEAD; git diff dev..HEAD --stat

Look hard at:
1. Did the rsync-with-excludes actually exclude what it claimed?
   (find evals -name '.git' -type d should return nothing)
2. Does the lift commit message point at a real commit in obra/drill?
3. Does the SUPERPOWERS_ROOT helper actually default correctly when
   the env var is unset? (cd evals && unset SUPERPOWERS_ROOT && uv
   run drill list — does it work?)
4. For each deleted bash test, does the corresponding drill scenario
   actually verify what the bash test asserted? Spot-check by reading
   the scenario YAML.
5. Are there active references in docs/, .github/, scripts/,
   lefthook.yml that still point at deleted bash test paths?
6. Did the drill pytest suite get updated for the new env-var contract,
   and does it pass?
7. Did the smoke scenario actually get run after path changes?
8. Is the drill repo unchanged? (cd ../drill && git status)

Verify before claiming. If you assert "X is broken", check on disk
first. Confidently-wrong claims count negatively.

Report format: numbered list, each with severity (critical/important/
minor/nitpick) and one-sentence explanation with file:line. Lead with
most serious. Cap at ~600 words.
```

- [ ] **步骤 3：处理发现**

对任一位评审者的每个合理发现，在单独提交中修复。修复后重新运行冒烟检查（任务 13）。

- [ ] **步骤 4：宣布胜者**

按跨平台 PR 模式，统计合理发现的数量（误报计为负分）。在回复总结中致谢胜者。

---

## 任务 15：推送并打开 PR

**文件：** 无

- [ ] **步骤 1：推送分支**

```bash
cd /Users/jesse/Documents/GitHub/superpowers/superpowers
git push -u origin f/evals-lift
```

- [ ] **步骤 2：针对 dev 打开 PR，附带完整描述**

```bash
gh pr create \
  --base dev \
  --head f/evals-lift \
  --reviewer arittr \
  --title "Lift drill into superpowers as evals/ harness" \
  --body "$(cat <<'EOF'
## What problem are you trying to solve?

Drill — the standalone Python skill-compliance benchmark at obra/drill — is already the de facto eval harness for superpowers. The PRI-1397 commit series lifted ~22 bash tests into drill scenarios, and the most recent superpowers commit (a2292c5) explicitly removed a redundant bash test with the message "replaced by drill behavioral coverage". Drill is a sibling repo today, requiring contributors to clone two checkouts and set SUPERPOWERS_ROOT manually. This PR completes the migration: drill becomes superpowers/evals/.

## What does this PR change?

- Lifts the obra/drill repo into superpowers as `evals/`, with explicit rsync excludes (.git, .venv, results, .env, __pycache__, *.egg-info, .private-journal). The lift commit records the source SHA.
- Adds a `_set_superpowers_root_default()` helper to drill/cli.py so SUPERPOWERS_ROOT defaults to the parent of evals/ — no manual env-var setup.
- Drops SUPERPOWERS_ROOT from required_env in codex.yaml/gemini.yaml (the helper supplies it). Claude*.yaml keep it because they interpolate ${SUPERPOWERS_ROOT} into --plugin-dir args.
- Deletes redundant bash tests under tests/skill-triggering/, tests/explicit-skill-requests/, tests/subagent-driven-dev/, and tests/claude-code/ — gated per-file by a subagent that compared each bash test's assertions to its drill scenario's verify block. Anything not 100% covered was kept.
- docs/testing.md split into Plugin tests + Skill behavior evals.
- README.md Contributing and CLAUDE.md gain pointers to evals/.

## Is this change appropriate for the core library?

Yes. Cross-runtime evaluation is core to superpowers, the migration to drill scenarios was already underway in this repo, and the eval harness needs to be discoverable in-tree to be findable.

## What alternatives did you consider?

- Vendored copy + sync script (drill repo continues independently). Rejected: divergence risk; single-source-of-truth wins.
- git subtree merge (preserves drill history in-tree). Rejected: superpowers' git history grows by 50+ commits, the merge commit is ugly, subtrees are operationally heavy.
- Keep drill as a sibling repo and just polish docs. Rejected: doesn't solve the discoverability problem.

## Does this PR contain multiple unrelated changes?

No — every change supports "drill is now evals/ inside superpowers". Multiple commits for atomicity (verbatim copy, env helper, YAML updates, docs) but one direction.

## Existing PRs

- [x] I have reviewed all open AND closed PRs for duplicates or prior art
- Related PRs: #1486 (obra/superpowers cross-platform PR — independent; no shared file changes besides README, which has no overlap)

## Environment tested

| Harness | Version | Model | Model ID |
|---------|---------|-------|----------|
| Claude Code | local install | Opus | claude-opus-4-7 (1M context) |

Drill's own pytest suite passes from the new location. `triggering-test-driven-development` drill scenario passes from `evals/` after the path-default changes. (Larger drill sweep deferred to release-cadence runs per the spec's deferred-CI policy.)

## Evaluation

- Initial prompt: see linked spec (`docs/superpowers/specs/2026-05-06-lift-drill-into-evals-design.md`).
- Drill's own pytest suite passes.
- One drill scenario re-run from the new location end-to-end (proves the SUPERPOWERS_ROOT default works).
- Per-deleted-file subagent verification recorded in each deletion commit's message.

## Rigor

- [x] If this is a skills change: this is not a skills change; it's a tooling/infrastructure migration. No behavior-shaping content modified.
- [x] Adversarial pressure-tested: two parallel reviewers on the spec; final adversarial pre-PR review on the implementation; spec already corrected for findings before implementation began.
- [x] Did not modify carefully-tuned content.

## Human review

- [x] A human has reviewed the COMPLETE proposed diff before submission

## Action items after merge

1. Archive obra/drill on GitHub (mark read-only, add README pointer to obra/superpowers/evals/).
2. The spec lists CI integration, scenario co-location with skills, and Python package rename as deferred work. Open issues for any of these you want tracked.
EOF
)"
```

- [ ] **步骤 3：确认 PR 已打开**

```bash
gh pr view --web
```

预期：浏览器打开新 PR。截屏或记下 URL 供后续跟进。

---

## 验证清单（在任务 15 之后运行）

- [ ] `git log --oneline dev..HEAD` 按顺序显示预期的提交
- [ ] lift 提交信息记录了源 SHA
- [ ] `find evals -name '.git' -type d` 无输出
- [ ] `cd evals && unset SUPERPOWERS_ROOT && uv run pytest` 通过
- [ ] `cd evals && unset SUPERPOWERS_ROOT && uv run drill list` 返回场景
- [ ] `cd evals && unset SUPERPOWERS_ROOT && uv run drill run triggering-test-driven-development -b claude` 通过
- [ ] `tests/brainstorm-server/server.test.js` 仍通过（非 LLM 测试的回归门禁）
- [ ] `git diff dev..HEAD docs/superpowers/plans/2026-04-06-worktree-rototill.md docs/superpowers/plans/2026-03-23-codex-app-compatibility.md RELEASE-NOTES.md` 只显示注释，无路径重写
- [ ] `cd ../drill && git log --oneline -1` 显示 obra/drill 与 lift 提交中记录的源 SHA 相比未变
- [ ] PR 正文列出合并后的归档行动项
```
