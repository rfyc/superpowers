# 基于用户反馈的技能改进

**日期：** 2025-11-28
**状态：** 草稿
**来源：** 在真实开发场景中使用 superpowers 的两个 Claude 实例

---

## 执行摘要

两个 Claude 实例提供了来自实际开发会话的详细反馈。他们的反馈揭示了当前技能中的**系统性漏洞**，这些漏洞使得本可避免的 Bug 在遵循技能的情况下仍然被发布。

**关键洞察：** 这些是问题报告，而不仅仅是解决方案提案。问题是真实的；解决方案需要仔细评估。

**关键主题：**
1. **验证缺口** - 我们验证操作成功，但没有验证它们是否达成了预期结果
2. **进程卫生** - 后台进程不断积累并在各 subagent 之间相互干扰
3. **上下文优化** - subagent 获得太多无关信息
4. **缺少自我反思** - 没有要求在手交接前对自己的工作进行批判的提示
5. **Mock 安全性** - Mock 可能在无人察觉的情况下偏离接口
6. **技能激活** - 技能存在但并未被读取/使用

---

## 已识别的问题

### 问题 1：配置变更验证缺口

**发生了什么：**
- Subagent 测试了 "OpenAI 集成"
- 设置了 `OPENAI_API_KEY` 环境变量
- 得到了 200 状态码响应
- 报告 "OpenAI 集成正常工作"
- **但是**响应中包含 `"model": "claude-sonnet-4-20250514"`——实际上使用的是 Anthropic

**根本原因：**
`verification-before-completion` 检查操作是否成功，但不检查结果是否反映了预期的配置变更。

**影响：** 高——集成测试产生虚假的信心，Bug 流入生产环境

**示例失败模式：**
- 切换 LLM 提供商 → 验证状态码 200 但不检查模型名称
- 启用功能开关 → 验证无错误但不检查功能是否生效
- 更改环境 → 验证部署成功但不检查环境变量

---

### 问题 2：后台进程积累

**发生了什么：**
- 会话期间调度了多个 subagent
- 每个 subagent 都启动了后台服务器进程
- 进程不断积累（4 个以上服务器在运行）
- 过时进程仍占用端口
- 之后的 E2E 测试命中了配置错误的过时服务器
- 产生了令人困惑/错误的测试结果

**根本原因：**
Subagent 是无状态的——不知道之前 subagent 的进程。没有清理协议。

**影响：** 中-高——测试命中错误的服务器，产生误报/误漏，调试混乱

---

### 问题 3：Subagent 提示中的上下文膨胀

**发生了什么：**
- 标准做法：把完整的计划文件给 subagent 阅读
- 实验做法：只给出任务 + 模式 + 文件 + 验证命令
- 结果：更快、更专注，单次尝试完成的概率更高

**根本原因：**
Subagent 把 token 和注意力浪费在无关的计划章节上。

**影响：** 中——执行变慢，失败尝试更多

**有效的方法：**
```
You are adding a single E2E test to packnplay's test suite.

**Your task:** Add `TestE2E_FeaturePrivilegedMode` to `pkg/runner/e2e_test.go`

**What to test:** A local devcontainer feature that requests `"privileged": true`
in its metadata should result in the container running with `--privileged` flag.

**Follow the exact pattern of TestE2E_FeatureOptionValidation** (at the end of the file)

**After writing, run:** `go test -v ./pkg/runner -run TestE2E_FeaturePrivilegedMode -timeout 5m`
```

---

### 问题 4：交接前没有自我反思

**发生了什么：**
- 添加了自我反思提示："用全新的眼光审视你的工作——哪些地方可以做得更好？"
- 任务 5 的实现者发现失败的测试是由实现 Bug 而非测试 Bug 导致的
- 追溯到第 99 行：`strings.Join(metadata.Entrypoint, " ")` 生成了无效的 Docker 语法
- 如果没有自我反思，实现者只会报告 "测试失败" 而不给出根本原因

**根本原因：**
实现者在报告完成前，不会自然地退后一步批判自己的作品。

**影响：** 中——实现者本可自己抓住的 Bug 被交给了审查者

---

### 问题 5：Mock 与接口漂移

