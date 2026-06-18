# tests/claude-code/ 深度分析报告

> 目录: `tests/claude-code/`（9 个文件，~57 KB）
> 角色: superpowers 技能在 Claude Code 上的"行为正确性"测试套件
> 测试对象: 13 个技能 + 实际 Claude Code 行为

---

## 1. 基本信息

| 字段 | 值 |
|------|-----|
| 目录路径 | `tests/claude-code/` |
| 文件数量 | 9（README + run-skill-tests + test-helpers + 6 测试） |
| 总规模 | ~57 KB |
| 测试对象 | Claude Code 行为 + 13 个 superpowers 技能 |
| 测试模式 | 单元测试（`claude -p` headless）+ 集成测试（执行实际工作流） |
| 外部依赖 | Claude Code CLI + python3（token 分析） |
| 测试哲学 | RED-GREEN-REFACTOR（参考 testing-skills-with-subagents.md） |

### 9 个文件清单

| 文件 | 行数 | 类型 | 角色 |
|------|------|------|------|
| `README.md` | 171 | 文档 | 测试套件使用文档 |
| `test-helpers.sh` | 203 | 库 | 通用 bash 辅助函数（`run_claude`、`assert_*`） |
| `run-skill-tests.sh` | 189 | 工具 | 测试运行器（支持 verbose、timeout、filter） |
| `test-subagent-driven-development.sh` | 166 | 单元 | 验证 SDD 技能内容（9 个断言） |
| `test-worktree-native-preference.sh` | 177 | 单元 | RED/GREEN/PRESSURE 三阶段 worktree 测试 |
| `test-subagent-driven-development-integration.sh` | 323 | 集成 | 真实执行 SDD 工作流（10-30 分钟） |
| `test-requesting-code-review.sh` | 215 | 集成 | 验证 code reviewer 子代理发现 planted bugs |
| `test-document-review-system.sh` | 178 | 集成 | 验证 spec 文档审查者发现错误 |
| `analyze-token-usage.py` | 169 | 工具 | 从 JSONL session 转录分析 token 用量 |

---

## 2. 目的与背景

superpowers 的核心价值主张是"教会 AI 如何使用技能"。但 AI 模型**不会自动**遵循技能说明——需要在训练 / 提示中显式声明。

**这个测试目录的存在是为了**：

1. **技能指令完整性**：每个技能包含必要的工作流步骤
2. **行为正确性**：Claude 真的能"按技能说的做"（不只是能"背诵技能内容"）
3. **REGRESSION 保护**：防止技能说明被悄悄修改
4. **工作流验证**：完整 subagent-driven-development 工作流真的能跑通
5. **Code reviewer 质量**：code review 子代理真的能发现 bug
6. **压力测试**：在 pressure 下行为不退化

**这是"技能 TDD"（RED-GREEN-REFACTOR 应用于技能文档）**的实践。

---

## 3. 核心技术决策

### 3.1 测试哲学：RED-GREEN-PRESSURE

来自 `test-worktree-native-preference.sh` 行 1-14：

```
# Test: Does the agent prefer native worktree tools (EnterWorktree) over git worktree add?
# Framework: RED-GREEN-REFACTOR per testing-skills-with-subagents.md
#
# RED:   Skill without Step 1a (no native tool preference). Agent should use git worktree add.
# GREEN: Skill with Step 1a (explicit tool naming + consent bridge). Agent should use EnterWorktree.
# PRESSURE: Same as GREEN but under time pressure with existing .worktrees/ dir.
#
# Key insight: the fix is Step 1a's text, not file separation. Three things make it work:
#   1. Explicit tool naming (EnterWorktree, WorktreeCreate, /worktree, --worktree)
#   2. Consent bridge ("user's consent = authorization to use native tool")
#   3. Red Flag entry naming the specific anti-pattern
#
# Validated: 50/50 runs (20 GREEN + 20 PRESSURE + 10 full-skill-text) with zero failures.
```

**核心概念**：

