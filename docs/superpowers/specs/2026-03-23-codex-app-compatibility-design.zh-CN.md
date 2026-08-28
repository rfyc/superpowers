# Codex App 兼容性：Worktree 与收尾技能适配

让 superpowers 技能在 Codex App 的沙箱化 worktree 环境中正常工作，同时不破坏既有 Claude Code 或 Codex CLI 的行为。

**工单：** PRI-823

## 动机

Codex App 在其管理的 git worktree 内运行代理——detached HEAD，位于 `$CODEX_HOME/worktrees/` 下，并带有一个阻止 `git checkout -b`、`git push` 和网络访问的 Seatbelt 沙箱。三个 superpowers 技能假设不受限制的 git 访问：`using-git-worktrees` 用命名分支创建手动 worktree，`finishing-a-development-branch` 按分支名合并/推送/创建 PR，`subagent-driven-development` 两者都需要。

Codex CLI（开源终端工具）没有此冲突——它没有内置的 worktree 管理。我们手动 worktree 的方法填补了那里的隔离缺口。问题特指 Codex App。

## 经验性发现

于 2026-03-23 在 Codex App 中测试：

| 操作 | workspace-write 沙箱 | Full access 沙箱 |
|---|---|---|
| `git add` | 可用 | 可用 |
| `git commit` | 可用 | 可用 |
| `git checkout -b` | **被阻止**（无法写入 `.git/refs/heads/`） | 可用 |
| `git push` | **被阻止**（网络 + `.git/refs/remotes/`） | 可用 |
| `gh pr create` | **被阻止**（网络） | 可用 |
| `git status/diff/log` | 可用 | 可用 |

额外发现：
- `spawn_agent` subagents **共享**父线程的文件系统（已通过标记文件测试确认）
- 无论 worktree 是从哪个分支启动的，App 头部都会出现 "Create branch" 按钮
- App 的原生收尾流程：Create branch → Commit modal → Commit and push / Commit and create PR
- `network_access = true` 配置在 macOS 上被静默破坏（issue #10390）

## 设计：只读环境检测

三条只读 git 命令无副作用地检测环境：

```bash
GIT_DIR=$(cd "$(git rev-parse --git-dir)" 2>/dev/null && pwd -P)
GIT_COMMON=$(cd "$(git rev-parse --git-common-dir)" 2>/dev/null && pwd -P)
BRANCH=$(git branch --show-current)
```

由此派生两个信号：

- **IN_LINKED_WORKTREE：** `GIT_DIR != GIT_COMMON` —— 代理位于由其他东西（Codex App、Claude Code Agent 工具、之前的技能运行或用户）创建的 worktree 中
- **ON_DETACHED_HEAD：** `BRANCH` 为空 —— 不存在命名分支

为什么用 `git-dir != git-common-dir` 而不是检查 `show-toplevel`：
- 在普通仓库中，两者都解析到同一个 `.git` 目录
- 在链接 worktree 中，`git-dir` 是 `.git/worktrees/<name>`，而 `git-common-dir` 是 `.git`
- 在子模块中，两者相等——避免了 `show-toplevel` 会产生的误报
- 通过 `cd && pwd -P` 解析可处理相对路径问题（`git-common-dir` 在普通仓库中返回相对路径 `.git`，但在 worktree 中返回绝对路径）以及符号链接（macOS `/tmp` → `/private/tmp`）

### 决策矩阵

| 链接 Worktree？ | Detached HEAD？ | 环境 | 行动 |
|---|---|---|---|
| 否 | 否 | Claude Code / Codex CLI / 普通 git | 完整技能行为（不变） |
| 是 | 是 | Codex App worktree（workspace-write） | 跳过 worktree 创建；收尾时交接 payload |
| 是 | 否 | Codex App（Full access）或手动 worktree | 跳过 worktree 创建；完整收尾流程 |
| 否 | 是 | 异常情况（手动 detached HEAD） | 正常创建 worktree；收尾时警告 |

## 变更

### 1. `using-git-worktrees/SKILL.md` —— 新增 Step 0（约 12 行）

"Overview" 与 "Directory Selection Process" 之间的新章节：

**Step 0: 检查是否已在隔离工作区中**

运行检测命令。如果 `GIT_DIR != GIT_COMMON`，完全跳过 worktree 创建。改为：
1. 跳转到 Creation Steps 下的 "Run Project Setup" 小节——`npm install` 等操作是幂等的，为安全起见值得运行
2. 然后 "Verify Clean Baseline"——运行测试
3. 报告分支状态：
   - 在分支上："Already in an isolated workspace at `<path>` on branch `<name>`. Tests passing. Ready to implement."
   - Detached HEAD："Already in an isolated workspace at `<path>` (detached HEAD, externally managed). Tests passing. Note: branch creation needed at finish time. Ready to implement."

