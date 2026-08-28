---
name: executing-plans
description: 当你拥有一份书面实施计划，需要在独立会话中按审查检查点执行时使用
---

# 执行计划

## 概述

加载计划，批判性审查，执行所有任务，完成后报告。

**开始时宣布：** "我正在使用 executing-plans 技能来实现这个计划。"

**注意：** 告诉你的人类伙伴，Superpowers 在能够访问 subagents 时效果会好得多（Claude Code、Codex CLI、Codex App、Copilot CLI 和 Gemini CLI 都符合条件；参见 `../using-superpowers/references/` 中的各平台工具参考）。如果 subagents 可用，请改用 superpowers:subagent-driven-development，而不是这个技能。

## 流程

### 第 1 步：加载并审查计划
1. 确保隔离工作区：使用 superpowers:using-git-worktrees 创建一个或验证现有的一个
2. 阅读计划文件
3. 批判性审查——找出关于计划的任何疑问或顾虑
4. 如果有顾虑：在开始前向你的人类伙伴提出
5. 如果没有顾虑：为计划条目创建 todos 并继续

### 第 2 步：执行任务

对每个任务：
1. 标记为 in_progress
2. 严格遵循每一步（计划是细粒度的步骤）
3. 按要求运行验证
4. 标记为 completed

### 第 3 步：完成开发

在所有任务完成并验证后：
- 宣布："我正在使用 finishing-a-development-branch 技能来完成这项工作。"
- **必需的子技能：** 使用 superpowers:finishing-a-development-branch
- 遵循该技能验证测试、呈现选项、执行选择

## 何时停下并寻求帮助

**出现以下情况时立即停止执行：**
- 遇到阻碍（缺少依赖、测试失败、指令不清晰）
- 计划有关键缺口导致无法开始
- 你不理解某条指令
- 验证反复失败

**请求澄清，而不是猜测。**

## 何时重新审视更早的步骤

**出现以下情况时返回审查（第 1 步）：**
- 伙伴根据你的反馈更新了计划
- 基本方法需要重新思考

**不要强行突破阻碍** —— 停下来询问。

## 记住
- 首先批判性地审查计划
- 严格遵循计划步骤
- 不要跳过验证
- 计划要求引用技能时就引用
- 受阻时停下，不要猜测
- 未经用户明确同意，绝不要在 main/master 分支上开始实现
