# tests/opencode/ 目录分析报告

> 对应源目录：`tests/opencode/`
> 包含 7 个测试脚本（1 runner + 1 setup + 5 测试）
> 报告定位：多文件分析，聚焦 OpenCode 插件的安装、缓存、工具与优先级

---

## 一、基本信息

| 字段 | 值 |
|------|-----|
| 目录路径 | `tests/opencode/` |
| 文件总数 | 7 个（5 个 `.sh` + 1 个 `.mjs` + 1 个 setup 脚本） |
| 总代码行数 | 约 700 行 |
| 测试方式 | 集成测试 + 单元测试（混合） |
| 关联生产代码 | `.opencode/plugins/superpowers.js`（OpenCode 插件的 main module） |
| 是否需要 OpenCode | 部分（`--integration` flag 启用） |
| 默认 CI 行为 | 跑 2 个非集成测试 |

`opencode/` 是 `tests/` 中**唯一为非 Claude Code 平台**专门建立的子目录——它测试的是项目在 OpenCode（另一款 AI IDE）下的兼容性。这与项目"跨平台工作"的核心理念一致。

---

## 二、目的与背景

### 2.1 测试目标

本目录存在是为验证 superpowers 插件在 **OpenCode** 平台下的行为：

1. **Plugin Loading**：插件能否被 OpenCode 正确识别和加载
2. **Bootstrap Caching**：`using-superpowers` 技能内容是否被缓存（避免每次 LLM 步骤都重读磁盘）
3. **Tool Functionality**：OpenCode 的原生 `use_skill` / `find_skills` 工具是否正确加载个人/项目/内置技能
4. **Priority Resolution**：同名技能在不同位置（个人/项目/内置）的优先级

### 2.2 关键设计约束

OpenCode 平台对插件的支持**与 Claude Code 不同**：
- 插件入口是 ESM `.js` 文件
- 插件通过 `experimental.chat.messages.transform` hook 注入内容
- 技能加载通过 OpenCode 原生 `skill` 工具
- 优先级语义与 Claude Code 不一致（这是 test-priority 的注释明确指出的"已知 bug"）

### 2.3 出现背景

根据 GitHub issue #1202（`test-bootstrap-caching.sh` 注释）的暗示，OpenCode 插件历史上存在"每个 LLM 步骤都重读 SKILL.md"的性能问题。本目录中的 `test-bootstrap-caching` 正是为该问题**修复后**添加的回归保护。

---

## 三、核心技术决策

### 3.1 Setup-Driven 测试隔离

```bash
# setup.sh 关键代码
TEST_HOME=$(mktemp -d)
export HOME="$TEST_HOME"
export XDG_CONFIG_HOME="$TEST_HOME/.config"
export OPENCODE_CONFIG_DIR="$TEST_HOME/.config/opencode"
```

**决策点**：通过覆盖 `HOME` / `XDG_CONFIG_HOME` / `OPENCODE_CONFIG_DIR` 实现**完全隔离**。

**理由**：
- OpenCode 读取 `$XDG_CONFIG_HOME/opencode/` 下的 `plugins/` 目录
- 把整个 config 目录搬到临时路径**完全不影响**用户的真实配置
- 每个测试在临时 `TEST_HOME` 中"假装"是一个干净系统

### 3.2 Setup 工厂 + 共享 trap

```bash
# setup.sh
cleanup_test_env() {
    if [ -n "${TEST_HOME:-}" ] && [ -d "$TEST_HOME" ]; then
        rm -rf "$TEST_HOME"
    fi
}
export -f cleanup_test_env
```

**决策点**：把 `cleanup_test_env` 定义在 setup.sh 中并 `export -f`，让所有测试用 `trap cleanup_test_env EXIT` 调用。

**理由**：
- 避免每个测试重写 cleanup 逻辑
- 单一 source of truth——如果清理策略要变，只改一处

### 3.3 真实插件复制 + 符号链接双轨

