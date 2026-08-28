---
name: using-git-worktrees
description: 当开始需要与当前工作区隔离的功能开发，或在执行实施计划之前使用——通过原生工具或 git worktree 回退方案确保隔离工作区存在
---

# 使用 Git Worktrees

## 概述

确保工作发生在隔离的工作区中。优先使用你所在平台的原生 worktree 工具。仅在没有可用原生工具时，才回退到手动 git worktrees。

**核心原则：** 首先检测是否已存在隔离。然后使用原生工具。然后回退到 git。绝不要与 harness 对抗。

**开始时宣布：** "我正在使用 using-git-worktrees 技能来设置隔离工作区。"

## 第 0 步：检测现有隔离

**在创建任何东西之前，先检查你是否已经处于隔离工作区中。**

```bash
GIT_DIR=$(cd "$(git rev-parse --git-dir)" 2>/dev/null && pwd -P)
GIT_COMMON=$(cd "$(git rev-parse --git-common-dir)" 2>/dev/null && pwd -P)
BRANCH=$(git branch --show-current)
```

**子模块保护：** 在 git 子模块内部，`GIT_DIR != GIT_COMMON` 同样成立。在得出"已在 worktree 中"的结论之前，请先确认你不在子模块中：

```bash
# If this returns a path, you're in a submodule, not a worktree — treat as normal repo
git rev-parse --show-superproject-working-tree 2>/dev/null
```

**如果 `GIT_DIR != GIT_COMMON`（且不是子模块）：** 你已经在链接的 worktree 中。跳到第 2 步（项目设置）。不要创建另一个 worktree。

报告分支状态：
- 在分支上："Already in isolated workspace at `<path>` on branch `<name>`."
- 分离的 HEAD："Already in isolated workspace at `<path>` (detached HEAD, externally managed). Branch creation needed at finish time."

**如果 `GIT_DIR == GIT_COMMON`（或在子模块中）：** 你处于正常的仓库检出状态。

你的指令中，用户是否已经表明了其 worktree 偏好？如果没有，请在创建 worktree 前征求同意：

> "你希望我建立一个隔离的 worktree 吗？它能保护你当前分支免受更改。"

尊重任何已声明的偏好，无需再问。如果用户拒绝同意，就在原地工作并跳到第 2 步。

## 第 1 步：创建隔离工作区

**你有两种机制。请按此顺序尝试。**

### 1a. 原生 Worktree 工具（首选）

用户已要求隔离工作区（第 0 步的同意）。你是否已经有创建 worktree 的方法？它可能是名为 `EnterWorktree`、`WorktreeCreate` 的工具、`/worktree` 命令，或 `--worktree` 标志。如果有，就使用它并跳到第 2 步。

原生工具会自动处理目录位置、分支创建和清理。当你有原生工具时仍使用 `git worktree add` 会创建你的 harness 无法看到或管理的幻影状态。

只有在没有可用的原生 worktree 工具时，才继续第 1b 步。

### 1b. Git Worktree 回退方案

**仅当第 1a 步不适用时才使用此方案** —— 即你没有可用的原生 worktree 工具。使用 git 手动创建 worktree。

#### 目录选择

按以下优先级顺序执行。用户明确表达的偏好始终优先于观察到的文件系统状态。

1. **检查你的指令中是否声明了 worktree 目录偏好。** 如果用户已经指定，直接使用，无需再问。

2. **检查是否已有项目本地的 worktree 目录：**
   ```bash
   ls -d .worktrees 2>/dev/null     # Preferred (hidden)
   ls -d worktrees 2>/dev/null      # Alternative
   ```
   如果找到，就使用它。如果两者都存在，`.worktrees` 优先。

3. **如果没有其他可用指引**，默认使用项目根目录下的 `.worktrees/`。

#### 安全验证（仅限项目本地目录）

**在创建 worktree 前必须验证目录已被忽略：**

```bash
git check-ignore -q .worktrees 2>/dev/null || git check-ignore -q worktrees 2>/dev/null
```

**如果未被忽略：** 将其添加到 .gitignore，提交更改，然后继续。

**为何至关重要：** 防止意外将 worktree 内容提交到仓库。

#### 创建 Worktree

```bash
# Determine path based on chosen location
path="$LOCATION/$BRANCH_NAME"

git worktree add "$path" -b "$BRANCH_NAME"
cd "$path"
```

**沙箱回退方案：** 如果 `git worktree add` 因权限错误（沙箱拒绝）而失败，告诉用户沙箱阻止了 worktree 的创建，你将在当前目录中工作。然后在原地运行设置和基线测试。

## 第 2 步：项目设置

自动检测并运行相应的设置：

```bash
# Node.js
if [ -f package.json ]; then npm install; fi

# Rust
if [ -f Cargo.toml ]; then cargo build; fi

# Python
if [ -f requirements.txt ]; then pip install -r requirements.txt; fi
if [ -f pyproject.toml ]; then poetry install; fi

# Go
if [ -f go.mod ]; then go mod download; fi
```

## 第 3 步：验证干净的基线

运行测试以确保工作区以干净状态开始：

```bash
# Use project-appropriate command
npm test / cargo test / pytest / go test ./...
```

**如果测试失败：** 报告失败情况，询问是继续还是调查。

**如果测试通过：** 报告就绪。

### 报告

```
Worktree ready at <full-path>
Tests passing (<N> tests, 0 failures)
Ready to implement <feature-name>
```

## 快速参考

| 情况 | 操作 |
|-----------|--------|
| 已在链接的 worktree 中 | 跳过创建（第 0 步） |
| 在子模块中 | 视为普通仓库（第 0 步保护） |
| 有可用的原生 worktree 工具 | 使用它（第 1a 步） |
| 没有原生工具 | Git worktree 回退方案（第 1b 步） |
| 存在 `.worktrees/` | 使用它（验证已被忽略） |
| 存在 `worktrees/` | 使用它（验证已被忽略） |
| 两者都存在 | 使用 `.worktrees/` |
| 都不存在 | 检查指令文件，然后默认 `.worktrees/` |
| 目录未被忽略 | 添加到 .gitignore + 提交 |
| 创建时权限错误 | 沙箱回退方案，原地工作 |
| 基线期间测试失败 | 报告失败情况 + 询问 |
| 没有 package.json/Cargo.toml | 跳过依赖安装 |

## 常见的自我合理化

| 借口 | 现实 |
|--------|---------|
| "我显然不在 worktree 中——无需检查" | 运行第 0 步。harness 创建的隔离和子模块都能骗过肉眼观察；检测命令可以确定结论。 |
| "`git worktree add` 比寻找原生工具更快" | 原生工具（如 `EnterWorktree`）负责位置、分支和清理。绕过它是第一大错误——它会创建你的 harness 无法看到或管理的幻影状态。 |
| "worktree 目录肯定已经被忽略了" | 运行 `git check-ignore`。未被忽略的 worktree 目录会把整个工作树提交到仓库。 |
| "任何目录名都行" | 明确的指令胜过已存在的项目本地目录，而项目本地目录又胜过 `.worktrees/` 默认值。 |
| "工作区是全新的——基线测试可以等等再说" | 不干净的基线会让之后每一个失败都变得含糊不清。现在就运行测试；在失败的情况下继续推进是你要交给人类伙伴的决定。 |
