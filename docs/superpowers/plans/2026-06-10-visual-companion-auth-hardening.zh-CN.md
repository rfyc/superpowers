# Visual Companion 认证加固实施计划

> **给 agent 工作者的提示：** 必需的子技能：使用 superpowers:subagent-driven-development（推荐）或 superpowers:executing-plans 来逐任务实施本计划。步骤使用复选框（`- [ ]`）语法进行跟踪。

**目标：** 加固 brainstorming 可视化伴侣（visual companion）的认证与重连流程，同时保留受信任的同源（same-origin）屏幕 JavaScript 与未来的本地化（vendored）UI 库。

**架构：** 带 key 的根路径加载成为引导（bootstrap）步骤：设置 cookie、将 key 存入标签页作用域的 `sessionStorage`，并导航到裸的 `/` 屏幕 URL。WebSocket 需要有效认证加上浏览器同源的 `Origin`，而 `/files/*` 使用 realpath 包含（containment）来防止内容目录逃逸。

**技术栈：** Node.js 内置模块（`http`、`fs`、`path`、`crypto`），零运行时依赖，现有 `ws` 测试依赖，Bash 启动/停止脚本，仓库 shell lint 脚本。

**重要提示：** 除非 Drew 明确要求，否则执行期间不要提交。本仓库的说明优先于通用计划模板的提交节奏。

---

## 文件映射

- 修改：`skills/brainstorming/scripts/server.cjs`
  - 添加引导（bootstrap）响应。
  - 添加共享安全响应头。
  - 添加 WebSocket Origin 校验。
  - 添加 `/files/*` realpath 包含。
- 修改：`skills/brainstorming/scripts/helper.js`
  - 读取存储的会话 key，并将其附加到 WebSocket URL。
- 修改：`tests/brainstorm-server/auth.test.js`
  - 添加引导、响应头、同源 WS、跨源 WS 以及 cookie/文件认证回归测试。
- 修改：`tests/brainstorm-server/helper.test.js`
  - 为基于 sessionStorage 的 WS URL 添加 mock 浏览器覆盖。
- 修改：`tests/brainstorm-server/server.test.js`
  - 为 `/files/*` 添加符号链接包含回归测试。
- 修改：`tests/brainstorm-server/lifecycle.test.js`
  - 让 start-server 超时标志测试强制使用后台模式。
  - 如果符合现有 lifecycle helper 的模式，添加重启重连凭证覆盖。
- 修改：`skills/brainstorming/scripts/start-server.sh`
  - 修复 shell lint。
- 修改：`skills/brainstorming/scripts/stop-server.sh`
  - 修复 shell lint。
- 修改：`.gitignore`
  - 添加 `.superpowers/`。
- 可选文档更新：`skills/brainstorming/visual-companion.md`
  - 如果代码行为变化需要面向操作者的说明，提及引导 URL 剥离与受信任的同源屏幕 JS。

## 任务 1：带 Key 的根路径加载引导

**文件：**
- 修改：`tests/brainstorm-server/auth.test.js`
- 修改：`skills/brainstorming/scripts/server.cjs`

- [ ] **步骤 1：为引导行为添加 RED 测试**

在 `tests/brainstorm-server/auth.test.js` 中，在现有有效 key 根路径测试之后添加测试：

```js
    await test('GET / with valid query returns bootstrap instead of screen content', async () => {
      const res = await get('/', { key: TOKEN });
      assert.strictEqual(res.status, 200);
      assert(res.body.includes('sessionStorage'), 'bootstrap should store the session key in tab storage');
      assert(res.body.includes('location.replace'), 'bootstrap should navigate to the bare root URL');
      assert(!res.body.includes('Secret screen'), 'bootstrap must not serve screen HTML at the keyed URL');
    });

    await test('GET / with valid cookie serves the screen after bootstrap', async () => {
      const res = await get('/', { cookie: `${COOKIE_NAME}=${TOKEN}` });
      assert.strictEqual(res.status, 200);
      assert(res.body.includes('Secret screen'), 'cookie-authenticated bare root should serve the screen');
      assert(!res.body.includes('sessionStorage'), 'bare screen response should not be the bootstrap page');
    });
```

