# tests/explicit-skill-requests/ 目录分析报告

> 对应源目录：`tests/explicit-skill-requests/`
> 包含 6 个 shell 脚本 + 9 个 prompt 文件
> 报告定位：多文件分析，聚焦"用户显式请求技能时 Claude 是否正确触发 Skill 工具"

---

## 一、基本信息

| 字段 | 值 |
|------|-----|
| 目录路径 | `tests/explicit-skill-requests/` |
| 文件总数 | 15 个（6 个 `.sh` 脚本 + 9 个 `.txt` 提示词） |
| 总代码行数 | 约 850 行（脚本）+ 30 行（提示词） |
| 测试方式 | **真实 Claude CLI 集成测试**（依赖 `claude` 命令、需要 API 凭据） |
| 测试目标 | 验证"用户用自然语言点名技能"时 Claude 是否触发 Skill 工具 |
| 调用入口 | `run-all.sh`（运行 4 个核心场景） |
| 是否需要联网 | **是**（调用 `claude -p` 真实模型推理） |

`explicit-skill-requests/` 是 `tests/` 下"测试真实 LLM 行为"最密集的子目录。它不是 fixture-based，而是**端到端集成测试**——直接调用 `claude` CLI 跑一个 prompt，然后用 `grep` 检查 JSON 输出是否包含 Skill 工具调用。

---

## 二、目的与背景

### 2.1 测试目标

这是一个"行为正确性"测试套件，关注的核心问题是：

> **当用户用自然语言说出技能名称（如 "use brainstorming"）时，Claude 是否真的会调用 Skill 工具来加载该技能，而不是假装已经"知道"然后开始行动？**

这种"幻觉式行动"（hallucination action）是 LLM 集成中的经典失败模式——模型可能自诩已经"知道"SDD（subagent-driven-development）是什么，然后直接开始写代码，**完全跳过加载技能的步骤**。这会导致：

1. 技能中的"硬规则"被忽略（例如"先与用户对齐目标"）
2. 技能的工具调用模式不被遵守
3. 用户以为触发了技能，实际上 Claude 在"凭印象工作"

### 2.2 与其他 tests/ 子目录的关系

| 子目录 | 测试什么 | 是否需要 LLM |
|--------|----------|--------------|
| `claude-code/` | 框架特性（worktree、subagent、code review） | **是**（integration 模式） |
| `brainstorm-server/` | WebSocket / HTTP / 文件监视 | 否（纯 JS 测试） |
| `codex-plugin-sync/` | sync 脚本的行为 | 否（纯 shell fixture） |
| `opencode/` | OpenCode 插件加载 / 缓存 / 优先级 | 部分（integration 模式） |
| **`explicit-skill-requests/`** | **LLM 是否触发技能** | **是（全部）** |
| `skill-triggering/` | 朴素 prompt 是否触发技能 | 是（全部） |
| `subagent-driven-dev/` | 端到端 SDD 工作流 | 是（全部） |

`explicit-skill-requests/` 与 `skill-triggering/` 是姊妹套件——前者测试"用户点名"，后者测试"自然语言触发"。

### 2.3 出现背景

这些测试来自一次**实际失败**的复盘——根据 `run-claude-describes-sdd.sh` 注释"This mimics the original failure scenario"，原始失败场景是：

1. 用户问 Claude "subagent-driven-development 是什么"
2. Claude 详细描述了 SDD
3. 用户接着说 "subagent-driven-development, please"
4. Claude **没有**调用 Skill 工具，而是直接开始干活（"我懂了"）

这组测试就是为了**回归保护**——确保这种失败不再发生。

---

## 三、核心技术决策

### 3.1 真实 Claude CLI 调用

```bash
claude -p "$PROMPT" \
    --plugin-dir "$PLUGIN_DIR" \
    --dangerously-skip-permissions \
    --max-turns "$MAX_TURNS" \
    --output-format stream-json \
    > "$LOG_FILE" 2>&1 || true
```

**决策点**：使用 `claude -p`（print mode）单次调用，而非交互式。

