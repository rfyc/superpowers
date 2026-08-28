---
name: finishing-a-development-branch
description: 当实现完成、所有测试通过，需要决定如何整合这些工作时使用
---

# 收尾一个开发分支

## 概述

**核心原则：** 验证测试 → 检测环境 → 呈现选项 → 执行选择 → 清理。

**开始时宣布：** "我正在使用 finishing-a-development-branch 技能来完成这项工作。"

## 第 1 步：验证测试

运行项目完整测试套件（`npm test` / `cargo test` / `pytest` / `go test ./...`）。

**如果测试失败**，汇报失败并停止——菜单在测试全绿之后才出现：

```
Tests failing (<N> failures). Must fix before completing:

[Show failures]
```

**如果测试通过：** 继续第 2 步。

## 第 2 步：检测环境

```bash
GIT_DIR=$(cd "$(git rev-parse --git-dir)" 2>/dev/null && pwd -P)
GIT_COMMON=$(cd "$(git rev-parse --git-common-dir)" 2>/dev/null && pwd -P)
# Capture now, while still inside the workspace — Step 5 changes directory
# before cleanup (Step 6) needs this value
WORKTREE_PATH=$(git rev-parse --show-toplevel)
```

这决定了显示哪个菜单以及清理如何工作：

| 状态 | 菜单 | 清理 |
|-------|------|---------|
| `GIT_DIR == GIT_COMMON`（普通仓库） | 标准 3 个选项 | 无需清理 worktree |
| `GIT_DIR != GIT_COMMON`，命名分支 | 标准 3 个选项 | 基于来源（见第 6 步） |
| `GIT_DIR != GIT_COMMON`，分离 HEAD | 精简 2 个选项（无合并） | 外部管理——保持原样 |

## 第 3 步：确定基础分支

基础分支就是这项工作的分叉来源——通常在计划、对话或分支的上游中指名。如果还不知道，就问："这个分支是从 <你的最佳猜测> 分出来的——对吗？"合并前要确认：合并进错误的基础分支，撤销代价很高。

## 第 4 步：呈现选项

**普通仓库和命名分支 worktree——正好呈现这 3 个选项：**

```
Implementation complete. What would you like to do?

1. Merge back to <base-branch> locally
2. Push and create a Pull Request
3. Keep the branch as-is (I'll handle it later)

Which option?
```

**分离 HEAD——正好呈现这 2 个选项：**

```
Implementation complete. You're on a detached HEAD (externally managed workspace).

1. Push as new branch and create a Pull Request
2. Keep as-is (I'll handle it later)

Which option?
```

按原样呈现菜单——简洁，每个选项都来自上面的列表。丢弃工作只在你的人类伙伴明确要求时才会发生（见下文"如果你的人类伙伴要求丢弃工作"）。等待他们的回答；集成的决定权在他们手里。

## 第 5 步：执行选择

### 选项 1：本地合并

```bash
# Get main repo root for CWD safety
MAIN_ROOT=$(git -C "$(git rev-parse --git-common-dir)/.." rev-parse --show-toplevel)
cd "$MAIN_ROOT"

# Merge first — verify success before removing anything
git checkout <base-branch>
git pull
git merge <feature-branch>

# Verify tests on merged result
<test command>
```

如果合并结果上的测试失败：停止，让 worktree 和分支保持原位，展开调查——尚未推送任何东西，所以合并是本地且可恢复的。

一旦合并结果全绿：清理 worktree（第 6 步），然后删除分支：

```bash
git branch -d <feature-branch>
```

### 选项 2：推送并创建 PR

```bash
git push -u origin <feature-branch>
# From a detached HEAD, name the new branch on the remote:
# git push origin HEAD:refs/heads/<new-branch>
```

然后用 forge 的工具针对 <base-branch> 创建 pull/merge request——有 CLI 就用 CLI，或者用大多数 forge 推送时打印的创建 URL——遵循仓库的 PR 模板和约定（如果有的话），并把 URL 汇报给你的人类伙伴。

保留 worktree——你的人类伙伴要在那里迭代 PR 反馈。

