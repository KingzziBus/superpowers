# tests/skill-triggering/ 目录分析报告

> 对应源目录：`tests/skill-triggering/`
> 包含 2 个 shell 脚本 + 6 个 prompt 文件
> 报告定位：多文件分析，聚焦"朴素自然语言 prompt 是否能让 Claude 自动触发技能"

---

## 一、基本信息

| 字段 | 值 |
|------|-----|
| 目录路径 | `tests/skill-triggering/` |
| 文件总数 | 8 个（2 个 `.sh` 脚本 + 6 个 `.txt` 提示词） |
| 总代码行数 | 约 150 行（脚本）+ 约 60 行（提示词） |
| 测试方式 | **真实 Claude CLI 集成测试**（与 `explicit-skill-requests/` 同源） |
| 测试目标 | 验证"用户**不**点名技能、只用朴素 prompt"时 Claude 能否自动触发对应技能 |
| 调用入口 | `run-all.sh`（运行 6 个技能 × 6 个 prompt） |
| 是否需要联网 | **是**（依赖 `claude` CLI + API） |

`skill-triggering/` 是 `explicit-skill-requests/` 的**姊妹套件**——前者测"用户明确点名"，后者测"用户**不**点名、纯自然语言"。两者合并构成"LLM 技能触发"的全谱系测试。

---

## 二、目的与背景

### 2.1 测试目标

这是一个"自动触发正确性"测试套件，核心问题是：

> **当用户用朴素的自然语言描述问题/任务时，Claude 能否根据 SKILL.md 的 description 自动识别并触发对应技能？**

例如，用户说："测试失败了这个错误，能帮我看看吗？"——Claude 应当**自动**调用 `systematic-debugging` 技能，而不是直接去读代码尝试修。

这种"自动触发"是 superpowers 插件的核心价值主张——如果 LLM 不能自动识别，技能就**形同虚设**。

### 2.2 与 `explicit-skill-requests/` 的对比

| 维度 | `explicit-skill-requests/` | `skill-triggering/` |
|------|---------------------------|---------------------|
| Prompt 形式 | 用户**点名**（"use systematic-debugging"） | 用户**描述问题**（"测试失败"） |
| 测试难度 | 简单（明确关键词） | **困难**（需要 LLM 推理） |
| 失败影响 | 用户没得到技能 | 用户没得到技能 + 技能系统形同虚设 |
| 测试数量 | 9 个 prompt | 6 个 prompt |
| 套件深度 | 4 + 3 脚本 | 1 + 1 脚本 |

### 2.3 出现背景

这是 `docs/testing.md` 中描述的"skill triggering tests"机制——验证 SKILL.md 文档的 `description` 字段是否**有效**（能否让 LLM 准确选择技能）。

`description` 字段是 SKILL.md 的"广告语"——如果写得不好，LLM 不会触发；如果写得太宽，会误触发。这个测试套件就是 description 调优的"基准线"。

---

## 三、核心技术决策

### 3.1 与 explicit-skill-requests/ 的代码复用

```bash
# run-test.sh (skill-triggering)
SKILL_NAME="$1"
PROMPT_FILE="$2"
MAX_TURNS="${3:-3}"

# 与 explicit-skill-requests/run-test.sh 几乎完全相同
```

**决策点**：`run-test.sh` 与 `explicit-skill-requests/run-test.sh` 结构高度相似（断言逻辑相同），仅 prompt 风格不同。

**理由**：
- 测试协议（"看是否调用 Skill 工具"）**通用**——与 prompt 风格无关
- 两套件可以共享一个核心 test 框架（但目前没有抽取）

### 3.2 单文件 orchestration 模式

```bash
# run-all.sh
SKILLS=(
    "systematic-debugging"
    "test-driven-development"
    "writing-plans"
    "dispatching-parallel-agents"
    "executing-plans"
    "requesting-code-review"
)
for skill in "${SKILLS[@]}"; do
    prompt_file="$PROMPTS_DIR/${skill}.txt"
    if "$SCRIPT_DIR/run-test.sh" "$skill" "$prompt_file" 3 2>&1 | tee /tmp/skill-test-$skill.log; then
        PASSED=$((PASSED + 1))
        RESULTS+=("✅ $skill")
    else
        FAILED=$((FAILED + 1))
        RESULTS+=("❌ $skill")
    fi
done
```