如果已有 cookie 测试，保留它；合并断言而不是重复相同的测试名。

- [ ] **步骤 2：验证 RED**

运行：

```bash
cd /Users/drewritter/prime-rad/superpowers/tests/brainstorm-server
node auth.test.js
```

预期：新的引导测试失败，因为当前 `GET /?key=...` 直接返回 `Secret screen`，并且不包含引导用的 `sessionStorage`/`location.replace` 代码。

- [ ] **步骤 3：实现最小的引导响应**

在 `skills/brainstorming/scripts/server.cjs` 中，在页面常量附近添加一个 helper：

```js
function bootstrapPage(key) {
  const jsonKey = JSON.stringify(String(key));
  return `<!DOCTYPE html>
<html>
<head><meta charset="utf-8"><title>Opening Brainstorm Companion</title></head>
<body>
<script>
sessionStorage.setItem('brainstorm-session-key', ${jsonKey});
location.replace('/');
</script>
</body>
</html>`;
}
```

然后在 `handleRequest` 中，在授权与设置 cookie 之后、返回屏幕 HTML 之前，检测根路径上的有效查询 key：

```js
function queryKey(url) {
  const q = url.indexOf('?');
  if (q < 0) return null;
  return new URLSearchParams(url.slice(q + 1)).get('key');
}
```

在 `handleRequest` 中使用它：

```js
  const pathname = pathnameOf(req.url);
  const keyFromQuery = queryKey(req.url);
  if (req.method === 'GET' && pathname === '/' && keyFromQuery && timingSafeEqualStr(keyFromQuery, TOKEN)) {
    res.writeHead(200, securityHeaders({ 'Content-Type': 'text/html; charset=utf-8' }));
    res.end(bootstrapPage(keyFromQuery));
    return;
  }
```

这假设任务 4 会引入 `securityHeaders`。如果先实现任务 1，临时使用：

```js
    res.writeHead(200, { 'Content-Type': 'text/html; charset=utf-8' });
```

并在任务 4 中替换它。

- [ ] **步骤 4：验证 GREEN**

运行：

```bash
cd /Users/drewritter/prime-rad/superpowers/tests/brainstorm-server
node auth.test.js
```

预期：所有认证测试通过，包括新的引导测试。

## 任务 2：WebSocket Origin 强制校验

**文件：**
- 修改：`tests/brainstorm-server/auth.test.js`
- 修改：`skills/brainstorming/scripts/server.cjs`

- [ ] **步骤 1：为同源与跨源 WS 添加 RED 测试**

在 `tests/brainstorm-server/auth.test.js` 中，扩展 `wsConnect` 以接受 `origin` 选项：

