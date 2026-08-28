# 将 drill 提升为 `evals/` 移入 superpowers——设计

## 背景

Drill 是一个 Python 技能合规性基准，存在于它自己的仓库 `obra/drill` 中。它驱动真实的 tmux 会话，运行一个 LLM actor 作为模拟用户，对生成的转录运行 LLM verifier，并逐场景报告通过/失败。它支持 Claude Code、Codex、Gemini CLI，以及（按最近的提交）OpenCode 和 Copilot CLI。

Drill 已经是 superpowers 事实上的 eval harness。drill 仓库中的 PRI-1397 提交系列将约 22 个 superpowers bash 测试提升为 drill 场景，而最近的 superpowers 提交（`a2292c5`）以 *"replaced by drill behavioral coverage"* 的消息明确移除了一个冗余的 bash 测试。迁移势头已存在；本规格完成它。

这项工作将 drill 以 `evals/` 移入 superpowers，在逐文件验证 drill 场景覆盖后删除冗余的 bash 测试，并更新文档以便贡献者落到新结构上。

## 目标

1. `evals/` 是 superpowers 中的规范 eval harness——完整的 drill 源码、场景、fixtures、提示词、后端配置和测试。
2. `superpowers/tests/` 中已逐个验证为 100% 被 drill 场景覆盖的 bash 测试被删除；其余被保留。
3. `tests/`（插件基础设施：bash + node + python 集成测试）与 `evals/`（带 actor + verifier 的 LLM 行为）之间的区分是有意义且已文档化的。
4. 顶层文档（`README.md`、`CLAUDE.md`、`docs/testing.md`）把贡献者指向正确位置。
5. 独立的 `obra/drill` 仓库继续存在（本 PR 不触碰它），并在本 PR 合并后作为单独的手动步骤归档。

## 非目标

- **CI 集成。** 此处仅手动。自然的后续是"分层"的：每个 PR 上跑快速子集，每晚 + 按需跑全量。这需要 API 预算决策、GitHub Actions secrets，以及安装了 `tmux` + `node` + `python` + `claude` / `codex` / `gemini` CLI 的 runner 镜像。超出范围。
- **场景与技能同处一地的布局。** 场景保持集中在 `evals/scenarios/`。如果我们稍后决定每个技能应拥有自己的场景，那是一次路径查找并重命名的操作；YAML 格式不变。
- **重命名内部 Python 包**（`drill` → `evals`）。目录是 `evals/`（面向用户）；Python 包保持其 `drill` 名称以保持 diff 很小。`evals/README.md` 中有一段简短说明。
- **Drill 仓库归档。** 本 PR 不触碰 `obra/drill`。合并后，drill 仓库被手动归档（GitHub 上只读，README 指向 `obra/superpowers/evals/`）。
- **将 `tests/claude-code/analyze-token-usage.py` 提升到 `evals/bin/`。** 有用的工具，不是测试代码。可以稍后移动；本 PR 不要求。

## 分支

从 `dev` 分出 `f/evals-lift`。这项工作独立于开放的 `f/cross-platform` PR——除可能的 `README.md` 外无共享文件更改，README 足够小，冲突时可在合并时解决。

## 移动后的架构

```
superpowers/
  evals/                              ← 新增（完整 drill 副本）
    pyproject.toml                    （Python 3.11，uv 管理）
    uv.lock
    .gitignore                        （drill 自己的；results/、.venv/、.env）
    README.md                         （原 drill 的 README；安装说明已更新）
    CLAUDE.md                         （原 drill 的 CLAUDE.md；路径已更新）
    docs/
      design.md                       （drill 的设计——原样保留，与本规格交叉链接）
      manual-testing.md
      pressure-and-red-testing.md
    drill/                            （Python 包；名称保留；cli、engine、actor、verifier 等）
    backends/                         （claude-*.yaml、codex.yaml、gemini.yaml）
    scenarios/                        （32+ 个 YAML 场景）
    setup_helpers/                    （15 个 Python 辅助程序；create_base_repo、sdd_*、spec_*、worktree 等）
    fixtures/                         （template-repo、sdd-go-fractals、sdd-svelte-todo）
    prompts/                          （actor.md、verifier.md）
    bin/                              （断言辅助脚本：tool-called、tool-count 等）
    tests/                            （drill 自己的 pytest 套件）

  tests/                              ← bash 测试默认保留
    brainstorm-server/                ← 保留（brainstorm-server JS 代码的 node 测试）
    opencode/                         ← 保留（插件加载测试）
    codex-plugin-sync/                ← 保留（同步验证）
    claude-code/                      ← 大部分保留——见删除门控
    explicit-skill-requests/          ← 除非验证已被替换，否则保留
    skill-triggering/                 ← 除非验证已被替换，否则保留
    subagent-driven-dev/              ← 除非验证已被替换，否则保留

  docs/
    testing.md                        ← 已更新（拆分为"插件测试" + "技能行为 evals"）
    superpowers/
      specs/
        2026-05-06-lift-drill-into-evals-design.md   ← 本规格

  README.md                           ← Contributing 部分中指向 evals/ 的简短指针
  CLAUDE.md                           ← 一行"Eval harness 位于 evals/"指针
```