**理由**：
- `-p` (print) 模式执行单次 prompt 然后退出，适合自动化
- `--output-format stream-json` 提供结构化 JSON 输出，便于 grep 解析
- `--dangerously-skip-permissions` 跳过权限检查（**测试专用**，不能用于生产）
- `> "$LOG_FILE" 2>&1 || true` 容忍非零退出（claude 可能因 max-turns 截断）

### 3.2 隔离的 HOME + 临时项目目录

```bash
TIMESTAMP=$(date +%s)
OUTPUT_DIR="/tmp/superpowers-tests/${TIMESTAMP}/explicit-skill-requests/${SKILL_NAME}"
mkdir -p "$OUTPUT_DIR"
PROJECT_DIR="$OUTPUT_DIR/project"
mkdir -p "$PROJECT_DIR/docs/superpowers/plans"
```

**决策点**：每个测试用例使用**全新时间戳**目录。

**理由**：
- 避免不同测试之间的 JSONL 会话文件相互污染
- 时间戳路径让多次运行可以并排比较
- 临时项目目录 + dummy plan 文件**模拟**真实规划流程

### 3.3 SKILL_PATTERN 兼容命名空间

```bash
SKILL_PATTERN='"skill":"([^"]*:)?'"${SKILL_NAME}"'"'
if grep -q '"name":"Skill"' "$LOG_FILE" && grep -qE "$SKILL_PATTERN" "$LOG_FILE"; then
```

**决策点**：正则同时匹配 `"skill":"X"` 和 `"skill":"namespace:X"`。

**理由**：
- Claude Code 在不同上下文（plugin 内部、plugin 外部）可能给技能加或不加命名空间前缀
- `([^"]*:)?` 让命名空间前缀可选
- 两条件 `&&` 确保：既确认调用了 Skill 工具（`"name":"Skill"`），又确认调用的是期望的技能

### 3.4 "Premature Action" 检测

```bash
FIRST_SKILL_LINE=$(grep -n '"name":"Skill"' "$LOG_FILE" | head -1 | cut -d: -f1)
if [ -n "$FIRST_SKILL_LINE" ]; then
    PREMATURE_TOOLS=$(head -n "$FIRST_SKILL_LINE" "$LOG_FILE" | \
        grep '"type":"tool_use"' | \
        grep -v '"name":"Skill"' | \
        grep -v '"name":"TodoWrite"' || true)
    if [ -n "$PREMATURE_TOOLS" ]; then
        echo "WARNING: Tools invoked BEFORE Skill tool:"
```

**决策点**：在第一次 Skill 工具调用**之前**的任何非 Skill / TodoWrite 工具都标记为"过早行动"。

**理由**：
- 这正是要捕获的失败模式
- TodoWrite 被豁免（合理：用户可能用 TodoWrite 来规划）
- 输出 WARNING 而非 FAIL——因为这是诊断信息，不一定算硬错误

### 3.5 多 turn 构造：模拟真实会话深度

```bash
claude -p "I want to add user authentication..."  # turn 1
claude -p "Let's use JWT tokens..."  --continue    # turn 2
claude -p "Great, write this up..."  --continue    # turn 3
claude -p "The plan looks good..."   --continue    # turn 4
claude -p "subagent-driven-development, please" --continue  # turn 5 (THE TEST)
```

**决策点**：`--continue` 标志串接多次 `-p` 调用，构建**累积会话历史**。

**理由**：
- 原始失败模式是"长会话后用户才点名技能"
- 仅 1 个 turn 不够——单 turn 测试永远会通过
- 5 轮（甚至 4 轮）让上下文窗口有足够"压力"

`run-extended-multiturn-test.sh` 名字直接说明意图："extended multi-turn test to reproduce the failure by building more context."

### 3.6 `set -e` + `|| true` 的妥协

```bash
timeout 300 claude -p "$PROMPT" ... > "$LOG_FILE" 2>&1 || true
```

**决策点**：claude 命令的非零退出被 `|| true` 吞掉。

**理由**：
- max-turns 限制会让 claude 自然"中断"，返回非零
- 测试目的是"看 JSON 输出里有什么"，而非"claude 退出码"
- 后续的 grep 才是真正的断言

### 3.7 Haiku 子测试：便宜模型压力测试

