# tests/codex-plugin-sync/ 目录分析报告

> 对应源目录：`tests/codex-plugin-sync/`
> 对应源文件：1 个文件（`test-sync-to-codex-plugin.sh`，616 行）
> 报告定位：单文件深度分析，聚焦于 `scripts/sync-to-codex-plugin.sh` 的端到端回归测试设计

---

## 一、基本信息

| 字段 | 值 |
|------|-----|
| 目录路径 | `tests/codex-plugin-sync/` |
| 文件数量 | 1 个文件（1 个 shell 测试） |
| 总代码行数 | 616 行 |
| 测试方式 | 端到端 shell 集成测试（不依赖任何外部库） |
| 关联生产代码 | `scripts/sync-to-codex-plugin.sh`（`sync` 脚本的真实"替身"被 cp 到 fixture 中运行） |
| 测试状态 | 单文件即单测试（`test-sync-to-codex-plugin.sh`），不拆分多文件 |
| 调用入口 | 直接执行 `bash test-sync-to-codex-plugin.sh` |

`codex-plugin-sync/` 是 `tests/` 下文件数最少、但单文件最大的子目录。这与被测对象 `sync-to-codex-plugin.sh` 的复杂性高度匹配——sync 脚本横跨 7 个不同场景（dry-run、bootstrap、dirty、noop、missing-manifest、stale-ignored、help），单文件测试可以将所有场景集中维护，保持断言的可对照性。

---

## 二、目的与背景

### 2.1 测试目标

本目录存在的唯一目的，是为 `scripts/sync-to-codex-plugin.sh` 维护一组**端到端回归断言**。它不测试单元（rsync、git 单独的命令行行为），而是测试：

1. **Preview 模式（`-n`）输出是否正确**：manifest 版本号、包含的文件、排除的目录、dry-run 不修改文件
2. **Bootstrap 模式行为**：自动创建 `plugins/superpowers/` 目录，dry-run 不写盘
3. **Apply 模式（`-y`）安全门**：脏 working tree 必须被拒绝，目标分支不被切换
4. **边界情况**：缺 manifest、stale 已忽略文件、目的仓库 OpenAI 元数据保留
5. **用户界面**：help 文本随参数变更同步更新
6. **源码层断言**：检测 `sync-to-codex-plugin.sh` 内部文案（如 "regenerated inline"）是否被彻底清理

### 2.2 必要性

`sync-to-codex-plugin.sh` 是项目的"重要但不紧急"基础设施——它执行 rsync、git checkout、git push，可能误删/误推用户仓库。回归测试对这类"破坏力大"的操作尤其关键。

### 2.3 与生产代码的同步约束

测试文件第 6 行通过 `SYNC_SCRIPT_SOURCE="$REPO_ROOT/scripts/sync-to-codex-plugin.sh"` 拉取**当前仓库**的最新脚本版本。这意味着：
- 修改 sync 脚本后，测试自动覆盖新行为
- 测试永远不被"冻结"在旧版脚本上
- 测试与生产代码同步演化，不会产生"测试已绿但生产已坏"的撕裂

---

## 三、核心技术决策

### 3.1 Fake Binary 注入：模拟 `gh` 行为

```bash
write_fake_gh() {
    local bin_dir="$1"
    mkdir -p "$bin_dir"
    cat > "$bin_dir/gh" <<'EOF'
#!/usr/bin/env bash
set -euo pipefail
if [[ "${1:-}" == "auth" && "${2:-}" == "status" ]]; then
    exit 0
fi
echo "unexpected gh invocation: $*" >&2
exit 1
EOF
    chmod +x "$bin_dir/gh"
}
```

**决策点**：通过 `PATH="$fake_bin:$PATH"`，将伪造的 `gh` 放在 PATH 最前。

**理由**：
- 真实 `gh` 在 CI 环境没有认证，会失败
- 但 sync 脚本在 `--local` 模式下不应调用 `gh`
- 通过 `set -euo pipefail` 配合 `exit 1` 显式报错，**任何意外的 `gh` 调用都会让测试失败**——这是一种"被调用时立刻报警"的保险丝

### 3.2 `set +e` 包裹的多场景捕获