**发生了什么：**
```typescript
// Interface defines close()
interface PlatformAdapter {
  close(): Promise<void>;
}

// Code (BUGGY) calls cleanup()
await adapter.cleanup();

// Mock (MATCHES BUG) defines cleanup()
vi.mock('web-adapter', () => ({
  WebAdapter: vi.fn().mockImplementation(() => ({
    cleanup: vi.fn().mockResolvedValue(undefined),  // Wrong!
  })),
}));
```
- 测试通过了
- 运行时崩溃："adapter.cleanup is not a function"

**根本原因：**
Mock 是根据有 Bug 的代码所调用的方法派生的，而不是根据接口定义派生的。TypeScript 无法捕获方法名错误的内联 mock。

**影响：** 高——测试给出虚假信心，运行时崩溃

**为什么 testing-anti-patterns 没有阻止这个问题：**
该技能涵盖了测试 mock 行为和不求甚解的 mock，但没有涵盖"从接口而非实现派生 mock"这一具体模式。

---

### 问题 6：代码审查者的文件访问

**发生了什么：**
- 派发了代码审查 subagent
- 找不到测试文件："该文件在仓库中似乎不存在"
- 文件实际存在
- 审查者不知道要先显式读取它

**根本原因：**
审查者提示中没有包含显式的文件读取指令。

**影响：** 低-中——审查失败或不完整

---

### 问题 7：修复工作流的延迟

**发生了什么：**
- 实现者在自我反思时发现了 Bug
- 实现者知道如何修复
- 当前工作流：报告 → 我派遣修复者 → 修复者修复 → 我验证
- 多余的往返增加了延迟，却没有增加价值

**根本原因：**
当实现者已经完成诊断时，实现者与修复者角色之间的僵化分离。

**影响：** 低——延迟，但没有正确性问题

---

### 问题 8：技能没有被读取

**发生了什么：**
- `testing-anti-patterns` 技能存在
- 人类和 subagent 在编写测试前都没有读取它
- 本可预防一些问题（虽然不是全部——见问题 5）

**根本原因：**
没有强制要求 subagent 读取相关技能。没有提示包含技能阅读要求。

**影响：** 中——如果不使用，技能投入就被浪费了

---

## 建议的改进

### 1. verification-before-completion：添加配置变更验证

**添加新章节：**

```markdown
## Verifying Configuration Changes

When testing changes to configuration, providers, feature flags, or environment:

**Don't just verify the operation succeeded. Verify the output reflects the intended change.**

### Common Failure Pattern

Operation succeeds because *some* valid config exists, but it's not the config you intended to test.

### Examples

| Change | Insufficient | Required |
|--------|-------------|----------|
| Switch LLM provider | Status 200 | Response contains expected model name |
| Enable feature flag | No errors | Feature behavior actually active |
| Change environment | Deploy succeeds | Logs/vars reference new environment |
| Set credentials | Auth succeeds | Authenticated user/context is correct |

### Gate Function

```
BEFORE claiming configuration change works:

1. IDENTIFY: What should be DIFFERENT after this change?
2. LOCATE: Where is that difference observable?
   - Response field (model name, user ID)
   - Log line (environment, provider)
   - Behavior (feature active/inactive)
3. RUN: Command that shows the observable difference
4. VERIFY: Output contains expected difference
5. ONLY THEN: Claim configuration change works

Red flags:
  - "Request succeeded" without checking content
  - Checking status code but not response body
  - Verifying no errors but not positive confirmation
```

**为什么这样有效：**
强制验证的是**意图**，而不仅仅是操作成功。

---

### 2. subagent-driven-development：为 E2E 测试添加进程卫生

**添加新章节：**

```markdown
## Process Hygiene for E2E Tests

When dispatching subagents that start services (servers, databases, message queues):

### Problem

Subagents are stateless - they don't know about processes started by previous subagents. Background processes persist and can interfere with later tests.

### Solution

**Before dispatching E2E test subagent, include cleanup in prompt:**

```
BEFORE starting any services:
1. Kill existing processes: pkill -f "<service-pattern>" 2>/dev/null || true
2. Wait for cleanup: sleep 1
3. Verify port free: lsof -i :<port> && echo "ERROR: Port still in use" || echo "Port free"

AFTER tests complete:
1. Kill the process you started
2. Verify cleanup: pgrep -f "<service-pattern>" || echo "Cleanup successful"
```

