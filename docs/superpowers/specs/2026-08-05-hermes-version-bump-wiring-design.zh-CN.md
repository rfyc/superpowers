# Hermes 版本升级接线设计

**日期：** 2026-08-05
**修订：** 2026-08-06
**状态：** 已批准

## 目标

把 `.hermes-plugin/plugin.yaml` 登记到 `.version-bump.json` 中，并教会 `scripts/bump-version.sh` 处理 YAML，而无需在 Bash 中实现 YAML 解析器，从而使其与仓库版本保持同步。

## 设计

- 向 `.version-bump.json` 添加 `{ "path": ".hermes-plugin/plugin.yaml", "field": "version" }`。
- 让 `.json` 走现有的 `jq` 辅助函数，`.yaml` 走 Mike Farah `yq` v4。YAML key 和值作为数据传入，而不是被插值进表达式。
- 只支持存在的顶层 YAML 字符串字段。嵌套字段和 `.yml` 在范围之外。
- 让 `--check`、`--audit` 和版本更新都走同一个小的读/写分派器。
- 在版本升级写入任何 manifest 之前，运行一次只读预检：验证所需工具，并通过分派器读取每个已声明存在的 manifest。这防止确定性的 YAML 失败发生在较早的 JSON 文件已被更新之后。缺失文件的行为保持不变，且 `--help` 在没有 `jq` 或 `yq` 时仍可工作。

预检是唯一新增的可靠性措施。它不会让脚本变为事务性的，也不会重新设计其现有的 audit 和错误状态行为。

## 测试

三个重点行为测试针对一个隔离的临时 fixture 运行真实脚本，并证明：

- 对齐的 JSON 和 YAML 通过 `--check` 和 `--audit`，且一次升级会同时更新两种格式；
- 一次真实的升级（JSON 声明在前，随后一个顶层 `version` 不是字符串的 YAML manifest）以非零退出，并让每个 manifest 逐字节保持不变；且
- 真实的 `.version-bump.json` 登记了 Hermes manifest。

验证还会针对仓库运行 shell lint 和 `scripts/bump-version.sh --check`。

## 非目标

- 不手写 YAML 解析器。
- 不支持 `.yml` 或嵌套 YAML。
- 不改 Hermes 运行时。
- 不做回滚框架、通用配置 schema 层、audit/status 重构，或穷举失败矩阵。
- 不改变审查期间发现的、独立的版本验证和 JSON 表达式问题。
