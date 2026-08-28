# 视觉伴侣指南

基于浏览器的头脑风暴视觉伴侣，用于展示 mockup、图表和选项。

## 何时使用

逐问题决定，而不是按会话决定。判据：**用户用看的会比用读的理解得更好吗？**

当内容本身是视觉的时候**使用浏览器**：

- **UI mockup** ——线框图、布局、导航结构、组件设计
- **架构图** ——系统组件、数据流、关系图
- **并排视觉对比** ——比较两种布局、两种配色方案、两个设计方向
- **设计打磨** ——当问题涉及观感、间距、视觉层级时
- **空间关系** ——以图表形式渲染的状态机、流程图、实体关系

当内容是文本或表格时**使用终端**：

- **需求和范围问题** ——"X 是什么意思？"、"哪些功能在范围内？"
- **概念性的 A/B/C 选择** ——在文字描述的方案之间做选择
- **权衡列表** ——利弊、对比表格
- **技术决策** ——API 设计、数据建模、架构方案选择
- **澄清问题** ——任何答案是文字、而非视觉偏好的内容

一个*关于* UI 主题的问题不自动是视觉问题。"你想要哪种向导？"是概念性的——用终端。"这些向导布局里哪种感觉对？"是视觉的——用浏览器。

## 工作原理

服务器监听一个目录中的 HTML 文件，并把最新的文件提供给浏览器。你把 HTML 内容写到 `screen_dir`，用户在浏览器中看到它并可点击选择选项。选择会记录到 `state_dir/events`，你在下一个回合读取。

**内容片段 vs 完整文档：** 如果你的 HTML 文件以 `<!DOCTYPE` 或 `<html` 开头，服务器会原样提供它（只注入辅助脚本）。否则，服务器会自动把你的内容包装进框架模板——加上页头、CSS 主题、连接状态和所有交互基础设施。**默认写内容片段。** 只有当你需要对页面有完全控制时才写完整文档。

## 启动一个会话

```bash
# 在用户批准伴侣之后启动。--open 会在第一屏时自动打开他们的浏览器；
# --project-dir 会持久化 mockup 并支持同端口重启。
scripts/start-server.sh --project-dir /path/to/project --open

# 返回: {"type":"server-started","port":52341,
#           "url":"http://localhost:52341/?key=ab12…",
#           "screen_dir":"/path/to/project/.superpowers/brainstorm/12345-1706000000/content",
#           "state_dir":"/path/to/project/.superpowers/brainstorm/12345-1706000000/state"}
```

从响应中保存 `screen_dir` 和 `state_dir`。使用 `--open` 时，你推送第一屏后浏览器会自动打开——你不需要请用户打开它，但仍然分享 URL 作为后备（无头/远程环境不会自动打开）。

**URL 包含会话密钥（`?key=…`）。** 服务器拒绝任何不带它的请求，所以永远给用户 `url` 字段里的**完整** URL——绝不要去掉查询字符串，也绝不要给出裸的 `http://host:port`。该密钥约束着 HTTP 和 WebSocket 访问，因此一个多余的浏览器标签页或网络上的另一台机器无法读取屏幕或注入事件。首次加载后浏览器会通过 cookie 记住密钥，所以刷新和 `/files/*` 资源无需重复输入它也能工作。

**查找连接信息：** 服务器把它的启动 JSON 写到 `$STATE_DIR/server-info`。如果你在后台启动了服务器且没有捕获 stdout，读取该文件以获取 URL 和端口。使用 `--project-dir` 时，在 `<project>/.superpowers/brainstorm/` 中查找会话目录。

**注意：** 把项目根目录作为 `--project-dir` 传入，这样 mockup 会持久化在 `.superpowers/brainstorm/` 中并在服务器重启后存活。不传的话，文件会放进 `/tmp` 并被清理。提醒用户把 `.superpowers/` 加入 `.gitignore`（如果还没加的话）。

**按平台启动服务器：**

**Claude Code:**
```bash
# 默认模式即可——脚本自己会把服务器放到后台。
scripts/start-server.sh --project-dir /path/to/project --open
```

在 Windows 上，脚本会自动检测并切换到前台模式（这会阻塞工具调用）。在 Bash 工具调用上使用 `run_in_background: true`，这样服务器能在对话回合之间存活，然后在下一个回合读取 `$STATE_DIR/server-info` 获取 URL 和端口。