如果 `GIT_DIR == GIT_COMMON`，继续完整的 worktree 创建流程（不变）。

当 Step 0 触发时跳过安全验证（.gitignore 检查）——对外部创建的 worktree 不适用。

更新 Integration 部分的 "Called by" 条目。将每条描述从上下文特定的文本改为："Ensures isolated workspace (creates one or verifies existing)"。例如，`subagent-driven-development` 条目从 "REQUIRED: Set up isolated workspace before starting" 改为 "REQUIRED: Ensures isolated workspace (creates one or verifies existing)"。

**沙箱回退：** 如果 `GIT_DIR == GIT_COMMON` 且技能继续进入 Creation Steps，但 `git worktree add -b` 因权限错误失败（例如 Seatbelt 沙箱拒绝），则将其视为延迟检测到的受限环境。回退到 Step 0 的"已在工作区中"行为——跳过创建，在当前目录中运行 setup 和基线测试，并相应报告。

在 Step 0 报告后，停止。不要继续到 Directory Selection 或 Creation Steps。

**其余一切不变：** Directory Selection、Safety Verification、Creation Steps、Project Setup、Baseline Tests、Quick Reference、Common Mistakes、Red Flags。

### 2. `finishing-a-development-branch/SKILL.md` —— 新增 Step 1.5 + 清理守卫（约 20 行）

**Step 1.5: 检测环境**（在 Step 1 "Verify Tests" 之后、Step 2 "Determine Base Branch" 之前）

运行检测命令。三条路径：

- **路径 A** 完全跳过 Step 2 和 3（不需要基准分支或选项）。
- **路径 B 和 C** 正常进行 Step 2（Determine Base Branch）和 Step 3（Present Options）。

**路径 A —— 外部管理的 worktree + detached HEAD**（`GIT_DIR != GIT_COMMON` 且 `BRANCH` 为空）：

首先，确保所有工作都已暂存并提交（`git add` + `git commit`）。Codex App 的收尾控件作用于已提交的工作。

然后向用户呈现以下内容（不呈现 4 选项菜单）：

```
Implementation complete. All tests passing.
Current HEAD: <full-commit-sha>

This workspace is externally managed (detached HEAD).
I cannot create branches, push, or open PRs from here.

⚠ These commits are on a detached HEAD. If you do not create a branch,
they may be lost when this workspace is cleaned up.

If your host application provides these controls:
- "Create branch" — to name a branch, then commit/push/PR
- "Hand off to local" — to move changes to your local checkout

Suggested branch name: <ticket-id/short-description>
Suggested commit message: <summary-of-work>
```

分支名推导：有工单 ID 就用（例如 `pri-823/codex-compat`），否则将计划标题的前 5 个单词 slug 化，再否则省略建议。避免在分支名中包含敏感内容（漏洞描述、客户名）。

跳到 Step 5（对外部管理的 worktree 而言，清理是空操作）。

**路径 B —— 外部管理的 worktree + 命名分支**（`GIT_DIR != GIT_COMMON` 且存在 `BRANCH`）：

正常呈现 4 选项菜单。（Step 5 的清理守卫会独立地重新检测外部管理状态。）

**路径 C —— 正常环境**（`GIT_DIR == GIT_COMMON`）：

照今天一样呈现 4 选项菜单（不变）。

**Step 5 清理守卫：**

在清理时重新运行 `GIT_DIR` 与 `GIT_COMMON` 的检测（不要依赖早期技能输出——收尾技能可能在不同的会话中运行）。如果 `GIT_DIR != GIT_COMMON`，跳过 `git worktree remove`——宿主环境拥有这个工作区。

否则，照今天一样检查和移除。注意：现有 Step 5 文本说 "For Options 1, 2, 4"，但 Quick Reference 表和 Common Mistakes 部分说 "Options 1 & 4 only"。新守卫加在此既有逻辑之前，不改变哪些选项触发清理。

**其余一切不变：** Options 1-4 逻辑、Quick Reference、Common Mistakes、Red Flags。

### 3. `subagent-driven-development/SKILL.md` 和 `executing-plans/SKILL.md` —— 各编辑 1 行

两个技能都有相同的 Integration 部分行。从：
```
- superpowers:using-git-worktrees - REQUIRED: Set up isolated workspace before starting
```
改为：
```
- superpowers:using-git-worktrees - REQUIRED: Ensures isolated workspace (creates one or verifies existing)
```

**其余一切不变：** 派发/审查循环、提示模板、模型选择、状态处理、red flags。

### 4. `codex-tools.md` —— 新增环境检测文档（约 15 行）

末尾的两个新章节：

**环境检测：**

