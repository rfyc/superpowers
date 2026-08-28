# 可视化头脑风暴伴侣 — 问题与变更目录

**日期：** 2026-06-09
**状态：** 分析 / 分类。我们自行实施这些；文中引用的社区 PR 是证据和参考资料，**并非**我们要合并的代码。

## 目的

用单一处所记录触及可视化头脑风暴伴侣（`skills/brainstorming/scripts/` 中的本地服务器）的每个未解决问题和 PR，提炼为底层问题以及我们要做的变更。每个条目都对照当前代码，而非 PR 作者的描述。

## 范围决策（Jesse，2026-06-09）

- **不引入 Alpine.js 的 vendored 版本。** PR #1639（通过 vendored Alpine 构建实现交互式原型）被**放弃**。参见 E3。
- **E1（终端与 HTML 的硬性门禁）是工作坊议题。** 我们会一起设计；这里不做规格。
- **E2（存储位置，#975/#977）暂时搁置。**
- **远程服务是一等场景。** Superpowers 是通用的；用户从远程连接（SSH 隧道、Tailscale、`--host 0.0.0.0`）。安全修复必须保护这些用户，而不只是 loopback。**决策：采用每会话密钥，而非 Host 白名单。** Host 白名单只能防御 loopback 上的浏览器"被混淆的代理"；直接远程客户端只需发送预期的 `Host`，因此白名单对远程暴露形同虚设。密钥是唯一能在 loopback、隧道和直接远程之间统一认证客户端的方式，它还能抵御 DNS 重绑定。参见 A1。

## 组件映射

| 文件 | 角色 |
|------|------|
| `skills/brainstorming/scripts/server.cjs` | 零依赖 HTTP + WebSocket 服务器（RFC 6455 手写实现）。提供最新屏幕服务，监听 `content/`，将事件记录到 `state/events`。 |
| `skills/brainstorming/scripts/helper.js` | 注入到每个页面。WebSocket 客户端、点击捕获、`window.brainstorm` API。 |
| `skills/brainstorming/scripts/frame-template.html` | 围绕内容片段包裹的框架（头部、主题 CSS、状态点、指示条）。 |
| `skills/brainstorming/scripts/start-server.sh` | 启动包装脚本。会话目录、host/url-host、owner-PID 解析、平台后台化。 |
| `skills/brainstorming/scripts/stop-server.sh` | 通过 PID 文件终止服务器，清理 `/tmp` 会话。 |
| `skills/brainstorming/visual-companion.md` | agent 在接受伴侣时阅读的操作指南。 |
| `skills/brainstorming/SKILL.md` | 提供伴侣以及每个问题的决策所在之处。 |

## 处置汇总

| ID | 条目 | 来源 | 处置 |
|----|------|--------|-------------|
| A1 | `/`、`/files/*` 和 WS 上的每会话密钥（取代 Host 白名单） | issues #1014、PRs #1110/#1553 | **做** — 选定方案 |
| A2 | Host 白名单；浏览器 WS Origin 检查 | PRs #1110/#1553 | Host 白名单放弃；认证后保留 WS Origin 检查，用于浏览器"被混淆的代理"防御 |
| A3 | `null` / 非对象 WS 负载导致崩溃 | PR #1504 | 做 |
| A4 | `decodeFrame` 中的帧长度上限 | issue #1446 | 已修复 — 验证 / 关闭 |
| B1 | 点文件屏幕被当作内容服务（`._*.html`） | PR #950 | 做 |
| B2 | `stop-server.sh` 杀死被复用/过期的 PID | PR #1703 | 做 |
| B3 | WS 客户端重连退避 + 状态指示器 | PR #856 | 做 |
| C1 | 空闲超时过短 / 不可配置；关闭时 WS 未关闭 | issue #1237（PR #1689） | 做 |
| C2 | 服务器死亡对用户/agent 不可见 | issue #1237（残余） | 做 |
| D1 | 永久选择退出伴侣 | issue #892 | 搁置 — 不在 PR #1720 中 |
| D2 | 来自浏览器的自由文本反馈 | issue #957 | 搁置 — 不在 PR #1720 中 |
| D3 | 自动打开伴侣 URL | PR #759（#755） | 已在 PR #1720 中通过 `--open` 完成 |
| D4 | 框架中的明/暗对比度辅助 | PR #1683 | 搁置 — 不在 PR #1720 中 |
| E1 | 每题终端与 HTML 的硬性门禁 | PR #1037 | **工作坊** |
| E2 | 将会话状态移出工作树 | issue #975（PR #977） | **搁置** |
| E3 | 为交互式原型引入 Alpine.js 的 vendored 版本 | PR #1639 | **放弃** |
| E4 | start/stop 脚本中的 shell-lint 警告 | PR #1677 | 仅顺带处理 |

