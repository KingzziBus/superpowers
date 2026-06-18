# tests/subagent-driven-dev/ 目录分析报告

> 对应源目录：`tests/subagent-driven-dev/`
> 包含 1 个 orchestrator + 2 个测试项目（每个含 design.md / plan.md / scaffold.sh）
> 报告定位：多文件分析，聚焦"端到端 SDD 工作流的可执行测试样本"

---

## 一、基本信息

| 字段 | 值 |
|------|-----|
| 目录路径 | `tests/subagent-driven-dev/` |
| 文件总数 | 8 个（1 个 `run-test.sh` + 2 个项目 × 3 个文件） |
| 总代码行数 | 约 107 行（`run-test.sh`）+ 82+46（go-fractals）+ 71+47（svelte-todo）+ 173+223（plan.md） |
| 测试方式 | **真实 Claude 端到端工作流** + 真实项目脚手架 |
| 测试目标 | 验证 `subagent-driven-development` 技能能否端到端执行完整 plan |
| 关联生产代码 | `skills/subagent-driven-development/SKILL.md` |
| 是否需要联网 | **是**（依赖 `claude` CLI + API） |
| 是否需要其他工具 | 部分（`go test` for go-fractals；`npm`/`npx` for svelte-todo） |

`subagent-driven-dev/` 是 `tests/` 下**项目级集成测试**的代表——它不是单技能测试，而是给 LLM 一个**完整 plan** 看它能否**多步骤**完成。

---

## 二、目的与背景

### 2.1 测试目标

本目录存在是为了验证 `subagent-driven-development`（SDD）技能的**端到端**能力：

1. **能否加载 SDD 技能**：LMM 是否调用 Skill 工具加载 `subagent-driven-development`
2. **能否遵循 plan.md 任务列表**：LMM 是否按 task 1 → 2 → 3 ... 顺序执行
3. **能否使用 subagent**：LMM 是否**实际**用 Agent 工具分派 subagent，而非自己单干
4. **能否完成真实任务**：Go fractals CLI / Svelte Todo List 能否成功 build/test/run
5. **能否产生可工作的代码**：用户拿到的代码能跑、测试能过

### 2.2 与其他 tests/ 子目录的关系

| 子目录 | 测试粒度 | 是否需要 LLM | 验证内容 |
|--------|----------|--------------|----------|
| `claude-code/test-subagent-driven-development.sh` | 单技能 | 是 | 是否调用 Skill 工具 |
| **`subagent-driven-dev/`** | **端到端** | **是** | **plan 能否执行完毕** |
| `claude-code/test-subagent-driven-development-integration.sh` | 端到端 | 是 | 是否产生 commits、是否走 TDD |
| `explicit-skill-requests/` | 触发正确性 | 是 | 是否调用 Skill 工具 |

`subagent-driven-dev/` 与 `claude-code/test-subagent-driven-development-integration.sh` 类似，但**重点不同**：
- `claude-code/integration` 验证**协议合规性**（commits、TDD、subagent 用法）
- `subagent-driven-dev/` 验证**项目完成度**（能否跑 `go test ./...`、能否 `npm test`）

### 2.3 出现背景

这是 superpowers 项目的"showcase 测试"——给潜在用户/贡献者一个**可亲手试**的样例。两个项目精心挑选：

- **go-fractals**：Go CLI（标准 `go test` 验证）
- **svelte-todo**：Svelte + Playwright（前端 + 端到端测试）

覆盖两种典型场景：CLI 工具 + Web 应用。

---

## 三、核心技术决策

### 3.1 真实脚手架 + 真实任务

```bash
# go-fractals/scaffold.sh
git init
cp "$SCRIPT_DIR/design.md" .
cp "$SCRIPT_DIR/plan.md" .
mkdir -p .claude
cat > .claude/settings.local.json << 'SETTINGS'
{
  "permissions": {
    "allow": ["Read(**)", "Edit(**)", "Write(**)", "Bash(go:*)", "Bash(mkdir:*)", "Bash(git:*)"]
  }
}
SETTINGS
git add . && git commit -m "Initial project setup with design and plan"
```