```js
function wsConnect({ key, cookie, origin } = {}) {
  const url = `ws://localhost:${TEST_PORT}/` + (key !== undefined ? `?key=${key}` : '');
  const headers = {};
  if (cookie) headers['Cookie'] = cookie;
  if (origin) headers['Origin'] = origin;
  const ws = new WebSocket(url, Object.keys(headers).length ? { headers } : {});
  return new Promise((resolve) => {
    let settled = false;
    const done = (outcome) => { if (!settled) { settled = true; resolve({ outcome, ws }); } };
    ws.on('open', () => done('opened'));
    ws.on('error', () => done('rejected'));
    ws.on('close', () => done('rejected'));
    setTimeout(() => done('rejected'), 1500);
  });
}
```

然后添加：

```js
    await test('WS upgrade with valid cookie and same-origin Origin opens', async () => {
      const { outcome, ws } = await wsConnect({
        cookie: `${COOKIE_NAME}=${TOKEN}`,
        origin: `http://localhost:${TEST_PORT}`
      });
      ws.close();
      assert.strictEqual(outcome, 'opened');
    });

    await test('WS upgrade with valid cookie but cross-origin Origin is rejected', async () => {
      const eventsFile = path.join(TEST_DIR, 'state', 'events');
      if (fs.existsSync(eventsFile)) fs.unlinkSync(eventsFile);

      const { outcome, ws } = await wsConnect({
        cookie: `${COOKIE_NAME}=${TOKEN}`,
        origin: 'http://localhost:9999'
      });
      if (outcome === 'opened') {
        ws.send(JSON.stringify({ type: 'choice', choice: 'attacker-injected', text: 'local attacker probe' }));
        await sleep(300);
      }
      ws.close();

      assert.strictEqual(outcome, 'rejected', 'cross-origin browser WS must not open even with cookie');
      assert(!fs.existsSync(eventsFile), 'cross-origin WS must not write state/events');
    });
