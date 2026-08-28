# Visual Companion 最终加固修正设计

**日期：** 2026-06-11
**状态：** 草稿，待 Drew 审阅

## 目标

完成 PR #1720 visual companion 的加固工作，使分支具备干净的安全行为、确定性的测试，且 PR diff 只包含 companion 相关工作，从而可以交给 Jesse 审阅。

这是在现有认证加固设计之上的修正。不应重新设计 companion，也不应扩展功能面。

## 背景

之前的加固工作新增了带 key 的会话、WebSocket 同源检查、URL key 剥离、`/files/*` 包含、减少泄漏的响应头、IPv6 URL 格式化、Windows 生命周期覆盖，以及 PR 证据更新。

最终审阅环节发现了五个剩余问题：

1. 根路由 `GET /` 的屏幕选择路径仍可能提供 `content/` 下指向内容目录之外的符号链接或硬链接。
2. 当首选端口被占用时，回退服务器可能复用持久化的 `.last-token`，造成两个活的、同项目的 companion 服务器持有同一个 bearer key。
3. 在缺少强所有权证明时，`stop-server.sh` 可能向一个无关的 `node server.cjs` 进程发送信号。
4. 一些测试可能针对错误的回退进程通过，失败时泄漏后台进程，或假设类 Windows 主机上支持符号链接。
5. PR 当前存在冲突，因为分支包含一个较早的 `evals` 子模块升级，而该升级已另行处理。

## 非目标

- 本轮不添加 HTTPS 隧道或 `wss://` 源语义。
- 不实现 opt-out、自由文本或对比度辅助（contrast-helper）等 companion 功能。
- 不 vendoring Alpine、Three.js 或任何其他 JavaScript 库。
- 不尝试沙箱化恶意的 agent 编写的屏幕 HTML。
- 不为陈旧的 stop-server PID 文件添加向后兼容，除非 Drew 明确批准这一权衡。

## 继承的安全不变量

本修正保留已设计并实现的认证加固：

- `.last-token` 和 `state/server-info` 仍是敏感的、仅属主可见的状态。
- 回退 token 可能出现在启动 JSON 和 `state/server-info` 中，但绝不能写入 `.last-token`。
- cookie 仍按端口命名，为 `HttpOnly`、`SameSite=Strict`，且作用域限定为 `/`。
- WebSocket 升级仍要求有效的 key 或 cookie。
- 当浏览器提供 `Origin` 头时，WebSocket 的 `Origin` 检查仍被强制执行。
- 直接的无 `Origin` 客户端仅在其携带会话 key 时才被允许。
- 生成的同源屏幕 JavaScript 以及未来的同源 vendored 库被视为可信。沙箱化恶意的屏幕 HTML 仍被延后处理。

## 设计

### 1. 变基到当前的 `dev`

在实现工作之前，将 `brainstorming-companion` 变基到当前的 `origin/dev`。取 `dev` 以解决 `evals` 子模块冲突。

变基之后：

- `evals` 不得出现在 PR diff 中。
- PR #1720 仍可提及在别处运行的 eval 证据，但必须包含精确的外部证据：eval 仓库提交、场景路径、命令、结果产物路径或 id，以及 RED/GREEN 结果。
- PR 正文不得暗示 evals 子模块升级属于本 PR。
- 任何暗示该子模块升级已包含在内的早期 PR 正文或评论，都必须由最终的 PR 正文证据取代。

### 2. 根屏幕包含

根屏幕路由必须使用与 `/files/*` 相同的包含边界。

`getNewestScreen()` 应忽略任何未通过"内容目录内常规文件"守卫的 `.html` 候选文件。该守卫必须解析真实路径，并确保所服务的文件位于 `CONTENT_DIR` 内部。它还必须保留现有的硬链接防护：当平台报告链接计数时，拒绝链接计数不为 1 的文件。

预期行为：

- 忽略 `content/` 下指向 `content/` 之外的符号链接。
- 当 `fs.linkSync` 成功且 `lstat.nlink > 1` 时，忽略 `content/` 下指向 `state/server-info` 的硬链接。
- 如果没有安全的屏幕文件剩余，则提供等待页面。
- 现有的 `/files/*` 包含行为保持不变：空名称、点文件、符号链接、硬链接和目录仍返回 404。

### 3. 回退 token 隔离

端口回退不得复用从持久化 `.last-token` 加载的 token。

代码中的 token 来源应明确：