**决策点**：每个测试项目自带完整脚手架（`scaffold.sh`），产出**真实可跑**的初始项目。

**理由**：
- LLM 拿到的是真实 git 仓库、真实 .claude settings、真实 plan.md
- 这是**模拟真实用户场景**而非 mock
- `.claude/settings.local.json` **明确授权** `Bash(go:*)`、`Bash(npm:*)` 等——免去 LLM 询问权限的中断

### 3.2 权限预授权避免 LLM 中断

```json
{
  "permissions": {
    "allow": [
      "Read(**)",
      "Edit(**)",
      "Write(**)",
      "Bash(go:*)",
      "Bash(mkdir:*)",
      "Bash(git:*)"
    ]
  }
}
```

**决策点**：在 `.claude/settings.local.json` 预授权所有相关 Bash 命令。

**理由**：
- LLM 跑 Go 项目需要 `go build`、`go test`、`go mod init` 等
- 如果不预授权，每个 Bash 命令都会**触发权限询问**——打断 LLM 节奏
- 这是测试**专用**的激进设置，生产环境**不能**这样

### 3.3 任务列表 + Do/Verify 格式

```markdown
### Task 1: Project Setup

**Do:**
- Initialize `go.mod` with module name `github.com/superpowers-test/fractals`
- Create directory structure: `cmd/fractals/`, `internal/sierpinski/`, ...
- Create minimal `cmd/fractals/main.go` that prints "fractals cli"
- Add `github.com/spf13/cobra` dependency

**Verify:**
- `go build ./cmd/fractals` succeeds
- `./fractals` prints "fractals cli"
```

**决策点**：每个 Task 都有 **Do**（做什么）+ **Verify**（如何验证）两部分。

**理由**：
- 这是 TDD 风格的 plan 模板
- **Do** 给 LLM 行为指引
- **Verify** 给 LLM 反馈机制（"我做完没"）
- 高度结构化让 LLM **易于解析**

### 3.4 `run-test.sh` 接受 `<test-name>` 参数

```bash
TEST_NAME="${1:?Usage: $0 <test-name> [--plugin-dir <path>]}"
```

**决策点**：`./run-test.sh go-fractals` 或 `./run-test.sh svelte-todo` 切换测试项目。

**理由**：
- 两个测试项目**相互独立**——不必同时跑
- 单文件 orchestrator + 多项目 directory = 关注点分离
- 添加第三个项目（如 `rust-web-server`）只需新建目录

### 3.5 `scaffold.sh` 接收目标路径参数

```bash
# run-test.sh
"$TEST_DIR/scaffold.sh" "$OUTPUT_DIR/project"

# scaffold.sh
TARGET_DIR="${1:?Usage: $0 <target-directory>}"
```

**决策点**：`scaffold.sh` 接收目标目录作为参数（不写死路径）。

**理由**：
- `run-test.sh` 决定**输出**到 `/tmp/superpowers-tests/<timestamp>/...`
- `scaffold.sh` 决定**怎么初始化**项目
- 解耦让 scaffold 可单独调试：`./scaffold.sh /tmp/test-go-fractals`

### 3.6 prompt 显式调用 SDD 技能

```bash
PROMPT="Execute this plan using superpowers:subagent-driven-development. The plan is at: $PLAN_PATH"
```

**决策点**：prompt **明确**要求 LLM 调用 `subagent-driven-development` 技能。

**理由**：
- 这是测试的核心目标——"SDD 技能能跑完一个 plan"
- 显式调用让失败原因可定位（"是技能没加载" vs "技能加载了但没用"）
- 模拟**真实用户**的 prompt 风格："用 SDD 执行这个 plan"

### 3.7 `--output-format stream-json --verbose` 双重输出

