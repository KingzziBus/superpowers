# testing.md 分析报告

## 基本信息

| 项目 | 内容 |
| --- | --- |
| 文档路径 | `docs/testing.md` |
| 字数规模 | 约 304 行 / 7.5KB |
| 文档类型 | 测试方法论 / 工程实践指南 |
| 主题 | 如何测试 Superpowers 技能（特别是涉及子代理的复杂技能） |
| 核心工具 | `tests/claude-code/` 下的 bash 集成测试 + Python token 分析 |
| 面向读者 | 编写新技能 / 修改现有技能的 Superpowers 维护者 |

---

## 一、文档目的

解决 Superpowers 项目中一个独特而根本的挑战：**如何测试"指导 AI 行为"的技能（skills）？**

传统软件测试假设被测对象是确定性的代码。但 Superpowers 的技能是**指导 Claude Code 行为的 Markdown 文档**——它们的"输出"是 AI 行为，而非具体的程序输出。

具体要回答 4 个问题：

1. **测什么**：复杂技能（涉及子代理、工作流、多轮交互）该怎么定义"通过/失败"
2. **怎么测**：用 headless 模式跑真实 Claude Code 会话，并通过会话记录验证行为
3. **怎么看成本**：每个子代理消耗多少 token / 多少钱
4. **怎么写新测试**：可复用的模板和最佳实践

定位上属于 **"AI 时代软件测试"** 的早期工程实践——在 LLM 编程工具爆发期，**如何测试"AI 行为"本身**还没有公认方法论，Superpowers 用**集成测试 + 行为验证 + 成本分析**的组合提供了一份参考答案。

---

## 二、核心技术决策

### 决策 1：用集成测试，而非单元测试

**为什么不用单元测试**：
- 技能的核心价值是"引导 AI 在真实场景下做正确的事"
- 把它拆成"调用 Skill 工具 → 检查返回"无法验证真实效果
- 单元测试会忽略多轮交互、子代理派发、TodoWrite 跟踪等关键行为

**为什么用集成测试**：
- 在 headless 模式下让 Claude Code 真的运行技能
- 创建临时项目，让技能"完成一个真实任务"
- 然后**检查这个任务是否被正确完成**

**代价**：
- 慢（10-30 分钟）
- 贵（$4-5 一次）
- 不稳定（LLM 行为有随机性）

**收益**：
- 高保真——能捕捉到"技能文本看起来 OK 但 Claude 没理解"的情况

### 决策 2：用"会话记录（session transcript）"作为行为证据

**为什么不 grep 用户可见输出**：
- 用户可见输出是"AI 决定呈现什么"，可能被简化、总结、跳过
- 真实行为（包括错误、retry、TodoWrite 调用）在 UI 输出中看不到

**会话记录的优势**：
- **完整**：每条消息、每个工具调用、每个结果都在
- **结构化**：JSONL 格式，易于解析
- **权威**：是 Claude Code 内部状态的真实反映

### 决策 3：基于"行为指标"而非"输出文本"做断言

测试 1-8 的 8 个验证点都是**行为指标**而非文本匹配：

| 测试 | 验证的行为 | 验证方式 |
| --- | --- | --- |
| 1 | Skill 工具被调用 | grep `"name":"Skill".*"skill":"..."` |
| 2 | 子代理被分派 | grep `Task` 工具调用 |
| 3 | TodoWrite 被使用 | 计数 TodoWrite 调用次数 |
| 6 | 实施文件被创建 | 检查 `src/math.js` 存在 |
| 6 | 测试通过 | 运行 `npm test` |
| 7 | Git 提交记录 | `git log` 检查 |
| 8 | 没有额外特性 | 对比预期文件列表 |

**意义**：这些是"AI 行为正确"的**可观测代理变量**（observable proxies）。

### 决策 4：Token / 成本分析作为一等公民

**为什么重要**：
- 集成测试每跑一次花 $4-5
- 技能可能催生大量子代理调用，成本可能失控
- **没有成本意识的 AI 工具测试 = 没有经济可持续性**