**决策点**：单文件 `run-all.sh` 用 `for` 循环串行跑 6 个技能。

**理由**：
- 6 个测试数量适中，不需要更复杂的调度
- 串行跑避免 LLM 配额突发
- `tee /tmp/skill-test-$skill.log` 把每技能日志留到 `/tmp`

### 3.3 `2>&1 | tee` 的双输出策略

```bash
"$SCRIPT_DIR/run-test.sh" "$skill" "$prompt_file" 3 2>&1 | tee /tmp/skill-test-$skill.log
```

**决策点**：管道 `tee` 同时输出到 stdout 和临时文件。

**理由**：
- 让 CI 日志实时可见（不需要等测试跑完）
- 同时把每技能的输出存档到 `/tmp/skill-test-$skill.log`
- 调试时**直接看** `/tmp/skill-test-skill-triggering.log` 即可

### 3.4 `tee` 后 `if` 的退出码判断

```bash
if "$SCRIPT_DIR/run-test.sh" ... 2>&1 | tee /tmp/skill-test-$skill.log; then
    PASSED=$((PASSED + 1))
fi
```

**决策点**：`if` 接收的是**管道**的退出码——而 `pipefail` 默认**未启用**。

**理由**：
- `tee` 几乎永远成功——`$?` 等于 `tee` 的退出码
- `run-test.sh` 的退出码（`exit 0`/`exit 1`）可能被 `tee` 吞掉
- 这是一个**潜在 bug**：如果 `run-test.sh` 失败，`PASSED` 仍会增加
- 修复需要 `set -o pipefail` 或 `${PIPESTATUS[0]}`

### 3.5 朴素 prompt 的"故障式"构造

```txt
# prompts/systematic-debugging.txt
The tests are failing with this error:

```
FAIL src/utils/parser.test.ts
  ● Parser › should handle nested objects
    TypeError: Cannot read property 'value' of undefined
      at parse (src/utils/parser.ts:42:18)
      at Object.<anonymous> (src/utils/parser.test.ts:28:20)
```

Can you figure out what's going wrong and fix it?
```

**决策点**：prompt 给出**真实错误信息**（不点名 systematic-debugging）。

**理由**：
- 测试 SKILL.md description 的"广告效果"——"systematic debugging" 应该让 Claude 看到 "tests are failing" 就触发
- 错误信息具体（TypeError + 行号）让测试更真实
- `Can you figure out what's going wrong and fix it?` 鼓励 LLM 走"调试"流程

### 3.6 6 个 prompt 的覆盖矩阵

| Prompt | 期望触发的技能 | 难度 |
|--------|---------------|------|
| `systematic-debugging.txt` | systematic-debugging | 中（关键词明显） |
| `test-driven-development.txt` | test-driven-development | 难（"implement" 可能让 Claude 直接写代码） |
| `writing-plans.txt` | writing-plans | 中（"plan" 是关键词） |
| `dispatching-parallel-agents.txt` | dispatching-parallel-agents | **极难**（无关键词） |
| `executing-plans.txt` | executing-plans | 中（"execute plan" 是关键词） |
| `requesting-code-review.txt` | requesting-code-review | 中（"review" 是关键词） |

`dispatching-parallel-agents` 是最难的一个——prompt 必须**强烈暗示**任务有多个独立部分（否则 Claude 不会想到并行）。

### 3.7 `RESULTS=()` 数组累积

```bash
RESULTS=()
for skill in "${SKILLS[@]}"; do
    ...
    RESULTS+=("✅ $skill")  # or ❌
done

# 末尾
for result in "${RESULTS[@]}"; do
    echo "  $result"
done
```

**决策点**：用 bash 数组累积结果（而非字符串）。

