# 平台中立的 README 排序——Phase C 设计

## 背景

Phase A 和 Phase B（见 `2026-05-05-platform-neutral-prose-design.md` 和 `2026-05-05-platform-neutral-config-refs-design.md`）已经中和了 README 中通用的 Claude 行文和配置文件引用。剩余的平台倾向信号是布局：README 的两个平台列表把 Claude Code 排在首位，且其他位置并非严格按字母顺序。

本阶段修复排序。不改动任何行文。

## 范围内

1. **Quickstart 平台列表**（`README.md:7`）——受支持 harness 的内联链接列表
2. **安装部分排序**（`README.md:35–152`）——每个 harness 的安装子部分

## 范围外

- 行文、市场名称、插件 ID、URL——现状事实正确。
- Claude Code 部分的视觉权重（它有两个子部分——官方 Anthropic 市场与 Superpowers 市场）。两者都是真实安装路径；折叠它们会隐藏准确信息。
- 每个安装块内的章节标题和内容——只有块的顺序改变。

## 替换

两个列表都重排为严格字母顺序：

| 旧顺序 | 新顺序 |
|-----------|-----------|
| Claude Code | Claude Code |
| Codex CLI | Codex App |
| Codex App | Codex CLI |
| Factory Droid | Cursor |
| Gemini CLI | Factory Droid |
| OpenCode | Gemini CLI |
| Cursor | GitHub Copilot CLI |
| GitHub Copilot CLI | OpenCode |

三次移动：Codex App 与 Codex CLI 互换；Cursor 上移两格；GitHub Copilot CLI 上移一格。

Claude Code 因字母巧合（`Cl…` 先于 `Co…`）而保持第一。

## 提交计划

一次原子提交覆盖两个列表，因为只改其中一个会造成 quickstart 与安装部分之间的不一致。

## 验证

- Quickstart 锚点（`#claude-code`、`#codex-app` 等）仍解析到现有的 `### …` 标题——未重命名任何标题。
- 每个安装子部分的正文前后逐字节相同；只有位置改变。
- `git diff README.md` 只显示章节移动，没有内容编辑。