- 来自环境的 `BRAINSTORM_TOKEN` 是有意的操作/测试覆盖。如果设置了显式环境 token 而首选端口被占用，服务器必须失败关闭（fail closed）而非回退，因为占用端口的服务器可能正在使用同一个显式 token。
- `.last-token` 是为了同端口重连便利而持久化的状态。如果服务器因首选端口被占用而回退，应丢弃该加载的 token，并为回退进程生成一个全新的、未持久化的 token。
- 一个不是从 `.last-token` 加载的新生成 token 可以在同一进程内复用，因为已知没有其他存活的进程持有它。

回退服务器必须继续避免覆盖 `.last-port` 和 `.last-token`。

### 4. Stop-Server 所有权证明

`start-server.sh` 应为每次启动创建一个服务器实例 id，并作为惰性命令行参数传给 Node，例如：

```text
node server.cjs --brainstorm-server-id=<id>
```

该 id 不是认证凭据。它只是供本地生命周期脚本使用的进程所有权证据。`server.cjs` 可以忽略该参数。

id 必须使用 shell/MSYS 安全的字母表，例如 `^[A-Za-z0-9_-]{32,64}$`。将其以仅属主权限存储在 `state/server-instance-id` 中。

`stop-server.sh` 应从状态中读取预期 id，并且仅当目标进程 argv 以完整 argv token（而非松散的子串）包含精确参数 `--brainstorm-server-id=<id>` 时，才向该 PID 发送信号。优先使用可用的 `/proc/<pid>/cmdline`，然后回退到宽格式的 `ps` 输出。即使 `server-info` 缺失或 `lsof` 不可用，匹配的实例 id 也足以作为证明。现有的端口到 PID 检查可保留为补充证据。

当无法证明所有权时失败关闭：

- 缺少 PID 文件
- 缺少或格式错误的服务器 id
- 目标命令行不可用
- 目标命令行不包含预期 id
- 缺少新 id 的陈旧会话元数据

这刻意倾向于留下一个陈旧的进程在运行，而不是杀掉一个无关的进程。

操作员可见的结果应明确：

- 缺少 PID 文件返回 `not_running`
- 缺少或格式错误的服务器 id 返回 `stale_pid`
- 命令行不可用返回 `stale_pid`
- argv id 错误或缺失返回 `stale_pid`
- 成功停止返回 `stopped`

在 `stale_pid` 和 `stopped` 结果下，移除 `server.pid` 和 `server-instance-id`，以免未来的停止尝试继续针对同一个有歧义的进程。不要移除持久的会话内容。

### 5. 测试加固

测试应能在 macOS 以及用于验证的 Windows Git Bash 主机上确定性地通过。

必需变更：

- 固定端口套件要么在服务器报告回退端口时快速失败，要么驱动所有客户端使用所报告的启动端口。
- `stop-server.test.sh` 需要在启动任何后台进程之前设置一个顶层清理 trap。
- 符号链接特定断言应探测符号链接能力，并在主机无法创建可用的测试符号链接时仅跳过该断言。
- 创建冒名进程的测试必须断言：当生命周期元数据缺失或不足时，冒名进程存活。
- Windows/MSYS 的 start-server 测试必须断言：类 Windows 检测仍会清除 `BRAINSTORM_OWNER_PID`，在适当情况下仍会自动进入前台模式，并仍精确地传入 instance-id argv。

### 6. 文档与 PR 一致性

在 Jesse 审阅之前，核对审阅者可见的文档与 PR 元数据：

- 更新问题目录，使处置结果与本 PR 实际交付的内容一致。
- 保持自动打开文档与已实现的 `--open` 行为一致。
- 在所有地方保持记录的默认空闲超时为 4 小时。
- 变基后对照模板复核 PR 正文。
- 在 PR 正文中记录 macOS、Windows、浏览器/手动以及外部 eval 证据，并附上具体命令和结果。

## 测试策略

每个行为变更使用 TDD：

1. 新增或收紧一个重点回归测试。
2. 运行并确认其因预期原因失败。
3. 实现最小的修复。
4. 重新运行重点测试。
5. 重新运行完整的 brainstorm-server 套件。

必需的重点回归测试：