```bash
# setup.sh
SUPERPOWERS_PLUGIN_FILE="$SUPERPOWERS_DIR/.opencode/plugins/superpowers.js"
cp "$REPO_ROOT/.opencode/plugins/superpowers.js" "$SUPERPOWERS_PLUGIN_FILE"
ln -sf "$SUPERPOWERS_PLUGIN_FILE" "$OPENCODE_CONFIG_DIR/plugins/superpowers.js"
```

**决策点**：
- 真实插件文件被 cp 到 fixture 内的 `superpowers/.opencode/plugins/superpowers.js`
- OpenCode 实际读取的是 `$OPENCODE_CONFIG_DIR/plugins/superpowers.js`（软链接）

**理由**：
- 模拟真实安装路径
- 软链接让"OpenCode 视角"和"用户视角"路径不同——这正是真实部署的样子
- 复用 `cp` + `ln -s` 比改插件内部"理解安装路径"更清晰

### 3.4 单元测试用 .mjs 模拟 fs 系统调用

```javascript
// test-bootstrap-caching.mjs 核心思路
let existsCount = 0;
let readCount = 0;
const originalExistsSync = fs.existsSync;
const originalReadFileSync = fs.readFileSync;

fs.existsSync = function (...args) {
  if (isBootstrapSkillPath(args[0])) {
    existsCount += 1;
  }
  return originalExistsSync.apply(this, args);
};
```

**决策点**：用 `fs.existsSync = function(...)` 替换原生方法，**侵入式地**计数所有调用。

**理由**：
- 这是 **monkey patching**——Node.js 测试中常见的"拦截系统调用"技术
- 不需要修改插件源码
- 通过 `isBootstrapSkillPath` filter，只关心对 `using-superpowers/SKILL.md` 的访问
- 完成后 `originalExistsSync.apply(this, args)` 委托回原函数

### 3.5 测试缓存而非测试时间

```javascript
// test-bootstrap-caching.mjs
if (result.firstReadCount !== 1) {
    failures.push(`expected first transform to read SKILL.md once, got ${result.firstReadCount}`);
}
if (result.secondReadCount !== result.firstReadCount) {
    failures.push(`expected cached second transform to do no additional reads, got ${result.secondReadCount - result.firstReadCount}`);
}
```

**决策点**：断言"两次 transform 之间**没有额外**的 `fs.readFileSync` 调用"。

**理由**：
- 缓存的语义是"第二次访问不走磁盘"——`readFileSync` 计数是最直接的证据
- 不依赖 wall-clock 时间测量（CI 上不稳定）
- 不依赖文件修改时间（mtime 可能受系统时钟影响）

### 3.6 "Present" 和 "Missing" 两种场景

```bash
# test-bootstrap-caching.sh
run_present_file_check() {
    node "$SCRIPT_DIR/test-bootstrap-caching.mjs" "$SUPERPOWERS_PLUGIN_FILE" present
}

run_missing_file_check() {
    mv "$SUPERPOWERS_SKILLS_DIR/using-superpowers/SKILL.md" "$TEST_HOME/using-superpowers.SKILL.md.bak"
    node "$SCRIPT_DIR/test-bootstrap-caching.mjs" "$SUPERPOWERS_PLUGIN_FILE" missing
}
```

**决策点**：把 SKILL.md 移走来模拟"文件不存在"场景。

**理由**：
- "Missing" 场景下插件应当**快速失败**（不读文件）然后**缓存**该失败
- 这比"present"场景更难——既要避免读盘，又要在下次 transform 时同样跳过
- 物理移除文件确保无法意外读到（避免 symlink 之类把戏）

### 3.7 集成测试的 "Best Effort" 模式

```bash
# test-tools.sh
if ! command -v opencode &> /dev/null; then
    echo "  [SKIP] OpenCode not installed - skipping integration tests"
    echo "  To run these tests, install OpenCode: https://opencode.ai"
    exit 0
fi
```