```bash
claude -p "..." --model haiku --plugin-dir "$PLUGIN_DIR" ...
```

**决策点**：`run-haiku-test.sh` 专门用便宜的 haiku 模型跑同样的多 turn 场景。

**理由**：
- 验证失败模式是否模型无关
- 如果 Opus 永远不失败、haiku 失败——可推测是模型能力问题
- 跑得快、便宜，适合 CI 频繁跑

---

## 四、文件结构剖析

### 4.1 6 个 shell 脚本的角色分工

| 文件 | 行数 | 角色 | 复杂度 |
|------|------|------|--------|
| `run-test.sh` | 137 | 核心单测：`<skill> <prompt>` 入口 | ⭐⭐⭐ |
| `run-all.sh` | 71 | 4 个核心场景的 orchestrator | ⭐⭐ |
| `run-multiturn-test.sh` | 144 | 3 turn 场景 | ⭐⭐⭐ |
| `run-extended-multiturn-test.sh` | 114 | 5 turn 场景（更深的上下文） | ⭐⭐⭐ |
| `run-haiku-test.sh` | 145 | haiku 模型下的 5 turn 场景 | ⭐⭐⭐ |
| `run-claude-describes-sdd.sh` | 101 | 复现"原始失败"：先描述再请求 | ⭐⭐⭐ |

### 4.2 `run-test.sh` 内部结构

```bash
# 1. 参数校验
SKILL_NAME="$1"
PROMPT_FILE="$2"
MAX_TURNS="${3:-3}"

# 2. 定位目录
SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
PLUGIN_DIR="$(cd "$SCRIPT_DIR/../.." && pwd)"

# 3. 准备输出
TIMESTAMP=$(date +%s)
OUTPUT_DIR="/tmp/superpowers-tests/${TIMESTAMP}/explicit-skill-requests/${SKILL_NAME}"
mkdir -p "$OUTPUT_DIR"
cp "$PROMPT_FILE" "$OUTPUT_DIR/prompt.txt"

# 4. 准备项目目录（含 dummy plan）
PROJECT_DIR="$OUTPUT_DIR/project"
mkdir -p "$PROJECT_DIR/docs/superpowers/plans"
cat > "$PROJECT_DIR/docs/superpowers/plans/auth-system.md" << 'EOF'
# Auth System Implementation Plan
...
EOF

# 5. 调用 claude
cd "$PROJECT_DIR"
timeout 300 claude -p "$PROMPT" ... > "$LOG_FILE" 2>&1 || true

# 6. 解析输出
SKILL_PATTERN='"skill":"([^"]*:)?'"${SKILL_NAME}"'"'
if grep -q '"name":"Skill"' "$LOG_FILE" && grep -qE "$SKILL_PATTERN" "$LOG_FILE"; then
    TRIGGERED=true
else
    TRIGGERED=false
fi

# 7. 诊断：过早行动检测
FIRST_SKILL_LINE=$(grep -n '"name":"Skill"' "$LOG_FILE" | head -1 | cut -d: -f1)
PREMATURE_TOOLS=$(head -n "$FIRST_SKILL_LINE" "$LOG_FILE" | grep ...)

# 8. 展示：被触发的所有技能
echo "Skills triggered in this run:"
grep -o '"skill":"[^"]*"' "$LOG_FILE" 2>/dev/null | sort -u

# 9. 退出码
[ "$TRIGGERED" = "true" ] && exit 0 || exit 1
```

### 4.3 `run-all.sh` 的 orchestrator 模式

```bash
SKILLS=("subagent-driven-development" "systematic-debugging" ...)
for skill in "${SKILLS[@]}"; do
    prompt_file="$PROMPTS_DIR/${skill}.txt"
    if "$SCRIPT_DIR/run-test.sh" "$skill" "$prompt_file" 3; then
        PASSED=$((PASSED + 1))
    else
        FAILED=$((FAILED + 1))
    fi
done
```

`run-all.sh` 是**线性 orchestrator**——按 skill 列表顺序调用 `run-test.sh`，统计 pass/fail 计数。注意它**没有**运行多 turn 测试（multiturn / extended / haiku）——那些是手动调用的，门槛更高。

### 4.4 prompts/ 目录的 9 个文件