---

## A. 服务器安全加固（`server.cjs`）

### A1 — 每会话密钥（选定方案）

**威胁模型。** 两个资产：所服务屏幕（`/`）和文件（`/files/*`）的机密性，以及 `state/events` 的完整性 — 带真值 `choice` 的 WebSocket 客户端会写入那里（`server.cjs:243-246`），agent 在下一轮将其作为用户的选择读取，即**对具有完整工具访问权限的实时会话进行提示注入**。触达者：在默认的 `127.0.0.1` 绑定下，用户浏览器中的恶意页面（一个被混淆的代理 — 既运行攻击者 JS，又能触达 loopback）；在远程绑定下（`--host 0.0.0.0`、tailnet/LAN），任何能路由到该端口的主机，直接、无同源策略挡路。当前 `handleUpgrade`（`server.cjs:176`）只检查 `Sec-WebSocket-Key`，而 `handleRequest`（`server.cjs:138`）什么都不检查 — 两者都门户大开。

**为什么用密钥而非 Host 白名单。** Host 白名单只能防御 loopback 上的浏览器代理。直接远程客户端只需发送预期的 `Host` 并伪造/省略 `Origin`，因此白名单对我们必须保护的远程场景恰恰形同虚设。每会话密钥在 loopback、SSH 隧道和直接远程之间统一地认证客户端，它还能消灭 DNS 重绑定（被重绑定的页面既不知道密钥，也收不到 host 作用域的 cookie）。因此密钥完全**取代** A1/A2 的 Host 白名单 — 无需 `BRAINSTORM_ALLOWED_HOSTS`。

**设计。** 随机令牌（`crypto.randomBytes(32)` 十六进制），在 `server.cjs` 启动时生成（可通过 `BRAINSTORM_TOKEN` 覆盖，用于确定性测试）：

1. **URL 携带它**，作为 `?key=<token>`。服务器已经在 `server-started` JSON（`server.cjs:351`）中构建 `url` 并写入 `state/server-info` — 在那里追加 `?key=` 意味着 `start-server.sh`（grep 并打印该 JSON）和技能（把该 URL 交给用户）都**无需变更**。
2. **Cookie 引导。** `/` 上有效的 `?key` 会设置 `brainstorm-key-<port>=<token>; HttpOnly; SameSite=Strict; Path=/`。浏览器随后将其自动附加到同源子资源（`/files/*`）和 WebSocket 握手，因此 agent 可以写出任何 URL 风格都能工作，且 `helper.js` 无需变更。Cookie 名**按端口**，以避免 Jupyter 多服务器冲突（cookie 不按端口作用域）。`SameSite=Strict` 对 CDN/Unsplash 内容是安全的 — 该 cookie 是 host 作用域的，因此出站的 CDN 请求从不携带它；SameSite 只约束返回我们源站的请求，而这些请求都是同站的。
3. **认证门禁** = `/`、`/files/*` 和 WS 升级上有效 `?key` **或** 有效 cookie（用 `crypto.timingSafeEqual` 比较）。缺失/错误密钥 → 友好的 **403 HTML 页面**（"此页面需要你的编程 agent 给你的完整 URL，包括 `?key=…`" — 使用通用的"编程 agent"，而非"Claude"，因为这也随 Codex/Gemini/Copilot 一起发布）。WS 升级 → 销毁 socket。

查询令牌是事实来源；cookie 是一种便利，从不需要承担初始认证负载。

**影响范围。** `server.cjs`（全部逻辑）。`helper.js` 可选的一行（在 cookie 被屏蔽时作为回退，从 `location.search` 向 WS URL 追加 `?key=`）。`start-server.sh` 无。`visual-companion.md` 文档说明（URL 现在带 `?key=`；不要剥离它）。测试更新为传入令牌。