**Codex:**
```bash
# Codex 会收割后台进程。脚本会自动检测 CODEX_CI 并切换到前台模式。
# 正常运行即可——不需要额外 flag。
scripts/start-server.sh --project-dir /path/to/project --open
```

**Gemini CLI:**
```bash
# 使用 --foreground 并在你的 shell 工具调用上设置 is_background: true，
# 这样进程能在回合之间存活
scripts/start-server.sh --project-dir /path/to/project --open --foreground
```

**Copilot CLI:**
```bash
# 用 Copilot CLI 的非阻塞/后台 shell 机制启动它，这样服务器能在回合之间
# 存活。保留 --foreground，让 harness 而不是脚本负责后台化。启动器是一个
# .sh，所以通过 bash 调用它（在 Windows 上，从 PowerShell 工具调用 Git Bash
# 的 bash.exe）。
bash scripts/start-server.sh --project-dir /path/to/project --open --foreground
```

**其他环境：** 服务器必须在对话回合之间持续在后台运行。如果你的环境会收割脱离的进程，使用 `--foreground` 并用你平台的后台执行机制启动该命令。

如果从你的浏览器无法访问该 URL（在远程/容器化环境中很常见），绑定一个非回环主机：

```bash
scripts/start-server.sh \
  --project-dir /path/to/project \
  --host 0.0.0.0 \
  --url-host localhost
```

使用 `--url-host` 控制返回的 URL JSON 中打印的主机名。

## 工作循环

1. **检查服务器是否存活**，然后把 **HTML 写**到 `screen_dir` 中的一个新文件：
   - **必需：在引用 URL 或推送屏幕之前，确认服务器存活。** 检查 `$STATE_DIR/server-info` 存在且 `$STATE_DIR/server-stopped` 不存在。如果它已关闭，用**相同的 `--project-dir`** 通过 `start-server.sh` 重启它——它会复用相同的端口，因此用户已打开的标签页会自动重新连接（服务器宕机期间会显示一个"paused"覆盖层），你也不需要发送新 URL。服务器空闲 4 小时后自动退出（可用 `--idle-timeout-minutes` 配置）。
   - 使用语义化的文件名：`platform.html`、`visual-style.html`、`layout.html`
   - **绝不复用文件名** ——每个屏幕都用一个新文件
   - 使用你的文件创建工具——**绝不用 cat/heredoc**（会把噪音倾倒进终端）
   - 服务器自动提供最新的文件

2. **告诉用户会看到什么并结束你的回合：**
   - 提醒他们 URL（每一步都提醒，不只是第一步）
   - 给一个简短的文字摘要说明屏幕上有什么（例如"正在展示首页的 3 个布局选项"）
   - 请他们在终端里回复："看看然后告诉我你的想法。如果想选某个选项，点击选中即可。"

3. **在你的下一个回合** ——在用户在终端里回复之后：
   - 如果 `$STATE_DIR/events` 存在就读取它——它包含用户的浏览器交互（点击、选择），格式为 JSON 行
   - 与用户在终端中的文字合并，得到全貌
   - 终端消息是主要反馈；`state_dir/events` 提供结构化的交互数据

4. **迭代或推进** ——如果反馈改变了当前屏幕，写一个新文件（例如 `layout-v2.html`）。只有当前步骤被验证后才进入下一个问题。

5. **回到终端时卸载** ——当下一步不需要浏览器时（例如澄清问题、权衡讨论），推送一个等待屏幕以清掉过期内容：

   ```html
   <!-- filename: waiting.html (or waiting-2.html, etc.) -->
   <div style="display:flex;align-items:center;justify-content:center;min-height:60vh">
     <p class="subtitle">Continuing in terminal...</p>
   </div>
   ```

   这可以防止对话已经继续后用户还盯着一个已解决的选择。当下一个视觉问题出现时，照常推送一个新的内容文件。

6. 重复直到完成。

## 写内容片段

只写进入页面的内容。服务器会自动把它包装进框架模板（页头、主题 CSS、连接状态和所有交互基础设施）。

**最小示例：**

```html
<h2>Which layout works better?</h2>
<p class="subtitle">Consider readability and visual hierarchy</p>

<div class="options">
  <div class="option" data-choice="a" onclick="toggleSelect(this)">
    <div class="letter">A</div>
    <div class="content">
      <h3>Single Column</h3>
      <p>Clean, focused reading experience</p>
    </div>
  </div>
  <div class="option" data-choice="b" onclick="toggleSelect(this)">
    <div class="letter">B</div>
    <div class="content">
      <h3>Two Column</h3>
      <p>Sidebar navigation with main content</p>
    </div>
  </div>
</div>
```