**决策点**：检测 `opencode` 命令存在性；缺失则 SKIP 而非 FAIL。

**理由**：
- CI 环境可能没装 OpenCode
- "SKIP" 状态明确告知"未运行，但合理"
- `exit 0` 不会让 CI 失败——这是 unit test 与 integration test 的清晰区分

### 3.8 `--integration` flag 显式启用

```bash
# run-tests.sh
RUN_INTEGRATION=false
integration_tests=("test-tools.sh" "test-priority.sh")
if [ "$RUN_INTEGRATION" = true ]; then
    tests+=("${integration_tests[@]}")
fi
```

**决策点**：默认只跑 2 个非集成测试；`--integration` 标志启用另外 2 个。

**理由**：
- 4 个测试中 2 个是离线、2 个需要 OpenCode
- 默认离线（CI 友好）
- 显式 flag 防止"自动跑"造成困惑

### 3.9 Priority 测试的"已知行为"哲学

```bash
# test-priority.sh 注释
# Documents current OpenCode duplicate-name behavior for local and bundled
# skills. The desired local-shadowing behavior is tracked separately; this
# test keeps the integration suite honest without adding a plugin workaround.
```

**决策点**：测试**不**尝试"修复"OpenCode 的优先级 bug，而是**记录**当前行为 + 标记 known issue。

**理由**：
- 修复 OpenCode 行为需要修改插件代码，**风险高**（可能破坏 Claude Code 兼容）
- 记录当前行为让团队"知情"——后续修复有据可查
- 用 `[INFO]` 而非 `[PASS]`/`[FAIL]` 区分

### 3.10 优先级测试的 "三副本 fixture"

```bash
# test-priority.sh
mkdir -p "$SUPERPOWERS_SKILLS_DIR/priority-test"
mkdir -p "$OPENCODE_CONFIG_DIR/skills/priority-test"  # personal
mkdir -p "$TEST_HOME/test-project/.opencode/skills/priority-test"  # project
```

**决策点**：在 3 个不同位置创建同名 `priority-test` 技能，每个有不同的 `PRIORITY_MARKER_*_VERSION` 文本。

**理由**：
- 让 OpenCode 调用 skill 工具时返回哪个版本**可断言**（包含哪个 marker）
- "包含期望 marker = PASS"；"包含 fallback marker = INFO (known bug)"；"都不包含 = FAIL"
- 三个 location 模拟 OpenCode 真实的 skill 发现机制

---

## 四、文件结构剖析

### 4.1 7 个文件的角色分工

| 文件 | 行数 | 角色 | 关键特性 |
|------|------|------|----------|
| `run-tests.sh` | 166 | orchestrator | argparse、统计 pass/fail、生成摘要 |
| `setup.sh` | 89 | 环境工厂 | TEST_HOME + XDG 路径 + cleanup 函数 |
| `test-plugin-loading.sh` | 83 | 离线测试 | 验证 symlink、skills 计数、JS 语法 |
| `test-bootstrap-caching.sh` | 33 | 离线测试 | 包装 .mjs，2 个场景 |
| `test-bootstrap-caching.mjs` | 125 | 离线核心 | monkey patch fs、统计调用次数 |
| `test-tools.sh` | 96 | 集成测试 | 验证 personal/project/bundled skill 加载 |
| `test-priority.sh` | 218 | 集成测试 | 3 location 优先级 + known bug 记录 |

### 4.2 run-tests.sh 的 argparse 设计

```bash
while [[ $# -gt 0 ]]; do
    case $1 in
        --integration|-i) RUN_INTEGRATION=true; shift ;;
        --verbose|-v)     VERBOSE=true; shift ;;
        --test|-t)        SPECIFIC_TEST="$2"; shift 2 ;;
        --help|-h)        show_help; exit 0 ;;
        *)                error "Unknown option: $1"; exit 1 ;;
    esac
done
```