| 文件 | 内容 | 用途 |
|------|------|------|
| `subagent-driven-development-please.txt` | "subagent-driven-development, please" | 直接点名 |
| `please-use-brainstorming.txt` | "please use the brainstorming skill..." | 直接点名 |
| `use-systematic-debugging.txt` | "use systematic-debugging..." | 直接点名 |
| `mid-conversation-execute-plan.txt` | 多行：先说有 plan，再说"SDD, please" | 模拟"中途请求" |
| `i-know-what-sdd-means.txt` | "I know what SDD means, just do it" | 抗干扰（用户自以为了解） |
| `claude-suggested-it.txt` | "you suggested SDD earlier, do it" | 抗干扰（Claude 自己提过） |
| `skip-formalities.txt` | "skip formalities, just SDD" | 抗干扰（用户要求跳步） |
| `action-oriented.txt` | "do something, but use SDD" | 抗干扰（混合指令） |
| `after-planning-flow.txt` | "刚完成 planning flow，现在用 SDD" | 抗干扰（连续场景） |

`run-all.sh` **只**调用 4 个核心 prompt。剩余 5 个是"抗干扰"prompt——它们的存在表明团队**预期**这种 prompt 会让 Claude 不触发技能，并把它当作**回归保护**。

### 4.5 多 turn 脚本的对话模板

每个多 turn 脚本都遵循相似模板：
- Turn 1：开场白（"我想加 auth，帮我思考")
- Turn 2：具体回答（"用 JWT 24h 过期"）
- Turn 3：要求生成 plan
- Turn 4：确认 plan，问执行选项
- Turn 5：**关键测试**（"subagent-driven-development, please"）

这是**精心设计**的脚本——它模拟了"完整 SDD 工作流上游"的真实用户行为。如果 Claude 在 turn 5 还能正确触发技能，说明**真的会**调用 Skill 工具，而不是凭 turn 1-4 的记忆。

---

## 五、优点亮点

### 5.1 真实 LLM 行为测试

这是最稀缺的测试类型。`run-test.sh` **不**测试 LLM 输出的内容正确性，只测试"是否调用了 Skill 工具"——这是**最关键的不变量**。技能内容会随 prompt 变化，但"先加载技能再行动"的协议**永远不能丢**。

### 5.2 "Premature Action" 检测

WARNING 信息"Tools invoked BEFORE Skill tool"是**行为正确性**的强信号。一个模型可能正确加载技能，但**先**调用 Read 工具读 plan——这种"过早读"也算失败。

### 5.3 多 turn 构造

5 轮对话构造了足够大的上下文，**才能暴露**真实失败模式。1 轮测试会全部通过，没有保护价值。

### 5.4 抗干扰 prompt 矩阵

9 个 prompt 覆盖了"用户可能怎样绕过技能调用"的所有角度。`i-know-what-sdd-means.txt` 直接挑战模型的"用户更懂"假设。

### 5.5 Haiku 子测试

便宜的模型做端到端 CI 是非常聪明的实践：失败模式在所有模型上都应被捕获，但**只**在 haiku 上做频繁测试可以节省成本。

### 5.6 时间戳目录隔离

`/tmp/superpowers-tests/${TIMESTAMP}/` 模式让**多次运行**可以**并排比较**（CI artifact、repro 调试）。

### 5.7 Dummy plan 文件策略

`cat > .../auth-system.md << 'EOF' ... EOF` 写入 dummy plan 模拟真实工作流。如果不写 plan，Claude 可能不会走 SDD 路径（因为没东西可执行）。

### 5.8 失败诊断的丰富输出

```
Skills triggered in this run:
  (sorted unique)
WARNING: Tools invoked BEFORE Skill tool:
  (list of tools)
First assistant response (truncated):
  (first 500 chars)
```

这种"失败时给出 3 个维度的证据"是优秀测试的标志——维护者无需重跑也能定位问题。

### 5.9 退出码语义

`[ "$TRIGGERED" = "true" ] && exit 0 || exit 1`——CI 系统会按退出码判断 pass/fail。**WARNING 不会让脚本失败**（这是设计选择：premature action 是诊断信息）。