```bash
claude -p "$PROMPT" \
  --plugin-dir "$PLUGIN_DIR" \
  --dangerously-skip-permissions \
  --output-format stream-json \
  --verbose \
  > "$LOG_FILE" 2>&1 || true
```

**决策点**：stream-json + verbose + || true 组合。

**理由**：
- `stream-json`：结构化输出便于后期分析
- `verbose`：人类可读输出便于调试
- `|| true`：SDD 可能长跑超过 max-turns，吞掉退出码

### 3.8 token 使用量跟踪

```bash
# 末尾
if command -v jq &> /dev/null; then
  echo ">>> Token usage:"
  jq -s '[.[] | select(.type == "result")] | last | .usage' "$LOG_FILE" 2>/dev/null || echo "(could not parse usage)"
fi
```

**决策点**：测试**展示**最终 token 使用量（虽然不做断言）。

**理由**：
- 让用户**看到**成本
- 不强制断言（不同 plan 长度不同）——避免"为了节省 token 而少做事"
- 用 `jq -s` 流式解析 stream-json

### 3.9 测试后给出 next steps 提示

```bash
echo ">>> Next steps:"
echo "1. Review the project: cd $OUTPUT_DIR/project"
echo "2. Review Claude's log: less $LOG_FILE"
echo "3. Check if tests pass:"
if [[ "$TEST_NAME" == "go-fractals" ]]; then
  echo "   cd $OUTPUT_DIR/project && go test ./..."
elif [[ "$TEST_NAME" == "svelte-todo" ]]; then
  echo "   cd $OUTPUT_DIR/project && npm test && npx playwright test"
fi
```

**决策点**：测试**引导**用户做后续验证（`go test`、`npm test`）。

**理由**：
- 这是**人类可读**的测试——结果是项目代码而非 pass/fail
- 用户拿到 `/tmp/superpowers-tests/.../project/` 后**自己跑测试**
- 项目类型特定的下一步（Go 用 go test，Svelte 用 npm test）

### 3.10 两个项目覆盖不同栈

- **go-fractals**：Go + Cobra（CLI）
- **svelte-todo**：Svelte + Vite + Playwright（Web + e2e）

**决策点**：选择**语言和范式都不同**的两个项目。

**理由**：
- Go：编译型、强类型、官方测试
- Svelte：解释型（transpiled）、前端、e2e 测试
- 验证 SDD 是否**栈无关**——不会因为 Go 测试模式或 Playwright 模式而失败

---

## 四、文件结构剖析

### 4.1 8 个文件的角色分工

| 文件 | 行数 | 角色 |
|------|------|------|
| `run-test.sh` | 107 | orchestrator（接受 test-name） |
| `go-fractals/design.md` | 82 | Go 项目设计规范 |
| `go-fractals/plan.md` | 173 | Go 项目 10 任务 plan |
| `go-fractals/scaffold.sh` | 46 | Go 项目脚手架 |
| `svelte-todo/design.md` | 71 | Svelte 项目设计规范 |
| `svelte-todo/plan.md` | 223 | Svelte 项目 12 任务 plan |
| `svelte-todo/scaffold.sh` | 47 | Svelte 项目脚手架 |

### 4.2 `run-test.sh` 流程