**决策点**：用 `case` 模式匹配 + `shift` 实现标准 CLI argparse。

**理由**：
- 比 `getopt` 简单（不依赖外部工具）
- `-i` 短形式 + `--integration` 长形式并存
- `shift 2` 处理带参数的 flag（`--test NAME`）

### 4.3 test-plugin-loading.sh 的 6 项检查

```bash
# Test 1: 插件 symlink 存在
if [ -L "$plugin_link" ]; then ...

# Test 2: symlink target 存在
if [ -f "$(readlink -f "$plugin_link")" ]; then ...

# Test 3: skills 目录非空
skill_count=$(find "$SUPERPOWERS_SKILLS_DIR" -name "SKILL.md" | wc -l)
if [ "$skill_count" -gt 0 ]; then ...

# Test 4: using-superpowers 技能存在（bootstrap 依赖）
if [ -f "$SUPERPOWERS_SKILLS_DIR/using-superpowers/SKILL.md" ]; then ...

# Test 5: JS 语法有效
if node --check "$SUPERPOWERS_PLUGIN_FILE"; then ...

# Test 6: bootstrap 文案不含旧路径
if grep -q 'configDir}/skills/superpowers/' "$SUPERPOWERS_PLUGIN_FILE"; then
    [FAIL] "Plugin still references old configDir skills path"
else
    [PASS]
fi
```

**亮点**：Test 6 是**反向断言**——确保旧 bug 路径（`configDir}/skills/superpowers/`）已被移除。这是"代码味道"测试的范本。

### 4.4 test-bootstrap-caching.mjs 的"状态断言"

```javascript
const firstOutput = makeOutput(`${scenario} bootstrap first step`);
await transform({}, firstOutput);
const afterFirst = { existsCount, readCount };

const secondOutput = makeOutput(`${scenario} bootstrap second step`);
await transform({}, secondOutput);
const afterSecond = { existsCount, readCount };

const result = {
    scenario,
    firstBootstrapParts: countBootstrapParts(firstOutput),
    secondBootstrapParts: countBootstrapParts(secondOutput),
    firstReadCount: afterFirst.readCount,
    secondReadCount: afterSecond.readCount,
    firstExistsCount: afterFirst.existsCount,
    secondExistsCount: afterSecond.existsCount,
};
```

**决策点**：在两次 transform **之间**记录状态，对比 before/after。

**理由**：
- 单点测量无法区分"每次都读"和"读一次缓存"——必须 delta
- 4 个独立计数器（firstRead / secondRead / firstExists / secondExists）让"缓存命中"和"未命中"都可量化
- 输出 JSON 让外部 shell 测试可读

### 4.5 test-priority.sh 的 "describe_priority_result" helper

```bash
describe_priority_result() {
    local output="$1"
    local expected_marker="$2"
    local fallback_marker="$3"
    local pass_message="$4"
    local known_bug_message="$5"

    loaded_skill="$(first_skill_tool_event "$output")"

    if [[ "$loaded_skill" == *"$expected_marker"* ]]; then
        echo "  [PASS] $pass_message"
    elif [[ "$loaded_skill" == *"$fallback_marker"* ]]; then
        echo "  [INFO] $known_bug_message"
        echo "  [INFO] Tracked separately: OpenCode bundled skills can shadow local skills with duplicate native names"
    else
        echo "  [FAIL] Could not verify priority marker in native skill tool output"
        ...
    fi
}
```

**亮点**：3 状态 helper（PASS / INFO / FAIL）让测试结果**梯度**表达——既不掩盖 bug，也不让已知问题阻断 CI。

---

## 五、优点亮点

### 5.1 monkey patch fs 的精准测量

`test-bootstrap-caching.mjs` 通过替换 `fs.existsSync` / `fs.readFileSync` 来**精确统计**插件对 `using-superpowers/SKILL.md` 的访问次数。这种"系统调用级"断言比"性能断言"更可靠。

