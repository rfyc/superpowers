# Worktree Rototill：检测并让位（Detect-and-Defer）

**日期：** 2026-04-06
**状态：** 草案
**工单：** PRI-974
**取代：** PRI-823（Codex App 兼容性）

## 问题

Superpowers 对 worktree 管理有明确的立场——特定的路径（`.worktrees/<branch>`）、特定的命令（`git worktree add`）、特定的清理（`git worktree remove`）。与此同时，Claude Code、Codex App、Gemini CLI 和 Cursor 都提供了各自的原生 worktree 支持，包括自己的路径、生命周期管理和清理机制。

这造成了三种失败模式：

1. **重复**——在 Claude Code 上，技能做了 `EnterWorktree`/`ExitWorktree` 已经在做的事情
2. **冲突**——在 Codex App 上，技能试图在一个已被管理的 worktree 内部再创建 worktree
3. **幻影状态**——技能创建的 `.worktrees/` 下的 worktree 对 harness 不可见；harness 创建的 `.claude/worktrees/` 下的 worktree 对技能不可见

对于没有原生支持的 harness（Codex CLI、OpenCode、Copilot 独立版），superpowers 填补了真实空白。技能不应消失——它只应在存在原生支持时让位（get out of the way）。

## 目标

1. 当原生 harness worktree 系统存在时让位于它们
2. 继续为缺乏原生支持的 harness 提供 worktree 支持
3. 修复 finishing-a-development-branch 中的三个已知 bug（#940、#999、#238）
4. 将 worktree 创建改为可选项而非强制项（#991）
5. 将硬编码的 `CLAUDE.md` 引用替换为平台中立措辞（#1049）

## 非目标

- 每个 worktree 的环境约定（`.worktree-env.sh`、端口偏移）——第四阶段
- 用于路径强制执行的 PreToolUse hooks——第四阶段
- 多仓库 worktree 文档——第四阶段
- 针对 worktree 的 brainstorming 检查清单变更——第四阶段
- `.superpowers-session.json` 元数据跟踪（PR #997 的想法很有趣，但 v1 不需要）
- 将 hooks 符号链接进 worktree（PR #965 的想法，属于另一个关注点）

## 设计原则

### 检测状态，而非平台

使用 `GIT_DIR != GIT_COMMON` 来判断"我是否已经在 worktree 中？"，而不是通过嗅探环境变量来识别 harness。这是一个稳定的 git 原语（自 2015 年 git 2.5 起），对所有 harness 都通用，且随着新 harness 的出现无需任何维护。

### 声明式意图，规定式兜底

技能描述目标（"确保工作发生在隔离的工作区中"），并在可用时让位于原生工具。只有当 harness 没有原生 worktree 支持时，技能才规定具体的 git 命令作为兜底。Step 1a 排在最前并明确点名原生工具（`EnterWorktree`、`WorktreeCreate`、`/worktree`、`--worktree`）；Step 1b 排在第二，提供 git 兜底。原始规格让 Step 1a 保持抽象（"你了解自己的工具集"），但 TDD 证明：当 Step 1a 过于模糊时，agent 会锚定在 Step 1b 的具体命令上。必须显式命名工具并建立同意授权桥接（consent-authorization bridge），偏好机制才能可靠生效。

### 基于来源的所有权

谁创建了 worktree，谁就负责它的清理。如果是 harness 创建的，superpowers 不去碰它。如果是 superpowers 创建的（通过 git 兜底），superpowers 负责清理。启发式规则：如果 worktree 位于 `.worktrees/` 或 `worktrees/` 下，superpowers 拥有它。其他任何路径（`.claude/worktrees/`、`~/.codex/worktrees/`、`.gemini/worktrees/`，或旧的用户全局 Superpowers 路径）都属于 harness 或用户，不得触碰。

## 设计

### 1. `using-git-worktrees` SKILL.md 重写

该技能在创建之前增加了三个新步骤，并简化了创建流程。

#### 步骤 0：检测现有隔离

```bash
GIT_DIR=$(cd "$(git rev-parse --git-dir)" 2>/dev/null && pwd -P)
GIT_COMMON=$(cd "$(git rev-parse --git-common-dir)" 2>/dev/null && pwd -P)
BRANCH=$(git branch --show-current)
```

三种结果：