```bash
# 1. 参数解析
TEST_NAME="${1:?Usage: $0 <test-name> [--plugin-dir <path>]}"
shift
# ... 解析 --plugin-dir

# 2. 默认 plugin dir
PLUGIN_DIR="$(cd "$SCRIPT_DIR/../.." && pwd)"

# 3. 验证测试存在
TEST_DIR="$SCRIPT_DIR/$TEST_NAME"
if [[ ! -d "$TEST_DIR" ]]; then
    echo "Available tests:"
    ls -1 "$SCRIPT_DIR" | grep -v '\.sh$' | grep -v '\.md$'
    exit 1
fi

# 4. 创建输出目录
TIMESTAMP=$(date +%s)
OUTPUT_BASE="/tmp/superpowers-tests/$TIMESTAMP/subagent-driven-development"
OUTPUT_DIR="$OUTPUT_BASE/$TEST_NAME"
mkdir -p "$OUTPUT_DIR"

# 5. 脚手架
"$TEST_DIR/scaffold.sh" "$OUTPUT_DIR/project"

# 6. 准备 prompt
PLAN_PATH="$OUTPUT_DIR/project/plan.md"
PROMPT="Execute this plan using superpowers:subagent-driven-development. The plan is at: $PLAN_PATH"

# 7. claude 调用
cd "$OUTPUT_DIR/project"
claude -p "$PROMPT" --plugin-dir "$PLUGIN_DIR" --dangerously-skip-permissions \
  --output-format stream-json --verbose > "$LOG_FILE" 2>&1 || true

# 8. 输出 token 使用量（如果 jq 可用）
if command -v jq &> /dev/null; then
  jq -s '[.[] | select(.type == "result")] | last | .usage' "$LOG_FILE"
fi

# 9. next steps
echo "1. Review the project: cd $OUTPUT_DIR/project"
echo "2. Review Claude's log: less $LOG_FILE"
echo "3. Check if tests pass:"
if [[ "$TEST_NAME" == "go-fractals" ]]; then
  echo "   cd $OUTPUT_DIR/project && go test ./..."
elif [[ "$TEST_NAME" == "svelte-todo" ]]; then
  echo "   cd $OUTPUT_DIR/project && npm test && npx playwright test"
fi
```

### 4.3 design.md 与 plan.md 的对照

| 项目 | design.md 行数 | plan.md 行数 | Task 数量 |
|------|---------------|-------------|-----------|
| go-fractals | 82 | 173 | 10 |
| svelte-todo | 71 | 223 | 12 |

**设计**（design）描述"做什么"（features、UI、架构），**计划**（plan）描述"怎么做"（task 列表、Do/Verify）。

两者**分离**让 LLM 拿到的是**结构化指令**：
1. 先读 design（理解目标）
2. 再执行 plan（按部就班）

### 4.4 scaffold.sh 的统一模板

两个 `scaffold.sh` 几乎相同（仅 settings.local.json 中的 `Bash(*)` 不同）：

```bash
# go-fractals/scaffold.sh (46 行)
TARGET_DIR="${1:?Usage: $0 <target-directory>}"
SCRIPT_DIR="$(cd "$(dirname "$0")" && pwd)"

mkdir -p "$TARGET_DIR"
cd "$TARGET_DIR"

git init
cp "$SCRIPT_DIR/design.md" .
cp "$SCRIPT_DIR/plan.md" .

mkdir -p .claude
cat > .claude/settings.local.json << 'SETTINGS'
{
  "permissions": {
    "allow": [
      "Read(**)", "Edit(**)", "Write(**)",
      "Bash(go:*)", "Bash(mkdir:*)", "Bash(git:*)"
    ]
  }
}
SETTINGS

git add . && git commit -m "Initial project setup with design and plan"
```

**重复模式**：`git init` + 复制 design/plan + 写入 settings + 首次 commit。

**潜在重构**：可以抽到 `_lib/scaffold-template.sh`，让每个项目只指定 settings 部分。

### 4.5 plan.md 的"四段式"Task 模板

每个 Task 都是相同结构：

```markdown
### Task N: 任务名称

任务的简短描述（一两句话说明目的）。

**Do:**
- 步骤 1
- 步骤 2
- 步骤 3
- ...

**Verify:**
- 验证命令 1
- 验证命令 2
- ...
```

这种**严格模板**让 LLM 解析任务列表**极其容易**——只要循环"读 Task 名 → 读 Do → 读 Verify → 执行 → 验证"。

### 4.6 go-fractals plan 的 10 任务全景

1. Project Setup（go.mod、目录结构、main.go）
2. CLI Framework with Help（cobra root）
3. Sierpinski Algorithm（递归分形）
4. Sierpinski CLI Integration
5. Mandelbrot Algorithm（ASCII 渲染）
6. Mandelbrot CLI Integration
7. Character Set Configuration
8. Input Validation and Error Handling
9. Integration Tests
10. README