### 5.2 Three-tier test status（PASS / INFO / FAIL）

`test-priority.sh` 引入 INFO 状态记录 known bug——这是承认"现实有缺陷但不阻塞开发"的成熟实践。完全对立的 PASS/FAIL 二元化反而会让团队"为了 CI 通过而 hack 掉真实问题"。

### 5.3 Setup-Driven 测试隔离

`setup.sh` 一次性创建 fixture、导出环境、定义 cleanup——5 个测试共享这套"环境脚手架"。`run-tests.sh` 用 `source setup.sh`（test-bootstrap-caching、test-priority、test-plugin-loading、test-tools）统一调用。

### 5.4 反向断言 Test 6

```bash
# test-plugin-loading.sh Test 6
if grep -q 'configDir}/skills/superpowers/' "$SUPERPOWERS_PLUGIN_FILE"; then
    [FAIL] "Plugin still references old configDir skills path"
else
    [PASS]
fi
```

这种"代码里**不应有**某字符串"的断言**专治**遗留路径——比"重构了但忘了改文案"更有效。

### 5.5 真实路径的"双轨"安装

`cp` 真实文件 + `ln -s` 注册符号——这种"实物 + 软链"模式让 fixture **完全模拟**真实 OpenCode 安装。测试的不是"被测对象能否跑"，而是"被测对象能否在真实环境配置下跑"。

### 5.6 双层测试：unit + integration

- Unit 测试（plugin-loading、bootstrap-caching）：不依赖 OpenCode CLI
- Integration 测试（tools、priority）：依赖 OpenCode CLI

这种分层让 CI 既能"快速跑非集成测试"又能"深度跑集成测试"。

### 5.7 详细的优先级 fixture

`priority-test` 技能在 3 个 location 创建相同名字 + 不同 marker——这让"OpenCode 实际加载了哪个版本"**可断言**。`PRIORITY_MARKER_SUPERPOWERS_VERSION` / `PRIORITY_MARKER_PERSONAL_VERSION` / `PRIORITY_MARKER_PROJECT_VERSION` 是优雅的"trace ID"。

### 5.8 缺失安装的优雅降级

```bash
if ! command -v opencode &> /dev/null; then
    echo "  [SKIP] OpenCode not installed"
    exit 0
fi
```

`exit 0` 让 SKIP 不阻塞 CI。这种"可降级"测试设计在多平台项目中至关重要。

### 5.9 输出 JSON 便于 shell 解析

```javascript
console.log(JSON.stringify(result, null, 2));
```

`test-bootstrap-caching.mjs` 输出 JSON 而非自由文本——便于 shell 测试 `grep` / `jq` 解析。`2` 是缩进（人类可读优先）。

### 5.10 `node --check` 静态校验

```bash
if node --check "$SUPERPOWERS_PLUGIN_FILE" 2>/dev/null; then
    [PASS] "Plugin JavaScript syntax is valid"
```

`node --check` 不执行，只检查语法。零成本 JS 语法验证。

---

## 六、缺点风险局限

### 6.1 OpenCode 行为未文档化

`test-priority.sh` 注释说"OpenCode bundled skills can shadow local skills with duplicate native names"——这是从行为反推的，不是 OpenCode 官方文档。如果 OpenCode 升级后**改变**该行为，测试**不会**报警（因为 fallback marker 路径走 INFO）。

### 6.2 fs monkey patch 是全局污染

```javascript
fs.existsSync = function (...args) { ... };
```

替换全局 `fs.existsSync` 后，**任何** Node.js 内部代码（assert、path、require 缓存）都可能受影响。如果插件或其依赖在 transform 之外调用 fs，会污染计数。`isBootstrapSkillPath` filter 是必要防线。

### 6.3 SKILL.md "Missing" 场景无法恢复

```bash
run_missing_file_check() {
    mv "$SUPERPOWERS_SKILLS_DIR/using-superpowers/SKILL.md" "$TEST_HOME/..."
}
```

