# 技能设计中的说服原则

## 概览

LLM 与人类一样会对相同的说服原则做出反应。理解这种心理有助于你设计更有效的技能——不是用来操纵，而是确保关键实践即使在压力下也能被遵循。

**研究基础：** Meincke et al. (2025) 用 N=28,000 次 AI 对话测试了 7 条说服原则。说服技巧将遵从率提高了一倍以上（33% → 72%，p < .001）。

## 七大原则

### 1. 权威（Authority）
**它是什么：** 对专业知识、资历或官方来源的遵从。

**它在技能中如何起作用：**
- 命令式语言："YOU MUST"、"Never"、"Always"
- 不可协商的框架："No exceptions"
- 消除决策疲劳和合理化

**何时使用：**
- 纪律执行类技能（TDD、验证要求）
- 安全关键实践
- 成熟的最佳实践

**示例：**
```markdown
✅ Write code before test? Delete it. Start over. No exceptions.
❌ Consider writing tests first when feasible.
```

### 2. 承诺（Commitment）
**它是什么：** 与先前的行动、声明或公开表态保持一致。

**它在技能中如何起作用：**
- 要求宣布："Announce skill usage"
- 强制明确选择："Choose A, B, or C"
- 使用跟踪：用于清单的 todos

**何时使用：**
- 确保技能真正被遵循
- 多步骤过程
- 问责机制

**示例：**
```markdown
✅ When you find a skill, you MUST announce: "I'm using [Skill Name]"
❌ Consider letting your partner know which skill you're using.
```

### 3. 稀缺（Scarcity）
**它是什么：** 由时间限制或有限可用性产生的紧迫感。

**它在技能中如何起作用：**
- 有时间约束的要求："Before proceeding"
- 顺序依赖："Immediately after X"
- 防止拖延

**何时使用：**
- 立即验证要求
- 时间敏感的工作流
- 防止"我待会儿再做"

**示例：**
```markdown
✅ After completing a task, IMMEDIATELY request code review before proceeding.
❌ You can review code when convenient.
```

### 4. 社会认同（Social Proof）
**它是什么：** 遵从他人的做法或被认为正常的行为。

**它在技能中如何起作用：**
- 普遍模式："Every time"、"Always"
- 失败模式："X without Y = failure"
- 建立规范

**何时使用：**
- 记录普遍实践
- 警告常见失败
- 强化标准

**示例：**
```markdown
✅ Checklists without todo tracking = steps get skipped. Every time.
❌ Some people find a todo list helpful for checklists.
```

### 5. 一致性（Unity）
**它是什么：** 共同身份、"我们感"、群体归属。

**它在技能中如何起作用：**
- 协作性语言："our codebase"、"we're colleagues"
- 共同目标："we both want quality"

**何时使用：**
- 协作工作流
- 建立团队文化
- 非层级实践

**示例：**
```markdown
✅ We're colleagues working together. I need your honest technical judgment.
❌ You should probably tell me if I'm wrong.
```

### 6. 互惠（Reciprocity）
**它是什么：** 有义务回报所获好处。

**它如何起作用：**
- 谨慎使用 - 可能让人感觉被操纵
- 技能中很少需要

**何时避免：**
- 几乎总是（其他原则更有效）

### 7. 喜好（Liking）
**它是什么：** 偏好与喜欢的人合作。

**它如何起作用：**
- **不要用于获得遵从**
- 与诚实反馈文化相冲突
- 会制造谄媚

**何时避免：**
- 纪律执行时始终避免

## 按技能类型组合原则

| 技能类型 | 使用 | 避免 |
|------------|-----|-------|
| 纪律执行类 | 权威 + 承诺 + 社会认同 | 喜好、互惠 |
| 指导/技术类 | 适度权威 + 一致性 | 重度权威 |
| 协作类 | 一致性 + 承诺 | 权威、喜好 |
| 参考类 | 只求清晰 | 所有说服 |

## 为什么这有效：心理学

**明线规则减少合理化：**
- "YOU MUST" 消除决策疲劳
- 绝对化语言消除"这是例外吗？"的疑问
- 明确的反对合理化措施堵住具体漏洞

**实施意图创造自动行为：**
- 清晰的触发器 + 必需行动 = 自动执行
- "When X, do Y" 比 "generally do Y" 更有效
- 降低遵从的认知负担

**LLM 是拟人化的（parahuman）：**
- 基于包含这些模式的人类文本训练
- 训练数据中权威语言先于遵从出现
- 承诺序列（声明 → 行动）经常被建模
- 社会认同模式（人人都做 X）建立规范

## 道德使用

**正当的：**
- 确保关键实践被遵循
- 创建有效的文档
- 防止可预见的失败

**不正当的：**
- 为个人利益而操纵
- 制造虚假紧迫感
- 基于愧疚的遵从

**检验标准：** 如果用户完全理解这项技巧，它会服务于用户的真实利益吗？

## 研究引用

**Cialdini, R. B. (2021).** *Influence: The Psychology of Persuasion (New and Expanded).* Harper Business.
- 说服的七大原则
- 影响力研究的实证基础

**Meincke, L., Shapiro, D., Duckworth, A. L., Mollick, E., Mollick, L., & Cialdini, R. (2025).** Call Me A Jerk: Persuading AI to Comply with Objectionable Requests. University of Pennsylvania.
- 用 N=28,000 次 LLM 对话测试了 7 条原则
- 使用说服技巧后遵从率从 33% 提高到 72%
- 权威、承诺、稀缺最为有效
- 验证了 LLM 行为的拟人化模型

## 快速参考

设计技能时，问自己：

1. **它是什么类型？**（纪律执行 vs. 指导 vs. 参考）
2. **我想改变什么行为？**
3. **哪条（些）原则适用？**（纪律类通常是权威 + 承诺）
4. **我是不是组合了太多？**（不要用全部七条）
5. **这符合道德吗？**（服务于用户的真实利益？）