**任务粒度**：每任务 1-2 个文件创建 + 1-2 个 verify 命令。**适中**——不是单文件 task（粒度太细），也不是"完成整个 CLI" task（粒度太粗）。

### 4.7 svelte-todo plan 的 12 任务全景

1. Project Setup（npm create vite、依赖）
2. Todo Store（store.ts）
3. localStorage Persistence（storage.ts）
4. TodoInput Component
5. TodoItem Component
6. TodoList Component
7. FilterBar Component
8. App Integration
9. Filter Functionality
10. Styling and Polish（CSS）
11. End-to-End Tests（Playwright）
12. README

**任务粒度**：每个组件独立 task。**很细**——12 任务适合 subagent 高度并行。

---

## 五、优点亮点

### 5.1 真实脚手架 + 真实任务

每个项目自带 `scaffold.sh` + 真实代码——LLM 拿到的是**真实可跑**的初始状态。

### 5.2 权限预授权避免 LLM 中断

`.claude/settings.local.json` 预授权 `Bash(go:*)`、`Bash(npm:*)` 等——LLM 不会被权限询问打断。

### 5.3 Do/Verify 模板让 TDD 自然

```markdown
**Do:**
- 创建 `internal/sierpinski/sierpinski.go`
- 实现 `Generate(size, depth int, char rune) []string`

**Verify:**
- `go test ./internal/sierpinski/...` passes
```

每个 task 都有**可执行的验证命令**——LLM 知道"做完没"。

### 5.4 双栈覆盖（Go CLI + Svelte Web）

两个项目**语言/范式都不同**——验证 SDD 栈无关。

### 5.5 Token 使用量展示

```bash
jq -s '[.[] | select(.type == "result")] | last | .usage' "$LOG_FILE"
```

让用户看到成本——是**透明性**的体现。

### 5.6 详细的 next steps 引导

```bash
echo "1. Review the project: cd $OUTPUT_DIR/project"
echo "2. Review Claude's log: less $LOG_FILE"
echo "3. Check if tests pass:"
echo "   cd $OUTPUT_DIR/project && go test ./..."
```

测试**引导**用户做后续——这是"人类可读"测试的范本。

### 5.7 错误处理：`scaffold.sh` 缺参数时报错

```bash
TARGET_DIR="${1:?Usage: $0 <target-directory>}"
```

`${1:?...}` 在 bash 中是"必填参数"语法——缺失即报错。

### 5.8 `Available tests:` 列表

```bash
if [[ ! -d "$TEST_DIR" ]]; then
    echo "Available tests:"
    ls -1 "$SCRIPT_DIR" | grep -v '\.sh$' | grep -v '\.md$'
    exit 1
fi
```

错误信息**主动列出**可用测试名——是优秀 CLI 错误处理的范本。

### 5.9 测试项目作为 showcase

`subagent-driven-dev/` 同时是：
- 测试套件（验证 SDD 技能）
- Showcase（展示 SDD 技能能做什么）
- 文档（`design.md` + `plan.md` 是 SDD 怎么用的范例）

---

## 六、缺点风险局限

### 6.1 测试结果依赖用户手动验证

```bash
echo "3. Check if tests pass:"
echo "   cd $OUTPUT_DIR/project && go test ./..."
```

测试**不**自动跑 `go test` 或 `npm test`——只让用户**自己**验证。这是**手动** E2E 测试。

### 6.2 LLM 采样非确定性

整个 plan 跑完要 30+ 分钟，消耗大量 token。如果 LLM 在某 task 上**不**调 subagent 而是直接干，结果会**波动**。

### 6.3 token 成本极高

完成 10-12 task 的 plan 估计消耗 **100K-500K tokens**（含 subagent 调用）。单次跑可能花 5-10 美元。

### 6.4 无并发控制