```

- [ ] **步骤 2：验证 RED**

运行：

```bash
cd /Users/drewritter/prime-rad/superpowers/tests/brainstorm-server
node auth.test.js
```

预期：跨源 cookie WS 测试失败，因为当前服务器无论 Origin 如何都接受任何 cookie 认证的 WS。

- [ ] **步骤 3：实现 Origin 校验**

在 `skills/brainstorming/scripts/server.cjs` 中，添加：

```js
function isAllowedWebSocketOrigin(req) {
  const origin = req.headers.origin;
  if (!origin) return true; // non-browser clients still need the session key
  const host = req.headers.host;
  if (!host) return false;
  return origin === 'http://' + host;
}
```

然后更新 `handleUpgrade`：

```js
function handleUpgrade(req, socket) {
  if (!isAuthorized(req) || !isAllowedWebSocketOrigin(req)) { socket.destroy(); return; }
```

- [ ] **步骤 4：验证 GREEN**

运行：

```bash
cd /Users/drewritter/prime-rad/superpowers/tests/brainstorm-server
node auth.test.js
```

预期：认证测试通过；跨源 WS 被拒绝；同源与直接使用 key 的 WS 仍然打开。

## 任务 3：Helper 使用存储的 key 进行重连

**文件：**
- 修改：`tests/brainstorm-server/helper.test.js`
- 修改：`skills/brainstorming/scripts/helper.js`

- [ ] **步骤 1：为 WebSocket URL key 添加 RED 测试**

在 `tests/brainstorm-server/helper.test.js` 中，在重连状态机测试附近添加一个 mock 浏览器测试：

```js
test('uses sessionStorage key in the WebSocket URL when present', () => {
  const e = makeEnv();
  e.state.sessionKey = 'stored-key-abc';
  e.boot();
  assert.strictEqual(e.sockets[0].url, 'ws://localhost:7777/?key=stored-key-abc');
});
```

更新 `makeEnv()`，使返回的对象暴露 `sockets`，并且 mock window 包含 sessionStorage：

```js
    window: {
      location: { host: 'localhost:7777', reload() { state.reloads++; } },
      sessionStorage: { getItem: (key) => key === 'brainstorm-session-key' ? state.sessionKey : null }
    },
```

再添加一个回退测试：

```js
test('uses cookie-only WebSocket URL when no sessionStorage key is present', () => {
  const e = makeEnv();
  e.state.sessionKey = null;
  e.boot();
  assert.strictEqual(e.sockets[0].url, 'ws://localhost:7777');
});
```

- [ ] **步骤 2：验证 RED**

运行：

```bash
cd /Users/drewritter/prime-rad/superpowers/tests/brainstorm-server
node helper.test.js
```

预期：存储 key 测试失败，因为当前 helper 使用 `ws://localhost:7777`。

- [ ] **步骤 3：实现存储 key 的 WS URL**

在 `skills/brainstorming/scripts/helper.js` 中，将：

```js
  const WS_URL = 'ws://' + window.location.host;
```

替换为：

```js
  function websocketUrl() {
    let key = null;
    try { key = window.sessionStorage && window.sessionStorage.getItem('brainstorm-session-key'); } catch (e) {}
    return 'ws://' + window.location.host + (key ? '/?key=' + encodeURIComponent(key) : '');
  }
```

然后将：

```js
    ws = new WebSocket(WS_URL);
```

替换为：

```js
    ws = new WebSocket(websocketUrl());
```

- [ ] **步骤 4：验证 GREEN**

运行：

```bash
cd /Users/drewritter/prime-rad/superpowers/tests/brainstorm-server
node helper.test.js
```

预期：helper 测试通过。

## 任务 4：安全响应头

**文件：**
- 修改：`tests/brainstorm-server/auth.test.js`
- 修改：`skills/brainstorming/scripts/server.cjs`

- [ ] **步骤 1：添加 RED 响应头测试**

在 `tests/brainstorm-server/auth.test.js` 中，添加：

```js
    await test('HTML responses include leak-reduction and anti-framing headers', async () => {
      const res = await get('/', { key: TOKEN });
      assert.strictEqual(res.headers['referrer-policy'], 'no-referrer');
      assert.strictEqual(res.headers['cache-control'], 'no-store');
      assert.strictEqual(res.headers['x-frame-options'], 'DENY');
      assert.strictEqual(res.headers['content-security-policy'], "frame-ancestors 'none'");
      assert.strictEqual(res.headers['cross-origin-resource-policy'], 'same-origin');
    });

    await test('403 responses include leak-reduction and anti-framing headers', async () => {
      const res = await get('/');
      assert.strictEqual(res.status, 403);
      assert.strictEqual(res.headers['referrer-policy'], 'no-referrer');
      assert.strictEqual(res.headers['cache-control'], 'no-store');
      assert.strictEqual(res.headers['x-frame-options'], 'DENY');
      assert.strictEqual(res.headers['content-security-policy'], "frame-ancestors 'none'");
      assert.strictEqual(res.headers['cross-origin-resource-policy'], 'same-origin');
    });
```

- [ ] **步骤 2：验证 RED**

运行：

```bash
cd /Users/drewritter/prime-rad/superpowers/tests/brainstorm-server
node auth.test.js
```

预期：响应头测试失败，因为当前响应不包含这些响应头。

- [ ] **步骤 3：实现共享响应头 helper**

在 `skills/brainstorming/scripts/server.cjs` 中，添加：

```js
function securityHeaders(headers = {}) {
  return {
    'Referrer-Policy': 'no-referrer',
    'Cache-Control': 'no-store',
    'X-Frame-Options': 'DENY',
    'Content-Security-Policy': "frame-ancestors 'none'",
    'Cross-Origin-Resource-Policy': 'same-origin',
    ...headers
  };
}
```

更新 `handleRequest` 中的响应写入：

```js
res.writeHead(403, securityHeaders({ 'Content-Type': 'text/html; charset=utf-8' }));
```

```js
res.writeHead(200, securityHeaders({ 'Content-Type': 'text/html; charset=utf-8' }));
```

```js
res.writeHead(200, securityHeaders({ 'Content-Type': contentType }));
```

对于 404：

```js
res.writeHead(404, securityHeaders());
```

- [ ] **步骤 4：验证 GREEN**

运行：

```bash
cd /Users/drewritter/prime-rad/superpowers/tests/brainstorm-server
node auth.test.js
```

预期：认证测试通过，响应头断言变绿。

## 任务 5：`/files/*` Realpath 包含

**文件：**
- 修改：`tests/brainstorm-server/server.test.js`
- 修改：`skills/brainstorming/scripts/server.cjs`

- [ ] **步骤 1：添加 RED 符号链接逃逸测试**

在 `tests/brainstorm-server/server.test.js` 中，在 `/files/` 空名称测试之后添加：

```js
    await test('does not serve symlinks that escape content dir via /files/', async () => {
      const target = path.join(STATE_DIR, 'server-info');
      const link = path.join(CONTENT_DIR, 'linked-server-info.txt');
      try { fs.unlinkSync(link); } catch (e) {}
      fs.symlinkSync(target, link);

      const res = await fetch(`http://localhost:${TEST_PORT}/files/linked-server-info.txt`);
      assert.strictEqual(res.status, 404, 'symlink to state/server-info must not be served');
      assert(!res.body.includes('server-started'), 'response must not include server-info body');
    });