**文档中的成本示例**：

```
main session:    $4.09 (持有完整上下文)
7 subagents:     $0.07-$0.09 each ($0.55 总计)
─────────────────────────────────
TOTAL:           $4.67
```

**意义**：把"AI 行为测试"从**纯定性**变成**定性 + 定量**。

### 决策 5：使用真实世界的小型项目作为测试夹具

测试创建了一个**最小化的 Node.js 项目**，让 Claude 实施一个 `add` + `multiply` 函数：

```
src/math.js
test/math.test.js
```

**为什么不是 mock**：
- Mock 会丢失"AI 在真实工程约束下能否工作"的信息
- 真实 npm install + npm test 才能验证"测试通过"
- 真实 git init + git commit 才能验证"提交记录"

### 决策 6：6 条最佳实践

| # | 实践 | 防止的问题 |
| --- | --- | --- |
| 1 | trap 清理临时目录 | 磁盘满 / 残留状态污染下次测试 |
| 2 | 解析 .jsonl 而非 grep 输出 | 误报 / 漏报 |
| 3 | bypassPermissions + --add-dir | 权限阻止导致测试假阴性 |
| 4 | 从插件目录运行 | 技能没加载 |
| 5 | 始终展示 token 分析 | 不知道成本变化 |
| 6 | 验证真实行为 | 假阳性（看似通过但不正确） |

---

## 三、文件结构

| 章节 | 内容 | 价值 |
| --- | --- | --- |
| Overview | 测试的核心挑战与思路 | 立场宣言 |
| Test Structure | tests/ 目录布局 | 让读者快速找到测试 |
| Running Tests | 集成测试的运行方法 | 实操入口 |
| Integration Test: subagent-driven-development | 具体测试样例的详细解释 | 范例驱动 |
| → What It Tests | 6 个验证点 | 行为契约 |
| → How It Works | 4 步流程 | 流程图 |
| → Test Output | 实际输出样例 | 期望管理 |
| Token Analysis Tool | 成本可视化 | 经济可持续性 |
| Troubleshooting | 4 类常见问题 | 救援指南 |
| Writing New Integration Tests | 模板 + 6 条最佳实践 | 知识传递 |
| Session Transcript Format | JSONL 结构说明 | 解析前置 |

**结构特点**：从"哲学（如何测试 AI 行为）"到"工程（怎么跑测试、怎么解析数据）"的完整闭环。

---

## 四、优点

### 1. 揭示了一个新兴领域的工程实践

**AI 行为测试目前没有标准方法论**。本文档提供的"集成测试 + 会话记录 + 行为指标"组合，是这个领域的早期 best practice。

**贡献**：
- 提出了"行为指标"作为测试契约
- 提供了"成本可视化"作为测试报告
- 探索了"LLM 行为 + 传统断言"的混合测试模式

### 2. 透明度极高

测试输出样例**不是美化过的**，而是真实运行结果：
- 包含 7 个子代理的成本（$0.07-$0.09 范围）
- 包含主会话的高 cache 读取（1.2M tokens）
- 包含 "Test 4" 和 "Test 5" 的缺失（编号跳跃说明原本有但被省略）

**意义**：让读者**真实预期**测试的输出、时长、成本。

### 3. 成本意识

文档不止一次强调"成本"：
- "集成测试可能需要 10-30 分钟"
- 整节专门讲 Token 分析
- 输出样例包含 `$4.67 估算成本`
- 最佳实践 #5："始终展示 token 分析"

**意义**：在 LLM 时代，**测试成本是工程决策的一部分**。

### 4. 模板即开即用

完整的 bash 模板可以直接复制：

```bash
SCRIPT_DIR="$(cd "$(dirname "$0")" && pwd)"
source "$SCRIPT_DIR/test-helpers.sh"
TEST_PROJECT=$(create_test_project)
trap "cleanup_test_project $TEST_PROJECT" EXIT
# ...
```