- **RED 阶段**：故意写"坏"技能（或无关键步骤），验证 agent 行为变坏
- **GREEN 阶段**：加入修复，验证 agent 行为变好
- **PRESSURE 阶段**：在压力下（时间紧迫、已有 directory）验证行为不退化

**关键洞察（行 9-13）**：

- 修复是**文本级别**的（Step 1a 的措辞）
- 不是文件分离（结构级别）
- 3 个元素让 Step 1a 工作：显式工具名 + 同意桥接 + 红旗列表

**这是"测试驱动的技能写作"最佳实践**——50/50 runs 0 failures = 真实证据。

### 3.2 测试运行器的多模式设计

`run-skill-tests.sh` 行 31-72 的参数解析：

```bash
while [[ $# -gt 0 ]]; do
    case $1 in
        --verbose|-v) VERBOSE=true; shift ;;
        --test|-t) SPECIFIC_TEST="$2"; shift 2 ;;
        --timeout) TIMEOUT="$2"; shift 2 ;;
        --integration|-i) RUN_INTEGRATION=true; shift ;;
        --help|-h) usage ;;
        *) echo "Unknown option: $1"; exit 1 ;;
    esac
done
```

**关键能力**：

- `--test` 跑特定测试
- `--timeout` 调整超时（CI 友好）
- `--integration` 包含慢测试（10-30 分钟）
- `--verbose` 看完整输出（默认只显示失败）

**这是"开发者 + CI 友好"的双模式**。

### 3.3 通用 assertion 库

`test-helpers.sh` 的 5 个核心函数：

1. `run_claude "prompt" [timeout] [allowed_tools]` — 调 Claude Code
2. `assert_contains output pattern name` — 输出包含
3. `assert_not_contains output pattern name` — 输出不包含
4. `assert_count output pattern count name` — 出现 N 次
5. `assert_order output pattern_a pattern_b name` — A 在 B 前

**`assert_order` 设计**特别精巧：

```bash
assert_order() {
    local line_a=$(echo "$output" | grep -n "$pattern_a" | head -1 | cut -d: -f1)
    local line_b=$(echo "$output" | grep -n "$pattern_b" | head -1 | cut -d: -f1)
    if [ "$line_a" -lt "$line_b" ]; then
        echo "  [PASS] ..."
    fi
}
```

**为什么 `assert_order` 重要**：

- superpowers 技能严格定义工作流顺序
- "spec compliance before code quality" = 必须先 spec 再 quality
- 单纯检查"两个词都存在"不够，必须**顺序正确**

**这是"工作流断言"的关键工具**。

### 3.4 集成测试用真实场景构造"假项目"

`test-subagent-driven-development-integration.sh` 行 31-105：

- 创建 `package.json`（Node.js 项目）
- 创建 `docs/superpowers/plans/implementation-plan.md`（2 任务）
- `git init` + 初始 commit
- 然后调 Claude 执行

**核心洞察**：

- 测试是"假项目 + 真实 Claude + 真实技能"
- 假项目 = 最小可工作示例（hello + goodbye）
- 真实 Claude = `claude -p` headless 模式
- 真实技能 = `--plugin-dir` 加载 superpowers

**这是"集成测试的金标准"**——用真实组件，只 mock 外部依赖（项目状态）。

### 3.5 Session JSONL 解析：从 transcripts 验证行为

`test-subagent-driven-development-integration.sh` 行 173-178：

```bash
TEST_PROJECT_REAL=$(cd "$TEST_PROJECT" && pwd -P)
SESSION_DIR="$HOME/.claude/projects/$(echo "$TEST_PROJECT_REAL" | sed 's|[^a-zA-Z0-9]|-|g')"
SESSION_FILE=$(ls -t "$SESSION_DIR"/*.jsonl 2>/dev/null | head -1 || true)
```

**核心思路**：

- Claude Code 把 session 记录到 `~/.claude/projects/<normalized-cwd>/*.jsonl`
- 测试**不解析 stdout**，而**读 JSONL**获取结构化数据
- 验证 `"name":"Skill"` 调用、`"name":"Agent"` subagent 调用、`"name":"TodoWrite"` 任务跟踪