### Example

```
Task: Run E2E test of API server

Prompt includes:
"Before starting the server:
- Kill any existing servers: pkill -f 'node.*server.js' 2>/dev/null || true
- Verify port 3001 is free: lsof -i :3001 && exit 1 || echo 'Port available'

After tests:
- Kill the server you started
- Verify: pgrep -f 'node.*server.js' || echo 'Cleanup verified'"
```

### Why This Matters

- Stale processes serve requests with wrong config
- Port conflicts cause silent failures
- Process accumulation slows system
- Confusing test results (hitting wrong server)
```

**权衡分析：**
- 增加了提示中的样板内容
- 但避免了极其令人困惑的调试
- 对 E2E 测试 subagent 来说是值得的

---

### 3. subagent-driven-development：添加上下文精简选项

**修改第 2 步：使用 Subagent 执行任务**

**之前：**
```
Read that task carefully from [plan-file].
```

**之后：**
```
## Context Approaches

**Full Plan (default):**
Use when tasks are complex or have dependencies:
```
Read Task N from [plan-file] carefully.
```

**Lean Context (for independent tasks):**
Use when task is standalone and pattern-based:
```
You are implementing: [1-2 sentence task description]

File to modify: [exact path]
Pattern to follow: [reference to existing function/test]
What to implement: [specific requirement]
Verification: [exact command to run]

[Do NOT include full plan file]
```