**理由**：
- 数组保留"顺序"——失败/成功的列表是有意义的时序
- 字符串累积（如 `RESULTS="$RESULTS\nPASS"）会有转义问题
- 末尾用 `for` 循环展开数组，输出与 run-test 顺序一致

### 3.8 emoji 状态的视觉清晰度

```bash
RESULTS+=("✅ $skill")
RESULTS+=("❌ $skill")
```

**决策点**：用 emoji 标记状态。

**理由**：
- 一眼区分 pass/fail（视觉扫描快）
- 与 `explicit-skill-requests/run-test.sh` 的纯文本 "PASS"/"FAIL" 不一致——后者用纯文本
- 风格不统一，但各自有理由（run-all 给摘要、run-test 给诊断）

---

## 四、文件结构剖析

### 4.1 2 个脚本的最小化结构

| 文件 | 行数 | 角色 |
|------|------|------|
| `run-test.sh` | 89 | 单技能测试（接受 skill + prompt 参数） |
| `run-all.sh` | 61 | 6 技能 orchestrator |

总计 150 行——是 `tests/` 下**最小**的子目录。

### 4.2 `run-test.sh` 流程

```bash
# 1. 参数解析
SKILL_NAME="$1"
PROMPT_FILE="$2"
MAX_TURNS="${3:-3}"

# 2. 路径定位
SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
PLUGIN_DIR="$(cd "$SCRIPT_DIR/../.." && pwd)"

# 3. 输出目录
TIMESTAMP=$(date +%s)
OUTPUT_DIR="/tmp/superpowers-tests/${TIMESTAMP}/skill-triggering/${SKILL_NAME}"
mkdir -p "$OUTPUT_DIR"
cp "$PROMPT_FILE" "$OUTPUT_DIR/prompt.txt"

# 4. claude 调用
cd "$OUTPUT_DIR"
timeout 300 claude -p "$PROMPT" \
    --plugin-dir "$PLUGIN_DIR" \
    --dangerously-skip-permissions \
    --max-turns "$MAX_TURNS" \
    --output-format stream-json \
    > "$LOG_FILE" 2>&1 || true

# 5. 断言
SKILL_PATTERN='"skill":"([^"]*:)?'"${SKILL_NAME}"'"'
if grep -q '"name":"Skill"' "$LOG_FILE" && grep -qE "$SKILL_PATTERN" "$LOG_FILE"; then
    TRIGGERED=true
else
    TRIGGERED=false
fi

# 6. 退出
[ "$TRIGGERED" = "true" ] && exit 0 || exit 1
```

与 `explicit-skill-requests/run-test.sh` **几乎完全相同**——主要差异是少了"premature action"检测（因为朴素 prompt 没有"用户期待立即行动"的前提）。

### 4.3 `run-all.sh` 流程

```bash
SKILLS=("systematic-debugging" "test-driven-development" "writing-plans" ...)

for skill in "${SKILLS[@]}"; do
    prompt_file="$PROMPTS_DIR/${skill}.txt"
    if [ ! -f "$prompt_file" ]; then
        echo "⚠️  SKIP: No prompt file for $skill"
        continue
    fi
    if "$SCRIPT_DIR/run-test.sh" "$skill" "$prompt_file" 3 2>&1 | tee /tmp/skill-test-$skill.log; then
        PASSED=$((PASSED + 1))
        RESULTS+=("✅ $skill")
    else
        FAILED=$((FAILED + 1))
        RESULTS+=("❌ $skill")
    fi
done
```

关键特点：
- `prompt_file` 不存在时**SKIP**而非 FAIL（保护新技能）
- 6 个技能串行跑（无并发）
- `tee` 双输出

### 4.4 prompts/ 的 6 个朴素 prompt

**systematic-debugging.txt**（11 行）：
```
The tests are failing with this error:
[真实错误]
Can you figure out what's going wrong and fix it?
```
→ 期望触发 systematic-debugging

**test-driven-development.txt**（7 行）：
```
I need to add a new feature to validate email addresses...
Can you implement this?
```
→ 期望触发 TDD（但可能让 Claude 直接写实现）

**writing-plans.txt** / **dispatching-parallel-agents.txt** / **executing-plans.txt** / **requesting-code-review.txt**：

每个都是**真实场景的朴素描述**——通过"特征"（如 "implement" → TDD、"review" → requesting-code-review）让 LLM 推理。

---

## 五、优点亮点

### 5.1 极简测试套件

仅 150 行 shell 代码 + 6 个 prompt，覆盖 6 个核心技能——**测试密度高**。

### 5.2 与 explicit-skill-requests 互补

两套件合并构成"技能触发"全谱系：
- `explicit-skill-requests/`：用户**明确**点名（基础功能）
- `skill-triggering/`：用户**模糊**描述（核心价值）

### 5.3 朴素 prompt 的"实战性"

每个 prompt 模拟**真实用户**的"懒得说细节"风格——不点技能名、不讲技术细节，只描述问题。这与生产环境高度一致。

### 5.4 临时日志文件留存

`tee /tmp/skill-test-$skill.log` 让每技能日志**独立存档**——CI 失败时直接 `cat /tmp/skill-test-systematic-debugging.log` 即可看到 claude 的完整输出。

### 5.5 缺失 prompt 的优雅降级

```bash
if [ ! -f "$prompt_file" ]; then
    echo "⚠️  SKIP: No prompt file for $skill"
    continue