本 PR 后，`tests/` 和 `evals/` 目录服务于明确不同的角色：

- **`tests/`**——插件的非 LLM 代码能用吗？brainstorm-server JS 代码、OpenCode 插件加载、codex-plugin-sync 同步验证的单元和集成测试。Bash + node + python。
- **`evals/`**——agent 在真实 LLM 会话上行为正确吗？带 actor + verifier 的 drill 场景。纯 Python，运行真实 tmux 会话。

## 删除门控（每个 bash 测试）

只有当 drill 场景可验证地覆盖 bash 测试做出的每一项断言时，该 bash 测试才会被删除。实现计划逐文件记录这一验证：阅读 bash 测试，列出其检查项，找到 drill 场景，确认每项检查都有匹配的 `verify.assertions` 或 `verify.criteria` 条目。即使只有一项检查缺失，选项是要么扩展 drill 场景，要么保留 bash 测试。默认保留。

**初步覆盖映射**（基于提交消息；任何删除前都需要逐文件验证）：

| Bash 测试 | 声称的 drill 替代 | 覆盖状态 |
|-----------|---------------------------|-----------------|
| `tests/skill-triggering/prompts/*`（6 个提示文件） | `triggering-*.yaml`（6 个场景） | 候选——删除前逐提示验证 |
| `tests/skill-triggering/run-test.sh`、`run-all.sh` | 不适用（运行器，不是测试） | **保留**——运行器脚本 |
| `tests/explicit-skill-requests/prompts/please-use-brainstorming.txt` | 需要验证——drill 尚无明显对应物 | 除非添加 drill 场景，否则很可能**保留** |
| `tests/explicit-skill-requests/prompts/use-systematic-debugging.txt` | 需要验证——drill 尚无明显对应物 | 除非添加 drill 场景，否则很可能**保留** |
| `tests/explicit-skill-requests/run-claude-describes-sdd.sh` | 部分 → `mid-conversation-skill-invocation.yaml` | 候选——逐脚本验证 |
| `tests/explicit-skill-requests/run-haiku-test.sh` | 无 drill 场景覆盖 Haiku 特定行为 | **保留** |
| `tests/explicit-skill-requests/run-multiturn-test.sh`、`run-extended-multiturn-test.sh` | 无 drill 场景覆盖多轮构建 | 除非添加 drill 场景，否则**保留** |
| `tests/explicit-skill-requests/run-test.sh`、`run-all.sh` | 不适用（运行器） | **保留** |
| `tests/subagent-driven-dev/go-fractals/`、`tests/subagent-driven-dev/svelte-todo/` | `sdd-go-fractals.yaml`、`sdd-svelte-todo.yaml` | 候选——删除前验证（这些包含关于测试套件通过的真实断言） |
| `tests/claude-code/test-document-review-system.sh` | `spec-reviewer-catches-planted-flaws.yaml` | 候选——删除前验证 |
| `tests/claude-code/test-requesting-code-review.sh` | `code-review-catches-planted-bugs.yaml` | 候选——删除前验证 |
| `tests/claude-code/test-subagent-driven-development-integration.sh` | `sdd-rejects-extra-features.yaml`（YAGNI 子集） | **部分**——bash 测试还断言 ≥3 个提交 / `npm test` 通过 / 运行 `analyze-token-usage.py`。Drill 场景断言禁止的导出 + reviewer 作为门控。大部分不相交——几乎可以肯定**保留 + 扩展 drill 场景**。 |
| `tests/claude-code/test-subagent-driven-development.sh` | 元/文档测试（要求 agent *描述* SDD）；无 drill 场景覆盖描述类测试 | 除非添加 drill 场景，否则**保留** |
| `tests/claude-code/test-worktree-native-preference.sh` | `worktree-creation-under-pressure.yaml` | 候选——删除前验证 |
| `tests/claude-code/test-helpers.sh`、`run-skill-tests.sh`、`analyze-token-usage.py` | 不适用（工具，不是测试） | **保留**——库/工具 |

## 验证协议（subagent 门控）

实现计划中的每项变更在提交前都经由独立 subagent 交叉检查。