**Use lean context when:**
- Task follows existing pattern (add similar test, implement similar feature)
- Task is self-contained (doesn't need context from other tasks)
- Pattern reference is sufficient (e.g., "follow TestE2E_FeatureOptionValidation")

**Use full plan when:**
- Task has dependencies on other tasks
- Requires understanding of overall architecture
- Complex logic that needs context
```

**示例：**
```
Lean context prompt:

"You are adding a test for privileged mode in devcontainer features.

File: pkg/runner/e2e_test.go
Pattern: Follow TestE2E_FeatureOptionValidation (at end of file)
Test: Feature with `"privileged": true` in metadata results in `--privileged` flag
Verify: go test -v ./pkg/runner -run TestE2E_FeaturePrivilegedMode -timeout 5m

Report: Implementation, test results, any issues."
```

**为什么这样有效：**
减少 token 使用量，提高专注度，在合适的情况下完成更快。

---

### 4. subagent-driven-development：添加自我反思步骤

**修改第 2 步：使用 Subagent 执行任务**

**在提示模板中添加：**

```
When done, BEFORE reporting back:

Take a step back and review your work with fresh eyes.

Ask yourself:
- Does this actually solve the task as specified?
- Are there edge cases I didn't consider?
- Did I follow the pattern correctly?
- If tests are failing, what's the ROOT CAUSE (implementation bug vs test bug)?
- What could be better about this implementation?

If you identify issues during this reflection, fix them now.

Then report:
- What you implemented
- Self-reflection findings (if any)
- Test results
- Files changed
```

**为什么这样有效：**
在交接前捕捉实现者自己能发现的 Bug。有记录的案例：通过自我反思识别出 entrypoint Bug。

**权衡：**
每个任务多花约 30 秒，但在审查前就捕捉了问题。

---

### 5. requesting-code-review：添加显式文件读取

**修改 code-reviewer 模板：**

**在开头添加：**

```markdown
## Files to Review

BEFORE analyzing, read these files:

1. [List specific files that changed in the diff]
2. [Files referenced by changes but not modified]

Use Read tool to load each file.

If you cannot find a file:
- Check exact path from diff
- Try alternate locations
- Report: "Cannot locate [path] - please verify file exists"

DO NOT proceed with review until you've read the actual code.
```

**为什么这样有效：**
显式指令可以防止"找不到文件"的问题。

---

### 6. testing-anti-patterns：添加 Mock-接口漂移反模式

**添加新的反模式 6：**

```markdown
## Anti-Pattern 6: Mocks Derived from Implementation

**The violation:**
```typescript
// Code (BUGGY) calls cleanup()
await adapter.cleanup();

// Mock (MATCHES BUG) has cleanup()
const mock = {
  cleanup: vi.fn().mockResolvedValue(undefined)
};

// Interface (CORRECT) defines close()
interface PlatformAdapter {
  close(): Promise<void>;
}
```

**Why this is wrong:**
- Mock encodes the bug into the test
- TypeScript can't catch inline mocks with wrong method names
- Test passes because both code and mock are wrong
- Runtime crashes when real object is used

**The fix:**
```typescript
// ✅ GOOD: Derive mock from interface

// Step 1: Open interface definition (PlatformAdapter)
// Step 2: List methods defined there (close, initialize, etc.)
// Step 3: Mock EXACTLY those methods

const mock = {
  initialize: vi.fn().mockResolvedValue(undefined),
  close: vi.fn().mockResolvedValue(undefined),  // From interface!
};

// Now test FAILS because code calls cleanup() which doesn't exist
// That failure reveals the bug BEFORE runtime
```

### Gate Function

```
BEFORE writing any mock:

  1. STOP - Do NOT look at the code under test yet
  2. FIND: The interface/type definition for the dependency
  3. READ: The interface file
  4. LIST: Methods defined in the interface
  5. MOCK: ONLY those methods with EXACTLY those names
  6. DO NOT: Look at what your code calls

  IF your test fails because code calls something not in mock:
    ✅ GOOD - The test found a bug in your code
    Fix the code to call the correct interface method
    NOT the mock

  Red flags:
    - "I'll mock what the code calls"
    - Copying method names from implementation
    - Mock written without reading interface
    - "The test is failing so I'll add this method to the mock"
```

**检测：**

当你看到运行时错误 "X is not a function" 且测试通过时：
1. 检查 X 是否被 mock
2. 将 mock 方法与接口方法进行对比
3. 查找方法名不匹配
```

**为什么这样有效：**
直接针对反馈中的失败模式。

---

### 7. subagent-driven-development：要求测试 subagent 阅读技能

**当任务涉及测试时，在提示模板中添加：**

```markdown
BEFORE writing any tests:

1. Read testing-anti-patterns skill:
   Use Skill tool: superpowers:testing-anti-patterns

2. Apply gate functions from that skill when:
   - Writing mocks
   - Adding methods to production classes
   - Mocking dependencies

This is NOT optional. Tests that violate anti-patterns will be rejected in review.
```

**为什么这样有效：**
确保技能被真正使用，而不仅仅是存在。

**权衡：**
每个任务都增加时间，但能防止整类 Bug。

---

### 8. subagent-driven-development：允许实现者修复自我发现的问题

**修改第 2 步：**

**当前：**
```
Subagent reports back with summary of work.
```

**提议：**
```
Subagent performs self-reflection, then:

IF self-reflection identifies fixable issues:
  1. Fix the issues
  2. Re-run verification
  3. Report: "Initial implementation + self-reflection fix"

ELSE:
  Report: "Implementation complete"

Include in report:
- Self-reflection findings
- Whether fixes were applied
- Final verification results
```

**为什么这样有效：**
当实现者已经知道修复方法时减少延迟。有记录的案例：本可为 entrypoint Bug 节省一次往返。

**权衡：**
提示稍复杂，但端到端更快。

---

## 实施计划

### 阶段 1：高影响、低风险（先做）

1. **verification-before-completion：配置变更验证**
   - 清晰的增补，不改变现有内容
   - 解决高影响问题（测试中的虚假信心）
   - 文件：`skills/verification-before-completion/SKILL.md`

2. **testing-anti-patterns：Mock-接口漂移**
   - 添加新的反模式，不修改现有内容
   - 解决高影响问题（运行时崩溃）
   - 文件：`skills/testing-anti-patterns/SKILL.md`

3. **requesting-code-review：显式文件读取**
   - 对模板的简单增补
   - 解决具体问题（审查者找不到文件）
   - 文件：`skills/requesting-code-review/SKILL.md`

### 阶段 2：中度变更（仔细测试）

4. **subagent-driven-development：进程卫生**
   - 添加新章节，不改变工作流
   - 解决中-高影响（测试可靠性）
   - 文件：`skills/subagent-driven-development/SKILL.md`

5. **subagent-driven-development：自我反思**
   - 修改提示模板（风险较高）
   - 但有记录证明能捕捉 Bug
   - 文件：`skills/subagent-driven-development/SKILL.md`

6. **subagent-driven-development：技能阅读要求**
   - 增加提示开销
   - 但确保技能被真正使用
   - 文件：`skills/subagent-driven-development/SKILL.md`

### 阶段 3：优化（先验证）

7. **subagent-driven-development：上下文精简选项**
   - 增加复杂性（两种方法）
   - 需要验证它不会造成混淆
   - 文件：`skills/subagent-driven-development/SKILL.md`

8. **subagent-driven-development：允许实现者修复**
   - 改变工作流（风险较高）
   - 是优化，不是 Bug 修复
   - 文件：`skills/subagent-driven-development/SKILL.md`

---

## 开放问题

1. **上下文精简方法：**
   - 是否应将其作为基于模式任务的默认方法？
   - 我们如何决定使用哪种方法？
   - 是否可能过于精简而遗漏重要上下文？

2. **自我反思：**
   - 是否会显著拖慢简单任务？
   - 是否只应适用于复杂任务？
   - 如何防止"反思疲劳"，即它变成例行公事？

3. **进程卫生：**
   - 应该放在 subagent-driven-development 中还是单独成一个技能？
   - 是否适用于 E2E 测试以外的其他工作流？
   - 如何处理进程本应持续存在的情况（开发服务器）？

4. **技能阅读强制：**
   - 是否应要求所有 subagent 阅读相关技能？
   - 如何防止提示变得过长？
   - 是否有过度记录、失去重点的风险？

---

## 成功指标

我们如何知道这些改进有效？

1. **配置验证：**
   - "测试通过但使用了错误配置"的情况为零
   - Jesse 不再说"那实际上不是你在测试的东西"

2. **进程卫生：**
   - "测试命中了错误的服务器"的情况为零
   - E2E 测试运行期间没有端口冲突错误

3. **Mock-接口漂移：**
   - "测试通过但运行时因缺失方法崩溃"的情况为零
   - mock 与接口之间没有方法名不匹配

4. **自我反思：**
   - 可量化：实现者的报告中是否包含自我反思发现？
   - 定性：进入代码审查的 Bug 是否更少？

5. **技能阅读：**
   - Subagent 报告是否引用技能闸门函数
   - 代码审查中的反模式违规是否更少

---

## 风险与缓解措施

### 风险：提示膨胀
**问题：** 添加所有这些要求会让提示难以承受
**缓解措施：**
- 分阶段实施（不要一次全部添加）
- 使部分增补有条件的（E2E 卫生只用于 E2E 测试）
- 考虑针对不同任务类型的模板

### 风险：分析瘫痪
**问题：** 过多的反思/验证拖慢执行
**缓解措施：**
- 保持闸门函数快速（几秒，而非几分钟）
- 初始阶段让上下文精简为可选加入
- 监控任务完成时间

### 风险：虚假的安全感
**问题：** 遵循清单并不能保证正确性
**缓解措施：**
- 强调闸门函数是最低要求，而非最高要求
- 在技能中保留"运用判断力"的表述
- 记录说明技能捕捉的是常见失败，而非所有失败

### 风险：技能分歧
**问题：** 不同技能给出相互矛盾的建议
**缓解措施：**
- 审查所有技能中的变更以确保一致性
- 记录技能如何交互（集成章节）
- 部署前用真实场景测试

---

## 建议

**立即推进阶段 1：**
- verification-before-completion：配置变更验证
- testing-anti-patterns：Mock-接口漂移
- requesting-code-review：显式文件读取

**在最终定稿前与 Jesse 测试阶段 2：**
- 获取关于自我反思影响的反馈
- 验证进程卫生方法
- 确认技能阅读要求值得增加开销

**在验证完成前暂缓阶段 3：**
- 上下文精简需要真实世界测试
- 实现者修复工作流的变更需要仔细评估

这些变更解决了用户记录的真实问题，同时最大程度降低了使技能变得更糟的风险。