| 行为 | 测试文件 | 重点命令 | 预期 RED | 预期 GREEN |
| --- | --- | --- | --- | --- |
| 根路由忽略符号链接逃逸 | `tests/brainstorm-server/server.test.js` | `node tests/brainstorm-server/server.test.js` | 已认证的 `GET /` 提供链接到 content 外部的文件 | 响应提供等待页面或安全屏幕 |
| 根路由忽略受支持的硬链接逃逸 | `tests/brainstorm-server/server.test.js` | `node tests/brainstorm-server/server.test.js` | 已认证的 `GET /` 提供硬链接的 `server-info` | 当 `nlink > 1` 时忽略硬链接候选 |
| `/files/*` 包含保持不变 | `tests/brainstorm-server/server.test.js` | `node tests/brainstorm-server/server.test.js` | 现有包含测试回归 | 空、点文件、目录、符号链接、硬链接情形仍返回 404 |
| 持久化 token 回退会轮换 token | `tests/brainstorm-server/lifecycle.test.js` | `node tests/brainstorm-server/lifecycle.test.js` | 回退 URL key 等于持久化的首选端口 key | 回退 URL key 不同且未写入 `.last-token` |
| 显式 token 回退失败关闭 | `tests/brainstorm-server/lifecycle.test.js` | `node tests/brainstorm-server/lifecycle.test.js` | 设置 `BRAINSTORM_TOKEN` 时服务器回退 | 进程以非零退出且不启动回退 |
| 回退 key 无法向原服务器认证 | `tests/brainstorm-server/lifecycle.test.js` | `node tests/brainstorm-server/lifecycle.test.js` | 回退 key 从原端口收到 200 | 原端口拒绝回退 key |
| 正确的实例 id 允许停止 | `tests/brainstorm-server/stop-server.test.sh` | `bash tests/brainstorm-server/stop-server.test.sh` | 真实 start-server 启动的服务器存活 | 停止返回 `stopped` 且进程退出 |
| 错误、缺失、格式错误或陈旧的 id 是安全的 | `tests/brainstorm-server/stop-server.test.sh` | `bash tests/brainstorm-server/stop-server.test.sh` | 冒名进程被发信号 | 停止返回 `stale_pid` 且冒名进程存活 |
| 固定端口套件不能经由回退通过 | `tests/brainstorm-server/server.test.js`、`tests/brainstorm-server/auth.test.js` | 各自的 `node` 命令 | 测试静默地连接到回退端口 | 测试清晰地失败或有意使用所报告的端口 |
| shell 清理 trap 在失败时运行 | `tests/brainstorm-server/stop-server.test.sh` | `bash tests/brainstorm-server/stop-server.test.sh` | 失败留下子进程 | trap 回收后台子进程 |
| Windows/MSYS 启动行为保持生命周期不变量 | `tests/brainstorm-server/start-server.test.sh`、`tests/brainstorm-server/windows-lifecycle.test.sh` | 在 macOS 和 `ballmer` 上运行 `bash` 测试命令 | 属主 PID 或 argv 处理回归 | 属主 PID 被清除，前台检测成立，id argv 存在 |

每个 RED/GREEN 循环都应为 PR 正文留下一段简短证据说明：重点命令、修复前的失败断言、修复后的通过断言，以及证据是在 macOS 还是 Windows 上收集的。

## 验证

在宣布修正完成之前，运行：

- `git fetch origin dev && git rebase origin/dev`
- `git diff --quiet origin/dev...HEAD -- evals`
- `gh pr view 1720 --json mergeStateStatus,statusCheckRollup,headRefOid`
- `cd tests/brainstorm-server && npm test`
- TDD 期间使用过的相关重点测试命令
- `git diff --check`
- 对被改动的 JavaScript 文件进行 Node 语法检查
- 对被改动的 shell 文件进行 shell lint
- 在 `ballmer` 上进行 Windows 验证：完整的可运行 brainstorm-server 套件，外加独立的 Windows 生命周期探针

手动/浏览器测试只在自动化通过之后进行。

## 验收标准

- PR #1720 可以干净地变基到当前的 `dev` 上。
- `evals` 不在 PR diff 中。
- 根屏幕服务无法通过符号链接或受支持的硬链接逃逸读取 `content/` 之外的内容。
- `/files/*` 包含保护保持不变。
- 不会有回退服务器运行着可能与占用首选端口的服务器共享的 token。
- 当所有权证明缺失或存在歧义时，`stop-server.sh` 不会向无关进程发送信号。
- 当 `server-info` 或 `lsof` 不可用时，`stop-server.sh` 仍能通过匹配的实例 id 停止合法的服务器。
- 每个回归都记录了重点 RED/GREEN 证据。
- PR 正文记录了 macOS 与 Windows 的验证证据。
- PR 正文准确描述了分支中的内容以及外部收集到的证据。