### A2 — 放弃 Host 白名单；保留浏览器 WS Origin

被 A1 吸收。密钥通过单一机制关闭 WS 注入向量（#1014）、HTTP/WS DNS 重绑定读取向量（PR #1553）和跨源 WS 向量（PR #1110），且与白名单不同，它实际保护了远程绑定场景。无 `BRAINSTORM_ALLOWED_HOSTS`，无 Host 白名单。最终实现仍在会话认证后检查浏览器 WebSocket `Origin`，使跨源的 localhost 标签页无法搭乘伴侣 cookie。

### A3 — 服务器在 `null` / 原始类型 WS 负载上崩溃

**问题。** `handleMessage`（`server.cjs:233`）先 `JSON.parse(text)`，然后在 `server.cjs:243` 做 `if (event.choice)`。客户端发送 4 字节文本帧 `null` 会得到 `event === null`，而 `null.choice` 抛异常。该异常**未被**捕获 — `handleMessage` 是从 `socket.on('data')` 处理器（`server.cjs:207`）调用的，位于 `try/catch` 之外，后者只包裹 `decodeFrame`。结果是未捕获异常并导致进程退出。任何本地客户端都能杀死服务器。

**变更。** 保护访问：`if (event && event.choice)`。最小且精确 — `JSON.parse` 无法产生 `undefined`，而原始类型对 `.choice` 返回 `undefined` 而不抛异常，因此只有 `null` 是实际风险。（避免更宽泛的修复 — 顶层 `try/catch` 或 `process.on('uncaughtException')` 会掩盖其他 bug。）

### A4 — `decodeFrame` 中的帧长度上限（相邻项）

PR #1504 将其引用为 #1446。当前代码**已经**限制了扩展帧长度：`MAX_FRAME_PAYLOAD_BYTES = 10MB`（`server.cjs:10`）在 `server.cjs:58-67` 的任何 `Buffer.alloc` 之前强制执行。行动：对照当前 `dev` 验证 #1446，若已解决则关闭，而非重新实现。

---

## B. 服务器健壮性 / 正确性

### B1 — macOS 资源分叉点文件被当作屏幕内容服务

**问题。** 最新屏幕选择器只按 `f.endsWith('.html')` 过滤（`server.cjs:127-128`）。在 macOS/ExFAT 上，`._screen.html` 资源分叉文件能通过该过滤器，且由于它们与真实文件一起写入，可能排在最前 — 于是浏览器拿到的是二进制元数据而非原型。四个读取位置共享这个弱过滤器：`getNewestScreen`（`server.cjs:127`）、`knownFiles` 初始化（`server.cjs:279`）、`fs.watch` 处理器（`server.cjs:286`）和 `/files/` 端点（`server.cjs:154-156`）。

**变更。** 在全部四个位置拒绝点文件（`!f.startsWith('.')`）。覆盖 `._*`、`.DS_Store` 等。

### B2 — `stop-server.sh` 可能杀死被复用的 PID

**问题。** `stop-server.sh` 从 `state/server.pid`（`stop-server.sh:20`）读取 PID 并对其 `kill`（`:23`，在 `:35` 升级到 `-9`），却不确认该 PID 仍属于我们的服务器。在重启或 PID 回绕之后，该文件可能指向无关进程，而我们会对其 SIGKILL。

**变更。** 在发信号之前验证所有权 — 该 PID 的命令是运行我们 `server.cjs` 的 `node`，理想情况下匹配本会话。若无法证明所有权，则失败关闭（报告 `stale_pid`，不 kill）。保留现有 `stopped` / `not_running` 输出以用于真实场景。

### B3 — WebSocket 客户端：静默重连、陈旧的"Connected"

**问题。** `helper.js` 在固定 1s 定时器上重连（`helper.js:21-23`），没有 `onerror` 处理器，关闭时从不将 `ws` 置空，也从不清除待处理的重连定时器。框架的状态元素硬编码为 "Connected"，圆点钉在 `var(--success)`（`frame-template.html:77,200`）。当笔记本休眠或服务器重启时，页面在死 socket 上仍显示 "Connected"，并在无反馈的情况下排队事件。