`run-test.sh` 串行跑——不能并行跑 go-fractals + svelte-todo。**不**像 npm test 可以并发。

### 6.5 `scaffold.sh` 模板未抽取

两个 `scaffold.sh` 95% 相同——仅 settings.local.json 的 `Bash(*)` 不同。**应当**抽取到 `_lib/scaffold-template.sh`。

### 6.6 settings.local.json 的过度授权

```json
"allow": ["Read(**)", "Edit(**)", "Write(**)", "Bash(go:*)", "Bash(git:*)", "Bash(mkdir:*)"]
```

`Read(**)` 允许读**任何**文件——可能读到 `/etc/passwd`。**测试专用**才安全。

### 6.7 LLM 跑 `go mod init` 网络下载

plan 中有"Add `github.com/spf13/cobra` dependency"——LLM 需要 `go get` 下载依赖。CI 环境需要网络。

### 6.8 不验证最终代码质量

测试**不**检查"代码是否符合 best practice"——只检查"项目能跑、测试能过"。**linting/format** 留给用户。

### 6.9 timeout 缺失

`claude -p` 没有 `timeout` 包裹（与 `explicit-skill-requests/` 不同）——如果 LLM 卡住，整个测试**永远跑不完**。

### 6.10 缺断言 = 缺回归保护

测试**不**断言任何东西（除了"exit 0"——claude 是否成功返回）。**真正的**验证在用户手动的 `go test ./...` 和 `npm test` 中。

这意味着：**如果 plan 被 LLM 完全搞错，run-test.sh 仍会"成功"退出**。

### 6.11 没有"Plan 完整性"断言

测试**不**检查"LLM 是否完成了所有 10/12 tasks"。只检查"是否到达终点"（不超时）。

### 6.12 "Available tests" 列表漂移

`ls -1 "$SCRIPT_DIR" | grep -v '\.sh$' | grep -v '\.md$'` 假设子目录命名不含 `.sh`、`.md` 后缀。如果有人命名 `tests-rust.sh/`，会被过滤掉。

---

## 七、关键洞察

### 7.1 这是"测试即 showcase"的设计

`subagent-driven-dev/` 不仅是测试——它**同时是**：
- 验证 SDD 技能能跑
- 展示 SDD 技能能做什么
- 文档化 SDD 技能怎么用

**三重价值**让 8 个文件的投入非常划算。

### 7.2 真实脚手架 vs Mock 脚手架

很多 LLM 测试用 **mock fixture**——`ls` 出的目录里只有 `prompt.txt`、`expected.txt`。本目录用**真实脚手架**——`scaffold.sh` 产出可编译的项目。

真实脚手架的代价是**慢**（每测试要 init 仓库 + 写 settings），但**真实性**远超 mock。

### 7.3 权限预授权的"测试哲学"

```json
"allow": ["Read(**)", "Edit(**)", "Write(**)", "Bash(go:*)", "Bash(git:*)", "Bash(mkdir:*)"]
```

这是**测试专用**的激进设置——给 LLM 自由以避免中断。**生产环境**绝不能这样。

### 7.4 Do/Verify 模板的 TDD 友好

每个 task 都有 `Verify:` 段——LLM 知道"做完没"。这是 TDD 的"红绿"循环在 plan 层的体现。

### 7.5 双栈覆盖 = 普适性证据

如果 SDD 在 Go 和 Svelte（两种范式）上都能成功，说明 SDD **栈无关**——可以推广到 Rust、Python、Java。

### 7.6 plan.md 的"任务粒度"哲学

- go-fractals: 10 任务
- svelte-todo: 12 任务

每任务 1-3 文件、1-2 verify 命令。这种粒度**适合** subagent 单次执行。

### 7.7 `ls -1 | grep -v` 的脆弱性

```bash
ls -1 "$SCRIPT_DIR" | grep -v '\.sh$' | grep -v '\.md$'
```

通过"反匹配文件后缀"识别子目录——脆弱。如果新文件类型加入，需更新 grep。