| 条件 | 含义 | 动作 |
|-----------|---------|--------|
| `GIT_DIR == GIT_COMMON` | 普通仓库检出 | 继续到步骤 0.5 |
| `GIT_DIR != GIT_COMMON`，具名分支 | 已在某个链接的 worktree 中 | 跳到步骤 3（项目设置）。报告："已在 `<path>` 的隔离工作区中，分支为 `<name>`。" |
| `GIT_DIR != GIT_COMMON`，分离 HEAD | 外部管理的 worktree（例如 Codex App sandbox） | 跳到步骤 3。报告："已在 `<path>` 的隔离工作区中（分离 HEAD，外部管理）。" |

步骤 0 不关心是谁创建了 worktree，也不关心正在运行的是哪个 harness。无论来源如何，worktree 就是 worktree。

**子模块防护：** 在 git 子模块内部，`GIT_DIR != GIT_COMMON` 也为真。在下结论"已在 worktree 中"之前，检查我们是否不在子模块中：

```bash
# 如果这返回一个路径，我们在子模块中，而不是 worktree 中
git rev-parse --show-superproject-working-tree 2>/dev/null
```

如果在子模块中，按 `GIT_DIR == GIT_COMMON` 处理（继续到步骤 0.5）。

#### 步骤 0.5：同意

当步骤 0 未发现现有隔离（`GIT_DIR == GIT_COMMON`）时，在创建之前询问：

> "你希望我设置一个隔离的 worktree 吗？这可以保护你当前分支免受更改影响。（y/n）"

如果同意，继续到步骤 1。如果不同意，就地工作——跳过到步骤 3，不创建 worktree。

当步骤 0 检测到现有隔离时，此步骤完全跳过（没有理由去问关于已存在之物的问题）。

#### 步骤 1a：原生工具（首选）

> 用户已请求隔离工作区（步骤 0 的同意）。检查你可用的工具——你是否有 `EnterWorktree`、`WorktreeCreate`、`/worktree` 命令或 `--worktree` 标志？如果是：用户创建 worktree 的同意即为使用它的授权。立即使用它并跳到步骤 3。

使用原生工具后，跳到步骤 3（项目设置）。

**设计说明——TDD 修订：** 原始规格使用了一个刻意简短、抽象的 Step 1a（"你了解自己的工具集——技能无需点名具体工具"）。TDD 验证推翻了这一点：agent 锚定在 Step 1b 的具体 git 命令上，而忽略了抽象指导（通过率 2/6）。三项变更修复了它（在 GREEN 和 PRESSURE 测试中通过率 50/50）：

1. **显式工具命名**——列出 `EnterWorktree`、`WorktreeCreate`、`/worktree`、`--worktree` 的名字，将决策从解读（"我有原生工具吗？"）转变为事实查找（"`EnterWorktree` 在我的工具列表中吗？"）。在没有这些工具的平台上，agent 只需检查、找不到、然后落到 Step 1b。未观察到误报。
2. **同意桥接**——"用户创建 worktree 的同意即为使用它的授权"直接针对 `EnterWorktree` 的工具级护栏（"仅当用户明确要求时"）。工具描述优先于技能指令（Claude Code #29950），因此技能必须把用户同意框定为工具所要求的授权。
3. **Red Flag 条目**——在 Red Flags 部分点名具体反模式（"当你拥有原生 worktree 工具时使用 `git worktree add`——这是第一大错误"）。

文件拆分（将 Step 1b 放在单独的技能中）经测试被证明没有必要。锚定问题通过 Step 1a 的文本质量解决，而不是通过物理分隔 git 命令。对完整的 240 行技能（所有 git 命令都可见）的对照测试通过了 20/20。

#### 步骤 1b：Git Worktree 兜底

当没有可用的原生工具时，手动创建 worktree。

**目录选择**（优先级顺序）：
1. 检查项目的 agent 指令文件（CLAUDE.md、GEMINI.md、AGENTS.md、.cursorrules 或等价物）中的 worktree 目录偏好。
2. 检查是否存在 `.worktrees/` 或 `worktrees/` 目录——如果找到，使用它。如果两者都存在，`.worktrees/` 优先。
3. 默认为 `.worktrees/`。

没有交互式目录选择提示。旧的用户全局 Superpowers worktree 路径不会被检测或提供；除非用户明确指定其他位置，否则新的手动 worktree 都是项目本地的。

**安全检查**（仅限项目本地目录）：

```bash
git check-ignore -q .worktrees 2>/dev/null
```

如果未被忽略，在继续之前添加到 `.gitignore` 并提交。