**意义**：降低"写新测试"的门槛。

### 5. 揭示了"会话记录"这个隐藏资产

很多 Claude Code 用户不知道 `~/.claude/projects/` 下存着自己的所有会话记录。文档把这个"内部 API"暴露出来：

```bash
SESSION_DIR="$HOME/.claude/projects/-Users-yourname-Documents-GitHub-superpowers-superpowers"
```

**意义**：让用户**复用 Claude 的内部数据**作为自己的"AI 行为分析平台"。

### 6. JSONL 格式的精准解析示例

```python
# 文档揭示的关键字段
{
  "type": "assistant",
  "message": {
    "usage": {
      "input_tokens": 27,
      "output_tokens": 3996,
      "cache_read_input_tokens": 1213703  ← 关键：缓存读取 = 省钱
    }
  }
}
```

**意义**：让读者理解 Claude Code 的"成本优化机制"——prompt caching 节省了 99% 的 input tokens。

---

## 五、缺点与风险

### 1. 测试不稳定（flakiness）风险

**根本问题**：LLM 行为有随机性。

**可能表现**：
- 同一份技能文档，两次跑测试可能 1 次通过 1 次失败
- 子代理可能"忘记"调用 TodoWrite
- 实施结果可能用了不同的命名（`math.js` vs `calculator.js`）

**文档如何应对**：
- 没明确说！这是个**重大隐患**
- 8 个测试断言有些是**硬编码**的（如检查 `add` 函数存在），但 LLM 可能输出 `addTwo`、`sum`、`plus`

**改进方向**：
- 用**模糊匹配**（regex 而不是 exact match）
- **多次跑取平均**通过率
- 对关键断言做**容错**（"至少调用了 5 次 TodoWrite"而不是"恰好 5 次"）

### 2. 成本高且需要付费 API

- 每次测试 $4-5
- 集成测试 10-30 分钟
- CI 跑全量测试 = 巨款

**缺失信息**：
- 文档没说"在 CI 上怎么控制成本"
- 没说"是否可以用 mock 模型"
- 没说"小改动要不要跑全套集成测试"

### 3. 测试覆盖范围不明确

**问题**：
- 文档说"测 8 件事"，但没说"这 8 件事是否覆盖了技能的所有重要行为"
- 比如 subagent-driven-development 技能的"review loop"机制，如果子代理不发起循环会怎样？测试会捕捉到吗？

**缺失**：
- 技能行为的"全部重要属性"清单
- 测试覆盖率指标

### 4. JSONL 格式可能不稳定

文档基于某个版本的 Claude Code 写的，但 Claude Code 升级后：
- 字段名可能改（`cache_read_input_tokens` → `cached_input_tokens`）
- 字段可能删
- 嵌套结构可能变

**风险**：测试和分析脚本可能在某次 Claude Code 升级后突然全挂。

### 5. 没有 mocking / 离线测试模式

**问题**：
- 所有测试都需要**真的能联网**的 Claude Code
- 不能在飞机上 / 离线环境开发测试
- 不能用 mock 行为快速验证"测试逻辑本身"是否正确

### 6. Python 分析脚本不在文档主目录

文档反复引用 `tests/claude-code/analyze-token-usage.py`，但**没在文档中给出该脚本的源码**。

读者要看完整实现需要：
- 切到 tests/claude-code 目录
- 读 Python 源码

**改进**：可以直接在文档中贴关键代码片段。

### 7. 缺少"对失败的处理"指导

测试输出样例只有 PASS，没展示：
- 失败时输出长什么样
- 怎么 debug
- 怎么区分"技能 bug" vs "测试 bug" vs "LLM 临时抽风"

### 8. "从插件目录运行"的强假设

```bash
cd "$SCRIPT_DIR/../.." && timeout 1800 claude -p "$PROMPT"
```

这意味着 `claude` 必须在 `PATH` 中且 cwd 是 superpowers 根目录。