**`|| true` 防止 pipefail 杀死脚本**：`ls` 可能被 `head -1` SIGPIPE。

**这是"通过 logs 验证 AI 行为"的工程模式**——比解析 stdout 精确得多。

### 3.6 "Planted bug" 测试：真实验证 reviewer

`test-requesting-code-review.sh` 行 56-90：

```javascript
// Baseline: parameterized findUserByEmail (安全)
// Second commit:
// 1. SQL injection (string concat)
// 2. 密码 hash 在登录成功时打印
// 3. hash() 是 identity function
```

**核心技巧**：

- **两次 commit**：baseline (安全) → planted (3 个 bug)
- 故意在第二次 commit 引入 3 个真实可识别 bug：
  1. SQL injection（字符串拼接）
  2. 日志打印 password_hash
  3. `hash(s) = s`（明文）
- 调 code reviewer
- 验证 reviewer 报告**至少一个**为 Critical/Important
- 验证 reviewer **不批准** merge

**"planted bug" 是"测试 reviewer"的最佳方法**——用真实可识别的 bug，而非人工写"看起来像 bug"的代码。

### 3.7 bash 的 `<(...)` + `cd` 隔离

`test-subagent-driven-development-integration.sh` 行 154-156：

```bash
cd "$TEST_PROJECT" && timeout 1800 claude -p "$PROMPT" --plugin-dir "$PLUGIN_DIR" --allowed-tools=all --permission-mode bypassPermissions 2>&1 | tee "$OUTPUT_FILE" || {
    echo ""
    echo "================================================================================"
    echo "EXECUTION FAILED (exit code: $?)"
    exit 1
}
```

**关键参数**：

- `cd $TEST_PROJECT`：让 Claude 在测试项目目录运行
- `--plugin-dir`：强制加载 superpowers 插件
- `--allowed-tools=all`：允许所有工具（headless 模式必须）
- `--permission-mode bypassPermissions`：绕过权限询问
- `tee`：同时输出到屏幕和文件
- `|| { ... exit 1; }`：错误处理（避免 pipefail 影响）

**这是"headless Claude 集成测试"的标准参数组合**。

### 3.8 Token 用量分析

`analyze-token-usage.py` 行 1-9：

```python
"""
Analyze token usage from Claude Code session transcripts.
Breaks down usage by main session and individual subagents.
"""
```

**核心功能**：

- 解析 JSONL session 文件
- 区分 main session 和 subagent usage
- 按 `agentId` 跟踪子代理
- 计算 `input_tokens` / `output_tokens` / `cache_creation` / `cache_read`
- 估算 cost（$3/$15 per M tokens）

**`cache_creation` / `cache_read`**：Anthropic API 的 prompt cache 机制——第二次调用若前缀相同，可读 cache 而非重新计算。

**这是一个"成本可观测性"工具**——帮开发者理解 skill 实际成本。

### 3.9 测试数据隔离：每个测试有独立 tmpdir

`test-helpers.sh` 行 127-130：

```bash
create_test_project() {
    local test_dir=$(mktemp -d)
    echo "$test_dir"
}
```

**关键优势**：

- `mktemp -d` 创建唯一 tmp 目录
- 测试间**无共享状态**
- 失败时不污染其他测试
- `cleanup_test_project "$test_dir"` 显式清理

**这是"测试隔离的金标准"**——无状态污染 = 可靠的测试结果。

---

## 4. 文件结构剖析