### 5.10 `run-multiturn-test.sh` 的 `--continue` 标志链

```bash
claude -p "..." --continue  # 后续所有 turn 共享 session
```

这种 session 复用是**真实多 turn 对话**的正确模拟——而非每 turn 独立开新窗口。

---

## 六、缺点风险局限

### 6.1 完全依赖外部服务

测试**必须**有：
- 安装的 `claude` CLI
- 有效的 API 凭据
- 网络连接
- 大量 token 消耗

如果 API 限流或模型升级导致 JSON schema 微调，所有测试可能误报。

### 6.2 非确定性

LLM 输出是**采样性**的。即使 SKILL.md 完全没变，temperature > 0 也会让每次响应不同。测试**可能**因为采样抖动而 flake（间歇性失败）。

### 6.3 token 成本

每个测试场景消耗数千 token。完整跑一遍 `run-all.sh`（4 场景）+ 3 个多 turn 脚本（每个 5 turn）= 约 20 次 claude 调用。如果每天跑 10 次 CI，每次 1-2 美元，单月就是几百美元。

### 6.4 硬编码 `/tmp`

```bash
OUTPUT_DIR="/tmp/superpowers-tests/${TIMESTAMP}/..."
```

在 Windows 上 `/tmp` 不存在（即使是 MSYS2 也是 `/tmp` 软链接到 `C:\Users\...\AppData\Local\Temp`）。直接 `cd /tmp` 在 Git Bash 外可能失败。

### 6.5 dummy plan 模板重复

每个多 turn 脚本都重写一个 4-task auth-system.md。如果要测另一个技能，需要重写 dummy plan。提取为 fixture 模板会更 DRY。

### 6.6 `claude -p` 的隐式 max-turns

```bash
timeout 300 claude -p "$PROMPT" --max-turns "$MAX_TURNS" ...
```

`timeout 300` 配合 `--max-turns 3`：claude 不会真跑 3 个 turn 5 分钟，**理论**上 3 turn 应该远小于 5 分钟。但 5 分钟是 5 turn 的安全余量。

### 6.7 没有 prompt 验证

如果 `prompts/foo-skill-please.txt` 文件不存在，`run-all.sh` 会在 `cat "$PROMPT_FILE"` 时 fail，但错误信息不友好。应该在前置 `if [ ! -f ... ]; then` 检查。

### 6.8 命名空间正则覆盖不全

```bash
SKILL_PATTERN='"skill":"([^"]*:)?'"${SKILL_NAME}"'"'
```

这要求 SKILL_NAME **是字面字符串**。如果 SKILL_NAME 包含正则特殊字符（`.`、`*`、`[`），可能误匹配。`subagent-driven-development` 没有特殊字符所以是安全的，但**通用**用法不安全。

### 6.9 `run-all.sh` 不覆盖多 turn 测试

只有 `run-test.sh`（单 turn）通过 `run-all.sh` 编排。多 turn / extended / haiku 都需要**手动**跑——这意味着**常规 CI 不会跑这些更深层的保护**。

### 6.10 `set -e` 与 `|| true` 妥协

`timeout 300 claude -p "$PROMPT" ... > "$LOG_FILE" 2>&1 || true` 让 claude 失败不会传播。如果 claude 因配置错误一直立即返回（比如无 API key），测试会**悄无声息地 fail**（不是 pass，而是 TRIGGERED=false 但无法区分原因）。

### 6.11 缺少负向断言

测试**只**断言"应该触发技能"。没有"应该不触发 foo 技能"或"应该**只**触发 SDD"——例如，用 brainstorming 的 prompt 测试时，不验证 Claude 是否错误地加载了 SDD。

---

## 七、关键洞察

### 7.1 真实 LLM 测试是"行为契约"测试

这些测试不验证 LLM 的输出内容正确性（那是不可能的——没有 ground truth），只验证**协议合规性**（"先调用 Skill 工具"）。这是 AI 集成测试的核心范式转变。

### 7.2 多 turn 是必需的不是可选的

单 turn 测试**永远**会通过——所有"用户点名"的 prompt 在 turn 1 都会触发技能。**多 turn 才是真正考验**——它模拟"长会话后用户才明确请求"。

