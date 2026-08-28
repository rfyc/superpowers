# 文档审查系统设计

## 概述

向 superpowers 工作流新增两个审查阶段：

1. **Spec 文档审查** —— 在 brainstorming 之后、writing-plans 之前
2. **计划文档审查** —— 在 writing-plans 之后、实现之前

两者都遵循实现审查所使用的迭代循环模式。

## Spec 文档审查器

**目的：** 验证 spec 是否完整、一致，并已为实施计划做好准备。

**位置：** `skills/brainstorming/spec-document-reviewer-prompt.md`

**它检查的内容：**

| 类别 | 检查什么 |
|----------|------------------|
| 完整性 | TODOs、占位符、"TBD"、未完成章节 |
| 覆盖面 | 缺失的错误处理、边界情况、集成点 |
| 一致性 | 内部矛盾、相互冲突的需求 |
| 清晰度 | 有歧义的需求 |
| YAGNI | 未要求的功能、过度设计 |

**输出格式：**
```
## Spec Review

**Status:** Approved | Issues Found

**Issues (if any):**
- [Section X]: [issue] - [why it matters]

**Recommendations (advisory):**
- [suggestions that don't block approval]
```

**审查循环：** 发现问题 -> brainstorming 代理修复 -> 复审 -> 重复直至获批。

**派发机制：** 使用 `subagent_type: general-purpose` 的 Task 工具。审查器提示模板提供完整提示。brainstorming 技能的控制器负责派发审查器。

## 计划文档审查器

**目的：** 验证计划是否完整、与 spec 相符，并具有恰当的任务分解。

**位置：** `skills/writing-plans/plan-document-reviewer-prompt.md`

**它检查的内容：**

| 类别 | 检查什么 |
|----------|------------------|
| 完整性 | TODOs、占位符、未完成任务 |
| Spec 一致性 | 计划覆盖 spec 需求，无范围蔓延 |
| 任务分解 | 任务原子化、边界清晰 |
| 任务语法 | 任务与步骤上的复选框语法 |
| 块大小 | 每个块不超过 1000 行 |

**块定义：** 块是计划文档中任务的逻辑分组，以 `## Chunk N: <name>` 标题分隔。writing-plans 技能基于逻辑阶段（例如"基础"、"核心功能"、"集成"）创建这些边界。每个块应足够独立，以便单独审查。

**Spec 一致性验证：** 审查器同时接收：
1. 计划文档（或当前块）
2. 供参考的 spec 文档路径

审查器阅读两者并比较需求覆盖面。

**输出格式：** 与 spec 审查器相同，但限定在当前块范围内。

**审查流程（逐块进行）：**
1. Writing-plans 创建块 N
2. 控制器携带块 N 内容与 spec 路径派发 plan-document-reviewer
3. 审查器阅读块与 spec，返回裁定
4. 如有问题：writing-plans 代理修复块 N，跳转到步骤 2
5. 如获批：继续块 N+1
6. 重复直至所有块获批

**派发机制：** 与 spec 审查器相同——`subagent_type: general-purpose` 的 Task 工具。

## 更新后的工作流

```
brainstorming -> spec -> SPEC REVIEW LOOP -> writing-plans -> plan -> PLAN REVIEW LOOP -> implementation
```

**Spec 审查循环：**
1. Spec 完成
2. 派发审查器
3. 如有问题：修复 -> 跳转到步骤 2
4. 如获批：继续

**计划审查循环：**
1. 块 N 完成
2. 为块 N 派发审查器
3. 如有问题：修复 -> 跳转到步骤 2
4. 如获批：下一个块或进入实现

## Markdown 任务语法

任务和步骤使用复选框语法：

```markdown
- [ ] ### Task 1: Name

- [ ] **Step 1:** Description
  - File: path
  - Command: cmd
```

## 错误处理

**审查循环终止：**
- 无硬性迭代限制——循环持续直到审查器批准
- 如果循环超过 5 次迭代，控制器应将此情况上报给人类以获取指导
- 人类可以选择：继续迭代、在已知问题下批准、或中止

**分歧处理：**
- 审查器是建议性的——它们标记问题但不阻塞
- 如果代理认为审查器反馈不正确，应在修复中说明原因
- 如果同一问题在 3 次迭代后仍有分歧，上报给人类

**审查器输出格式异常：**
- 控制器应验证审查器输出具有必需字段（Status、适用时的 Issues）
- 如果格式异常，带关于预期格式的说明重新派发审查器
- 2 次格式异常响应后，上报给人类

## 需要变更的文件

**新文件：**
- `skills/brainstorming/spec-document-reviewer-prompt.md`
- `skills/writing-plans/plan-document-reviewer-prompt.md`

**修改的文件：**
- `skills/brainstorming/SKILL.md` - 在 spec 编写完成后加入审查循环
- `skills/writing-plans/SKILL.md` - 加入逐块审查循环，更新任务语法示例
