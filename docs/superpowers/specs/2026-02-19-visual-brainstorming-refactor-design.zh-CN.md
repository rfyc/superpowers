# 可视化头脑风暴重构：浏览器显示、终端命令

**日期：** 2026-02-19
**状态：** 已批准
**范围：** `lib/brainstorm-server/`、`skills/brainstorming/visual-companion.md`、`tests/brainstorm-server/`

## 问题

在可视化头脑风暴期间，Claude 会以后台任务方式运行 `wait-for-feedback.sh`，并阻塞在 `TaskOutput(block=true, timeout=600s)` 上。这会完全占据 TUI——在可视化头脑风暴运行期间，用户无法向 Claude 输入内容。浏览器成为唯一的输入通道。

Claude Code 的执行模型是基于回合（turn-based）的。Claude 无法在单个回合内同时监听两个通道。阻塞式 `TaskOutput` 模式是错误的原语——它模拟了平台并不支持的事件驱动行为。

## 设计

### 核心模型

**浏览器 = 交互式显示。** 展示 mockup，让用户点击选择选项。选择结果记录在服务器端。

**终端 = 对话通道。** 始终不阻塞、始终可用。用户在这里与 Claude 对话。

### 循环

1. Claude 将一个 HTML 文件写入会话目录
2. 服务器通过 chokidar 检测到它，向浏览器推送 WebSocket reload（不变）
3. Claude 结束其回合——告诉用户查看浏览器并在终端中回复
4. 用户查看浏览器，可选地点击选择一个选项，然后在终端中输入反馈
5. 在下一个回合，Claude 读取 `$SCREEN_DIR/.events` 获取浏览器交互流（点击、选择），与终端文本合并
6. 迭代或推进

无后台任务。无 `TaskOutput` 阻塞。无轮询脚本。

### 关键删除：`wait-for-feedback.sh`

完全删除。它的用途是桥接"服务器将事件记录到 stdout"与"Claude 需要接收这些事件"。`.events` 文件取代了它——服务器直接写入用户交互事件，Claude 用平台提供的任何文件读取机制读取它们。

### 关键新增：`.events` 文件（每屏事件流）

服务器将所有用户交互事件写入 `$SCREEN_DIR/.events`，每行一个 JSON 对象。这为 Claude 提供当前屏幕的完整交互流——不只是最终选择，还有用户的探索路径（点击了 A，然后 B，最终停在 C）。

用户探索选项后的示例内容：

```jsonl
{"type":"click","choice":"a","text":"Option A - Preset-First Wizard","timestamp":1706000101}
{"type":"click","choice":"c","text":"Option C - Manual Config","timestamp":1706000108}
{"type":"click","choice":"b","text":"Option B - Hybrid Approach","timestamp":1706000115}
```

- 在单个屏幕内仅追加。每个用户事件作为新行追加。
- 当 chokidar 检测到新 HTML 文件（新屏幕被推送）时，该文件被清空（删除），防止过期事件残留。
- 如果 Claude 读取时文件不存在，则没有发生浏览器交互——Claude 只使用终端文本。
- 该文件只包含用户事件（`click` 等）——不包含服务器生命周期事件（`server-started`、`screen-added`）。这使其保持小巧且专注。
- Claude 可以读取完整流以理解用户的探索模式，或者只查看最后一个 `choice` 事件获取最终选择。

## 按文件的变更

### `index.js`（服务器）

**A. 将用户事件写入 `.events` 文件。**

在 WebSocket `message` 处理器中，在将事件记录到 stdout 之后：通过 `fs.appendFileSync` 将事件作为 JSON 行追加到 `$SCREEN_DIR/.events`。只写入用户交互事件（那些带 `source: 'user-event'` 的事件），不写服务器生命周期事件。

**B. 在新屏幕时清空 `.events`。**

在 chokidar `add` 处理器中（检测到新的 `.html` 文件），如果 `$SCREEN_DIR/.events` 存在则删除它。这是明确的"新屏幕"信号——比在每次 reload 都会触发的 GET `/` 上清空更好。

**C. 替换 `wrapInFrame` 的内容注入。**

当前的正则锚定在 `<div class="feedback-footer">` 上，该元素正在被移除。替换为注释占位符：移除 `#claude-content` 内既有的默认内容（`<h2>Visual Brainstorming</h2>` 和副标题段落），替换为单个 `<!-- CONTENT -->` 标记。内容注入变为 `frameTemplate.replace('<!-- CONTENT -->', content)`。更简单，且如果模板格式变化也不会破坏。

### `frame-template.html`（UI 框架）

**移除：**
- `feedback-footer` div（textarea、Send 按钮、label、`.feedback-row`）
- 关联的 CSS（`.feedback-footer`、`.feedback-footer label`、`.feedback-row`、其中包含的 textarea 和按钮样式）