**创建：**

```bash
git worktree add "$path" -b "$BRANCH_NAME"
cd "$path"
```

**Hooks 感知：** git worktree 不会继承父仓库的 hooks 目录。通过 1b 创建 worktree 后，如果主仓库存在 hooks 目录，则将其符号链接到 worktree：

```bash
if [ -d "$MAIN_ROOT/.git/hooks" ]; then
    ln -sf "$MAIN_ROOT/.git/hooks" "$path/.git/hooks"
fi
```

这可以防止 pre-commit 检查、linter 和其他 hooks 在工作转移到 worktree 时静默停止。（想法来自 PR #965。）

**Sandbox 兜底：** 如果 `git worktree add` 因权限错误而失败，按受限环境处理。跳过创建，在当前目录工作，继续到步骤 3。

**步骤编号说明：** 当前技能的步骤 1-4 是平铺列表。此重设计使用 0、0.5、1a、1b、3、4。没有步骤 2——它是旧的单一整体"创建隔离工作区"，现已拆分为 1a/1b 结构。实现时应干净地重新编号（例如，0 → "步骤 0：检测"，0.5 → 在步骤 0 流程内，1a/1b → "步骤 1"，3 → "步骤 2"，4 → "步骤 3"），或保留当前编号并加注说明。由实现者决定。

#### 步骤 3-4：项目设置和基线测试（不变）

无论哪条路径创建了工作区（步骤 0 检测到现有、步骤 1a 原生工具、步骤 1b git 兜底，或完全没有 worktree），执行都汇聚到：

- **步骤 3：** 自动检测并运行项目设置（`npm install`、`cargo build`、`pip install`、`go mod download` 等）
- **步骤 4：** 运行测试套件。如果测试失败，报告失败并询问是否继续。

### 2. `finishing-a-development-branch` SKILL.md 重写

完成技能新增环境检测并修复三个 bug。

#### 步骤 1：验证测试（不变）

运行项目的测试套件。如果测试失败，停止。不提供完成选项。

#### 步骤 1.5：检测环境（新增）

重新运行与创建时步骤 0 相同的检测：

```bash
GIT_DIR=$(cd "$(git rev-parse --git-dir)" 2>/dev/null && pwd -P)
GIT_COMMON=$(cd "$(git rev-parse --git-common-dir)" 2>/dev/null && pwd -P)
```

三条路径：

| 状态 | 菜单 | 清理 |
|-------|------|---------|
| `GIT_DIR == GIT_COMMON`（普通仓库） | 标准 4 个选项 | 无需清理的 worktree |
| `GIT_DIR != GIT_COMMON`，具名分支 | 标准 4 个选项 | 基于来源（见步骤 5） |
| `GIT_DIR != GIT_COMMON`，分离 HEAD | 精简菜单：推为新分支 + PR、原样保留、丢弃 | 无合并选项（无法从分离 HEAD 合并） |

#### 步骤 2：确定基础分支（不变）

#### 步骤 3：呈现选项

**普通仓库和具名分支 worktree：**

1. 本地合并回 `<base-branch>`
2. 推送并创建 Pull Request
3. 原样保留分支（我稍后处理）
4. 丢弃此工作

**分离 HEAD：**

1. 推为新分支并创建 Pull Request
2. 原样保留（我稍后处理）
3. 丢弃此工作

#### 步骤 4：执行选择

**选项 1（本地合并）：**

```bash
# 为 CWD 安全获取主仓库根目录（Bug #238 修复）
MAIN_ROOT=$(git -C "$(git rev-parse --git-common-dir)/.." rev-parse --show-toplevel)
cd "$MAIN_ROOT"

# 先合并，在移除任何东西之前验证成功
git checkout <base-branch>
git pull
git merge <feature-branch>
<run tests>

# 仅在合并成功后：移除 worktree，然后删除分支（Bug #999 修复）
git worktree remove "$WORKTREE_PATH"  # 仅当 superpowers 拥有它时
git branch -d <feature-branch>
```

顺序至关重要：合并 → 验证 → 移除 worktree → 删除分支。旧技能在移除 worktree 之前删除分支（这会失败，因为 worktree 仍引用该分支）。先移除 worktree 的朴素修复也是错误的——如果随后合并失败，工作目录已消失，更改丢失。

**选项 2（创建 PR）：**

推送分支，创建 PR。不要清理 worktree——用户需要它进行 PR 迭代。（Bug #940 修复：移除自相矛盾的"然后：清理 worktree"措辞。）