```
tests/claude-code/
├── README.md (171 行) ──────────── 文档
├── test-helpers.sh (203 行) ────── 通用库
├── run-skill-tests.sh (189 行) ─── 测试运行器
│
├── 单元测试 (快速，~2 分钟)
│   ├── test-subagent-driven-development.sh (166 行) ─ 9 断言
│   │     - 技能识别
│   │     - 工作流顺序（spec compliance before code quality）
│   │     - self-review 需求
│   │     - plan 读一次
│   │     - reviewer 怀疑态度
│   │     - review loops
│   │     - 任务上下文传递
│   │     - worktree 前提
│   │     - main branch 红旗
│   │
│   └── test-worktree-native-preference.sh (177 行) ─ RED/GREEN/PRESSURE
│         - 50/50 runs 验证 0 失败
│         - "Step 1a 文本级别修复"
│
├── 集成测试 (慢速，10-30 分钟)
│   ├── test-subagent-driven-development-integration.sh (323 行) ─ 真实工作流
│   │     - 假 Node.js 项目
│   │     - 真实 claude -p 执行 SDD
│   │     - 验证 JSONL session 中:
│   │       - Skill 工具调用
│   │       - Agent 子代理 ≥ 2
│   │       - TodoWrite 使用
│   │       - src/math.js 创建
│   │       - npm test 通过
│   │       - 多 commit
│   │       - 无额外功能
│   │
│   ├── test-requesting-code-review.sh (215 行) ─ planted bug 测试
│   │     - 假项目 + 两次 commit
│   │     - 第一次: parameterized SQL (安全)
│   │     - 第二次: 3 个 planted bug
│   │     - 验证 reviewer 报告 Critical
│   │     - 验证 reviewer 不批准 merge
│   │
│   └── test-document-review-system.sh (178 行) ─ spec 审查
│         - 创建带 TODO + "specified later" 的假 spec
│         - 验证 reviewer 捕获错误
│
└── analyze-token-usage.py (169 行) ──── 工具
      - 解析 JSONL
      - 按 subagent 分解
      - cost 估算
```

---

## 5. 优点亮点

- **RED-GREEN-PRESSURE 三阶段测试**：worktree 测试 50/50 runs 0 failures
- **真实工作流集成测试**：执行完整 plan 而非 mock
- **planted bug 测试**：code reviewer 用 3 个真实可识别 bug 验证
- **Session JSONL 解析**：从 logs 验证 AI 行为而非 stdout
- **`assert_order` 工具**：验证工作流顺序
- **多种 timeout 控制**：单元 30s、集成 1800s
- **CI 友好**：默认只跑快速测试，`--integration` 跑慢的
- **verbose / 安静双模式**：开发者 / CI 友好
- **token 用量分析**：成本可观测性
- **跨测试隔离**：`mktemp -d` 独立 tmp 目录
- **headless Claude 集成**：`--plugin-dir --allowed-tools=all --permission-mode bypassPermissions`
- **显式 cleanup**：`trap cleanup_test_project EXIT`
- **`tee` 输出**：同时 stdout + file
- **`|| true` 防 pipefail**：避免 SIGPIPE 杀死脚本
- **`pwd -P` 真实路径**：macOS 兼容
- **JSONL 自动发现**：`ls -t ... | head -1` 找最新 session
- **路径归一化**：`sed 's|[^a-zA-Z0-9]|-'` 符合 Claude 命名

---

## 6. 缺点风险局限

- **集成测试非常慢**：SDD 测试 10-30 分钟
- **`--permission-mode bypassPermissions` 安全风险**：CI 隔离环境下可接受
- **依赖 Claude Code 模型行为**：模型升级可能让测试 flaky
- **`assert_count` 用 `grep -c` 含 exit 1 行为**：用 `|| echo "0"` 兜底
- **没有 deterministic 控制**：模型输出有随机性
- **`run_claude` 用 `mktemp` 文件**：tmp 不清理可能累积（虽然有 `rm -f`）
- **断言靠文本匹配**：模型改变措辞可能让测试失败
- **没有 metric 历史**：跨运行的 pass rate 无法追踪
- **CI 缺 mock 集成测试**：实际跑真实模型（贵）
- **token 估算价格可能过时**：$3/$15 per M 是 Anthropic 当时价格
- **没有结构性 XML 解析**：纯 grep 提取 JSONL 字段
- **未隔离 cost 风险**：意外循环调用可能产生大账单
- **planted bug 测试用 Node.js 风格**：codebase 实际可能用不同语言
- **没有针对 Cursor / Codex 的等价测试**：只在 Claude Code 上

