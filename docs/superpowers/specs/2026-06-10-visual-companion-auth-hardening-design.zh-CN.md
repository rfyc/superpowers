# Visual Companion 认证加固设计

**日期：** 2026-06-10
**状态：** 草稿，待 Drew 审阅

## 目标

修复 PR #1720 中 brainstorming 可视化 companion 存在的安全与可靠性缺口，且不改变 companion 的核心工作流，也不新增运行时依赖。

修复必须先写测试，并必须留下清晰的自动化证据，证明：

- 跨源浏览器标签页无法借助 cookie 注入 companion 事件
- 重启重连不再只依赖浏览器 cookie 行为
- bearer key 在 bootstrap 之后不再残留于可见 URL 中
- `/files/*` 无法提供内容目录之外的文件
- 未来同源（same-origin）的 vendored UI 库仍可正常工作

## 威胁模型

companion 为单个 brainstorming 会话提供由 agent 生成的本地 UI。重要资产包括：

- companion 提供的屏幕内容
- 会话 key
- `state/events`，agent 将其读取为用户反馈
- companion 会话目录下的本地文件

范围内的攻击者：

- 另一个 `localhost` 端口上的恶意浏览器标签页
- 一个可以向 companion 发起请求、但不应当能以 companion UI 身份完成认证的浏览器页面
- 当服务器绑定到非 loopback 接口时的直接远程客户端
- 通过 URL 历史、referrer 或已提交的本地状态造成的意外泄漏
- 内容目录中的符号链接或路径技巧，借此逃逸 `/files/*`

本次修复范围之外：

- 恶意的 agent 编写的屏幕 HTML
- companion 屏幕加载的恶意同源 vendored JavaScript

这一范围边界是刻意设定的。companion 屏幕属于 agent UI 表面的一部分。它们今天可能使用内联脚本，将来可能使用同源 vendored 库（如 Alpine 或 Three.js）。要防御恶意的屏幕 HTML，需要更大型的 sandboxed-iframe 架构并配合一个狭窄的消息桥；这不在本次 PR 加固的范围之内。

## 当前失败点

自动化测试与有头浏览器测试在 PR 分支上发现了以下失败点：

1. 跨源的 localhost 页面可以在真实 companion 页面设置 cookie 之后，打开一个基于 cookie 认证的 WebSocket，并把攻击者控制的选择写入 `state/events`。
2. `/files/*` 提供指向 `content/` 之外的符号链接，包括指向包含带 key URL 的 `state/server-info` 的符号链接。
3. 会话 key 仍然留在实际屏幕页面的 URL 中，因此同源的屏幕 JavaScript 以及意外的 referrer/历史记录都能看到它。
4. helper 使用无 key 的 `ws://host` URL 重连。在有头 Chrome 中，经过同端口/同 token 的重启后，浏览器不再向重启后的服务器出示 cookie，因此打开的标签页会一直停留在 tombstone 页面上，直到手动刷新。
5. shell lint 与生命周期测试需要清理，以便测试在 Codex 中稳定通过。

## 设计

### 1. Bootstrap 携带 key 的加载

`GET /?key=<token>` 变为一个 bootstrap 响应，而不是屏幕响应。

当 key 有效时，服务器：

1. 像现在一样设置 HttpOnly 会话 cookie
2. 返回一个小的 HTML bootstrap 页面
3. bootstrap 页面把 key 存储在标签页作用域的 `sessionStorage` 中
4. bootstrap 页面使用 `location.replace('/')` 跳转到 `/`

之后，可见的屏幕 URL 是裸的 `/`，而不是 `/?key=...`。

带有有效 cookie 的 `GET /` 提供当前屏幕。不带有效 cookie 的 `GET /` 仍返回友好的 403 页面。`GET /?key=<wrong>` 返回 403。

为什么用 `sessionStorage`：helper 需要一个能够在同端口重启后存续、且不只依赖 cookie 行为的重连凭据。由于屏幕 HTML 是可信的同源 UI，把 key 存储在标签页作用域存储中对于这个威胁模型是可以接受的。它比把 key 留在地址栏、历史记录和 referrer 表面上要实质性地更好。

### 2. WebSocket 同源强制

WebSocket 升级必须同时通过两项检查：

1. 通过查询 key 或 cookie 完成有效的会话认证
2. 如果存在 `Origin` 头，它必须与请求目标源匹配

源检查应比较：

```text
Origin === "http://" + req.headers.host
```

浏览器攻击者页面示例：