**变更。**
- `helper.js`：指数退避（500ms → ×2 → 上限 30s，open 时重置）；`onerror` 委托给 `onclose`；关闭时 `ws = null`；重连前 `clearTimeout`。
- `frame-template.html`：用 `--status-color` 自定义属性驱动状态点，使 JS 能在 Connected（绿）/ Reconnecting（黄）/ Disconnected（红）之间切换。

---

## C. 生命周期 / 超时（issue #1237）

### C1 — 空闲超时过短、不可配置、WS 使进程保持存活

**问题。** `IDLE_TIMEOUT_MS` 硬编码为 30 分钟（`server.cjs:258`），由 60s 生命周期检查（`server.cjs:329-332`）强制执行。单个头脑风暴问题可能因用户思考或离开而停留超过 30 分钟，于是服务器在会话中途死亡。另外，`shutdown()`（`server.cjs:310-321`）调用 `server.close()`，但从不关闭 `clients`（`server.cjs:174`）中升级后的 socket，因此打开中的浏览器连接可以在关闭后继续让 Node 进程存活。

**变更。**
- 将默认值提高到 4 小时并使其可配置：`start-server.sh` 中的 `--idle-timeout-minutes` → 环境变量 → `IDLE_TIMEOUT_MS`，并针对 Node 定时器溢出做校验。
- 在启动 JSON / `state/server-info` 中暴露生效的超时值。
- 在 `shutdown()` 中关闭 `clients` 中的每个 socket，使进程真正退出。

### C2 — 服务器死亡不可见

**问题。** 服务器退出时写入 `state/server-stopped` 并移除 `state/server-info`（`server.cjs:312-317`），技能被**告知**检查这些文件（`visual-companion.md:108`）— 但这是模型会跳过的软性指导，而浏览器只显示通用的"无法访问"。用户手动诊断；agent 继续引用一个已死的 URL。

**变更（两部分，独立于 C1）：**
- **浏览器端墓碑。** 在最后服务的 URL 上留下某种东西，显示"此伴侣已过期 — 请让 Claude 重启它"，而非连接错误。可权衡的方案：`helper.js` 在 socket 超过退避仍掉线时渲染横幅（仅在页面加载期间有效），对比更复杂的做法（保持一个最小响应器存活以服务墓碑页）。
- **更强的技能检查。** 收紧 `visual-companion.md` / `SKILL.md`，使"在引用 URL 或推送屏幕之前检查 `server-info`/`server-stopped`"成为必需步骤，而非一句说明。保持轻量 — 可能是一个 agent 总是运行的单行助手。

---

## D. 功能

### D1 — 永久选择退出可视化伴侣（issue #892）

**问题。** 伴侣每个会话都作为单独消息被提供（`SKILL.md:25,151-152`）。永不想要它的用户每次都要付出那次往返 — 以及 HTML 生成 — 的代价。没有"永不再提供这个"的途径。

**变更。** 在提供步骤之前，技能检查用户级设置，当选择退出被设置时完全跳过提供。

**设计选择待定。** 机制尚未定：
- 技能被告知读取的环境变量（如 `SUPERPOWERS_VISUAL_COMPANION=off`）— 最简单，符合 issue 的要求，位于 `.zshrc`。
- 插件设置文件（`.claude/superpowers.local.md` frontmatter）— 更结构化，支持按项目，但更重且按项目作用域。
- issue 中的可靠性告诫：一个单独的"no-companion"技能会在触发词上竞争，且不可靠 — 已否决。

选定机制后，就是一个小的 `SKILL.md` 变更加上一个文档化旋钮。

### D2 — 来自浏览器的自由文本反馈（issue #957）

**问题。** 客户端只捕获对 `[data-choice]` 的点击（`helper.js:36-62`）。想给原型做批注（"蓝色色调不对"）的用户不得不切到终端，破坏了视觉流程。

**变更。** 添加一个反馈 `<textarea>`，其提交通过现有的 `window.brainstorm.send` 路径（`helper.js:82-85`）发出 `{"type":"feedback","text":...,"timestamp":...}`。