### 选项 3：保持原样

汇报："保留分支 <name>。Worktree 保留在 <path>。"

### 如果你的人类伙伴要求丢弃工作

这条路径只作为对明确要求扔掉工作的回应而存在。先确认：

```
This will permanently delete:
- Branch <name>
- All commits: <commit-list>
- Worktree at <path>

Type 'discard' to confirm.
```

等待那个精确的确认。确认到达后：

```bash
MAIN_ROOT=$(git -C "$(git rev-parse --git-common-dir)/.." rev-parse --show-toplevel)
cd "$MAIN_ROOT"
```

然后清理 worktree（第 6 步）并强制删除分支：

```bash
git branch -D <feature-branch>
```

## 第 6 步：清理工作区

**对选项 1 和已确认的丢弃执行。** 选项 2 和 3 总是保留 worktree。两个调用方都已把目录切换到主仓库根目录——worktree 删除必须在 worktree 外部运行——并使用第 2 步捕获的、在该目录切换之前的 `GIT_DIR`/`GIT_COMMON`/`WORKTREE_PATH` 值。

**如果 `GIT_DIR == GIT_COMMON`：** 普通仓库，没有 worktree 需要清理。完成。

**如果 `WORKTREE_PATH` 位于 `.worktrees/` 或 `worktrees/` 之下：** Superpowers 创建了这个 worktree——清理归我们：

```bash
git worktree remove "$WORKTREE_PATH"
git worktree prune  # Self-healing: clean up any stale registrations
```

**如果删除被拒绝**（`contains modified or untracked files`）：worktree 中持有别处都不存在的文件——未提交的计划、笔记或草稿工作。永远不要自行 `--force`。把利害关系展示给你的人类伙伴并询问：

```bash
git -C "$WORKTREE_PATH" status --porcelain -uall
```

```
Worktree removal refused — these files were never committed:

<file list>

1. Commit them to <branch> before cleanup
2. Move them into <main repo root>
3. Delete them (unrecoverable)

Which?
```

执行选择，然后删除 worktree。

**其他情况：** 宿主环境拥有这个工作区——保持原样。如果你的平台提供工作区退出工具，使用它。

## 快速参考

| 选项 | 合并 | 推送 | 保留 Worktree | 清理分支 |
|--------|-------|------|---------------|----------------|
| 1. 本地合并 | 是 | - | - | 是 |
| 2. 创建 PR | - | 是 | 是 | - |
| 3. 保持原样 | - | - | 是 | - |
| 丢弃（仅限明确请求） | - | - | - | 是（强制） |

## 常见合理化

| 借口 | 现实 |
|--------|---------|
| "本会话早些时候测试通过了" | 在你即将集成的这棵树上运行测试套件。一次全绿只能证明它运行过的那棵树。 |
| "他们显然想合并" | 集成是你的人类伙伴的决定。呈现菜单并等待。 |
| "他们似乎对这个功能做完了——我提议丢弃它" | 菜单按原样是完整的。丢弃只在你的人类伙伴亲口要求时才发生。 |
| "'嗯，删了吧'算作确认" | 只有打出来的 `discard` 一词才授权删除。 |
| "PR 已经提交了，所以 worktree 现在是个累赘" | PR 反馈要在那个 worktree 里修复。它一直保留到工作落地。 |
| "这个别的 worktree 看起来很陈旧——我也一并清理掉" | 只清理 `.worktrees/` 或 `worktrees/` 下的 worktree。其他一切都属于宿主。 |
| "删除被拒绝——`--force` 只是把清理做完" | 被拒绝意味着文件只存在于那个 worktree。`--force` 会永久销毁它们。展示给你的人类伙伴并询问。 |
| "合并结果的失败可能是偶发的" | 合并结果失败会停下一切。在你调查期间，分支和 worktree 保持原位。 |
| "基础分支显然是 main" | 确认分叉点或询问。合并进错误的基础分支，撤销代价很高。 |
| "推送被拒绝了——force-push 会修复它" | 推送被拒绝意味着远端已移动。调查；只有在你的人类伙伴明确要求时才 force-push。 |
