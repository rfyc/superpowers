---
name: verification-before-completion
description: 当即将声称工作已完成、已修复或已通过，在提交或创建 PR 之前使用——要求在做出任何成功声明前先运行验证命令并确认输出；始终先有证据再有断言
---

# 完成前的验证

## 概述

**核心原则：** 先有证据，再下断言，始终如此。

**违反这条规则的字面意思，就是违反它的精神。**

## 铁律

```
NO COMPLETION CLAIMS WITHOUT FRESH VERIFICATION EVIDENCE
```

如果你没有在本消息中运行过验证命令，你就不能声称它通过。

## 守门函数

```
BEFORE claiming any status or expressing satisfaction:

1. IDENTIFY: What command proves this claim?
2. RUN: Execute the FULL command (fresh, complete)
3. READ: Full output, check exit code, count failures
4. VERIFY: Does output confirm the claim?
   - If NO: State actual status with evidence
   - If YES: State claim WITH evidence
5. ONLY THEN: Make the claim

Skip any step = lying, not verifying
```

## 常见失败

| 声称 | 需要 | 不充分 |
|-------|----------|----------------|
| 测试通过 | 测试命令输出：0 个失败 | 之前的运行结果、"应该能通过" |
| Linter 干净 | Linter 输出：0 个错误 | 部分检查、外推 |
| 构建成功 | 构建命令：退出码 0 | Linter 通过、日志看起来正常 |
| Bug 已修复 | 测试原始症状：通过 | 改了代码、假设已修复 |
| 回归测试有效 | 红绿循环已验证 | 测试通过了一次 |
| Agent 已完成 | VCS diff 显示变更 | Agent 报告"成功" |
| 需求已满足 | 逐行核对清单 | 测试通过 |

## 危险信号 - 立即停止

- 使用"应该"、"大概"、"似乎"
- 在验证之前表达满意（"太棒了！"、"完美！"、"完成了！"等）
- 在没有验证的情况下准备 commit/push/PR
- 信任 agent 的成功报告
- 依赖部分验证
- 心想"就这一次"
- 疲惫，想尽快结束工作
- **任何暗示成功却未运行验证的措辞**

## 合理化辩解防范

| 借口 | 现实 |
|--------|---------|
| "现在应该能用了" | 运行验证 |
| "我有信心" | 信心 ≠ 证据 |
| "就这一次" | 没有例外 |
| "Linter 通过了" | Linter ≠ 编译器 |
| "Agent 说成功了" | 独立验证 |
| "我太累了" | 疲惫 ≠ 借口 |
| "部分检查就够了" | 部分证明不了任何东西 |
| "换了种说法所以规则不适用" | 精神高于字面 |

## 关键模式

**测试：**
```
✅ [运行测试命令] [看到：34/34 通过] "所有测试通过"
❌ "现在应该能通过" / "看起来正确"
```

**回归测试（TDD 红绿）：**
```
✅ 编写 → 运行（通过）→ 还原修复 → 运行（必须失败）→ 恢复 → 运行（通过）
❌ "我已经写了回归测试"（未经红绿验证）
```

**构建：**
```
✅ [运行构建] [看到：退出码 0] "构建通过"
❌ "Linter 通过了"（linter 不检查编译）
```

**需求：**
```
✅ 重读计划 → 创建清单 → 逐项验证 → 报告缺口或完成
❌ "测试通过，阶段完成"
```

**Agent 委派：**
```
✅ Agent 报告成功 → 检查 VCS diff → 验证变更 → 报告实际状态
❌ 信任 agent 的报告
```

## 何时应用

**在以下情况之前始终应用：**
- 任何形式的成功/完成声称
- 任何满意的表达
- 任何关于工作状态的正面陈述
- 提交、创建 PR、任务完成
- 进入下一个任务
- 委派给 agent

**规则适用于：**
- 确切的措辞
- 改写和同义词
- 对成功的暗示
- 任何暗示完成/正确的沟通
