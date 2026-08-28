---
name: requesting-code-review
description: 当完成任务、实现重要功能，或在合并之前用于验证工作是否满足需求时使用
---

# 请求代码审查

派遣一个代码审查子代理（code reviewer subagent）来在问题扩散之前捕获它们。审查者会拿到经过精确构造的评估上下文——绝不是你的会话历史。

**核心原则：** 及早审查，经常审查。

## 何时请求审查

**必须：**
- 在子代理驱动的开发中，每完成一个任务之后
- 完成主要特性之后
- 合并到 main 之前

**可选但有价值：**
- 卡住时（新的视角）
- 重构之前（基线检查）
- 修复复杂 Bug 之后

## 如何请求

**1. 获取 git SHA：**
```bash
BASE_SHA=$(git rev-parse HEAD~1)  # or origin/main
HEAD_SHA=$(git rev-parse HEAD)
```

**2. 派遣代码审查子代理：**

派遣一个 `general-purpose` 子代理，填写 [code-reviewer.md](code-reviewer.md) 中的模板

**占位符：**
- `{DESCRIPTION}` - 你所构建内容的简要总结
- `{PLAN_OR_REQUIREMENTS}` - 它应该做什么
- `{BASE_SHA}` - 起始提交
- `{HEAD_SHA}` - 结束提交

**3. 根据反馈采取行动：**
- 立即修复 Critical（严重）问题
- 在继续之前修复 Important（重要）问题
- 将 Minor（次要）问题记下稍后处理
- 如果审查者错了，提出异议（并附上理由）

## 示例

```
[刚刚完成任务 2：添加验证函数]

你：让我在继续之前请求代码审查。

BASE_SHA=$(git log --oneline | grep "Task 1" | head -1 | awk '{print $1}')
HEAD_SHA=$(git rev-parse HEAD)

[派遣代码审查子代理]
  DESCRIPTION: Added verifyIndex() and repairIndex() with 4 issue types
  PLAN_OR_REQUIREMENTS: Task 2 from docs/superpowers/plans/deployment-plan.md
  BASE_SHA: a7981ec
  HEAD_SHA: 3df7661

[子代理返回]：
  优点：架构清晰，测试真实
  问题：
    重要：缺少进度指示器
    次要：报告间隔的魔法数字 (100)
  评估：可以继续

你：[修复进度指示器]
[继续到任务 3]
```

## 常见的合理化辩解

| 借口 | 现实 |
|--------|---------|
| "我自己审查 diff 就行，不必派遣审查者" | 你是协调者——在对话中审查 diff 会消耗你驱动工作所需的上下文窗口。派遣一个审查子代理：diff 和评估都放在它的上下文里，只有发现结果会回到你这里。 |
| "审查者需要我的整个会话历史才能理解这个变更" | 给它精确构造的上下文，绝不是你的会话历史。这样能让审查者专注于工作成果，而不是你的思考过程。 |

## 危险信号

**永远不要：**
- 因为"它很简单"就跳过审查
- 忽略 Critical 问题
- 带着未修复的 Important 问题继续
- 与合理的技术反馈争论

**如果审查者错了：**
- 用技术推理提出异议
- 展示证明它有效的代码/测试
- 请求澄清

参见模板：[code-reviewer.md](code-reviewer.md)