```bash
set +e
preview_output="$(run_preview "$upstream" "$dest" "$fake_bin")"
preview_status=$?
bootstrap_output="$(run_bootstrap_preview "$upstream" "$bootstrap_dest" "$fake_bin")"
bootstrap_status=$?
...
set -e
```

**决策点**：所有运行 sync 脚本的调用都包在 `set +e ... set -e` 区间。

**理由**：
- 测试核心目标是"内容断言"而非"退出码断言"
- 某些边缘情况（missing manifest、dirty apply）脚本会非零退出，但测试仍要检查输出包含哪些字串
- 不取消防御性 set -e 包裹，避免中间命令静默失败

注释 510-511 行明确说明："This regression test is about dry-run content, so capture the preview output even if the current script exits nonzero in --local mode." 这是工程师对自己妥协的清晰记录。

### 3.3 版本号常量与"假装"manifest

```bash
PACKAGE_VERSION="1.2.3"
MANIFEST_VERSION="9.8.7"
```

**决策点**：测试使用两个明显不同的版本号（package.json 是 1.2.3，manifest 是 9.8.7）。

**理由**：
- 第 537-538 行用 `assert_contains "Version:  $MANIFEST_VERSION"` 验证**应当**显示 manifest 版本
- 紧接着用 `assert_not_contains "Version:  $PACKAGE_VERSION"` 验证**不应当**显示 package.json 版本
- 通过一个数字同时检验"包含"和"不包含"，确保"用错版本源"这种 bug 立即暴露

### 3.4 临时目录 + trap 清理

```bash
TEST_ROOT="$(mktemp -d)"
trap cleanup EXIT
...
cleanup() {
    if [[ -n "$TEST_ROOT" && -d "$TEST_ROOT" ]]; then
        rm -rf "$TEST_ROOT"
    fi
}
```

**决策点**：`mktemp -d` 创建临时目录，trap EXIT 自动清理。

**理由**：
- 6 个 fixture 仓库（upstream / dest / mixed_only_* / stale / dirty_apply / noop_apply / bootstrap）每个都是完整 git 仓库
- 这些是 `git init` 创建的"假仓库"，不是用户的真实代码
- 即使测试中途 `set -e` 失败退出，trap 也会清理

### 3.5 真实 sync 脚本的"复制粘贴"机制

```bash
write_upstream_fixture() {
    ...
    cp "$SYNC_SCRIPT_SOURCE" "$repo/scripts/sync-to-codex-plugin.sh"
    ...
}
```

**决策点**：把 `tests/codex-plugin-sync/../..scripts/sync-to-codex-plugin.sh` 复制到 fixture 仓库的 `scripts/` 子目录中，然后从 fixture 仓库里运行它。

**理由**：
- 测试运行的脚本路径在 fixture 内（`$upstream/scripts/sync-to-codex-plugin.sh`）
- 这样 sync 脚本可以用 `cd "$upstream"` 的方式定位自己
- 同时也确保脚本能找到 `$upstream/.codex-plugin/plugin.json`（相对路径）
- 这就是为什么 fixture 必须用 cp 注入真实脚本——而不能用 symlink 或 path override

### 3.6 不重复造轮子的 `set +e` 恢复

测试使用 `set +e ... set -e` 模式，但全局包裹不破坏后续断言。如果在 `set -e` 之后又调用会失败的命令，必须重新用 `set +e` 包裹——这种"切换语义"是 shell 测试的常见模式。

---

## 四、文件结构剖析

### 4.1 单文件单测：6 阶段组织

虽然是一个 bash 文件，但内容按"主函数 + 多个 fixture 工厂 + 多个 assertion helper + main()"组织清晰：

| 行号区间 | 内容 | 角色 |
|----------|------|------|
| 1-149 | shebang + 通用 helpers | `assert_equals`、`assert_contains`、`assert_matches`、`assert_path_absent`、`assert_branch_absent`、`assert_current_branch`、`assert_file_equals`、`cleanup`、`configure_git_identity`、`init_repo` |
| 151-242 | 上游 fixture 工厂 | `commit_fixture`、`checkout_fixture_branch`、`write_upstream_fixture` |
| 244-343 | 下游 fixture 工厂 | `write_destination_fixture`、`add_openai_agent_metadata_fixture`、`dirty_tracked_destination_skill`、`write_synced_destination_fixture`、`write_stale_ignored_destination_fixture` |
| 345-411 | 运行 helper | `write_fake_gh`、`run_preview`、`run_bootstrap_preview`、`run_preview_without_manifest`、`run_preview_with_stale_ignored_destination`、`run_apply`、`run_help` |
| 413-420 | bootstrap fixture | `write_bootstrap_destination_fixture` |
| 422-615 | `main()` | 全部测试逻辑 |
| 615 | 入口 | `main "$@"` |