fi
```

新技能即使没有 prompt 文件也不会阻断其他技能测试——**优雅降级**。

### 5.6 emoji 视觉扫描

`✅` / `❌` 让 CI 摘要**一眼可读**。在 monospace 日志中尤其有效。

### 5.7 SKILLS 数组的"配置即代码"

```bash
SKILLS=(
    "systematic-debugging"
    "test-driven-development"
    ...
)
```

技能列表**集中**在文件顶部——添加新技能只需 1 行。配合 prompts 目录的 .txt 文件，新技能可"零代码"接入。

### 5.8 max-turns 默认 3

`MAX_TURNS="${3:-3}"` 默认 3 turn——给 LLM 足够空间"读 SKILL.md → 加载技能 → 行动"。

### 5.9 时间戳目录隔离

与 `explicit-skill-requests/` 一致——`/tmp/superpowers-tests/${TIMESTAMP}/skill-triggering/${SKILL_NAME}/` 让每次运行独立。

---

## 六、缺点风险局限

### 6.1 pipefail 未启用导致退出码误判

```bash
if "$SCRIPT_DIR/run-test.sh" ... 2>&1 | tee /tmp/skill-test-$skill.log; then
    PASSED=$((PASSED + 1))
```

**问题**：`if` 判断的是 `tee` 的退出码（永远 0），而非 `run-test.sh` 的退出码。

**修复**：
```bash
if "${PIPESTATUS[0]}" = "0" 2>&1 | tee ...
```
或
```bash
set -o pipefail
```

### 6.2 与 explicit-skill-requests/run-test.sh 高度重复

两个 `run-test.sh` 95% 相同——应当**抽取**为共享库（如 `tests/_lib/run-test.sh`）。

### 6.3 LLM 采样非确定性

朴素 prompt 触发本质是"LLM 推理"——temperature > 0 让结果**波动**。同一 prompt 跑 10 次可能 8/10 通过、2/10 失败。

### 6.4 缺少 parallel 跑选项

6 个串行跑 ≈ 6 × 5min = 30 分钟。CI nightly 可接受，但**本地调试**时希望"只跑 1 个技能"。

### 6.5 SKILLS 数组与 prompts/ 内容易 drift

如果有人在 `prompts/` 加了新 .txt 但**没**在 SKILLS 数组添加对应项——测试不会跑该 .txt。**反向**情况同理。

### 6.6 "FAIL" 输出未提供修复线索

```bash
echo "❌ FAIL: Skill '$SKILL_NAME' was NOT triggered"
TRIGGERED=false
```

只说"未触发"——不说"为什么"。需要人类**自己**看 `/tmp/skill-test-$skill.log` 找原因。

### 6.7 时间戳目录无清理

`/tmp/superpowers-tests/${TIMESTAMP}/` 永不清空——历史 run 累积。最终 `/tmp` 可能撑满（虽然 `/tmp` 在多数系统是 tmpfs 自动清理）。

### 6.8 `RESULT += "❌"` 的字符渲染依赖

emoji 在某些终端（特别是 ssh without UTF-8）会显示为乱码。CI 终端一般 OK，但开发者本地可能有差异。

### 6.9 没有 `--suite` flag

`run-all.sh` **必须**跑全部 6 个测试。如果只改了一个技能的 SKILL.md 也得跑全部——CI 成本浪费。

### 6.10 emoji 与纯文本混用

`run-all.sh` 用 emoji，run-test.sh 用纯文本。**风格不统一**。

### 6.11 LLM 限流脆弱

连续 6 次 `claude -p` 调用可能触发**API 限流**。没有 retry 机制。

### 6.12 缺 pytest/junit XML 输出

CI 系统的"测试报告"通常需要 junit XML——bash 测试没有这种结构化输出。

---

## 七、关键洞察

### 7.1 skill-triggering 是 superpowers 的"价值证明"

如果用户**每次都要**说"please use systematic-debugging"，技能就形同虚设。`skill-triggering/` 验证的是**自动识别**能力——这是 superpowers 真正的价值。

### 7.2 SKILL.md 的 description 字段是"广告位"

测试间接验证每个技能的 description 写得**好不好**——description 是 LLM 决策的唯一依据。写得差的 description 会让 LLM 选错技能。

### 7.3 "dispatching-parallel-agents" 是最难的 prompt

该 prompt 必须**强烈暗示**任务有多个独立部分——但用户很少这样描述。这是**当前 LLM 能力的边界**。

### 7.4 pipefail 的微妙陷阱

```bash
if cmd1 | tee log; then
```

`if` 接收 `tee` 的退出码——这是 bash 经典陷阱。`set -o pipefail` 是必备防线。

### 7.5 "重写而非复用"的工程债务

两个 `run-test.sh` 高度重复——目前是**技术债**。长期应当抽取到 `_lib/`。

### 7.6 朴素 prompt 的"双重作用"

朴素 prompt 既测"LLM 自动识别能力"也测"SKILL.md 描述质量"——这是**双重测试**。改进 SKILL.md 的 description 应当通过这种测试验证。

### 7.7 缺省是 3 turn 的合理性

朴素 prompt 期望 LLM 走"读 SKILL → 加载技能 → 行动"路径——3 turn 足够。
> 但对于"加载多步工具"可能不够。3 turn 是**保守选择**。

### 7.8 `tee` 双输出的时代意义

现代 CI 强调"实时日志"——`tee` 是**简单但有效**的实时流方案。

### 7.9 bash 测试的 100 行哲学

150 行覆盖 6 个技能的 LLM 行为——bash 的"小而精"哲学在此体现。

### 7.10 缺失 prompt 的 SKIP 是"前瞻性"设计

将来添加新技能时，`SKILLS` 数组添加条目但**忘**加 .txt 文件——SKIP 让测试**仍可运行**。

---

## 八、关联文档

### 8.1 关联生产代码

- `skills/systematic-debugging/SKILL.md` — 主要测试目标之一
- `skills/test-driven-development/SKILL.md` — TDD 测试目标
- `skills/writing-plans/SKILL.md` — writing-plans 测试目标
- `skills/dispatching-parallel-agents/SKILL.md` — 最难测试目标
- `skills/executing-plans/SKILL.md` — executing-plans 测试目标
- `skills/requesting-code-review/SKILL.md` — requesting-code-review 测试目标
- 所有 6 个技能的 `description:` 字段——LLM 决策的依据

### 8.2 关联其他 tests/

- `tests/explicit-skill-requests/` — 姊妹套件（用户点名）
- `tests/claude-code/` — Claude Code 平台测试
- `docs/testing.md` — 顶层测试文档

### 8.3 项目元文件

- `CLAUDE.md` / `AGENTS.md` — 项目级 AI 指令
- `docs/testing.md` — 测试策略文档

### 8.4 缺失文档

- 本目录**没有 README.md**
- 没有 prompt 编写指南（"如何写一个能触发 X 技能的 prompt"）
- 没有 SKILL.md description 调优指南

---

## 九、状态评价

### 9.1 健康度

| 维度 | 评分 | 备注 |
|------|------|------|
| 覆盖核心技能 | ⭐⭐⭐⭐⭐ | 6 个核心技能全覆盖 |
| 朴素 prompt 实战性 | ⭐⭐⭐⭐⭐ | 模拟真实用户 |
| 代码复用 | ⭐⭐ | 与 explicit-skill-requests 重复 |
| pipefail 安全 | ⭐⭐ | 退出码误判陷阱 |
| 失败诊断 | ⭐⭐ | 需手动查 /tmp 日志 |
| CI 友好 | ⭐⭐⭐ | 结构化输出缺失 |
| 文档化 | ⭐⭐ | 缺 README |
| 进度反馈 | ⭐⭐⭐⭐ | emoji 视觉清晰 |

### 9.2 适用场景

- **CI nightly** 跑全套 6 技能
- **技能 SKILL.md 改 description 后** 验证是否仍触发
- **模型升级时**看是否仍能自动识别
- **新人 onboarding 时**作为 SKILL.md 写作参考

### 9.3 不适用场景

- 离线/沙箱（必须联网）
- 严格确定性（LLM 采样会波动）
- 性能基准（无计时）

---

## 十、给读者的启示

### 10.1 skill-triggering 测试是 SKILL.md 的"质量雷达"

每个技能的 description 写得好不好——这个测试**直接给答案**。description 调优的反馈循环应基于此测试。

### 10.2 pipefail 是 bash 测试的必备武器

```bash
set -o pipefail
if cmd1 | tee log; then
```

不启用 pipefail 等于**主动放弃**对 cmd1 失败的检测。

### 10.3 bash 测试的极简主义

150 行测 6 个技能——bash 适合"小而精"测试，不适合"大而全"。

### 10.4 emoji 状态码的视觉优势

`✅` / `❌` 比 `[PASS]` / `[FAIL]` 视觉扫描快得多——CI 日志中尤其重要。

### 10.5 "配置即代码"的可维护性

`SKILLS=("..." "...")` 数组让"添加新技能"零代码——降低 contributor 门槛。

### 10.6 双套件设计模式

"明确触发" + "自动触发"两套件合并构成"完整测试矩阵"——其他 LLM 集成可参考。

### 10.7 SKILL.md 的 description 是产品决策

不只是"写技术"——它直接决定"用户不点名时 LLM 会不会选你的技能"。是**产品定位**而非技术细节。

### 10.8 朴素 prompt 的"双重作用"

朴素 prompt 同时验证"LLM 能力"和"SKILL.md 质量"——这是**测试效率**的体现。

---

## 十一、后续工作

### 11.1 立即可做

- [ ] 在 `run-all.sh` 顶部添加 `set -o pipefail` 修复退出码误判
- [ ] 添加 README.md 说明 6 个 prompt 的覆盖矩阵
- [ ] 把 `run-test.sh` 抽取到 `tests/_lib/run-test.sh`，与 explicit-skill-requests 共享
- [ ] 添加 `--skill <name>` flag 让只跑单技能

### 11.2 中期可做

- [ ] 添加 parallel 跑（`xargs -P 2`）减少总时长
- [ ] 添加 retry 机制容忍 LLM 限流
- [ ] 输出 junit XML 让 CI 系统结构化解析
- [ ] 把 `tee` 输出加上时间戳

### 11.3 长期演进

- [ ] 抽取共享 bash test 框架（含 set -o pipefail + cleanup + assertion library）
- [ ] 引入 OpenAI/Claude SDK 替代 `claude -p` 黑盒调用
- [ ] 写"prompt 编写指南"——如何写一个**会触发** X 技能的 prompt
- [ ] SKILL.md description 调优文档

### 11.4 跨项目借鉴

`skill-triggering/` 的以下模式值得其他 LLM 集成项目借鉴：

1. **朴素 prompt 自动触发测试**：直接验证 description 质量
2. **pipefail 必备**：bash 测试的正确性基础
3. **双套件设计**：明确触发 + 自动触发互补
4. **emoji 状态码**：CI 日志视觉扫描优化
5. **SKILLS 数组配置即代码**：低门槛添加新技能
6. **SKIP 优雅降级**：新技能不会破坏老测试

---

**报告完成时间**：2026-06-03
**分析对象**：`tests/skill-triggering/` 全部 2 个 shell + 6 个 prompt
**分析深度**：多文件全深度，核心 run-test.sh / run-all.sh 详尽剖析
**对应源目录**：[`tests/skill-triggering/`](file://e:\WorkStation\MyProject\superpowers\tests\skill-triggering)
**姊妹套件**：[`tests/explicit-skill-requests/`](file://e:\WorkStation\MyProject\superpowers\tests\explicit-skill-requests) — 用户点名
**关联文档**：[`docs/testing.md`](file://e:\WorkStation\MyProject\superpowers\docs\testing.md)