**问题**：
- CI 容器里需要先 `cd` 到正确目录
- 不同的开发环境路径不同
- 没有跨平台的路径处理（Windows vs Unix）

---

## 六、关键洞察

### 洞察 1：这是"AI 时代软件测试"的早期范式

**传统软件测试**：
```
Unit Test → Integration Test → E2E Test
测代码、测模块、测系统
```

**AI 技能测试**：
```
Skill Document
   ↓ loaded by
Claude Code (LLM)
   ↓ runs in
Real Project
   ↓ produces
Session Transcript
   ↓ verified by
Assertions on transcript
```

**本质区别**：被测对象从"代码"变成"文档+LLM 的组合"。

### 洞察 2：会话记录是"AI 行为的数据库"

`~/.claude/projects/` 下的所有 `.jsonl` 文件，本质上是**AI 行为的完整数据库**。

**类比**：
- 传统软件 = 源代码 + 日志
- AI 系统 = 提示词 + 会话记录

**推论**：未来会出现"AI 行为分析平台"——专门解析 `.jsonl`、可视化行为、检测异常。

### 洞察 3：成本 = 行为复杂度

```
main session:           $4.09
7 subagents:            $0.55 total
─────────────────────────────────
main 占总成本 88%       ← 协调者成本主导
```

**意义**：复杂技能的成本**主要由协调者驱动**，不是子代理。这暗示：
- 优化"主会话上下文"对降本最有效
- 子代理成本相对可预测（每个 $0.07-$0.09）

### 洞察 4：缓存是 LLM 测试的"经济杠杆"

```
Total input (incl cache): 1,515,639
Cache creation tokens:      132,742
Cache read tokens:        1,382,835  ← 91% 是缓存读取
```

**意义**：Claude Code 的 prompt caching 节省了大量成本。但这也意味着：
- 第一次跑测试贵
- 后续跑测试便宜（如果上下文变化不大）
- 测试的"边际成本"快速下降

### 洞察 5：6 条最佳实践是"防 flakiness"清单

| # | 实践 | 防的是什么 |
| --- | --- | --- |
| 1 | trap 清理 | 状态污染 |
| 2 | 解析 .jsonl | 表面文本误判 |
| 3 | bypassPermissions | 权限阻塞 |
| 4 | 从插件目录运行 | 技能未加载 |
| 5 | token 分析 | 成本失控 |
| 6 | 验证真实行为 | 假阳性 |

**每一都对应一类 flakiness 来源**。

### 洞察 6：测试是"技能质量的反馈环"

```
Skill MD 改动
   ↓
跑集成测试
   ↓
观察行为 + 成本
   ↓
调整技能文本
   ↓
再跑
```

**意义**：技能开发从"凭感觉"变成"凭数据"。这是 **Data-Driven Skill Engineering** 的雏形。

---

## 七、关联文档

| 文档/文件 | 关系 |
| --- | --- |
| `tests/claude-code/test-helpers.sh` | 共享工具（`create_test_project`、`cleanup_test_project`） |
| `tests/claude-code/analyze-token-usage.py` | Token 分析脚本（文档主推的工具） |
| `tests/claude-code/test-subagent-driven-development-integration.sh` | 文档主推的范例测试 |
| `tests/claude-code/run-skill-tests.sh` | 测试运行器 |
| `skills/writing-skills/SKILL.md` | 编写新技能的指南（也涉及测试） |
| `skills/writing-skills/testing-skills-with-subagents.md` | 用子代理测试技能的具体方法 |
| `docs/plans/` 下的设计文档 | 测试策略的演化记录 |

---

## 八、状态评价