### 4.2 `main()` 函数：6 个阶段串联

`main()` 内部按功能分为 6 块（每一块对应一个"阶段"）：

1. **Setup 阶段（457-507）**：
   - 创建 6 个 fixture 仓库
   - 注入 fake `gh`
   - 打印 `=== Test: sync-to-codex-plugin dry-run regression ===`

2. **Preview assertions（534-550）**：20+ 条断言，验证 dry-run 输出
3. **Mixed-directory assertions（552-556）**：3 条，验证纯被忽略目录的子场景
4. **Convergence assertions（558-561）**：2 条，验证 stale 文件收敛
5. **Bootstrap assertions（563-571）**：7 条，验证 bootstrap 行为
6. **Apply assertions（573-588）**：12+ 条，验证 -y 模式
7. **Missing manifest assertions（590-593）**：2 条
8. **Help assertions（595-597）**：1 条
9. **Source assertions（599-603）**：3 条
10. **Final tally（605-612）**：统计 FAILURES 并决定 exit code

总断言数约 50 条，是单 shell 文件中的高密度测试。

### 4.3 Fixture 仓库的角色分工

```
TEST_ROOT/
├── upstream/                     # 完整上游仓库（manifest 9.8.7）
├── mixed-only-upstream/          # 缺 .private-journal 的上游变体
├── destination/                  # 普通下游（含 OpenAI 元数据 + dirty 跟踪文件）
├── mixed-only-destination/       # 不带 OpenAI 元数据
├── stale-destination/            # 仅有 stale 的 .private-journal/leak.txt
├── dirty-apply-destination/      # 模拟"将要 apply 但 working tree 脏"的目标
├── noop-apply-destination/       # 干净的目标，apply 应无变化
└── bootstrap-destination/        # 没有 plugins/superpowers/ 的目标（测试 bootstrap）
└── bin/gh                        # fake gh
```

每个 fixture 的存在都不是冗余的——它们各自覆盖一个独立场景。

### 4.4 Fixture 复用与变异

- `write_upstream_fixture` 通过可选参数 `with_pure_ignored="${2:-1}"` 复用为 `upstream` 和 `mixed-only-upstream` 两个场景
- `write_synced_destination_fixture` 是 "已经 sync 过的下游"，可复用为 `dirty-apply-destination` 和 `noop-apply-destination`
- 这种"工厂函数参数化"是 shell 测试减少代码量的常见手法

---

## 五、优点亮点

### 5.1 50 条断言的高密度

单 616 行文件容纳 50+ 断言，平均每 12 行一条。这种密度反映了 sync 脚本本身的行为密度——它必须在 6 个场景下都正确工作。

### 5.2 注释即文档

```bash
# This regression test is about dry-run content, so capture the preview
# output even if the current script exits nonzero in --local mode.
set +e
```

测试中大量行内注释（"What"、"Why"、"because"）让维护者立即理解每个 `set +e` 的目的。**这与"测试应当自解释"的工程原则高度一致**。

### 5.3 失败模式对称设计

`dirty-apply` 场景要求脚本：
- **退出码 1**（保护性失败）
- **报错信息精确**（"ERROR: local checkout has uncommitted changes under 'plugins/superpowers'"）
- **不切换分支**
- **不创建 sync branch**
- **保留工作区文件内容**（不覆盖用户修改）

这 5 个独立断言组合起来形成"安全门"的完整描述，缺一不可。

### 5.4 Fake `gh` 的"未调用保险丝"

```bash
if [[ "${1:-}" == "auth" && "${2:-}" == "status" ]]; then
    exit 0
fi
echo "unexpected gh invocation: $*" >&2
exit 1
```