| 变更类别 | Subagent 验证 |
|----------------|----------------------|
| 每个 bash 测试删除 | 派发一个 subagent，输入包括：（a）bash 测试文件内容，（b）候选 drill 场景 YAML，（c）提示词：*"列出 bash 测试做出的每一项断言。列出 drill 场景中的每一项 verify 条目。对每条 bash 断言，找到匹配的 drill 检查或将其报告为未匹配。输出逐断言表格。"* subagent 的输出就是门控——只有当每条 bash 断言都有匹配时才删除。 |
| 初始 `evals/` 副本 | Subagent 验证：（a）被复制的 drill SHA 记录在提升提交消息中，以保证来源可审计；（b）**逐文件 SHA-256 校验和**与 drill 仓库中的每个文件匹配（不只是文件数）；（c）排除路径（`.git/`、`.venv/`、`results/`、`.env`、`__pycache__/`、`*.egg-info/`、任何 `.private-journal/`）不在 `evals/` 中；（d）所有后端 YAML 引用的路径在移动后存在；（e）`pyproject.toml`、`uv.lock`、`.gitignore` 完好。 |
| Drill 自己的 pytest 套件 | Subagent 在路径默认值变更后运行 `cd evals && uv run pytest`。Drill 在 `evals/tests/` 附有其自己的 pytest 套件，包括练习 `SUPERPOWERS_ROOT` 环境变量行为的 `test_backend.py`——这些测试必须更新以匹配辅助程序并继续通过。 |
| 删除后的引用清理 | Subagent 在整个 superpowers 树（排除 `node_modules/`、`.venv/` 和 `evals/`）中 grep 对已删除 bash 测试路径的引用。搜索目标：`docs/`、`docs/superpowers/plans/`、`RELEASE-NOTES.md`、`CLAUDE.md`、`GEMINI.md`、`AGENTS.md`、`README.md`、`.github/`、`scripts/`、`.opencode/INSTALL.md`、`.codex-plugin/INSTALL.md`、`lefthook.yml`。任何命中要么被更新，要么暴露出一个被遗漏的依赖。 |
| 路径默认值变更（`SUPERPOWERS_ROOT` 默认值） | Subagent 在路径变更后运行至少一个廉价的 drill 场景（例如 `triggering-test-driven-development`）并确认其仍通过。真实验证，不只是代码审查。 |
| 最终 PR 前对抗性审查 | 两个 subagent 并行，采用"找到最合法问题者得 5 分"框架——与 cross-platform PR 所用的协议相同。验证源码和行为两者。 |

每个 subagent 任务在实现计划中都有其自己的要点，含显式输入和通过标准。Subagent 的输出在相关提交消息中汇总（"Subagent verification: …"），使痕迹可审计。

## 具体的路径/配置编辑

**在本规格编写前已验证。** `drill/cli.py` 定义 `PROJECT_ROOT = Path(__file__).parent.parent`。移动后，`cli.py` 位于 `evals/drill/cli.py`，因此 `PROJECT_ROOT` 解析为 `evals/`，`PROJECT_ROOT.parent` 解析为 superpowers 仓库根目录。那就是 `SUPERPOWERS_ROOT` 应取的默认值。

**YAML 替换审计。** 只有四个 `claude*.yaml` 后端配置将 `${SUPERPOWERS_ROOT}` 插值到 `args` 中（用于 `--plugin-dir` 标志）；`codex.yaml` 和 `gemini.yaml` 只在 `required_env` 中列出 `SUPERPOWERS_ROOT`（由运行前/运行后 hooks 中 `engine.py:233` / `setup.py:25` 的 `os.environ["SUPERPOWERS_ROOT"]` 查找消耗）。辅助程序的 `os.environ` 变更覆盖两条代码路径。