| 维度 | 评分 | 说明 |
| --- | --- | --- |
| **问题定义** | ⭐⭐⭐⭐⭐ (5/5) | 清晰识别"AI 行为测试"的独特挑战 |
| **方法论创新** | ⭐⭐⭐⭐ (4/5) | 集成测试+会话记录+成本分析是新颖组合 |
| **可操作性** | ⭐⭐⭐⭐ (4/5) | 模板可直接用，但成本和 flakiness 需自行应对 |
| **完整性** | ⭐⭐⭐ (3/5) | 缺失败处理、离线模式、CI 集成 |
| **稳定性建议** | ⭐⭐ (2/5) | 没充分讨论 LLM 测试的 flakiness 问题 |
| **经济意识** | ⭐⭐⭐⭐⭐ (5/5) | 显式讨论 token 成本 |
| **可演进性** | ⭐⭐⭐ (3/5) | 依赖 Claude Code 的 JSONL 格式，可能不稳定 |

**总体评价**：在"AI 行为测试"这个**尚未标准化的领域**，Superpowers 提供了**一份扎实的方法论文档**。它在工程意识（成本分析）和实战指导（可复制模板）上都做得好，但在 LLM 特有的 flakiness 问题上**坦白得不够**，这是后续需要补强的地方。

---

## 九、给读者的启示

### 如果你正在为 Superpowers 贡献新技能
- **必读**：Running Tests、What It Tests、Writing New Integration Tests
- **必做**：每次技能改动都跑一次集成测试
- **必看**：Token 分析——如果你的技能让成本翻倍，就是个问题

### 如果你正在测试其他 LLM 驱动的工具
- **借鉴**：
  - "集成测试 + 行为指标"模式
  - "会话记录作为行为证据"思路
  - "成本可视化作为测试报告"实践
- **警惕**：
  - LLM flakiness（同一测试可能 1/10 失败）
  - 测试成本（每次 $4-5）
  - JSONL 格式锁定（供应商可能改格式）

### 如果你在做"AI 测试工具"或"AI 行为分析平台"
- **市场机会**：目前没有标准工具
- **技术栈建议**：JSONL 解析 + 行为模式识别 + 成本归因
- **数据来源**：Claude Code / OpenCode / Codex / Cursor 各自的会话记录

### 如果你维护 Superpowers
- **改进建议 1**：增加"测试稳定性策略"章节（重试、容错、CI 集成）
- **改进建议 2**：把 `analyze-token-usage.py` 的关键代码贴到文档中
- **改进建议 3**：增加 mock 模式（不调 LLM，跑测试逻辑本身）
- **改进建议 4**：增加"测试失败时怎么 debug"指南

### 如果你刚开始接触 AI 编程工具
- **学到**：你的每次对话都被**完整记录**在 `~/.claude/projects/`
- **学到**：AI 行为可以通过**回放**来分析
- **学到**：prompt caching 是**降本关键**

---

## 十、后续工作

按重要性排序：

1. **高优先级**：补充 LLM flakiness 应对策略（重试、容错、多次跑取平均）
2. **高优先级**：把 `analyze-token-usage.py` 的核心代码片段贴到文档中
3. **中优先级**：补充"CI 集成模式"（GitHub Actions、预算控制、按改动范围跑子集）
4. **中优先级**：增加"失败调试"指南（失败输出长什么样、怎么定位是技能 bug 还是 LLM 抽风）
5. **中优先级**：探索"非 Claude Code"的测试能力（OpenCode、Codex 也能复用这个模式吗？）
6. **低优先级**：增加 mock 模式（不调 LLM 也能跑测试逻辑，验证断言本身）
7. **可选**：把"AI 行为测试"作为独立研究方向，发布一份 best practice 白皮书

---

**总结**：这份文档是 **"AI 时代软件测试方法论"** 的早期探索。它在缺乏标准答案的情况下，提供了**集成测试 + 会话记录 + 成本分析**的扎实组合，并积累了真实的测试样例与故障排查经验。其最大的未解决问题是 **LLM flakiness**——这是整个 LLM 测试领域的共同挑战，但文档坦白得不够，留下了显著的改进空间。**对任何严肃做 LLM 应用测试的人来说，这都是必读资料。**