`mv` 后文件**永久移走**——`trap cleanup_test_env EXIT` 会清理 TEST_HOME，但若 trap 失败（极端情况），文件丢失。需要 `mv` + `trap "mv back" EXIT` 双保险。

### 6.4 set -e + 隐式失败混合

```bash
set -euo pipefail
...
if ! command -v opencode &> /dev/null; then
    echo "  [SKIP] ..."
    exit 0
fi
```

虽然 `set -e` 包裹了开头，但 SKIP 分支用 `exit 0` 主动退出。如果 SKIP 检测因故不触发，集成测试会真跑 opencode，CI 可能卡住。

### 6.5 集成测试的 prompt 是硬编码

`test-tools.sh` 的 3 个测试都 hardcode "personal-test"、"project-test"、"brainstorming"——如果有人改名测试 fixture，集成测试不会自动跟踪。

### 6.6 120s timeout 可能不够

```bash
OPENCODE_TEST_TIMEOUT_SECONDS="${OPENCODE_TEST_TIMEOUT_SECONDS:-120}"
```

`test-tools.sh` 3 个集成测试 + `test-priority.sh` 4 个测试 = 7 次 opencode run，每次 120s。完整跑要 14 分钟。CI 可能超时。

### 6.7 fixture 工厂和测试混杂

`test-priority.sh` 自带 fixture 创建 + 测试断言——这与 `setup.sh` 共享 fixture 模式**不一致**。`setup.sh` 创建的 `personal-test` 和 `project-test` 在 priority 测试里**没用到**（priority 创建自己的 `priority-test`）。

### 6.8 result.scenario 是字符串

```javascript
if (failures.length > 0) {
    console.error(JSON.stringify(result, null, 2));
    ...
}
```

失败时打印 result JSON，但**不**用 `jq` 等结构化工具定位——需要人眼读。

### 6.9 `.mjs` 文件的 ESM 限制

`test-bootstrap-caching.mjs` 用 `import` + `await import()`——如果未来想用 CommonJS 测试，将需要重写。

### 6.10 没有 `--dry-run` 选项

`run-tests.sh` 直接跑测试，没有"只列出将要跑什么"模式。对于调试"哪些测试会被跑"不友好。

### 6.11 priority 测试的 "non-colliding" 部分松散

```bash
# test-priority.sh Test 4
run_opencode output "$TEST_HOME/test-project" "Call the skill tool with name \"superpowers-only-test\"..."
```

这条测试与前 3 条优先级测试**目的不同**——它验证"非冲突技能仍可加载"。但被混在 priority 测试文件里，逻辑上应拆分为 `test-skill-availability.sh`。

---

## 七、关键洞察

### 7.1 monkey patch 的精准度 vs 风险

`fs.existsSync = function(...)` 是**黑盒**测量插件行为的强手段。它不依赖插件暴露"指标"——任何代码改不改，monkey patch 都能看到 fs 真实访问。但**全局污染**是代价。

### 7.2 Three-tier status 是"成熟工程"的标志

`[PASS]` / `[INFO]` / `[FAIL]` 三态胜过二态——承认"已知但不阻塞"是诚实的工程文化。如果所有问题都必须 FAIL，会让团队 hack 掉真问题来让 CI 绿。

### 7.3 fixture 复用 = 减少 drift

`setup.sh` 创建 `personal-test` 和 `project-test` 一次——多个测试共享。**但** `test-priority.sh` 创建自己的 `priority-test`——这种"按需创建"模式更灵活，但增加了 fixture 数量。

### 7.4 `--integration` 显式启用是 CI 哲学

默认跑非集成测试让 PR 检查**便宜**；`--integration` flag 让 nightly 跑**深度**。这种"渐进式深入"是测试金字塔的工程体现。

### 7.5 "Reverse assertion" 模式