```

- [ ] **步骤 2：验证 RED**

运行：

```bash
cd /Users/drewritter/prime-rad/superpowers/tests/brainstorm-server
node server.test.js
```

预期：符号链接测试失败，因为当前 `/files/*` 会跟随符号链接并返回 `server-info`。

- [ ] **步骤 3：实现包含 helper**

在 `skills/brainstorming/scripts/server.cjs` 中，添加：

```js
function isRegularFileInsideContentDir(filePath) {
  let stat, realContentDir, realFilePath;
  try {
    stat = fs.lstatSync(filePath);
    if (stat.isSymbolicLink()) return false;
    if (!stat.isFile()) return false;
    realContentDir = fs.realpathSync(CONTENT_DIR);
    realFilePath = fs.realpathSync(filePath);
  } catch (e) {
    return false;
  }
  return realFilePath.startsWith(realContentDir + path.sep);
}
```

将 `/files/*` 守卫替换为：

```js
    if (!fileName || fileName.startsWith('.') || !isRegularFileInsideContentDir(filePath)) {
      res.writeHead(404, securityHeaders());
      res.end('Not found');
      return;
    }
```

- [ ] **步骤 4：验证 GREEN**

运行：

```bash
cd /Users/drewritter/prime-rad/superpowers/tests/brainstorm-server
node server.test.js
```

预期：服务器测试通过，包括符号链接拒绝。

## 任务 6：重启重连回归测试

**文件：**
- 修改：`tests/brainstorm-server/lifecycle.test.js`
- 修改：`skills/brainstorming/scripts/server.cjs`
- 修改：`skills/brainstorming/scripts/helper.js`

- [ ] **步骤 1：为重启后通过 WS 使用相同 key 添加 RED 集成测试**

在 `tests/brainstorm-server/lifecycle.test.js` 中，在端口/令牌持久化测试之后添加一个测试：

```js
  await test('stored key can authenticate WebSocket after same-port restart', async () => {
    const dir = fs.mkdtempSync('/tmp/bs-reconnect-');
    const portFile = path.join(dir, '.last-port');
    const tokenFile = path.join(dir, '.last-token');
    const env = { ...process.env, BRAINSTORM_PORT_FILE: portFile, BRAINSTORM_TOKEN_FILE: tokenFile, BRAINSTORM_LIFECYCLE_CHECK_MS: 100000 };

    const a = spawn('node', [SERVER], { env: { ...env, BRAINSTORM_DIR: path.join(dir, 's1') } });
    let outA = ''; a.stdout.on('data', d => outA += d.toString());
    for (let i = 0; i < 60 && !outA.includes('server-started'); i++) await sleep(50);
    const infoA = firstServerStarted(outA);
    const keyA = new URL(infoA.url).searchParams.get('key');
    a.kill(); await sleep(400);

    const b = spawn('node', [SERVER], { env: { ...env, BRAINSTORM_DIR: path.join(dir, 's2') } });
    let outB = ''; b.stdout.on('data', d => outB += d.toString());
    for (let i = 0; i < 60 && !outB.includes('server-started'); i++) await sleep(50);
    const infoB = firstServerStarted(outB);

    const ws = new WebSocket(`ws://localhost:${infoB.port}/?key=${keyA}`, {
      headers: { Origin: `http://localhost:${infoB.port}` }
    });
    const opened = await new Promise(resolve => {
      ws.on('open', () => resolve(true));
      ws.on('error', () => resolve(false));
      setTimeout(() => resolve(false), 1500);
    });

    try {
      assert.strictEqual(infoB.port, infoA.port, 'restart should reuse same port');
      assert(opened, 'stored key should authenticate WS after restart');
    } finally {
      try { ws.close(); } catch (e) {}
      b.kill(); await sleep(100);
      fs.rmSync(dir, { recursive: true, force: true });
    }
  });
```

一旦任务 2 和任务 3 实现，此测试可能已经通过。如果它在代码更改之前就通过，保留它作为覆盖，但不要称之为 RED。真实的浏览器重连行为主要由任务 3 加上最终的手动/无头浏览器验证覆盖。

- [ ] **步骤 2：验证行为**

运行：

```bash
cd /Users/drewritter/prime-rad/superpowers/tests/brainstorm-server
node lifecycle.test.js
```

任务 2 和任务 3 之后预期：lifecycle 测试通过。如果失败，在继续之前修复认证/重启路径。

## 任务 7：Lifecycle 挂起与 Shell Lint

**文件：**
- 修改：`tests/brainstorm-server/lifecycle.test.js`
- 修改：`skills/brainstorming/scripts/start-server.sh`
- 修改：`skills/brainstorming/scripts/stop-server.sh`

- [ ] **步骤 1：复现 shell lint 失败**

运行：

```bash
cd /Users/drewritter/prime-rad/superpowers
scripts/lint-shell.sh skills/brainstorming/scripts/start-server.sh skills/brainstorming/scripts/stop-server.sh tests/brainstorm-server/stop-server.test.sh
```

预期的当前失败：

```text
SC2164: skills/brainstorming/scripts/start-server.sh line 128: cd "$SCRIPT_DIR"
SC2034: skills/brainstorming/scripts/start-server.sh line 166: for i in {1..50}
SC2034: skills/brainstorming/scripts/stop-server.sh line 57: for i in {1..20}
```

- [ ] **步骤 2：最小化修复 shell lint**

在 `skills/brainstorming/scripts/start-server.sh` 中，将：

```bash
cd "$SCRIPT_DIR"
```

改为：

```bash
cd "$SCRIPT_DIR" || exit 1
```

将未使用的循环变量从 `i` 改为 `_`（在未被读取的地方）：

```bash
for _ in {1..50}; do
```

在 `skills/brainstorming/scripts/stop-server.sh` 中，将：

```bash
for i in {1..20}; do
```

改为：

```bash
for _ in {1..20}; do
```

- [ ] **步骤 3：修复 lifecycle start-server 挂起**

在 `tests/brainstorm-server/lifecycle.test.js` 中，更新 `start-server.sh --idle-timeout-minutes sets the timeout` 测试命令：

```js
const out = execFileSync('bash', [START, '--project-dir', dir, '--idle-timeout-minutes', '5', '--background'], { encoding: 'utf8' });
```

这可以防止当 `CODEX_CI` 触发 start-server 前台模式时测试挂起。

- [ ] **步骤 4：验证 lint 与 lifecycle**

运行：

```bash
cd /Users/drewritter/prime-rad/superpowers
scripts/lint-shell.sh skills/brainstorming/scripts/start-server.sh skills/brainstorming/scripts/stop-server.sh tests/brainstorm-server/stop-server.test.sh
cd tests/brainstorm-server
node lifecycle.test.js
```

预期：shell lint 以 0 退出；lifecycle 测试以 0 退出且不挂起。

## 任务 8：Gitignore 持久的 Companion 状态

**文件：**
- 修改：`.gitignore`

- [ ] **步骤 1：验证当前的忽略缺口**

运行：

```bash
cd /Users/drewritter/prime-rad/superpowers
git check-ignore .superpowers/brainstorm/.last-token || true
```

预期的当前输出：无匹配的忽略规则。

- [ ] **步骤 2：添加忽略规则**

在 `.gitignore` 中添加这一行：

```gitignore
.superpowers/
```

- [ ] **步骤 3：验证 GREEN**

运行：

```bash
cd /Users/drewritter/prime-rad/superpowers
git check-ignore .superpowers/brainstorm/.last-token
```

预期输出：

```text
.superpowers/brainstorm/.last-token
```

## 任务 9：完整的自动化验证

**文件：**
- 本任务无代码更改。

- [ ] **步骤 1：运行聚焦的测试套件**

运行：

```bash
cd /Users/drewritter/prime-rad/superpowers/tests/brainstorm-server
node auth.test.js
node helper.test.js
node server.test.js
node lifecycle.test.js
```

预期：四个命令都以 0 退出。

- [ ] **步骤 2：运行完整的 brainstorm-server 测试套件**

运行：

```bash
cd /Users/drewritter/prime-rad/superpowers/tests/brainstorm-server
npm test
```

预期：所有测试通过，包括 ws-protocol、helper、auth、server、lifecycle 和 stop-server。

- [ ] **步骤 3：重复测试套件以排查 lifecycle/watch 不稳定**

运行：

```bash
cd /Users/drewritter/prime-rad/superpowers/tests/brainstorm-server
for i in 1 2 3; do npm test || exit 1; done
```

预期：所有三次重复都通过且不挂起。

- [ ] **步骤 4：运行 shell lint**

运行：

```bash
cd /Users/drewritter/prime-rad/superpowers
scripts/lint-shell.sh skills/brainstorming/scripts/start-server.sh skills/brainstorming/scripts/stop-server.sh tests/brainstorm-server/stop-server.test.sh
```

预期：以 0 退出。

## 任务 10：重新运行安全探测

**文件：**
- 本任务无代码更改。

- [ ] **步骤 1：重建跨源攻击者探测**

如果可用，使用之前的临时探测：

```bash
node /tmp/superpowers-pr1720-security-drewritter/probe-pr1720.cjs
```

如果临时探测不可用，在 `/tmp` 下重建一个最小探测，它需要：

- 使用固定令牌启动 companion
- 在无头 Chrome 中加载带 key 的 URL
- 在另一个 localhost 端口启动攻击者页面
- 尝试 `new WebSocket('ws://localhost:<companion-port>/')`
- 发送 `{"type":"choice","choice":"attacker-injected"}`
- 检查 `state/events`

修复后的预期：

- 无 key 与错误 key 的 HTTP 仍然返回 403
- 同源 helper 达到 Connected
- 跨源 WebSocket 无法打开
- `state/events` 不包含 `attacker-injected`
- 指向 `server-info` 的符号链接返回 404
- 带 key 的浏览器加载以裸 `/` 结束

- [ ] **步骤 2：仅在自动化探测通过后重跑手动/浏览器流程**

手动流程：

1. 使用 `--project-dir --open` 启动 companion
2. 推送一个屏幕
3. 确认 URL 剥离为 `/`
4. 确认状态达到 Connected
5. 点击一个选择并验证 `state/events`
6. 停止并重启同一项目
7. 验证打开的标签页自动重连

预期：所有步骤通过，无需手动刷新 URL。

## 自审清单

- 规格覆盖：每个设计需求都映射到至少一个任务。
- 占位符扫描：本计划不包含未解决的占位符标记或未指定的边界情况步骤。
- TDD 顺序：每个生产更改任务都以一个聚焦的失败测试或一个能演示当前失败的命令开始。
- 信任模型：本计划保留受信任的同源屏幕 JavaScript 与未来的同源本地化（vendored）库。
- 不提交规则：除非 Drew 明确要求，否则执行期间不提交。
