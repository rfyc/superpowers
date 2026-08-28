# Hermes 版本号更新接线实施计划

> **给 agent 工作者的提示：** 必需的子技能：使用 superpowers:subagent-driven-development（推荐）或 superpowers:executing-plans 来逐任务实施本计划。步骤使用复选框（`- [ ]`）语法进行跟踪。

**目标：** 让 Hermes YAML 清单（manifest）的版本号与其他所有已声明的发布清单保持同步。

**规格：** `docs/superpowers/specs/2026-08-05-hermes-version-bump-wiring-design.md`

**架构：** 用基于扩展名的小型分发器扩展现有发布脚本：JSON 继续通过 `jq`，而 `.yaml` 使用 Mike Farah `yq` v4。在变异的更新循环之前，通过该分发器读取每个存在的清单，使得确定性的格式或字段失败发生在第一次写入之前。

**技术栈：** 兼容 Bash 3.2 的 shell，`jq`，Mike Farah `yq` v4，现有 shell-lint 工具。

## 全局约束

- 仅支持 `.json` 与 `.yaml`；`.yml` 与其他扩展名仍不受支持。
- YAML 字段是位于顶层的字符串；嵌套的 YAML 字段不在范围内。
- 通过环境数据传递 YAML 字段与新值，绝不将二者插值进 `yq` 表达式。
- 将 `yq` 限制在维护者的发布工具中；不要添加插件运行时依赖。
- 保留现有的缺失文件行为：`--check` 报告缺失文件，bump 跳过它们。
- 仅对变异式 bump 路径做预检；不添加回滚或事务性写入。
- 不更改审计状态行为、版本校验或现有的 JSON 字段表达式实现。

---

## 文件映射

- 创建：`tests/version-bump/test-bump-version.sh`
  - 在临时 JSON/YAML 夹具中练习真实脚本，并检查真实的注册表。
- 修改：`scripts/bump-version.sh`
  - 添加 YAML 读/写 helper、格式分发与仅 bump 的读取预检。
- 修改：`.version-bump.json`
  - 在顶层字段 `version` 下注册 `.hermes-plugin/plugin.yaml`。

### 任务 1：将 Hermes 接入现有的版本号更新脚本

**文件：**
- 创建：`tests/version-bump/test-bump-version.sh`
- 修改：`scripts/bump-version.sh`
- 修改：`.version-bump.json`

**接口：**
- 消费：形状为 `{ "path": string, "field": string }` 的 `.version-bump.json` 记录。
- 产出：`read_manifest_field FILE FIELD`、`write_manifest_field FILE FIELD VALUE` 与 `preflight_manifests` Bash helper。

- [ ] **步骤 1：拉取当前的开发基线**

运行：

```bash
git fetch origin dev
```

预期：命令以 0 退出并刷新 `origin/dev`。

- [ ] **步骤 2：Rebase 任务分支**

运行：

```bash
git rebase origin/dev
```

预期：命令以 0 退出，且 `git status --short --branch` 不再报告分支落后于 `origin/dev`。

- [ ] **步骤 3：添加初始的失败行为测试**

用 happy-path 夹具与真实注册表断言创建 `tests/version-bump/test-bump-version.sh`：

```bash
#!/usr/bin/env bash
set -euo pipefail

SCRIPT_DIR="$(cd "$(dirname "$0")" && pwd)"
REPO_ROOT="$(cd "$SCRIPT_DIR/../.." && pwd)"
SCRIPT_SOURCE="$REPO_ROOT/scripts/bump-version.sh"
TEST_ROOT="$(mktemp -d)"

cleanup() {
  rm -rf "$TEST_ROOT"
}
trap cleanup EXIT

fail() {
  echo "FAIL: $*" >&2
  exit 1
}

make_fixture() {
  local repo="$1"
  local yaml_body="$2"

  mkdir -p "$repo/scripts" "$repo/.hermes-plugin"
  cp "$SCRIPT_SOURCE" "$repo/scripts/bump-version.sh"
  cat >"$repo/.version-bump.json" <<'JSON'
{
  "files": [
    { "path": "package.json", "field": "version" },
    { "path": ".hermes-plugin/plugin.yaml", "field": "version" }
  ],
  "audit": { "exclude": [] }
}
JSON
  cat >"$repo/package.json" <<'JSON'
{
  "name": "fixture",
  "version": "1.2.3"
}
JSON
  printf '%s\n' "$yaml_body" >"$repo/.hermes-plugin/plugin.yaml"
}

happy_repo="$TEST_ROOT/happy"
make_fixture "$happy_repo" $'name: superpowers\nversion: 1.2.3'

/bin/bash "$happy_repo/scripts/bump-version.sh" --check >"$TEST_ROOT/check.out"
/bin/bash "$happy_repo/scripts/bump-version.sh" --audit >"$TEST_ROOT/audit.out"
/bin/bash "$happy_repo/scripts/bump-version.sh" 2.3.4 >"$TEST_ROOT/bump.out"

[[ "$(jq -r '.version' "$happy_repo/package.json")" == "2.3.4" ]] \
  || fail "JSON manifest was not bumped"
[[ "$(yq -r '.version' "$happy_repo/.hermes-plugin/plugin.yaml")" == "2.3.4" ]] \
  || fail "YAML manifest was not bumped"

jq -e '
  any(.files[];
    .path == ".hermes-plugin/plugin.yaml" and .field == "version")
' "$REPO_ROOT/.version-bump.json" >/dev/null \
  || fail "Hermes manifest is not registered"

echo "Version-bump tests passed"
```

- [ ] **步骤 4：运行测试以验证 RED**

运行：