```bash
if grep -q 'OLD_PATH' "$plugin"; then [FAIL] else [PASS]
```

这种"代码里不应有 X"断言在 refactor 后特别有价值——**专治**"代码已迁移但文案没改"。

### 7.6 "Present" vs "Missing" 双向验证

缓存的两个失败模式：
- 总是重读（缓存未生效）
- 总是报错（缺失文件路径未优化）

`present` 场景验前者，`missing` 场景验后者。**双向**才能全面保护。

### 7.7 OpenCode 优先级"接受现实"哲学

`test-priority.sh` 不尝试 hack OpenCode 行为——它**记录**行为 + 标记 issue。这种"接受现实、留待未来"是工程现实的体现。

### 7.8 bash + mjs 双语

`run-tests.sh`（bash 编排）+ `test-bootstrap-caching.mjs`（JS 单元）共存——bash 处理 IO 隔离，JS 处理系统调用 mocking。**各取所长**。

### 7.9 缺失命令的优雅降级

`command -v opencode` + `exit 0` 让测试**在没装 OpenCode 的环境也能跑**——这是 cross-platform 项目的必备。

---

## 八、关联文档

### 8.1 关联生产代码

- [`.opencode/plugins/superpowers.js`](file://e:\WorkStation\MyProject\superpowers\.opencode\plugins\superpowers.js) — OpenCode 插件入口
- `skills/using-superpowers/SKILL.md` — bootstrap 注入的源
- `.opencode/INSTALL.md` — OpenCode 安装指南

### 8.2 关联其他 tests/

- `tests/claude-code/` — Claude Code 平台（对比基线）
- `tests/brainstorm-server/` — 另一个非 Claude 集成

### 8.3 关联设计文档

- `docs/plans/2025-11-22-opencode-support-design.md` — OpenCode 支持设计
- `docs/plans/2025-11-22-opencode-support-implementation.md` — 实现
- `docs/README.opencode.md` — OpenCode 用户文档

### 8.4 缺失文档

- 本目录**没有 README.md**——`run-tests.sh` 的 `--help` 文本是唯一的运行说明
- 没有 OpenCode 版本兼容性矩阵
- 没有 `setup.sh` 的环境变量文档

---

## 九、状态评价

### 9.1 健康度

| 维度 | 评分 | 备注 |
|------|------|------|
| 跨平台测试覆盖 | ⭐⭐⭐⭐ | OpenCode 平台独立子目录 |
| 单元 vs 集成分层 | ⭐⭐⭐⭐⭐ | `--integration` 显式 |
| Monkey patch 精准度 | ⭐⭐⭐⭐⭐ | fs 系统调用拦截 |
| 已知问题管理 | ⭐⭐⭐⭐⭐ | INFO 状态 + 注释 |
| Fixture 复用 | ⭐⭐⭐ | setup 共享但 priority 自创 |
| CI 友好 | ⭐⭐⭐⭐ | 离线/集成分层 |
| 文档化 | ⭐⭐ | 缺 README |
| 反向断言 | ⭐⭐⭐⭐⭐ | Test 6 范本 |

### 9.2 适用场景

- **PR check** 跑非集成测试（plugin-loading、bootstrap-caching）—— 5 秒
- **nightly** 跑全部 4 个测试 —— 14 分钟
- **OpenCode 升级后** 跑 priority 测试看行为是否变化
- **新技能添加后** 跑 bootstrap-caching 看缓存是否仍生效

### 9.3 不适用场景

- 性能基准（无计时）
- 并发安全（单线程）
- Windows 原生（依赖 OpenCode CLI 安装）

---

## 十、给读者的启示

### 10.1 monkey patch fs 是 Node.js 性能断言的最佳武器

```javascript
let count = 0;
const original = fs.readFileSync;
fs.readFileSync = function(...args) {
    if (isTargetPath(args[0])) count++;
    return original.apply(this, args);
};
```

这种"拦截系统调用 + 计数"对缓存、I/O 优化类测试**特别有效**。比"测时间"更稳定。

### 10.2 三态测试结果是成熟的工程文化

`[PASS]` / `[INFO]` / `[FAIL]` 胜过二态。`[INFO]` 状态让 known issues **可见但不阻塞**。

### 10.3 "Reverse assertion" 防遗留路径

```bash
if grep -q 'OLD' "$code"; then [FAIL] else [PASS] fi
```

这种断言专治"重构了功能但忘了改文案"。

### 10.4 setup-driven 测试隔离的工厂模式

`setup.sh` + `trap cleanup_test_env EXIT` + `source` 模式是 shell 测试的"框架雏形"——比每个测试手写 cleanup 优雅。

### 10.5 集成测试的"显式 opt-in"是 CI 成本控制

`--integration` 标志让"昂贵的集成测试"在 PR 阶段不跑、nightly 跑。

### 10.6 missing/present 双向验证

缓存测试不能只测"文件存在时"——必须测"文件不存在时"。后者捕获的是"未实现 early-exit"类 bug。

### 10.7 `node --check` 零成本语法验证

任何 JS 项目都应在 CI 跑 `node --check`——5ms 内发现语法错误。

### 10.8 接受现实 ≠ 放弃改进

`test-priority.sh` 不 hack OpenCode 行为——它**记录**行为。这种"留待未来"的处理是工程现实主义。

---

## 十一、后续工作

### 11.1 立即可做

- [ ] 添加 README.md 说明 4 个测试的覆盖矩阵和 `--integration` 标志
- [ ] 在 `setup.sh` 中显式 export 文档化所有环境变量
- [ ] 把 priority 测试拆为 `test-priority.sh`（冲突场景）+ `test-skill-availability.sh`（非冲突场景）
- [ ] 添加 `node --check` 失败时的 `exit 1`（当前用 `2>/dev/null` 吞掉错误）

### 11.2 中期可做

- [ ] 引入 jest/vitest 替代手写 .mjs 测试
- [ ] 编写 OpenCode 行为变更的回归矩阵（升级前 baseline）
- [ ] 添加 fixture 创建到 setup.sh（避免 priority 测试自创 fixture）
- [ ] 记录 OpenCode 行为文档（"我们的实证"）

### 11.3 长期演进

- [ ] 提供 Docker 镜像（带 OpenCode）让 CI 一键跑全部
- [ ] 添加 perf benchmark 跟踪 bootstrap 性能
- [ ] 探索在 OpenCode 升级时跑 shadow test
- [ ] 把 `setup.sh` 升级为 bats-core

### 11.4 跨项目借鉴

`opencode/` 的以下模式值得其他跨平台 AI 集成项目借鉴：

1. **Setup-driven 测试隔离**：`TEST_HOME` + `XDG_*` 环境变量
2. **Monkey patch fs**：Node.js 性能断言的标准武器
3. **三态测试结果**：`[PASS]` / `[INFO]` / `[FAIL]` 显式分级
4. **Reverse assertion**：专治遗留路径
5. **集成测试 opt-in**：`--integration` 显式启用
6. **known issue 文档化**：INFO 状态 + 注释跟踪
7. **present/missing 双向验证**：缓存测试的完整保护

---

**报告完成时间**：2026-06-03
**分析对象**：`tests/opencode/` 全部 7 个文件
**分析深度**：7 个文件全深度，2 个核心 .mjs/.sh 详尽剖析
**对应源目录**：[`tests/opencode/`](file://e:\WorkStation\MyProject\superpowers\tests\opencode)
**对应生产代码**：[`.opencode/plugins/superpowers.js`](file://e:\WorkStation\MyProject\superpowers\.opencode\plugins\superpowers.js)
**关联设计**：[`docs/plans/2025-11-22-opencode-support-design.md`](file://e:\WorkStation\MyProject\superpowers\docs\plans\2025-11-22-opencode-support-design.md)