| 文件 | 当前 | 之后 |
|------|---------|-------|
| `drill/cli.py` | 在模块导入时 `load_dotenv(PROJECT_ROOT / ".env")`；关于 `SUPERPOWERS_ROOT` 什么都没有 | 在 `load_dotenv` 之后，调用新的辅助程序 `_set_superpowers_root_default()`，当且仅当尚未设置时，将 `os.environ["SUPERPOWERS_ROOT"]` 设为 `str(PROJECT_ROOT.parent)`。顺序：`load_dotenv` → 设置默认值 → click 组定义。 |
| `drill/engine.py:233`、`drill/setup.py:25` | 直接 `os.environ["SUPERPOWERS_ROOT"]` 访问（未设置则 KeyError） | 不变。CLI 启动 hook 保证在 engine/setup 执行时环境变量已设置。 |
| `backends/claude*.yaml`（5 个文件） | `${SUPERPOWERS_ROOT}` 在 `args` 中为 `--plugin-dir` 替换 | 不变。YAML 替换在后端加载时读取 `os.environ`，那在 CLI 启动之后。 |
| `backends/codex.yaml`、`backends/gemini.yaml` | `SUPERPOWERS_ROOT` 仅在 `required_env` 中 | 从 `required_env` 中移除（辅助程序提供它）。`claude*.yaml` 为向后兼容保留 `required_env`（环境变量可作为覆盖）。 |
| `evals/tests/test_backend.py` | 测试断言 `SUPERPOWERS_ROOT` 在 `required_env` 列表中，外加路径解析测试 | 更新测试以匹配新契约：辅助程序提供的默认值、环境变量覆盖仍有效、codex/gemini 不再要求 `required_env`。 |
| `evals/README.md` | "export SUPERPOWERS_ROOT=/path/to/superpowers" | 移除 export 行；注明环境变量自动默认到 `evals/` 的父目录；说明唯一要求的设置是 `ANTHROPIC_API_KEY`（或 `OPENAI_API_KEY` / Gemini 认证）。 |
| `evals/CLAUDE.md` | 相同 | 相同 |
| `evals/.gitignore` | drill 现有的模式（`results/`、`.venv/`、`__pycache__/`、`.env`、`*.pyc`、`*.egg-info/`、`dist/`、`build/`、`.claude/`） | 原样复制。模式相对于文件位置，因此它们会在 `evals/` 下正确应用。 |
| `evals/lefthook.yml` | drill 附带定义 `pre-commit: uv run ruff check && uv run ty check` 的 `lefthook.yml` | 移到 `evals/lefthook.yml`。要么（a）在 superpowers 根目录安装 lefthook 并让其代理到 `evals/lefthook.yml`，要么（b）记录贡献者手动运行 `cd evals && lefthook run pre-commit`。**实现决策：为简洁起见选选项（b）**——superpowers 的顶层工作流不变。 |

`.env` 位置：保留 `evals/.env`（被 gitignore）。贡献者从那里 source 它，或在 shell 环境中设置 `ANTHROPIC_API_KEY`。

**需要小幅添加的顶层 superpowers 文件：**

- `superpowers/.gitignore`：添加 `evals/results/`、`evals/.venv/`、`evals/.env`（双保险；evals/.gitignore 已在本地覆盖这些）。
- `superpowers/CLAUDE.md`：添加一行指针"Eval harness 位于 `evals/`——见 `evals/README.md`"，以便 agent 发现它。
- `superpowers/docs/testing.md`：拆分为"## Plugin tests"（现有 tests/ 内容，剪除已删除测试的引用）和"## Skill behavior evals"（一段摘要 + 指向 `evals/` 的指针）。
- `superpowers/README.md`：在 Contributing 部分添加一行，指向 `evals/` 以进行技能行为测试。

## 迁移顺序

每个步骤是一个单独的提交（或一小小组提交）。步骤 2 是最大的单个提交（原样 drill 副本）；后续步骤小而原子。