**新增：**
- `#claude-content` 内的 `<!-- CONTENT -->` 占位符，替换默认文本
- 一个选择指示条，位于原 footer 位置，具有两种状态：
  - 默认："Click an option above, then return to the terminal"
  - 选择后："Option B selected — return to terminal to continue"
- 指示条的 CSS（微妙，与现有 header 视觉权重相近）

**保持不变：**
- 带 "Brainstorm Companion" 标题和连接状态的 Header 条
- `.main` 包装器和 `#claude-content` 容器
- 所有组件 CSS（`.options`、`.cards`、`.mockup`、`.split`、`.pros-cons`、占位符、mock 元素）
- 深色/浅色主题变量和媒体查询

### `helper.js`（客户端脚本）

**移除：**
- `sendToClaude()` 函数和 "Sent to Claude" 页面接管
- `window.send()` 函数（曾绑定到被移除的 Send 按钮）
- 表单提交处理器——没有反馈 textarea 就没有用途，只会增加日志噪音
- 输入变更处理器——原因相同
- `pageshow` 事件监听器（曾用于修复 textarea 持久化——现在没有 textarea 了）

**保留：**
- WebSocket 连接、重连逻辑、事件队列
- reload 处理器（服务器推送时 `window.location.reload()`）
- 用于选择高亮的 `window.toggleSelect()`
- `window.selectedChoice` 跟踪
- `window.brainstorm.send()` 和 `window.brainstorm.choice()`——它们与被移除的 `window.send()` 不同。它们调用 `sendEvent`，通过 WebSocket 记录到服务器。适用于自定义整文档页面。

**收窄：**
- 点击处理器：只捕获 `[data-choice]` 点击，而非所有按钮/链接。宽泛捕获在浏览器是反馈通道时是必要的；现在它只用于选择跟踪。

**新增：**
- 在 `data-choice` 点击时，更新选择指示条文本以显示选中了哪个选项。

**从 `window.brainstorm` API 中移除：**
- `brainstorm.sendToClaude` —— 不再存在

### `visual-companion.md`（技能指令）

**重写 "The Loop" 部分**为上述非阻塞流程。移除对所有以下内容的引用：
- `wait-for-feedback.sh`
- `TaskOutput` 阻塞
- 超时/重试逻辑（600s 超时、30 分钟上限）
- 描述 `send-to-claude` JSON 的 "User Feedback Format" 部分

**替换为：**
- 新循环（写 HTML → 结束回合 → 用户在终端回复 → 读取 `.events` → 迭代）
- `.events` 文件格式文档
- 指引：终端消息是主要反馈；`.events` 提供完整的浏览器交互流作为补充上下文

**保留：**
- 服务器启动/关闭指令
- 内容片段与整文档的指引
- CSS 类参考和可用组件
- 设计技巧（按问题调整保真度、每屏 2-4 个选项等）

### `wait-for-feedback.sh`

**完全删除。**

### `tests/brainstorm-server/server.test.js`

需要更新的测试：
- 断言片段响应中存在 `feedback-footer` 的测试——更新为断言选择指示条或 `<!-- CONTENT -->` 替换
- 断言 `helper.js` 包含 `send` 的测试——更新以反映收窄后的 API
- 断言 `sendToClaude` CSS 变量用法的测试——移除（函数不再存在）

## 平台兼容性

服务器代码（`index.js`、`helper.js`、`frame-template.html`）完全平台无关——纯 Node.js 和浏览器 JavaScript。无任何 Claude Code 特有引用。已通过后台终端交互在 Codex 上验证可用。

技能指令（`visual-companion.md`）是平台自适应层。每个平台的 Claude 使用自己的工具启动服务器、读取 `.events` 等。非阻塞模型在各平台上自然工作，因为它不依赖任何平台特定的阻塞原语。

## 这带来的能力

- **可视化头脑风暴期间 TUI 始终可响应**
- **混合输入** —— 浏览器中点击 + 终端中输入，自然合并
- **优雅降级** —— 浏览器挂了或用户没打开？终端仍然可用
- **更简单的架构** —— 无后台任务、无轮询脚本、无超时管理
- **跨平台** —— 同一套服务器代码可用于 Claude Code、Codex 和任何未来平台

## 这放弃的能力

- **纯浏览器反馈工作流** —— 用户必须返回终端才能继续。选择指示条引导他们，但与旧的点击-Send-并等待流程相比多了一步。
- **浏览器的行内文本反馈** —— textarea 已被移除。所有文本反馈都通过终端进行。这是有意的——终端比框架中的小 textarea 是更好的文本输入通道。
- **浏览器 Send 上的即时响应** —— 旧系统在用户点击 Send 的瞬间就让 Claude 响应。现在用户切换到终端时有一个间隙。实际上这只是几秒，而且用户能在终端消息中添加上下文。