**跨切面 — 需要服务器变更。** `handleMessage` 只在 `event.choice` 为真值时才持久化事件（`server.cjs:243`）。`feedback` 事件没有 `choice`，所以今天它会被记录，但**绝不会写入 `state/events`**，agent 也就看不到。持久化条件还必须接受 `feedback` 事件。在 `visual-companion.md`（Browser Events Format，`:247-259`）中记录新事件形状。决定提交触发器（按钮 vs blur vs 两者）以及 textarea 渲染位置（框架级 vs 每屏选择加入）。

### D3 — 自动打开伴侣 URL（PR #759，issue #755）

**问题。** `start-server.sh` 只打印 URL；用户手动打开。尤其在 WSL2 中，人们期望浏览器自动打开。

**变更。** 在 `server-started` JSON 被解析后进行尽力而为的打开器：Windows/WSL → `rundll32.exe url.dll,FileProtocolHandler <url>`，macOS → `open`，Linux → 仅当设置了 `DISPLAY`/`WAYLAND_DISPLAY` 时 `xdg-open`。吞掉失败，绝不阻塞启动，继续回显 URL。在 `visual-companion.md` 中记录。（考虑为无头/远程运行提供选择退出，因为在那里弹出浏览器是错误的 — 与 D1 的配置机制相关联。）

### D4 — 明/暗对比度辅助（PR #1683）

**问题。** 内容片段被包裹在感知 OS 的框架中（`frame-template.html`）。在暗色模式下，快速原型常用白色内联背景，同时继承低对比度的框架文本，使卡片/面板难以阅读。

**变更。** 添加 `.light-surface` / `.dark-surface` 辅助类，以及为常见内联浅色背景提供的保守回退，并在 `visual-companion.md` 的 CSS 参考中记录它们。纯 CSS，位于 `frame-template.html`。

---

## E. 工作坊 / 搁置 / 放弃

### E1 — 每题终端与 HTML 的硬性门禁（PR #1037）— 工作坊

软性指导已存在："每题决定"，浏览器 vs 终端测试位于 `SKILL.md:156-161` 和 `visual-companion.md:5-25`。抱怨在于模型为纯文本内容（A/B 列表、澄清问题）渲染 HTML，浪费令牌和一轮。PR #1037 将决定包裹在 `<HARD-GATE>` 中。**按 Jesse 的意见，我们会一起打磨措辞/机制** — 这是塑造行为的技能内容，这里不做规格。

### E2 — 将会话状态移出工作树（issue #975 / PR #977）— 搁置

今天 `--project-dir` 将会话状态写到 `<project>/.superpowers/brainstorm/`（`start-server.sh:80-84`），技能告诉用户将其加入 gitignore（`visual-companion.md:58`）。诉求是仓库之外的 `--state-dir` / `SUPERPOWERS_STATE_DIR` 默认值（XDG），保留 `--project-dir` 作为别名。**Jesse 暂时搁置。** 记录下来以免丢失。

### E3 — 为交互式原型引入 Alpine.js 的 vendored 版本（PR #1639）— 放弃

添加一个 vendored Alpine 构建，使原型无需手写 JS 即可交互（标签页、手风琴、表单）。**按 Jesse 意见放弃** — 我们不在伴侣运行时中承担 vendored 第三方依赖。底层需求（交互式原型）不以这种方式推进。

### E4 — shell-lint 警告（PR #1677）— 顺带处理

`start-server.sh` / `stop-server.sh` 中的 SC2034（等）。微不足道；在我们编辑这些脚本时并入 B2/C1/D3，而非作为单独变更。

---

## 建议的实施分组

这些聚成几个连贯的批次（每批可对照 `tests/brainstorm-server/` 独立测试）：

1. **安全批次**（进行中，分支 `brainstorm-companion-session-key`）— A1 每会话密钥（取代 A2）+ A3 null 崩溃保护。验证/关闭 A4。*最高优先级。*
2. **生命周期批次** — C1 + C2 一起（两者都触及 `shutdown()` 和服务器死亡的故事）。
3. **健壮性批次** — B1、B2、B3（独立、小）。
4. **搁置功能批次** — D1、D2、D4 不属于 PR #1720。D3 通过 `--open` 流程发布。

E1 是单独的工作坊会话。E2/E3 超出本轮范围。