```
1. 从 `dev` 分出分支（f/evals-lift）

2. 将 drill 仓库复制到 evals/（单个提交，易于回滚）
   ├─ 复制时记录 drill SHA → 提交消息
   ├─ 使用 `rsync -a --exclude=.git --exclude=.venv --exclude=results
   │  --exclude=.env --exclude=__pycache__ --exclude='*.egg-info'
   │  --exclude=.private-journal /path/to/drill/ evals/`
   │  （选 rsync 而非 `cp -r` 是为显式排除；用
   │  `find evals -name '.git' -type d` 验证返回为空）
   ├─ Subagent 门控：每个未排除文件的 SHA-256 校验和与 drill 仓库匹配；排除路径不在 evals/ 中
   └─ 冒烟检查：`cd evals && uv sync` 成功（仅证明安装；
      不是行为测试）

3. 更新路径默认值
   ├─ 向 drill/cli.py 添加 _set_superpowers_root_default() 辅助程序
   ├─ 在 load_dotenv 之后、click 组定义之前接线
   ├─ 更新 evals/README.md 和 evals/CLAUDE.md（移除 SUPERPOWERS_ROOT 安装步骤）
   ├─ 从 codex.yaml/gemini.yaml 的 required_env 中移除 SUPERPOWERS_ROOT
   │  （保留在 claude*.yaml 中作为覆盖）
   └─ 更新 evals/tests/test_backend.py 以匹配新契约

4. 从新位置验证（两项检查）
   ├─ 运行 drill 自己的 pytest：`cd evals && uv run pytest`——必须通过
   └─ 运行廉价的 drill 场景：`cd evals && uv run drill run
      triggering-test-driven-development -b claude`——必须通过。
      真实行为验证，不只是代码审查。

5. bash 测试删除阶段——逐文件并带 subagent 门控
   对候选删除列表中的每个文件：
   a. Subagent 对比 bash 测试断言与 drill 场景的 verify 块
   b. 通过标准：每条 bash 断言都有匹配的 drill 检查
   c. 如果通过 → 删除 bash 测试文件（每个文件或每个连贯组一个提交）
   d. 如果失败 → 要么扩展 drill 场景（单独提交 + 验证），要么
      保留 bash 测试（无提交）

6. 过期引用清理
   ├─ Subagent 在 superpowers 树（排除 node_modules/、.venv/、
   │  evals/）中 grep 已删除的文件路径
   ├─ 搜索目标：docs/、docs/superpowers/plans/、RELEASE-NOTES.md、
   │  CLAUDE.md、GEMINI.md、AGENTS.md、README.md、.github/、scripts/、
   │  .opencode/INSTALL.md、.codex-plugin/INSTALL.md、lefthook.yml
   ├─ 更新活跃引用（例如 docs/testing.md、README.md 安装）
   └─ docs/superpowers/plans/*.md 和 RELEASE-NOTES.md 中的历史引用以简短的
      注释保留（"(test removed; behavior covered by drill scenario X)"）而非
      重写——这些是带日期的产物，不是活跃文档。

7. 顶层文档
   ├─ docs/testing.md 拆分
   ├─ CLAUDE.md 指针
   └─ README.md Contributing 部分

8. 重新运行冒烟检查（回归门控）
   ├─ `cd evals && uv run pytest`
   └─ `cd evals && uv run drill run triggering-test-driven-development -b claude`

9. 最终对抗性审查
   └─ 两个并行 subagent，完整 diff，"找到最合法问题者得
      5 分"框架。推送前处理发现的问题。

10. 推送分支 + 针对 dev 打开 PR
    └─ PR 描述包括：复制时固定的 drill SHA、归档动作
       项（"合并后：归档 obra/drill，添加指向
       obra/superpowers/evals/ 的 README 指针"）、逐删除文件的覆盖凭据。
```

## 验证（实现后）

实现计划必须展示：

- 步骤 2 后所有未排除的 drill 源文件都存在于 `evals/`（subagent **逐文件 SHA-256 校验和 diff**，对比 `obra/drill@<recorded-sha>`）。
- 排除路径（`.git/`、`.venv/`、`results/`、`.env`、`__pycache__/`、`*.egg-info/`、`.private-journal/`）不在 `evals/` 中。
- 步骤 2 提交消息记录 drill 源 SHA。
- `cd evals && uv sync` 在未设置 `SUPERPOWERS_ROOT` 的情况下成功。
- `cd evals && uv run pytest` 通过（drill 自己的 pytest 套件）。
- `cd evals && uv run drill list` 返回与记录 SHA 处独立 drill 仓库相同的场景数。
- `cd evals && uv run drill run triggering-test-driven-development -b claude` 通过（证明路径默认值端到端工作）。
- 对每个已删除的 bash 测试：提交消息中有 subagent 验证表，展示每条断言都映射到 drill 检查。
- 步骤 6 后，在活跃的 superpowers 文档中 grep 已删除文件路径返回零命中；`docs/superpowers/plans/*.md` 和 `RELEASE-NOTES.md` 中的历史引用已注释而非重写。
- `docs/testing.md` 同时有"Plugin tests"和"Skill behavior evals"两个章节。
- drill 仓库的历史未被触碰；`obra/drill` 不受本 PR 影响。
- PR 描述点名合并后归档 `obra/drill` 的动作项。

## 开放问题

无。所有澄清决策都已做出：

| 问题 | 决策 |
|----------|----------|
| Drill 在 superpowers 中位于何处？ | `evals/`（从 drill 重命名）；独立仓库作为单独步骤归档 |
| 冗余 bash 测试的命运？ | 逐文件删除，并带 subagent 覆盖验证；默认保留 |
| 场景布局？ | 集中在 `evals/scenarios/` |
| Python 工具链位置？ | 在 `evals/` 内自包含 |
| CI 集成？ | 本 PR 仅手动；记录了未来路径 |
| 迁移机制？ | 直接复制；drill 仓库的历史保留在归档仓库中，不在树内 |
| 内部 Python 包名？ | 保持 `drill`（目录是 `evals/`） |
| 分支策略？ | 独立于 `dev`（不叠加在 `f/cross-platform` 上） |