就这些。不需要 `<html>`、不需要 CSS、不需要 `<script>` 标签。服务器会提供全部。

## 可用的 CSS 类

框架模板为你的内容提供这些 CSS 类：

### 选项（A/B/C 选择）

```html
<div class="options">
  <div class="option" data-choice="a" onclick="toggleSelect(this)">
    <div class="letter">A</div>
    <div class="content">
      <h3>Title</h3>
      <p>Description</p>
    </div>
  </div>
</div>
```

**多选：** 给容器加上 `data-multiselect` 让用户选择多个选项。每次点击切换该条目的选中样式。

```html
<div class="options" data-multiselect>
  <!-- same option markup — users can select/deselect multiple -->
</div>
```

### 卡片（视觉设计）

```html
<div class="cards">
  <div class="card" data-choice="design1" onclick="toggleSelect(this)">
    <div class="card-image"><!-- mockup content --></div>
    <div class="card-body">
      <h3>Name</h3>
      <p>Description</p>
    </div>
  </div>
</div>
```

### Mockup 容器

```html
<div class="mockup">
  <div class="mockup-header">Preview: Dashboard Layout</div>
  <div class="mockup-body"><!-- your mockup HTML --></div>
</div>
```

### 分屏视图（并排）

```html
<div class="split">
  <div class="mockup"><!-- left --></div>
  <div class="mockup"><!-- right --></div>
</div>
```

### 利弊

```html
<div class="pros-cons">
  <div class="pros"><h4>Pros</h4><ul><li>Benefit</li></ul></div>
  <div class="cons"><h4>Cons</h4><ul><li>Drawback</li></ul></div>
</div>
```

### Mock 元素（线框图构建块）

```html
<div class="mock-nav">Logo | Home | About | Contact</div>
<div style="display: flex;">
  <div class="mock-sidebar">Navigation</div>
  <div class="mock-content">Main content area</div>
</div>
<button class="mock-button">Action Button</button>
<input class="mock-input" placeholder="Input field">
<div class="placeholder">Placeholder area</div>
```

### 排版与分区

- `h2` ——页面标题
- `h3` ——分区标题
- `.subtitle` ——标题下方的次要文本
- `.section` ——带底部外边距的内容块
- `.label` ——小号大写标签文本

## 浏览器事件格式

当用户在浏览器中点击选项时，他们的交互会被记录到 `$STATE_DIR/events`（每行一个 JSON 对象）。当你推送新屏幕时，该文件会自动清空。

```jsonl
{"type":"click","choice":"a","text":"Option A - Simple Layout","timestamp":1706000101}
{"type":"click","choice":"c","text":"Option C - Complex Grid","timestamp":1706000108}
{"type":"click","choice":"b","text":"Option B - Hybrid","timestamp":1706000115}
```

完整的事件流展示用户的探索路径——他们可能会在做出决定前点击多个选项。最后一个 `choice` 事件通常就是最终选择，但点击模式可能揭示值得追问的犹豫或偏好。

如果 `$STATE_DIR/events` 不存在，说明用户没有与浏览器交互——只用他们的终端文字。

## 设计建议

- **让保真度匹配问题** ——布局问题用线框图，打磨问题用精修图
- **在每个页面上说明问题** ——"哪个布局看起来更专业？"而不只是"选一个"
- **推进前先迭代** ——如果反馈改变了当前屏幕，写一个新版本
- **每屏最多 2-4 个选项**
- **在重要时使用真实内容** ——对于摄影作品集，使用真实图片（Unsplash）。占位内容会掩盖设计问题。
- **保持 mockup 简单** ——聚焦于布局和结构，而不是像素级完美

## 文件命名

- 使用语义化的名称：`platform.html`、`visual-style.html`、`layout.html`
- 绝不复用文件名 ——每个屏幕都必须是一个新文件
- 迭代时：追加版本后缀，如 `layout-v2.html`、`layout-v3.html`
- 服务器按修改时间提供最新文件

## 清理

```bash
scripts/stop-server.sh $SESSION_DIR
```

如果会话使用了 `--project-dir`，mockup 文件会持久化在 `.superpowers/brainstorm/` 中供以后参考。只有 `/tmp` 会话会在停止时被删除。

## 参考

- 框架模板（CSS 参考）：`scripts/frame-template.html`
- 辅助脚本（客户端）：`scripts/helper.js`