任何不在 `auth status` 范围内的 `gh` 调用都会让测试以非零退出，并打印实际调用。这种"沉默即安全"的反模式（默认不报警）和"显式即安全"的正模式（任何意外都立即报错）对比鲜明。

### 5.5 数字版本号 + 双重反向断言

`assert_contains` 配合 `assert_not_contains` 用同一个数字（`9.8.7` 和 `1.2.3`）做正反两个方向，确保"用错版本源"这种 regression 立刻可观察。

### 5.6 Source 文本断言：检测废弃代码

```bash
assert_not_contains "$script_source" "regenerated inline" "Source drops regenerated inline phrasing"
assert_not_contains "$script_source" "Brand Assets directory" "Source drops Brand Assets directory phrasing"
assert_not_contains "$script_source" "--assets-src" "Source drops --assets-src"
```

这是**元测试**：测试被测对象**没有**某些字符串。这保证"重构时同步清理了用户界面文案"——避免 UI 字符串"孤魂野鬼"地停留在生产代码中。

### 5.7 复合断言的语义化命名

`assert_branch_absent` 接受 repo + branch pattern + description 三个参数，但 description 文本本身就是断言的"规范"——这让测试输出易读。

---

## 六、缺点风险局限

### 6.1 依赖真实 git/rsync/mktemp

测试假设 `/bin/bash`、`git`、`rsync`、`mktemp` 可用。在 Windows MSYS2 或 Alpine 这种精简环境上，`rsync` 可能缺失。需要 CI 镜像明确。

### 6.2 路径硬编码 `/bin/bash`

```bash
BASH_UNDER_TEST="/bin/bash"
```

不兼容 NixOS 之类的"bash 不在 /bin"系统。应当用 `$(command -v bash)` 或 `$BASH`。

### 6.3 fixture 工厂函数与断言深度耦合

`write_synced_destination_fixture`、`dirty_tracked_destination_skill` 等 fixture 函数名非常长，并且默认参数会"假定下游已经有 plugins/superpowers/"。如果将来需要测一个新场景（比如 dry-run on missing scripts dir），可能要重写 fixture。

### 6.4 `set +e` 区间内的命令静默失败

虽然有 `set -e` 在外层保护，但 `set +e` 期间内任何命令失败都被吞掉。如果某次 `run_preview` 调用本身 typo，错误会延迟到断言阶段才暴露，而不是在调用现场立即触发。

### 6.5 缺断言总数

虽然 `FAILURES` 计数器存在，但没有 `TOTAL_ASSERTIONS` 计数器。如果某次重构意外导致一段断言被跳过（比如新 fixture 工厂替换了旧的，但没改 main() 的调用），从 PASS 输出看不出"少测了 X 条"。

### 6.6 Source 断言易脆

```bash
assert_not_contains "$script_source" "--assets-src" "Source drops --assets-src"
```

这种"代码里不应有 X 字符串"的断言对 refactor 非常敏感。如果把 `--assets-src` 改成 `--asset-source`，断言立刻 false-positive（实际是改进但测试 fail）。

### 6.7 没有 sub-shell 隔离

`TEST_ROOT` 内的多个 fixture 共享同一份 shell 进程。如果 sync 脚本"污染"了父 shell 的 cwd 或 environment（不太可能但可能），可能跨 fixture 影响。

### 6.8 长函数 main() 难以单元测试

`main()` 内联 615 行的 6 个阶段。如果某阶段失败，没有"只重跑某阶段"的机制（不像 npm test 可以用 --test 过滤）。

### 6.9 没有并发安全保证

虽然 `TEST_ROOT="$(mktemp -d)"` 给出唯一目录，但脚本本身不假定串行运行（`run_opencode` 等其他 tests 也是 mktemp 模式，但本文件没有并发问题）。

### 6.10 Fake `gh` 的边界 case 未覆盖

`if [[ "${1:-}" == "auth" && "${2:-}" == "status" ]]` 只匹配 "auth status" 这一种调用。如果 sync 脚本在 `--local` 模式下还调用 `gh api ...` 或 `gh repo ...`，fake gh 也会让脚本失败——但这种失败不会区分"sync 脚本意外调用 gh"和"sync 脚本需要 gh 但 --local 不应该"。