---

## 7. 关键洞察

1. **"RED-GREEN-PRESSURE" 是技能 TDD 的核心**：50/50 runs 0 failures 是真实证据，不是断言
2. **修复是文本级别（Step 1a）而非结构级别**：3 元素（工具名+桥接+红旗）让 skill 工作
3. **`<(...)` process substitution + `cd` 隔离**：让 Claude 在测试项目目录运行
4. **Session JSONL 是 AI 行为的"数据库"**：从 logs 验证行为比 stdout 精确
5. **Planted bug 是 reviewer 测试的金标准**：用真实可识别 bug 而非 mock
6. **`assert_order` 验证工作流顺序**：superpowers 强依赖顺序
7. **`mktemp -d` 跨测试隔离**：无状态污染 = 可靠结果
8. **`tee` 同时输出到屏幕和文件**：调试友好
9. **`pwd -P` 真实路径**：macOS `/var/folders/` ↔ `/private/var/folders/` 陷阱
10. **`|| true` 防 pipefail**：`head -1` 的 SIGPIPE 不杀脚本
11. **Token cache `cache_creation` / `cache_read`**：Anthropic API 的成本优化
12. **`--allowed-tools=all` + `bypassPermissions`**：headless 模式的必需组合
13. **3 planted bug 类别（SQL 注入、密码日志、identity hash）**：覆盖常见安全漏洞
14. **`$HOME/.claude/projects/<normalized-cwd>/*.jsonl`**：Claude 的 session 组织方式
15. **`--plugin-dir` 而非全局安装**：让测试用本地 superpowers 副本
16. **bash 的 `set -euo pipefail`**：严格模式，但需 `|| true` 防过度
17. **`grep -n` 给行号**：让 `assert_order` 能比较先后
18. **`head -1 | cut -d: -f1`**：取第一匹配的行号
19. **`pwd -P` + `sed` normalize**：把绝对路径变成 Claude 用的目录名

---

## 8. 关联文档