```text
Origin: http://localhost:9999
Host: localhost:58088
```

即使浏览器发送了 companion cookie，也必须拒绝这一情况。

合法 companion 页面示例：

```text
Origin: http://localhost:58088
Host: localhost:58088
```

当 key 或 cookie 有效时，应接受这一情况。

直接的、非浏览器的客户端可以省略 `Origin`；它们仍需要会话 key。

### 3. Helper 重连凭据

`helper.js` 应从 `sessionStorage` 读取标签页作用域的 key，并将其追加到 WebSocket URL：

```text
ws://<host>/?key=<stored-key>
```

如果不存在已存储的 key，helper 将回退到当前的纯 cookie 行为 `ws://<host>`。这保持了与已加载页面的兼容性——这些页面确实有有效 cookie，但没有存储条目。

### 4. `/files/*` 包含（containment）

文件服务器应继续拒绝空名称和点文件。它还必须确保该文件是 `CONTENT_DIR` 内部的真实常规文件。

以 realpath 包含作为边界：

- 计算 `realContentDir = fs.realpathSync(CONTENT_DIR)`
- 计算 `realFilePath = fs.realpathSync(filePath)`
- 仅当 `realFilePath` 等于 `realContentDir` 的后代时才提供服务
- 对符号链接以及内容目录之外的任何内容返回 404 予以拒绝

服务器应继续使用 `path.basename`，因此嵌套路径仍不受支持。

### 5. 减少泄漏的响应头

添加保守的响应头，不阻止内联脚本或未来的同源 vendored 库：

```text
Referrer-Policy: no-referrer
Cache-Control: no-store
X-Frame-Options: DENY
Content-Security-Policy: frame-ancestors 'none'
Cross-Origin-Resource-Policy: same-origin
```

本轮不添加限制性的 `script-src` CSP。companion 目前会注入内联 helper JavaScript，未来的屏幕可能加载同源 vendored 库。

### 6. Gitignore 持久化会话状态

在仓库根目录的 `.gitignore` 中添加 `.superpowers/`，以便使用 `--project-dir` 时不会意外提交持久化的 companion 状态和 `.last-token`。

### 7. 测试稳定性与 Lint

清理被改动的启动/停止脚本中的 shell lint 警告。

更新调用 `start-server.sh --idle-timeout-minutes` 的生命周期测试，使其在 Codex 的 `CODEX_CI` 前台自动检测机制下不会挂起。当测试期望脚本返回启动 JSON 时，应使用 `--background` 强制后台模式。

## 测试策略

所有行为变更都应采用 TDD：

1. 编写失败的重点测试
2. 运行并确认其因预期原因失败
3. 实现最小修复
4. 重新运行重点测试
5. 重新运行完整的 brainstorm-server 套件

必需的重点回归测试：

- 带有效 key 的 `/` 返回 bootstrap，而非屏幕内容
- bootstrap 把 key 存储在 `sessionStorage` 中并去除 URL 中的 key
- 仅带 cookie 的 `/` 仍提供屏幕内容
- helper 为 WebSocket URL 使用 `sessionStorage` 中的 key
- 同源的 cookie WebSocket 可以打开
- 跨源的 cookie WebSocket 被拒绝且不写入任何事件
- 不带 `Origin` 的直接 key WebSocket 仍可打开
- `content/` 下指向 `state/server-info` 的符号链接返回 404
- 安全响应头存在于正常的 HTML、bootstrap、403 和文件响应上
- 同端口/token 重启后，可用已存储的 key 对重连进行认证
- 被改动的 shell 脚本通过 shell lint
- 生命周期套件在 Codex 下不挂起

## 验收标准

- `cd tests/brainstorm-server && npm test` 反复通过且不挂起。
- 之前从另一个 localhost 源写入 `attacker-injected` 的安全探针现在无法打开 WebSocket，并保持 `state/events` 不变。
- 指向 `server-info` 的符号链接探针返回 404。
- 有头或无头浏览器携带 key 的加载最终落在裸 `/` URL 上，且状态胶囊（status pill）达到 Connected。
- 同端口/同 token 的重启无需手动刷新即可自动重连。
- 被改动的 shell 脚本通过 `scripts/lint-shell.sh`。

## 延后工作

如果项目将来需要把屏幕 HTML 视为不可信，请另行设计一个独立的 sandboxed iframe 架构。该架构应在独立源或 sandboxed frame 上隔离生成的屏幕，并只为用户选择暴露一个狭窄的 `postMessage` 桥。不要把它捆绑到本次修复中。