---

## 七、关键洞察

### 7.1 单元测试 vs 集成测试的边界判断

本文件**完全不做**单元测试。sync 脚本本质是"组合命令的胶水"——rsync + git + mktemp + diff——单元化它就是测 rsync、git 的命令行行为，没有价值。

真正的价值在"这些命令按什么顺序、什么参数、什么条件组合"——这是 integration 测试的用武之地。

### 7.2 fixture 是测试的"数据"，但 fixture 工厂是"代码"

文件 40% 的代码量是 fixture 工厂函数（`write_upstream_fixture`、`write_destination_fixture` 等），这暴露一个事实：**当被测对象是"组合器"时，fixture 的复杂度等于被测对象的复杂度**。

### 7.3 数字版本号的"反断言"哲学

`assert_contains ... MANIFEST_VERSION` 后面紧跟 `assert_not_contains ... PACKAGE_VERSION`——这种"包含 A 同时不包含 B"的成对断言在 shell 测试中相对少见，但**能精确捕获"用错源"类 bug**。

### 7.4 trap 的"必须"使用

`trap cleanup EXIT` 在第 460 行紧跟 `TEST_ROOT="$(mktemp -d)"`——这是 shell 测试的"卫生"原则：任何 mktemp 后必须 trap，否则会留下垃圾目录。

### 7.5 注释的价值

`# This regression test is about dry-run content, so capture the preview output even if the current script exits nonzero in --local mode.`——这种"我当时为什么这样写"的注释比"这是 set +e 区间"更值钱，因为它记录了**意图**而非语法。

### 7.6 Source 断言的"反 introspection"

测试不仅检查 sync 脚本**做什么**（行为），还检查 sync 脚本**不说什么**（文案）。这种"代码味道检测器"是非常规但极其实用的回归防线——专治"重构了功能但忘了改 UI 文案"。

### 7.7 "Branch 残留"是隐性破坏

`assert_branch_absent "$dest" "sync/superpowers-*"` 验证 sync 脚本失败时**不在目标仓库里创建 sync/superpowers-* 分支**。这是 GitOps 风格脚本常见的"残余分支"问题——脚本中断后留下半成品分支，下次运行出错。

---

## 八、关联文档

### 8.1 关联生产代码