### 7.8 测试"项目级"而非"组件级"

其他 tests/ 子目录多测单文件、单函数。本目录测**完整项目**——这是 E2E 测试的金标准。

### 7.9 token 跟踪的"成本可见性"

```bash
jq -s '[.[] | select(.type == "result")] | last | .usage'
```

让用户**看到** token 用量——成本透明是 LLM 工具的必备特性。

### 7.10 缺断言 ≠ 测试失败

`run-test.sh` 没有任何 `assert_*`——**真正的**验证在用户手动跑 `go test` / `npm test`。这是"测试是脚手架"而非"测试是断言机"的设计。

---

## 八、关联文档

### 8.1 关联生产代码

- [`skills/subagent-driven-development/SKILL.md`](file://e:\WorkStation\MyProject\superpowers\skills\subagent-driven-development\SKILL.md) — 被测技能
- `skills/subagent-driven-development/implementer-prompt.md` — subagent 模板
- `skills/subagent-driven-development/code-quality-reviewer-prompt.md`
- `skills/subagent-driven-development/spec-reviewer-prompt.md`

### 8.2 关联其他 tests/

- `tests/claude-code/test-subagent-driven-development.sh` — 单技能测试
- `tests/claude-code/test-subagent-driven-development-integration.sh` — 端到端断言版
- `tests/claude-code/analyze-token-usage.py` — token 用量分析工具
- `tests/explicit-skill-requests/run-multiturn-test.sh` — 多 turn 对话构造（姊妹模式）

### 8.3 项目元文件

- `CLAUDE.md` / `AGENTS.md` — 项目级 AI 指令
- `docs/testing.md` — 顶层测试策略

### 8.4 缺失文档

- 本目录**没有 README.md**——但 `run-test.sh` 注释中说明用法
- 没有"如何添加新测试项目"的指南
- 没有"如何解读测试结果"——用户拿到项目后**自己**判断

---

## 九、状态评价

### 9.1 健康度

| 维度 | 评分 | 备注 |
|------|------|------|
| 真实性 | ⭐⭐⭐⭐⭐ | 真实脚手架 + 真实 plan |
| 跨栈覆盖 | ⭐⭐⭐⭐⭐ | Go CLI + Svelte Web |
| 可重跑 | ⭐⭐⭐⭐ | 时间戳目录隔离 |
| 失败诊断 | ⭐⭐ | 缺断言，用户手动验证 |
| 成本控制 | ⭐⭐ | 高 token 消耗 |
| 双角色（测试+展示） | ⭐⭐⭐⭐⭐ | 同时是 showcase |
| 文档化 | ⭐⭐ | 缺 README |
| 并发安全 | ⭐⭐ | 串行 + 缺 timeout |

### 9.2 适用场景

- **新人 onboarding**：看 SDD 技能怎么用
- **CI nightly**：完整 plan 跑一遍
- **技能升级时**：看是否仍能完成 plan
- **模型升级时**：看是否仍能 follow 10+ task 计划

### 9.3 不适用场景

- PR check（太慢、太贵）
- 离线/沙箱
- 高频 CI

---

## 十、给读者的启示

### 10.1 "测试即 showcase" 的高价值设计

让测试**同时**是 showcase——降低维护成本（同一份代码两用）。**值得**其他 LLM 集成项目借鉴。

### 10.2 真实脚手架是端到端测试的金标准

不要 mock 项目结构——让 LLM 拿到**真实可编译**的代码。**真实性**让测试**真正**捕获集成问题。

### 10.3 Do/Verify 是 plan 模板的 TDD 体现

```markdown
**Do:**
- ...

**Verify:**
- 验证命令
```

让 LLM 知道"做完没"——是 plan 层的 TDD 实践。

### 10.4 权限预授权是测试专用设置

```json
"allow": ["Read(**)", "Edit(**)", "Write(**)", "Bash(go:*)"]
```

`Read(**)`、`Bash(*)` 的激进授权**仅限测试**——生产环境必须收紧。

### 10.5 任务粒度的"分形"哲学

每任务 1-3 文件、1-2 verify——这种粒度**适合** subagent 并行。粒度太粗不能并行；太细 overhead 高。

### 10.6 Token 使用量是"成本透明性"

```bash
jq -s '[.[] | select(.type == "result")] | last | .usage'
```

让用户看到 token 成本——LLM 工具的必备透明性。

### 10.7 双栈覆盖 = 普适性证据

Go + Svelte 是不同范式——SDD 能跑两者说明**栈无关**。**值得**其他 LLM 集成项目借鉴。

### 10.8 缺断言 ≠ 测试失败

本目录**没有**自动断言——真正的验证在用户手动跑测试。这是"测试是脚手架"的设计，让 LLM 失败由**人类**判断。

### 10.9 next steps 引导

```bash
echo "1. Review the project: cd $OUTPUT_DIR/project"
echo "2. Review Claude's log: less $LOG_FILE"
echo "3. Check if tests pass:"
```

测试**引导**用户做后续验证——是优秀 CLI 设计的范本。

### 10.10 "Available tests:" 错误处理

```bash
echo "Available tests:"
ls -1 "$SCRIPT_DIR" | grep -v '\.sh$' | grep -v '\.md$'
```

错误信息**主动列出**可用选项——是优秀 CLI 错误处理。

---

## 十一、后续工作

### 11.1 立即可做

- [ ] 添加 README.md 说明两个项目、运行方式、token 成本预期
- [ ] 抽取共享 `scaffold-template.sh` 减少重复
- [ ] 给 `claude -p` 添加 `timeout 1800` 防止 LLM 卡住
- [ ] 添加 `--parallel` flag 并行跑两个项目

### 11.2 中期可做

- [ ] 添加第三个测试项目（如 `rust-web-server` 覆盖另一栈）
- [ ] 引入"自动验证"阶段：跑完 plan 后**自动**跑 `go test` / `npm test`，把结果断言进 exit code
- [ ] 添加 plan 完成度检查（"是否所有 task 都有 commit"）
- [ ] 输出 junit XML 让 CI 系统解析

### 11.3 长期演进

- [ ] 把 plan.md / design.md 抽为模板，contributors 用模板添加新项目
- [ ] 写"如何添加新测试项目"指南
- [ ] 引入 token 成本预算（>X tokens 警告）
- [ ] 探索 LLM "judge" 模型对最终代码打分

### 11.4 跨项目借鉴

`subagent-driven-dev/` 的以下模式值得其他 LLM 集成项目借鉴：

1. **测试即 showcase**：同一份代码两用
2. **真实脚手架**：不 mock 真实项目结构
3. **Do/Verify 模板**：TDD 在 plan 层的体现
4. **权限预授权**：测试专用的激进设置
5. **任务粒度的"分形"**：1-3 文件/任务适合 subagent
6. **Token 透明性**：让用户看到成本
7. **双栈覆盖**：验证栈无关性
8. **next steps 引导**：优秀 CLI 设计
9. **`Available tests:` 错误处理**：主动列出选项
10. **缺断言 = 人类判断**：测试是脚手架而非断言机

---

**报告完成时间**：2026-06-03
**分析对象**：`tests/subagent-driven-dev/` 全部 8 个文件
**分析深度**：多文件全深度，1 orchestrator + 2 项目完整剖析
**对应源目录**：[`tests/subagent-driven-dev/`](file://e:\WorkStation\MyProject\superpowers\tests\subagent-driven-dev)
**关联技能**：[`skills/subagent-driven-development/SKILL.md`](file://e:\WorkStation\MyProject\superpowers\skills\subagent-driven-development\SKILL.md)
**姊妹测试**：[`tests/claude-code/test-subagent-driven-development-integration.sh`](file://e:\WorkStation\MyProject\superpowers\tests\claude-code\test-subagent-driven-development-integration.sh)