**选项 3（原样保留）：** 不采取任何动作。

**选项 4（丢弃）：** 要求输入"discard"确认。然后移除 worktree（如果 superpowers 拥有它），强制删除分支。

#### 步骤 5：清理（更新）

```
if GIT_DIR == GIT_COMMON:
    # Normal repo, no worktree to clean up
    done

if worktree path is under .worktrees/ or worktrees/:
    # Superpowers created it — we own cleanup
    cd to main repo root       # Bug #238 fix
    git worktree remove <path>

else:
    # Harness created it — hands off
    # If platform provides a workspace-exit tool, use it
    # Otherwise, leave the worktree in place
```

清理只针对选项 1 和 4 运行。选项 2 和 3 始终保留 worktree。（Bug #940 修复。）

**过期 worktree 修剪：** 在任何 `git worktree remove` 之后，运行 `git worktree prune` 作为自愈步骤。Worktree 目录可能被带外删除（例如，由 harness 清理、手动 `rm` 或 `.claude/` 清理），留下导致令人困惑错误的过期注册。一行命令，防止静默腐烂。（想法来自 PR #1072。）

### 3. 集成更新

#### `subagent-driven-development` 和 `executing-plans`

两者目前在集成部分都将 `using-git-worktrees` 列为 REQUIRED。改为：

> `using-git-worktrees`——确保隔离工作区（创建一个或验证已有）

技能本身现在处理同意（步骤 0.5）和检测（步骤 0），因此调用技能无需门控或提示。

#### `writing-plans`

移除过时声明"应在专用 worktree 中运行（由 brainstorming 技能创建）"。Brainstorming 是设计技能，不创建 worktree。Worktree 提示在执行时通过 `using-git-worktrees` 发生。

### 4. 平台中立的指令文件引用

所有 worktree 相关技能中硬编码的 `CLAUDE.md` 实例都被替换为：

> "你项目的 agent 指令文件（CLAUDE.md、GEMINI.md、AGENTS.md、.cursorrules 或等价物）"

这适用于 Step 1b 中的目录偏好检查。

## 捆绑的 Bug 修复

| Bug | 问题 | 修复 | 位置 |
|-----|---------|-----|----------|
| #940 | 选项 2 的措辞说"然后：清理 worktree（步骤 5）"，但快速参考说保留它。步骤 5 说"针对选项 1、2、4"，但 Common Mistakes 说"仅限选项 1 和 4。" | 从选项 2 移除清理。步骤 5 仅适用于选项 1 和 4。 | finishing SKILL.md |
| #999 | 选项 1 在移除 worktree 之前删除分支。`git branch -d` 可能失败，因为 worktree 仍引用该分支。 | 重新排序为：合并 → 验证测试 → 移除 worktree → 删除分支。在任何内容被移除之前，合并必须成功。 | finishing SKILL.md |
| #238 | 如果 CWD 在被移除的 worktree 内部，`git worktree remove` 会静默失败。 | 添加 CWD 防护：在 `git worktree remove` 之前 `cd` 到主仓库根目录。 | finishing SKILL.md |

## 已解决的问题

| 问题 | 解决方式 |
|-------|-----------|
| #940 | 直接修复（Bug #940） |
| #991 | Step 0.5 中的可选同意 |
| #918 | Step 0 检测 + Step 1.5 完成检测 |
| #1009 | 由 Step 1a 解决——agent 使用原生工具（例如 `EnterWorktree`），其创建于 harness 原生的路径。依赖 Step 1a 正常工作；见风险。 |
| #999 | 直接修复（Bug #999） |
| #238 | 直接修复（Bug #238） |
| #1049 | 平台中立的指令文件引用 |
| #279 | 通过检测并让位解决——因为不覆盖原生路径，所以原生路径得到尊重 |
| #574 | **已推迟。** 本规格未触及该 bug 所在的 brainstorming 技能。完整修复（在 brainstorming 的检查清单中添加 worktree 步骤）属于第四阶段。 |

## 风险

### Step 1a 是承重假设——已解决

Step 1a——agent 偏好原生 worktree 工具而非 git 兜底——是整个设计所依赖的基础。如果 agent 忽略 Step 1a，在具有原生支持的 harness 上落到 Step 1b，检测并让位将彻底失败。