- [`scripts/sync-to-codex-plugin.sh`](file://e:\WorkStation\MyProject\superpowers\scripts\sync-to-codex-plugin.sh) — 被测对象（460 行的 sync 工具）
- `.codex-plugin/plugin.json` — manifest（测试使用 `MANIFEST_VERSION="9.8.7"` 模拟）
- `package.json` — package.json（测试使用 `PACKAGE_VERSION="1.2.3"` 模拟）

### 8.2 关联其他 tests/

- `tests/claude-code/test-helpers.sh` — 不同的 assertion 库
- `tests/brainstorm-server/` — 另一个 fixture-heavy 测试集

### 8.3 项目元文件

- `CLAUDE.md` / `AGENTS.md` — 项目级 AI 指令
- `.version-bump.json` — 版本号管理（sync 脚本可能要读它）

### 8.4 缺失文档

- 本目录**没有 README.md**。其他子目录（claude-code/、brainstorm-server/）都有 README，但 codex-plugin-sync/ 缺少运行指引。
- 没有 `package.json` 或 `Makefile`，完全靠手动 `bash test-sync-to-codex-plugin.sh` 执行。

---

## 九、状态评价

### 9.1 健康度

| 维度 | 评分 | 备注 |
|------|------|------|
| 断言密度 | ⭐⭐⭐⭐⭐ | 50+ 断言 / 616 行 |
| 可读性 | ⭐⭐⭐⭐ | 函数命名长但语义清晰 |
| 可维护性 | ⭐⭐⭐ | fixture 工厂较多，修改时需小心 |
| 跨平台 | ⭐⭐ | 硬编码 /bin/bash，依赖 git/rsync |
| 与生产同步 | ⭐⭐⭐⭐⭐ | 通过 `cp` 注入当前生产脚本 |
| 失败信息可读 | ⭐⭐⭐⭐⭐ | "expected: / actual:" 模式成熟 |
| 文档化 | ⭐⭐ | 缺少 README；通过注释补充 |
| Fixture 复用 | ⭐⭐⭐⭐ | 参数化工厂函数控制良好 |

### 9.2 适用场景

- **CI 中**作为"代码改动后立即跑"的快速回归
- **重构 sync 脚本时**的"金标准"
- **新人 onboarding 时**的"实际行为说明书"

### 9.3 不适用场景

- 性能基准测试（无计时断言）
- 模糊测试（无随机化）
- 大规模并发场景（单进程）

---

## 十、给读者的启示

### 10.1 写 shell 测试时优先 fixture-driven

与其写 10 个独立 `assert_X.sh` 文件，不如 1 个 600 行文件用 fixture 工厂组合场景。`codex-plugin-sync/` 是 fixture-driven shell 测试的范本。

### 10.2 fake binary 的"未调用即失败"模式

```bash
if [[ expected_case ]]; then exit 0; fi
echo "unexpected invocation: $*" >&2
exit 1
```

任何在 fake binary 覆盖范围内但脚本未预期的调用都会让测试 fail。这种"白名单 + 默认拒绝"是 mock 的最佳实践。

### 10.3 数字版本号的反断言

如果生产代码有多个"版本源"（package.json、manifest、CHANGELOG、commit SHA），测试应当用**差异明显的常量**（如 1.2.3 vs 9.8.7）作为正反断言的标尺。

### 10.4 注释记录"妥协"

`# ... so capture the preview output even if the current script exits nonzero ...` 这种注释不是冗余——它告诉未来的维护者"这里有 set +e 不是因为疏忽，而是有意为之"。

### 10.5 fixture 工厂的参数化

```bash
write_upstream_fixture "$upstream"          # 完整版
write_upstream_fixture "$mixed_only_upstream" 0  # 缺 .private-journal
```

`with_pure_ignored="${2:-1}"` 这种默认参数是 shell 测试减少 fixture 重复的关键技巧。

### 10.6 重视 trap 卫生

每个 `mktemp -d` 后必须 `trap cleanup EXIT`。否则失败路径会留下垃圾。

---

## 十一、后续工作

### 11.1 立即可做

- [ ] 添加 README.md 说明运行方法和测试覆盖矩阵
- [ ] 替换 `BASH_UNDER_TEST="/bin/bash"` 为 `$(command -v bash)`
- [ ] 添加 `TOTAL_ASSERTIONS` 计数器，使 pass 输出包含断言总数

### 11.2 中期可做

- [ ] 把 `main()` 拆为 6 个 phase 函数，便于单独运行
- [ ] 添加 `--filter=<phase>` 选项，跳过/只跑某 phase
- [ ] 添加 `set -x` / `--trace` 调试模式
- [ ] 引入 shellcheck 静态分析作为 pre-commit

### 11.3 长期演进

- [ ] 写一个 generator，根据 sync 脚本的 help 文本自动生成"help 断言"
- [ ] 引入 bats-core（Bash Automated Testing System）替代纯手写 main()
- [ ] 为 Windows MSYS2 / WSL 提供 CI matrix
- [ ] Source 断言改成 AST 级别（用 bash AST 库），避免字符串匹配易脆

### 11.4 跨项目借鉴

`codex-plugin-sync/test-sync-to-codex-plugin.sh` 的以下模式值得其他项目借鉴：

1. **fake binary 注入 + 默认拒绝**：任何 shell 集成测试都可套用
2. **fixture 工厂函数 + 参数化**：减少代码重复
3. **数字常量作正反断言**：版本源多时必备
4. **Source 文本断言**：防止"重构但忘了改 UI 文案"

---

**报告完成时间**：2026-06-03
**分析对象**：`tests/codex-plugin-sync/test-sync-to-codex-plugin.sh`（616 行）
**分析深度**：单文件深度剖析，50+ 断言全部覆盖
**对应源目录**：[`tests/codex-plugin-sync/`](file://e:\WorkStation\MyProject\superpowers\tests\codex-plugin-sync)
**对应生产代码**：[`scripts/sync-to-codex-plugin.sh`](file://e:\WorkStation\MyProject\superpowers\scripts\sync-to-codex-plugin.sh)