```bash
/bin/bash tests/version-bump/test-bump-version.sh
```

预期：在 `Version-bump tests passed` 之前 FAIL；当前的仅 JSON 读取器无法处理 YAML 夹具。

- [ ] **步骤 5：添加最小的 YAML 分发并注册 Hermes**

在 `scripts/bump-version.sh` 中，在 `write_json_field` 之后添加这些 helper：

```bash
require_tool() {
  command -v "$1" >/dev/null 2>&1 || {
    echo "error: required tool '$1' is not on PATH" >&2
    return 1
  }
}

read_yaml_field() {
  local file="$1" field="$2"
  require_tool yq || return 1
  FIELD="$field" yq -er '.[strenv(FIELD)] | select(tag == "!!str")' "$file"
}

write_yaml_field() {
  local file="$1" field="$2" value="$3"
  FIELD="$field" VALUE="$value" \
    yq -i '.[strenv(FIELD)] = strenv(VALUE)' "$file"
}

read_manifest_field() {
  local file="$1"

  case "$file" in
    *.json) read_json_field "$@" ;;
    *.yaml) read_yaml_field "$@" ;;
    *)
      echo "error: unsupported manifest format: $file" >&2
      return 1
      ;;
  esac
}

write_manifest_field() {
  local file="$1"

  case "$file" in
    *.json) write_json_field "$@" ;;
    *.yaml) write_yaml_field "$@" ;;
    *)
      echo "error: unsupported manifest format: $file" >&2
      return 1
      ;;
  esac
}
```

将三处命令路径中对 `read_json_field` 的调用替换为 `read_manifest_field`，并将 bump 路径中对 `write_json_field` 的调用替换为 `write_manifest_field`。

在 `.version-bump.json` 中，紧接 `package.json` 之后添加这一精确条目：

```json
{ "path": ".hermes-plugin/plugin.yaml", "field": "version" },
```

- [ ] **步骤 6：运行初始测试以验证 GREEN**

运行：

```bash
/bin/bash tests/version-bump/test-bump-version.sh
```

预期：以 `Version-bump tests passed` 通过。

- [ ] **步骤 7：添加失败的无部分写入回归测试**

在 `tests/version-bump/test-bump-version.sh` 的最终成功消息之前插入这个块：

```bash
invalid_repo="$TEST_ROOT/invalid"
make_fixture "$invalid_repo" $'name: superpowers\nversion: 123'
cp "$invalid_repo/package.json" "$TEST_ROOT/package.before"
cp "$invalid_repo/.hermes-plugin/plugin.yaml" "$TEST_ROOT/plugin.before"

if /bin/bash "$invalid_repo/scripts/bump-version.sh" 2.3.4 \
  >"$TEST_ROOT/invalid.out" 2>&1; then
  fail "bump accepted a non-string YAML version"
fi

cmp -s "$TEST_ROOT/package.before" "$invalid_repo/package.json" \
  || fail "JSON manifest changed before YAML validation failed"
cmp -s "$TEST_ROOT/plugin.before" "$invalid_repo/.hermes-plugin/plugin.yaml" \
  || fail "invalid YAML manifest changed"
```

- [ ] **步骤 8：运行回归测试以验证 RED**

运行：

```bash
/bin/bash tests/version-bump/test-bump-version.sh
```

预期：以 `JSON manifest changed before YAML validation failed` 失败；没有预检时，JSON 清单会在之后的 YAML 读取器拒绝其非字符串版本之前被写入。

- [ ] **步骤 9：添加仅 bump 的预检**

在 `scripts/bump-version.sh` 的 `declared_files` 之后添加这个 helper：

```bash
preflight_manifests() {
  local path field fullpath

  require_tool jq || return 1
  while IFS=$'\t' read -r path field; do
    fullpath="$REPO_ROOT/$path"
    [[ -f "$fullpath" ]] || continue

    if ! read_manifest_field "$fullpath" "$field" >/dev/null; then
      echo "error: cannot read declared manifest: $path ($field)" >&2
      return 1
    fi
  done < <(declared_files)
}
```

在版本格式校验之后、第一个 bump 输出或写入之前调用它：

```bash
  preflight_manifests

  echo "Bumping all declared files to $new_version..."
```

- [ ] **步骤 10：运行聚焦验证**

运行：

```bash
/bin/bash tests/version-bump/test-bump-version.sh
scripts/lint-shell.sh scripts/bump-version.sh tests/version-bump/test-bump-version.sh
scripts/bump-version.sh --check
git diff --check
```

预期：

- 行为测试打印 `Version-bump tests passed`。
- Shell lint 报告两个脚本均无错误。
- `--check` 列出八个声明的清单，包括 `.hermes-plugin/plugin.yaml`，全部为 `6.2.0`。
- `git diff --check` 不打印任何内容。

- [ ] **步骤 11：审查并提交实现**

运行：

```bash
git status --short
git diff -- .version-bump.json scripts/bump-version.sh tests/version-bump/test-bump-version.sh
git add .version-bump.json scripts/bump-version.sh tests/version-bump/test-bump-version.sh
git commit \
  -m "fix(release): wire Hermes into version bumps" \
  -m "Register the Hermes YAML manifest alongside the existing JSON manifests. Route manifest reads and writes by extension through jq or Mike Farah yq v4, with field names and values passed as data." \
  -m "Preflight every present manifest before the mutating bump loop so a deterministic YAML read failure cannot leave earlier JSON manifests partially updated. Cover check, audit, bump, registry wiring, and byte-for-byte no-partial-write behavior with one focused fixture test."
```

预期：提交成功，且只暂存了三个实现路径。