```markdown
## Environment Detection

Skills that create worktrees or finish branches should detect their
environment with read-only git commands before proceeding:

\```bash
GIT_DIR=$(cd "$(git rev-parse --git-dir)" 2>/dev/null && pwd -P)
GIT_COMMON=$(cd "$(git rev-parse --git-common-dir)" 2>/dev/null && pwd -P)
BRANCH=$(git branch --show-current)
\```

- `GIT_DIR != GIT_COMMON` → already in a linked worktree (skip creation)
- `BRANCH` empty → detached HEAD (cannot branch/push/PR from sandbox)

See `using-git-worktrees` Step 0 and `finishing-a-development-branch`
Step 1.5 for how each skill uses these signals.
```

**Codex App 收尾：**

```markdown
## Codex App Finishing

When the sandbox blocks branch/push operations (detached HEAD in an
externally managed worktree), the agent commits all work and informs
the user to use the App's native controls:

- **"Create branch"** — names the branch, then commit/push/PR via App UI
- **"Hand off to local"** — transfers work to the user's local checkout

The agent can still run tests, stage files, and output suggested branch
names, commit messages, and PR descriptions for the user to copy.
```

## 不变的内容

- `implementer-prompt.md`、`spec-reviewer-prompt.md`、`code-quality-reviewer-prompt.md` —— subagent 提示未改动
- `executing-plans/SKILL.md` —— 仅 1 行 Integration 描述变更（与 `subagent-driven-development` 相同）；所有运行时行为不变
- `dispatching-parallel-agents/SKILL.md` —— 无 worktree 或收尾操作
- `.codex/INSTALL.md` —— 安装流程不变
- 4 选项收尾菜单 —— 为 Claude Code 和 Codex CLI 精确保留
- 完整 worktree 创建流程 —— 为非 worktree 环境精确保留
- Subagent 派发/审查/迭代循环 —— 不变（文件系统共享已确认）

## 范围汇总

| 文件 | 变更 |
|---|---|
| `skills/using-git-worktrees/SKILL.md` | +12 行（Step 0） |
| `skills/finishing-a-development-branch/SKILL.md` | +20 行（Step 1.5 + 清理守卫） |
| `skills/subagent-driven-development/SKILL.md` | 1 行编辑 |
| `skills/executing-plans/SKILL.md` | 1 行编辑 |
| `skills/using-superpowers/references/codex-tools.md` | +15 行 |

5 个文件共约新增/变更 50 行。零新文件。零破坏性变更。

## 未来考量

如果第三个技能需要相同的检测模式，将其抽取到共享的 `references/environment-detection.md` 文件（方案 B）。现在不需要——只有 2 个技能使用它。

## 测试计划

### 自动化（实现后在 Claude Code 中运行）

1. 普通仓库检测 —— 断言 IN_LINKED_WORKTREE=false
2. 链接 worktree 检测 —— `git worktree add` 测试 worktree，断言 IN_LINKED_WORKTREE=true
3. Detached HEAD 检测 —— `git checkout --detach`，断言 ON_DETACHED_HEAD=true
4. 收尾技能交接输出 —— 在受限环境中验证交接消息（而非 4 选项菜单）
5. **Step 5 清理守卫** —— 创建一个链接 worktree（`git worktree add /tmp/test-cleanup -b test-cleanup`），`cd` 进入其中，运行 Step 5 清理检测（`GIT_DIR` 与 `GIT_COMMON`），断言它不会调用 `git worktree remove`。然后 `cd` 回到主仓库，运行相同检测，断言它会调用 `git worktree remove`。之后清理测试 worktree。

### 手动 Codex App 测试（5 项测试）

1. Worktree 线程中的检测（workspace-write）—— 验证 GIT_DIR != GIT_COMMON，分支为空
2. Worktree 线程中的检测（Full access）—— 相同检测，不同的沙箱行为
3. 收尾技能交接格式 —— 验证代理发出交接 payload，而非 4 选项菜单
4. 完整生命周期 —— 检测 → 提交 → 收尾检测 → 正确行为 → 清理
5. **Local 线程中的沙箱回退** —— 启动一个 Codex App **Local 线程**（workspace-write 沙箱）。提示语："Use the superpowers skill `using-git-worktrees` to set up an isolated workspace for implementing a small change." 预检：`git checkout -b test-sandbox-check` 应以 `Operation not permitted` 失败。预期：技能检测到 `GIT_DIR == GIT_COMMON`（普通仓库），尝试 `git worktree add -b`，遇到 Seatbelt 拒绝，回退到 Step 0 的"已在工作区中"行为——运行 setup、基线测试、从当前目录报告就绪。通过：代理优雅恢复，无晦涩错误消息。失败：代理打印原始 Seatbelt 错误、重试或带着令人困惑的输出放弃。

### 回归

- 既有 Claude Code 技能触发测试仍然通过
- 既有 subagent-driven-development 集成测试仍然通过
- 正常 Claude Code 会话：完整 worktree 创建 + 4 选项收尾仍然可用