| 文档/文件 | 关系 |
|----------|------|
| [`skills/using-superpowers/SKILL.md`](file://e:/WorkStation/MyProject/superpowers/skills/using-superpowers/SKILL.md) | 测试验证"如何用 superpowers" |
| [`skills/subagent-driven-development/SKILL.md`](file://e:/WorkStation/MyProject/superpowers/skills/subagent-driven-development/SKILL.md) | 多个测试对象 |
| [`skills/using-git-worktrees/SKILL.md`](file://e:/WorkStation/MyProject/superpowers/skills/using-git-worktrees/SKILL.md) | test-worktree-native-preference.sh 的目标 |
| [`skills/requesting-code-review/SKILL.md`](file://e:/WorkStation/MyProject/superpowers/skills/requesting-code-review/SKILL.md) | test-requesting-code-review.sh 的目标 |
| [`skills/brainstorming/spec-document-reviewer-prompt.md`](file://e:/WorkStation/MyProject/superpowers/skills/brainstorming/spec-document-reviewer-prompt.md) | test-document-review-system.sh 的目标 |
| [`skills/writing-skills/testing-skills-with-subagents.md`](file://e:/WorkStation/MyProject/superpowers/skills/writing-skills/testing-skills-with-subagents.md) | RED-GREEN-REFACTOR 框架来源 |
| `docs/testing.md` | 项目级 testing 指南 |
| `~/.claude/projects/<cwd>/*.jsonl` | Claude session 存储位置 |
| Anthropic API prompt cache 文档 | `cache_creation` / `cache_read` 来源 |

---

## 9. 状态评价

| 维度 | 评分 | 说明 |
|------|------|------|
| 完整性 | 9/10 | 覆盖单元 + 集成 + 性能 + reviewer |
| 正确性 | 9/10 | 真实场景，标准测试向量 |
| 可维护性 | 9/10 | 通用库 + 专用测试分离 |
| 可读性 | 9/10 | 测试名即文档 |
| 性能 | 7/10 | 集成测试 10-30 分钟 |
| 健壮性 | 9/10 | 多层 cleanup、隔离、错误处理 |
| 可移植性 | 9/10 | macOS / Linux 全覆盖 |
| 文档化 | 10/10 | README + 注释 + 头部说明 |
| 教育价值 | 10/10 | RED-GREEN-PRESSURE 范式 |

**总体评价**: 9/10 - 一个精心设计的、多层次的"技能 TDD"测试套件。

**改进优先级**:

1. **中**: 缓存模型输出结果（让 deterministic 测试稳定）
2. **中**: 添加更多 planted bug 类别（XSS、CSRF、path traversal）
3. **中**: 添加 cross-platform 集成测试（Cursor / Codex）
4. **低**: 减少 grep 依赖，用 jq 解析 JSONL
5. **低**: 自动追踪 pass rate 历史

---

## 10. 给读者的启示

- **"RED-GREEN-PRESSURE" 是技能 TDD 的核心范式**：50/50 runs 0 failures 是真实证据
- **修复是文本级别而非结构级别**：3 元素（工具名+桥接+红旗）让 skill 工作
- **Session JSONL 是 AI 行为的"数据库"**：从 logs 验证比 stdout 精确
- **Planted bug 是 reviewer 测试的金标准**：真实可识别 bug
- **`assert_order` 验证工作流顺序**：superpowers 强依赖顺序
- **`mktemp -d` 跨测试隔离**：无状态污染
- **`<(...)` + `cd` 让 Claude 在测试目录运行**：集成测试必备
- **`--allowed-tools=all` + `bypassPermissions` 是 headless 模式必需**
- **Token cache 机制**：`cache_creation` / `cache_read` 是成本优化
- **3 类 planted bug**：SQL 注入、密码日志、identity hash
- **`pwd -P` 解析真实路径**：macOS 符号链接陷阱
- **`|| true` 防 pipefail 误杀**：`head -1` SIGPIPE
- **JSONL 路径归一化**：`sed 's|[^a-zA-Z0-9]|-'` 符合 Claude 命名

---

## 11. 后续工作

### 11.1 短期改进（2-3 小时）

- [ ] 缓存模型输出（deterministic 测试）
- [ ] 添加 XSS、CSRF、path traversal 类 planted bug
- [ ] 添加 Cursor / Codex 等价测试
- [ ] 用 jq 替代 grep 解析 JSONL
- [ ] 自动追踪 pass rate 历史

### 11.2 中期改进（半天）

- [ ] 添加 performance benchmark（每个 skill 的 token cost）
- [ ] 添加 visual regression（截图对比）
- [ ] 添加"模型升级影响"分析
- [ ] 探索 prompt caching 利用
- [ ] 添加 assertion for reasoning（不只是 final output）

### 11.3 长期演进

- [ ] 多模型对比（Haiku / Sonnet / Opus）
- [ ] Property-based testing（自动生成 planted bug）
- [ ] 探索 shadow testing（新模型在生产流量下对比）
- [ ] 添加 failure 模式分类（model drift / skill bug / env issue）

### 11.4 风险预警

- ⚠️ Claude 模型升级可能让测试 flaky
- ⚠️ `--permission-mode bypassPermissions` CI 隔离要求高
- ⚠️ 集成测试 10-30 分钟可能超 CI timeout
- ⚠️ token 估算价格可能过时
- ⚠️ 路径归一化对中文/特殊字符支持

### 11.5 教育价值挖掘

- [ ] 录制"planted bug"工作坊
- [ ] 写博客："为什么 fix 是文本级别不是结构级别"
- [ ] 演示 RED-GREEN-PRESSURE 在 skill 写作的应用
- [ ] 写技术博客："JSONL session 是 AI 行为数据库"

### 11.6 跨平台 CI 矩阵

- [ ] macOS / Linux 矩阵
- [ ] 不同 Claude 模型版本
- [ ] 不同 Node.js 版本
- [ ] 收集 flaky test 统计

---
