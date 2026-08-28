# Pi 扩展与 Evals 实施计划

> **适用于智能体工作者：** 必读子技能：使用 superpowers:subagent-driven-development（推荐）或 superpowers:executing-plans 逐步实施本计划。步骤使用复选框（`- [ ]`）语法进行跟踪。

**目标：** 为 Superpowers 增加一流的 Pi 包支持，并将 Pi 添加为 Drill eval 后端。

**架构：** Pi 包在根 `package.json` 中声明，并加载现有的 `skills/` 以及一个小的 Pi 扩展。该扩展在会话启动和压缩（compaction）之后，将 `using-superpowers` 引导注入提供方上下文，作为 user-role 消息，并带有 Pi 专属的工具映射。Drill 增加一个 `pi` 后端、Pi 会话日志规范化以及测试。

**技术栈：** Pi TypeScript 扩展 API、Node 内置测试运行器、Drill Python eval 框架、pytest。

---

### 任务 1：Pi 包清单与扩展测试

**文件：**
- 修改：`package.json`
- 创建：`tests/pi/test-pi-extension.mjs`

- [ ] **步骤 1：编写会失败的包 / 扩展测试**

创建 `tests/pi/test-pi-extension.mjs`，其测试导入 `extensions/superpowers.ts`，注册假的 Pi 处理器，并断言：
- 根 `package.json` 的 `keywords` 包含 `pi-package`
- 根 `package.json` 具有 `pi.skills: ["./skills"]`
- 根 `package.json` 具有 `pi.extensions: ["./extensions/superpowers.ts"]`
- 该扩展注册了 `resources_discover`、`session_start`、`session_compact`、`context` 和 `agent_end`
- 启动 `context` 恰好注入一条 user-role 引导消息
- `agent_end` 清除启动注入
- `session_compact` 重新启用注入
- 该扩展不注册 `session_before_compact`

- [ ] **步骤 2：运行测试并验证 RED**

运行：`node --experimental-strip-types --test tests/pi/test-pi-extension.mjs`

预期：FAIL，因为 `extensions/superpowers.ts` 不存在且 `package.json` 缺少 `pi` 清单。

- [ ] **步骤 3：实现清单字段**

更新 `package.json`，添加 `description`、`keywords`、`pi.extensions` 和 `pi.skills`，同时保留现有的 `name`、`version`、`type` 和 `main`。

- [ ] **步骤 4：实现 `extensions/superpowers.ts`**

创建一个零运行时依赖的扩展，它：
- 从 `import.meta.url` 定位包根目录
- 读取 `skills/using-superpowers/SKILL.md`
- 剥离 YAML frontmatter
- 追加 Pi 专属的工具映射
- 通过 `resources_discover` 暴露 skills 路径
- 在 `session_start` 和 `session_compact` 时标记引导待处理
- 在 `context` 中注入一条 user-role 引导消息
- 在开头的 `compactionSummary` 消息之后插入压缩后的引导
- 在 `agent_end` 时清除待处理的引导

- [ ] **步骤 5：运行测试并验证 GREEN**

运行：`node --experimental-strip-types --test tests/pi/test-pi-extension.mjs`

预期：PASS。

### 任务 2：Pi 工具映射参考文档

**文件：**
- 创建：`skills/using-superpowers/references/pi-tools.md`
- 修改：`tests/pi/test-pi-extension.mjs`

- [ ] **步骤 1：为 Pi 参考文档编写会失败的测试**

添加断言：`skills/using-superpowers/references/pi-tools.md` 存在，并记录了 `Skill`、`Task`、`TodoWrite` 以及内置工具名的映射。

- [ ] **步骤 2：运行测试并验证 RED**

运行：`node --experimental-strip-types --test tests/pi/test-pi-extension.mjs`

预期：FAIL，因为 `pi-tools.md` 不存在。

- [ ] **步骤 3：添加 Pi 参考文档**

创建 `skills/using-superpowers/references/pi-tools.md`，解释 Pi 原生的技能、可选的 `pi-subagents`、没有规范的 todo/tasklist 插件，以及内置的小写工具。

- [ ] **步骤 4：运行测试并验证 GREEN**

运行：`node --experimental-strip-types --test tests/pi/test-pi-extension.mjs`

预期：PASS。

### 任务 3：Drill Pi 后端与会话日志规范化

**文件：**
- 创建：`evals/backends/pi.yaml`
- 修改：`evals/drill/backend.py`
- 修改：`evals/drill/engine.py`
- 修改：`evals/drill/normalizer.py`
- 修改：`evals/tests/test_backend.py`
- 修改：`evals/tests/test_normalizer.py`

- [ ] **步骤 1：编写会失败的后端 / 规范化测试**

添加 pytest 覆盖：
- `load_backend("pi")` 返回 `family == "pi"`
- Pi 后端命令以 `pi` 开头并包含 `-e ${SUPERPOWERS_ROOT}`
- Pi 的 `_resolve_log_dir()` 指向 `~/.pi/agent/sessions` 之下
- `filter_pi_logs_by_cwd()` 只保留头部 `cwd` 与场景工作目录匹配的会话文件
- `normalize_pi_logs()` 从 Pi assistant 会话条目中提取 `toolCall` 块，并将内置小写工具映射为规范名称

- [ ] **步骤 2：运行测试并验证 RED**

运行：`uv run pytest evals/tests/test_backend.py evals/tests/test_normalizer.py -q`

预期：FAIL，因为 Pi 后端和规范化器不存在。

- [ ] **步骤 3：添加 `evals/backends/pi.yaml`**

配置后端运行 `pi -e ${SUPERPOWERS_ROOT}`，使用宽松的 TUI 就绪检查、`/quit` 关闭方式，以及 Pi 会话日志位置。

- [ ] **步骤 4：实现 Pi family 支持**

更新 `Backend.family`、`Engine._resolve_log_dir`、`Engine._collect_tool_calls` 和 `normalizer.py`，加入 Pi 日志过滤与规范化。

- [ ] **步骤 5：运行测试并验证 GREEN**

运行：`uv run pytest evals/tests/test_backend.py evals/tests/test_normalizer.py -q`

预期：PASS。

### 任务 4：文档与完整验证

**文件：**
- 修改：`README.md`
- 修改：`evals/README.md`

- [ ] **步骤 1：记录 Pi 安装与 eval 后端**

将 Pi 添加到 README 快速开始 / 安装列表，并在 `evals/README.md` 中添加后端条目 / 用法。

- [ ] **步骤 2：运行验证**

运行：
```bash
node --experimental-strip-types --test tests/pi/test-pi-extension.mjs
uv run pytest evals/tests/test_backend.py evals/tests/test_setup.py evals/tests/test_normalizer.py -q
```

预期：全部测试通过。