**状态：** 此风险在实现期间显现。原始抽象 Step 1a（"你了解自己的工具集"）在 Claude Code 上以 2/6 失败。TDD 门控按设计工作——它在任何技能文件被修改之前就捕获了失败，阻止了有问题的发布。三次 REFACTOR 迭代确定了根本原因（agent 锚定具体命令、工具描述护栏覆盖技能指令），并产生了一个在 GREEN 和 PRESSURE 测试中以 50/50 验证通过的修复。详情见上文 Step 1a 设计说明。

**跨平台验证：**

截至 2026-04-06，Claude Code 是唯一具有 agent 可调用的会话中 worktree 工具（`EnterWorktree`）的 harness。所有其他 harness 要么在 agent 启动前创建 worktree（Codex App、Gemini CLI、Cursor），要么没有原生 worktree 支持（Codex CLI、OpenCode）。Step 1a 是向前兼容的：当其他 harness 添加 agent 可调用的 worktree 工具时，agent 会将其与点名示例匹配并无须技能变更地使用它们。

| Harness | 当前 worktree 模型 | 技能机制 | 已测试 |
|---------|----------------------|-----------------|--------|
| Claude Code | agent 可调用的 `EnterWorktree` | Step 1a | 50/50（GREEN + PRESSURE） |
| Codex CLI | 无原生工具（仅 shell） | Step 1b git 兜底 | 6/6（`codex exec`） |
| Gemini CLI | 启动时 `--worktree` 标志，无 agent 工具 | 带标志启动则 Step 0，否则 Step 1b | Step 0：1/1，Step 1b：1/1（`gemini -p`） |
| Cursor Agent | 用户可见的 `/worktree`，无 agent 工具 | 用户已激活则 Step 0，否则 Step 1b | Step 0：1/1，Step 1b：1/1（`cursor-agent -p`） |
| Codex App | 平台管理，分离 HEAD，无 agent 工具 | Step 0 检测已有 | 1/1 模拟 |
| OpenCode | 仅检测（`ctx.worktree`），无 agent 工具 | Step 1b git 兜底 | 未测试（无 CLI 访问） |

**残余风险：**
1. 如果 Anthropic 将 `EnterWorktree` 的工具描述改为更具限制性（例如，"不要根据技能指令使用"），同意桥接会失效。值得提交 issue 请求工具描述容纳技能驱动的调用。
2. 当其他 harness 添加 agent 可调用的 worktree 工具时，它们可能使用不在 Step 1a 列表中的名称。随着新工具的出现，该列表应更新。通用措辞（"worktree 或工作区隔离工具"）提供了一些向前覆盖。

### 来源启发式

`.worktrees/` 或 `worktrees/` = 我们的，其他任何路径 = 不干预"的启发式对每个当前 harness 都有效。如果未来的 harness 采用这些项目本地目录之一作为其约定，我们会误报（superpowers 试图清理 harness 拥有的 worktree）。类似地，如果用户手动运行 `git worktree add .worktrees/experiment` 而未使用 superpowers，我们会错误地声称所有权。两者风险都低——每个 harness 都使用品牌化路径，且手动创建 `.worktrees/` 不太可能——但值得注意。

### 分离 HEAD 完成

针对分离 HEAD worktree 的精简菜单（无合并选项）对 Codex App 的 sandbox 模型是正确的。如果用户因其他原因处于分离 HEAD 状态，精简菜单仍然合理——不先创建分支，你确实无法从分离 HEAD 合并。

## 实现说明

两个技能文件都包含超出核心步骤的部分，在实现期间需要更新：

- **Frontmatter**（`name`、`description`）：更新以反映检测并让位行为
- **快速参考表**：重写以匹配新的步骤结构和 bug 修复
- **Common Mistakes 部分**：更新或移除引用旧行为的条目（例如，"跳过 CLAUDE.md 检查"现在已错误）
- **Red Flags 部分**：更新以反映新优先级（例如，"当步骤 0 检测到现有隔离时，绝不创建 worktree"）
- **集成部分**：更新技能之间的交叉引用

本规格描述*变更什么*；实现计划将指定这些次要部分的具体编辑。

## 未来工作（不在本规格中）

- **第三阶段剩余：** `$TMPDIR` 目录选项（#666）、缓存和环境继承的设置文档（#299）
- **第四阶段：** 用于路径强制执行的 PreToolUse hooks（#1040）、每个 worktree 的环境约定（#597）、brainstorming 检查清单的 worktree 步骤（#574）、多仓库文档（#710）