### 7.3 "Premature Action" 是隐性失败模式

测试不只检查"是否调用了技能"，还检查"是否**先**调用了其他工具"。这种"调用顺序正确性"是 LLM 行为的微妙契约。

### 7.4 抗干扰 prompt 矩阵的工程价值

9 个 prompt 不是冗余——它们各自捕获一种"用户尝试绕过技能调用"的方式。这是**对抗性测试设计**的雏形。

### 7.5 Haiku 子测试的"模型无关性"哲学

如果失败模式是"用户压力下 Claude 跳过技能"，那么它在 haiku 上应当**更容易**发生（模型能力越弱，偷懒倾向越大）。如果只在 Opus 上出现，haiku 上不出现，可能暗示问题是 Opus-specific 的。

### 7.6 `run-claude-describes-sdd.sh` 是 "regression of regression"

它复现的是**最初的失败场景**——Claude 先描述 SDD，然后用户请求 SDD，Claude 不再触发技能。这种"先入为主"是 LLM 的典型失败（"我已经懂了"）。

### 7.7 JSONL 作为不透明日志

测试把 stream-json 当作**不透明文件**，只 grep 关键字段（`"name":"Skill"`、`"skill":"X"`）。这种"协议最小依赖"测试对 JSON schema 变化不敏感——只要 Skill 工具调用还包含 `"name":"Skill"` 字段，测试就稳定。

### 7.8 时间戳目录作为"测试历史"

`/tmp/superpowers-tests/${TIMESTAMP}/` 不清理——所有历史运行都保留。这在调试时**极有价值**——可以对比"今天失败的 JSONL"和"昨天成功的 JSONL"，看到底是 prompt 改坏了还是模型波动。

---

## 八、关联文档

### 8.1 关联生产代码

- `skills/using-superpowers/SKILL.md` — 触发技能加载的入口
- `skills/subagent-driven-development/SKILL.md` — 主要测试目标
- `skills/systematic-debugging/SKILL.md` — 次要测试目标
- `skills/brainstorming/SKILL.md` — brainstorming 测试目标

### 8.2 关联其他 tests/

- `tests/skill-triggering/` — 姊妹套件（朴素 prompt 触发）
- `tests/claude-code/test-subagent-driven-development-integration.sh` — SDD 端到端
- `tests/claude-code/test-subagent-driven-development.sh` — SDD 单元

### 8.3 项目元文件

- `CLAUDE.md` / `AGENTS.md` — 项目级 AI 指令
- `.claude-plugin/plugin.json` — 插件 manifest
- `package.json` — 顶层 npm 包配置

### 8.4 缺失文档

- 本目录**没有 README.md**。`run-all.sh` 注释只说"Run all explicit skill request tests"，但没说明每个脚本的覆盖矩阵。
- 没有 `prompts/` 的索引文档（哪些 prompt 测什么）。

---

## 九、状态评价

### 9.1 健康度

| 维度 | 评分 | 备注 |
|------|------|------|
| 覆盖真实 bug | ⭐⭐⭐⭐⭐ | 直接复现已知失败 |
| 多 turn 深度 | ⭐⭐⭐⭐⭐ | 5 turn 构造 |
| 抗干扰 prompt 矩阵 | ⭐⭐⭐⭐⭐ | 9 个 prompt |
| 跨模型验证 | ⭐⭐⭐⭐ | haiku 子测试 |
| 失败诊断可读性 | ⭐⭐⭐⭐⭐ | 3 维度输出 |
| CI 友好 | ⭐⭐ | 完全依赖外部服务+token 消耗 |
| 非确定性容忍 | ⭐⭐ | LLM 采样可能 flake |
| 文档化 | ⭐⭐ | 缺 README；提示词目录未索引 |

### 9.2 适用场景

- **CI nightly** 跑完整套（不阻塞 PR）
- **PR check** 只跑 `run-all.sh`（4 个核心场景）
- **模型升级时**对比 Opus / haiku 子测试结果
- **技能重写时**确认不破坏"先加载再行动"契约

### 9.3 不适用场景

- 单元测试（这种测试没有"单元"可言）
- 离线 / 沙箱环境（必须联网 + API）
- 严格确定性要求（不能保证每次 100% 复现）

