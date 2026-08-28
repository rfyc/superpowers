# 测试 Superpowers

Superpowers 有两种不同类型的测试，每种都在自己的目录中：

- **`tests/`** —— 插件的非 LLM 代码是否正常工作？针对 brainstorm-server JS、OpenCode 插件加载、codex-plugin 同步和分析工具的 Bash + node + python 集成测试。
- **`evals/`** —— 代理在真实的 LLM 会话中是否表现正确？Python 宿主环境驱动 Claude Code / Codex / Gemini CLI 的真实 tmux 会话，由 LLM 演员和验证器评判技能合规性。

## 插件测试

位于 `tests/`。目前包括：

- `tests/brainstorm-server/` —— brainstorm server JS 代码的 node 测试套件。
- `tests/opencode/` —— 针对 OpenCode 插件加载、引导缓存和工具注册的 bash 测试。
- `tests/codex-plugin-sync/` —— bash 同步验证。
- `tests/kimi/` —— 针对 Kimi 插件清单接线的 bash/Python 检查。
- `tests/claude-code/test-helpers.sh`、`analyze-token-usage.py` —— 其余 bash 测试使用的工具。
- `tests/claude-code/test-subagent-driven-development.sh` —— 代理能描述 SDD 的测试（没有 drill 对应版本；测试的是描述记忆，而非行为）。
- `tests/claude-code/test-subagent-driven-development-integration.sh` —— 扩展的 SDD 集成测试，带 token 分析（drill 覆盖 YAGNI 子集；bash 增加提交计数、Claude Code 任务跟踪和 token 遥测断言）。
- `tests/claude-code/test-worktree-native-preference.sh` —— 针对 worktree 技能的 RED-GREEN-REFACTOR 验证（drill 覆盖 PRESSURE 阶段；bash 还覆盖 RED/GREEN 基线）。
- `tests/explicit-skill-requests/` —— 针对 Haiku 特定、多轮和技能名提示的测试，这些未被 drill 覆盖。

通过相关目录的 `run-*.sh` 或 `npm test` 运行插件测试。

## 技能行为评估

位于 `evals/`。Drill 是宿主环境；场景位于 `evals/scenarios/*.yaml`。设置方法见 `evals/README.md`。快速开始：

```bash
cd evals
uv sync --extra dev
export ANTHROPIC_API_KEY=sk-...
uv run drill run triggering-test-driven-development -b claude
```

Drill 场景很慢（每个 3-30+ 分钟），并运行真实的 LLM 会话。它们目前不属于 CI 的一部分；自然的后续方案是分层模型（PR 上跑快速子集，每晚加按需跑完整扫描）。
