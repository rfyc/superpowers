---
name: writing-plans
description: 当你拥有一份多步骤任务的规格或需求、在接触代码之前使用
---

# 编写计划

## 概述

编写全面的实现计划，假设工程师对我们代码库零上下文、品味也存疑。把他们在每个任务中需要知道的一切写清楚：要动哪些文件、代码、测试、他们可能需要查阅的文档、如何测试。把整个计划以一口大小的任务交给他。遵循 DRY、YAGNI、TDD。频繁提交。

假设他们是有经验的开发者，但对我们的工具集或问题域几乎一无所知。假设他们对好的测试设计理解不深。

**开始时宣布：** "我正在使用 writing-plans 技能来创建实现计划。"

**上下文：** 如果在隔离的 worktree 中工作，它应当在执行时通过 `superpowers:using-git-worktrees` 技能创建。

**计划保存到：** `docs/superpowers/plans/YYYY-MM-DD-<feature-name>.md`
- （用户对计划存放位置的偏好优先于这个默认值）

## 范围检查

如果 spec 覆盖多个相互独立的子系统，它应当在头脑风暴阶段就被拆成子项目 spec。如果没有，建议把它拆成多个独立计划——每个子系统一个。每个计划都应能独立产出可运行、可测试的软件。

## 文件结构

在定义任务之前，先规划哪些文件会被创建或修改、每个文件负责什么。拆解决策在这里被锁定。

- 设计边界清晰、接口定义良好的单元。每个文件应有一个清晰的职责。
- 你能更好地推理一次能装进上下文的代码，文件聚焦时你的编辑也更可靠。偏好更小、更聚焦的文件，而不是做得太多的大文件。
- 一起变化的文件应该待在同一个地方。按职责拆分，而不是按技术分层。
- 在既有代码库中，遵循既定模式。如果代码库用大文件，不要单方面重构——但如果你正在修改的文件已经臃肿，在计划中包含一次拆分是合理的。

这个结构为任务拆解提供依据。每个任务都应产生自洽、可独立理解的变更。

## 任务规模合理化

任务是承载自己的测试循环、且值得一次全新审阅者门槛的最小单元。划定任务边界时：把搭建、配置、脚手架和文档步骤并入需要它们的那个任务的交付物里；只在审阅者可以合理地批准某个任务的同时拒绝其相邻任务的地方进行拆分。每个任务都以一个可独立测试的交付物结束。

## 一口大小的任务粒度

**每个步骤是一个动作（2-5 分钟）：**
- "写一个失败的测试" —— 步骤
- "运行它，确保它失败" —— 步骤
- "实现让测试通过的最简代码" —— 步骤
- "运行测试，确保它们通过" —— 步骤
- "提交" —— 步骤

## 计划文档头部

**每个计划必须以这个头部开头：**

```markdown
# [Feature Name] Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** [One sentence describing what this builds]

**Architecture:** [2-3 sentences about approach]

**Tech Stack:** [Key technologies/libraries]

**Spec:** [path to the spec/design doc this plan implements — the plan
argues from the spec, so the spec travels with it; executors read both]

## Global Constraints

[The spec's project-wide requirements — version floors, dependency limits,
naming and copy rules, platform requirements — one line each, with exact
values copied verbatim from the spec. Every task's requirements implicitly
include this section.]

---
```

## 任务结构

````markdown
### Task N: [Component Name]

**Files:**
- Create: `exact/path/to/file.py`
- Modify: `exact/path/to/existing.py:123-145`
- Test: `tests/exact/path/to/test.py`

**Interfaces:**
- Consumes: [what this task uses from earlier tasks — exact signatures]
- Produces: [what later tasks rely on — exact function names, parameter
  and return types. A task's implementer sees only their own task; this
  block is how they learn the names and types neighboring tasks use.]

- [ ] **Step 1: Write the failing test**

```python
def test_specific_behavior():
    result = function(input)
    assert result == expected
```

- [ ] **Step 2: Run test to verify it fails**

Run: `pytest tests/path/test.py::test_name -v`
Expected: FAIL with "function not defined"

- [ ] **Step 3: Write minimal implementation**

```python
def function(input):
    return expected
```

- [ ] **Step 4: Run test to verify it passes**

Run: `pytest tests/path/test.py::test_name -v`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add tests/path/test.py src/path/file.py
git commit -m "feat: add specific feature"
```
````

## 禁止占位符

每个步骤必须包含工程师需要的实际内容。这些是**计划失败**——永远不要写它们：
- "TBD"、"TODO"、"稍后实现"、"补充细节"
- "添加适当的错误处理" / "添加校验" / "处理边界情况"
- "为上述内容写测试"（没有实际测试代码）
- "与任务 N 类似"（重复代码——工程师可能会乱序阅读任务）
- 只描述做什么、不展示怎么做的步骤（代码步骤必须有代码块）
- 引用任何任务中都没有定义的类型、函数或方法

## 自审

写完完整计划后，用全新的眼光看 spec，并按它核对计划。这是你自己运行的清单——不是派出子代理。

**1. Spec 覆盖：** 快速浏览 spec 中的每个小节/需求。你能指出实现它的任务吗？列出任何缺口。

**2. 占位符扫描：** 在你的计划中搜索红旗——上面"禁止占位符"一节中的任何模式。修掉它们。

**3. 类型一致性：** 你在后面任务中使用的类型、方法签名和属性名与你在前面任务中定义的一致吗？任务 3 里叫 `clearLayers()` 而任务 7 里叫 `clearFullLayers()` 的函数就是一个 bug。

如果发现问题，就地修复。无需重新审阅——修完继续前进。如果你发现某个 spec 需求没有对应任务，就添加该任务。

## 执行交接

保存计划后，提供执行方式选择：

**"计划已完成并保存到 `docs/superpowers/plans/<filename>.md`。有两种执行方式：**

**1. 子代理驱动（推荐）** ——我为每个任务派出一个全新子代理，在任务之间进行审阅，快速迭代

**2. 内联执行** ——在本会话中使用 executing-plans 执行任务，带检查点的批量执行

**选哪种？"**

**如果选择子代理驱动：**
- **必需子技能：** 使用 superpowers:subagent-driven-development
- 每个任务一个全新子代理 + 两阶段审阅

**如果选择内联执行：**
- **必需子技能：** 使用 superpowers:executing-plans
- 带检查点的批量执行以供审阅