---

## 十、给读者的启示

### 10.1 LLM 行为测试的"协议契约"哲学

不要试图测试"LLM 输出是否正确"——那是 open-ended 问题。测试**"LLM 是否遵守协议"**（先调用工具 X，再做 Y）——这是有 ground truth 的。

### 10.2 多 turn 构造是真实场景的最低门槛

单 turn 测试对 LLM 行为验证**几乎无价值**。LLM 的失败模式几乎全部出现在"长上下文+明确指令"的多 turn 场景中。

### 10.3 "Premature Action" 是一种通用失败模式

```bash
# 在关键工具 X 调用之前调用了其他工具？
PREMATURE_TOOLS=$(head -n "$FIRST_X_LINE" "$LOG_FILE" | grep '"type":"tool_use"' | grep -v '"name":"X"')
```

这个模式可以推广到所有"必须按顺序调用"的工作流——例如"先 Read 再 Edit"、"先 Skill 再 Bash"。

### 10.4 抗干扰 prompt 矩阵

对每个用户指令设计**正交干扰**——"我已经懂了"、"我自己做过"、"我要求跳过"——这种 adversarial 设计是真实用户行为的模拟。

### 10.5 Haiku 子测试作为便宜回归

对 LLM 行为的关键不变量，至少在便宜模型上做完整覆盖；这能在不破产的前提下提升 CI 频率。

### 10.6 `set -e` + `|| true` 是测试设计的妥协

承认 LLM 命令会"自然失败"，用 `|| true` 吞掉退出码；让**断言**成为唯一的 pass/fail 权威。

### 10.7 时间戳目录的"测试考古"价值

不清空 `/tmp/superpowers-tests/` 是有意为之——历史 run 是宝贵的诊断资料。

---

## 十一、后续工作

### 11.1 立即可做

- [ ] 添加 README.md 说明 4 个核心脚本 + 3 个深度脚本的覆盖矩阵
- [ ] 把 dummy plan 模板抽到 `prompts/_fixtures/auth-system.md` 共享
- [ ] 添加 prompt 文件存在性前置检查
- [ ] 把 `OUTPUT_DIR` 改为环境变量可覆盖（默认 `/tmp/superpowers-tests/`）

### 11.2 中期可做

- [ ] 在 `run-all.sh` 中加入 `--extended`、`--haiku`、`--claude-describes` 三个 flag
- [ ] 把 SKILL_NAME 改用 `grep -F`（fixed string）而非正则，避免特殊字符问题
- [ ] 添加 `--model` 参数支持跨模型 CI matrix
- [ ] 引入 retry 机制：失败时重跑 1 次以容忍采样抖动

### 11.3 长期演进

- [ ] 写"统计置信度"：每个场景跑 N 次，N>=8/9 视为 pass
- [ ] 引入 JSONL 解析库（jq）做更细粒度的断言
- [ ] 跟踪 token 成本仪表板
- [ ] 探索"ground truth LLM"作为第二意见的对比测试

### 11.4 跨项目借鉴

`explicit-skill-requests/` 的以下模式值得其他 LLM 集成项目借鉴：

1. **多 turn 构造**：`--continue` 链式构建真实会话
2. **抗干扰 prompt 矩阵**：adversarial user behavior 模拟
3. **协议契约断言**：测试"是否调用了工具 X"而非"输出是否正确"
4. **"Premature Action" 检测**：调用顺序的隐式契约
5. **便宜模型压力测试**：haiku 子测试作为成本控制
6. **时间戳目录历史保留**：可对比、可复现的测试考古

---

**报告完成时间**：2026-06-03
**分析对象**：`tests/explicit-skill-requests/` 全部 6 个 shell + 9 个 prompt
**分析深度**：多文件全深度，6 个脚本逐个剖析
**对应源目录**：[`tests/explicit-skill-requests/`](file://e:\WorkStation\MyProject\superpowers\tests\explicit-skill-requests)
**姊妹套件**：[`tests/skill-triggering/`](file://e:\WorkStation\MyProject\superpowers\tests\skill-triggering) — 朴素 prompt 触发
