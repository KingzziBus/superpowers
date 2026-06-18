# writing-skills/testing-skills-with-subagents.md 深度分析报告

> 本报告分析 superpowers 项目的"使用子代理测试技能"文档。它是 writing-skills 体系中**测试方法论**的具体化。

---

## 1. 基本信息

| 项 | 值 |
|----|----|
| **文件名** | `testing-skills-with-subagents.md` |
| **总行数** | 385 行 |
| **文档定位** | 技能测试的具体方法论（writing-skills 的"操作手册"） |
| **核心命题** | "Testing skills is just TDD applied to process documentation" |
| **依赖技能** | `superpowers:test-driven-development`（必需背景） |
| **完整示例** | `examples/CLAUDE_MD_TESTING.md` |
| **核心创新** | 压力场景 + 合理化表 + 元测试 |

---

## 2. 目的与背景

### 2.1 文档的桥梁作用

本文档在 superpowers 知识体系中扮演**方法论桥梁**：

```
[抽象层面]
superpowers:test-driven-development
    ↓
[理论映射]
writing-skills/SKILL.md
    ↓
[具体操作]  ← 本文档
testing-skills-with-subagents.md
    ↓
[实际案例]
examples/CLAUDE_MD_TESTING.md
```

它把抽象的"TDD for skills"转化为**可操作的具体步骤**。

### 2.2 三大核心命题

1. **"If you didn't watch an agent fail without the skill, you don't know if the skill prevents the right failures"**（未观察失败 = 不知防什么）
2. **"Tests pass once ≠ bulletproof"**（一次通过 ≠ 防弹）
3. **"Continue REFACTOR cycle until no new rationalizations"**（继续 REFACTOR 直到无新合理化）

### 2.3 解决的三个核心问题

| 问题 | 本文档提供的方案 |
|------|------------------|
| 不知道测试什么 | 3+ 组合压力 + 7 种压力类型 |
| 不知道如何记录失败 | 逐字捕获合理化 + 4 种漏洞堵法 |
| 不知道何时停止 | 4 个防弹标志 + 元测试 |

---

## 3. 核心技术决策

### 3.1 决策 1：场景必须"看起来真实"

```markdown
IMPORTANT: This is a real scenario. You must choose and act.
Don't ask hypothetical questions - make the actual decision.
```

**关键要素：**
- "This is real" 心理暗示
- "You must choose" 强制选择
- "Make the actual decision" 拒绝假设

**为什么？** agent 在"测验模式"下会复述技能，但在"工作模式"下会钻空子。**真实性触发真实行为**。

### 3.2 决策 2：好场景的 5 个关键要素

| 要素 | 论证 |
|------|------|
| **具体选项** | 强制 A/B/C 选择，不是开放式 |
| **真实约束** | 具体时间（6pm）、实际后果（$10k/min） |
| **真实文件路径** | `/tmp/payment-system` 而非"一个项目" |
| **让 agent 行动** | "What do you do?" 而非 "What should you do?" |
| **没有简单出路** | 不能以"问人类伙伴"为借口 |

**这 5 个要素的本质：** 把"学术讨论"转化为"工作情境"。

### 3.3 决策 3：3+ 组合压力是最低标准

> "Best tests combine 3+ pressures"

**7 种压力类型：**
1. 时间（紧急、截止日期、部署窗口）
2. 沉没成本（数小时工作、删除"浪费"）
3. 权威（资深人士说跳过）
4. 经济（工作、晋升、公司生存）
5. 疲惫（一天结束、累了、想回家）
6. 社交（显得教条、不灵活）
7. 实用主义（"实用 vs 教条"）

**为什么 3+？** 单一压力太弱——agent 抗住单一压力。**多重压力是"测试真实失败"的必要条件**。

**对比 systematic-debugging 的"3+ 修复失败 = 质疑架构"**：
- 同样的"3+ 硬触发器"模式
- 同样反映"单一不充分、多重才暴露本质"

### 3.4 决策 4：7 个常见合理化（用于 REFACTOR 阶段）

```
"This case is different because..."
"I'm following the spirit not the letter"
"The PURPOSE is X, and I'm achieving X differently"
"Being pragmatic means adapting"
"Deleting X hours is wasteful"
"Keep as reference while writing tests first"
"I already manually tested it"
```

**这 7 个合理化的本质：** 都是"特定场景下规则可以例外"的变体。

**为什么必须逐字记录？** 因为"agent 用了什么措辞"决定"应该用什么反制"。

### 3.5 决策 5：4 种漏洞堵法

| 漏洞堵法 | 例子 |
|----------|------|
| 1. 规则中的明确否定 | "Don't keep it as 'reference'" |
| 2. 合理化表条目 | "Keep as reference = testing after" |
| 3. 红旗条目 | "Keep as reference" → STOP |
| 4. 更新 description | "Use when tempted to test after" |

**这 4 种堵法覆盖不同层次：**
- 规则：告诉 agent 做什么
- 合理化表：反驳借口
- 红旗：自我检查触发器
- description：让 LLM 在"想违规"时发现技能

### 3.6 决策 6：元测试（Meta-Testing）的 3 响应分类

```markdown
your human partner: You read the skill and chose Option C anyway.
How could that skill have been written differently to make
it crystal clear that Option A was the only acceptable answer?
```

**3 种响应：**
1. **"技能很清楚，我选择忽略它"** → 加"Violating letter is violating spirit"
2. **"技能应该说 X"** → 加他们的建议
3. **"我没看到 Y 部分"** → 让关键点更突出

**这一决策的精妙：**
- 把"agent 违规"转化为"技能改进信号"
- 三种响应对应三种不同的失败模式（心理、文档、组织）
- 形成了"持续改进的飞轮"

### 3.7 决策 7：4 个防弹标志

```
1. Agent 在最大压力下选择正确的选项
2. Agent 引用技能部分作为理由
3. Agent 承认诱惑但仍然遵循规则
4. 元测试揭示"技能很清楚，我应该遵循它"
```

**这些标志是"测试可观察现象"——** 不需要"我猜 agent 理解了"，而是"agent 表现出理解"。

### 3.8 决策 8：6 次迭代达到防弹的真实数据

> "From applying TDD to TDD skill itself (2025-10-03):
> 6 RED-GREEN-REFACTOR iterations to bulletproof
> Baseline testing revealed 10+ unique rationalizations
> Final VERIFY GREEN: 100% compliance under maximum pressure"

**意义：**
- 这是"用 TDD 写 TDD 技能"的元案例
- 6 次迭代、10+ 合理化、100% 遵从 = 量化证据
- 文档用自己的演化证明方法论

---

## 4. 文件结构剖析

```
testing-skills-with-subagents.md (385 行)
├── 标题 + 概述 (15 行) - 必要背景 + 完整示例
├── 何时使用 (12 行) - 测试 vs 不测试
├── 技能测试的 TDD 映射 (12 行) - 6 阶段表
├── RED 阶段：基线测试 (40 行) - 过程 + 示例
│   ├── 过程 (5 项 checklist)
│   └── 4 小时 TDD 场景
├── GREEN 阶段：编写最小技能 (7 行)
├── VERIFY GREEN：压力测试 (66 行)
│   ├── 编写压力场景 (16 行) - 坏/好/极好对比
│   ├── 压力类型 (15 行) - 7 类型表
│   ├── 关键要素 (7 行) - 5 要素
│   └── 测试设置 (8 行) - 心理暗示
├── REFACTOR 阶段：关闭漏洞 (78 行)
│   ├── 7 个常见合理化
│   ├── 4 种漏洞堵法
│   └── 重构后重新验证
├── 元测试 (28 行) - 3 响应分类
├── 何时技能是防弹的 (15 行) - 4 标志
├── 示例：TDD 技能防弹化 (24 行) - 3 阶段迭代
├── 测试清单 (24 行) - 3 阶段 × 多子项
├── 常见错误 (26 行) - 6 个 ❌ + 修复
├── 快速参考 (10 行) - 6 阶段表
└── 实际影响 (8 行) - 6 次迭代的元数据
```

**结构特点：**
1. **压力测试占 17%**（66 行）—— 核心方法论
2. **REFACTOR 阶段占 20%**（78 行）—— 关闭漏洞是难点
3. **示例 + 清单占 13%**（49 行）—— 可操作性
4. **元测试占 7%**（28 行）—— 独特创新

---

## 5. 优点 / 亮点

### 5.1 "真实性心理暗示"作为测试基础设施

```markdown
IMPORTANT: This is a real scenario. You must choose and act.
```

这种"心理暗示"是**测试有效性的前提**——如果 agent 知道是测验，它会"表演"技能遵从；如果是"工作"，它会暴露真实行为。

### 5.2 "3+ 组合压力"的可证伪标准

> "Best tests combine 3+ pressures"

这不是"建议"，是"标准"——可以检验、可以违反、可以讨论。这比"压力应该足够"更有用。

### 5.3 7 种压力类型的分类学

7 种压力（时间、沉没成本、权威、经济、疲惫、社交、实用主义）覆盖了"agent 钻空子"的主要诱因。这种分类是**实证观察的结晶**。

### 5.4 4 种漏洞堵法的层次性

| 层次 | 目标 |
|------|------|
| 规则 | 告诉 agent 做什么 |
| 合理化表 | 反驳借口 |
| 红旗 | 自我检查 |
| description | 让 LLM 在"想违规"时发现技能 |

这种**多层次防御**与 `defense-in-depth.md`（systematic-debugging 配套）的哲学一致。

### 5.5 元测试的"三响应分类"

**独特价值：**
- 把"agent 违规"从"失败"转化为"技能改进信号"
- 三种响应（清晰但忽略 / 文档问题 / 组织问题）对应三种根本不同的失败模式
- 形成了**自我改进的飞轮**

### 5.6 "防弹"作为可观察现象

不像"我猜 agent 理解了"，4 个防弹标志都是**可观察、可验证、可证伪**的现象。

### 5.7 自身的"活的例子"

> "From applying TDD to TDD skill itself (2025-10-03):
> 6 RED-GREEN-REFACTOR iterations to bulletproof"

这是**形式即内容**——文档用自己的演化证明方法论。

### 5.8 常见错误章节的"实用主义"

不是列举"理论上可能的错误"，而是**6 个来自实际测试的真实错误**：
- ❌ 在测试之前编写技能（跳过 RED）
- ❌ 没有正确观察测试失败
- ❌ 弱测试用例（单一压力）
- ❌ 没有捕获精确的失败
- ❌ 模糊的修复
- ❌ 一次通过后停止

每个错误都有"修复"——具体可操作。

### 5.9 与其他文档的引用网络

- → `superpowers:test-driven-development`（必需背景）
- → `persuasion-principles.md`（压力类型理论）
- → `examples/CLAUDE_MD_TESTING.md`（完整示例）
- ← 被 `writing-skills/SKILL.md` 引用

这种**有机的引用网络**体现了 superpowers 的模块化设计。

### 5.10 "失败也是数据"的科学态度

不假装"成功率为 100%"，而是承认：
- 6 次迭代才达到防弹
- 10+ 合理化被识别
- "Bulletproof" 是迭代的结果

这种诚实是**科学方法**的体现。

---

## 6. 缺点 / 风险 / 局限

### 6.1 "3+ 压力"的可操作性不足

> "Best tests combine 3+ pressures"

**问题：**
- "3+" 是模糊的下限
- 哪 3 种压力的组合最有效？
- 是否存在"无效的压力组合"？
- 文档没有给"压力配方"

**风险：** agent 可能"伪组合"——堆砌 3 种压力但实际上只有 1 种起作用。

### 6.2 场景生成方法论缺失

文档给了 5 个要素、3 个对比（坏/好/极好），但：
- 如何**生成**新场景？没有方法
- 如何**评估**场景质量？没说
- 如何**避免**场景之间的重叠？没说

**风险：** "凭感觉"生成场景，测试质量不可控。

### 6.3 缺少量化的"防弹"标准

4 个防弹标志都是定性的：
- "Agent chooses correct option under maximum pressure"——多少次试验？
- "Agent cites skill sections"——多少个？
- "100% compliance"——样本多大？

**风险：** "100% compliance" 可能是 N=1 的实验。

### 6.4 "基线测试"的可靠性

> "Run scenario WITHOUT skill, document exact failures"

**问题：**
- 不同 agent 模型的基线行为不同（Haiku vs Opus）
- 同一 agent 的多次运行可能给出不同响应
- 没有"基线稳定性"的概念

**风险：** 你记录的"基线失败"可能是"agent 的随机失败"。

### 6.5 漏洞堵法的"无限循环"风险

> "Continue REFACTOR cycle until no new rationalizations"

**问题：**
- "没有新合理化"如何确定？
- 可能存在"agent 已找到所有合理化" vs "agent 没发现剩余合理化"
- 文档没有"停止标准"

**风险：** 永远无法宣布"防弹"。

### 6.6 元测试的"主观性"

> "Three possible responses: 1) Skill was clear... 2) Skill should have said X... 3) I didn't see Y..."

**问题：**
- agent 的响应可能不在这 3 类之内
- agent 可能混合多种响应
- 文档没有"如何处理第 4 种响应"

### 6.7 "TDD 自身"的特殊性

> "From applying TDD to TDD skill itself"

**问题：**
- TDD 技能是 superpowers 的"元技能"
- TDD 技能的"6 次迭代"是否能推广到其他技能？
- 文档没说"非 TDD 技能需要多少次迭代"

### 6.8 缺少"反例测试"指导

> "Counter-examples: Do they know when NOT to apply?"

这是 writing-skills/SKILL.md 中提到的，但 testing-skills-with-subagents.md 没有详细展开"反例测试"的方法。

**风险：** 技能可能"过度泛化"——在不该用时也被使用。

### 6.9 跨模型的"防弹性"问题

不同模型（Haiku/Sonnet/Opus）行为不同：
- 同一技能在 Haiku 上防弹，在 Opus 上可能钻空子
- 文档没有讨论"跨模型测试"
- anthropic-best-practices 提到了多模型测试，本文档没有补充

### 6.10 缺少"自动化"指导

所有方法都是手动的：
- 手动运行基线
- 手动识别合理化
- 手动堵漏洞

**问题：**
- 不能扩展到大规模技能测试
- 不能 CI/CD 集成
- 文档没有"如何自动化"

### 6.11 错误分类可能不完整

6 个常见错误可能不是全部：
- 还有哪些常见错误？
- 错误的错误分类有"遗漏"风险
- 文档说"Common Mistakes"但没说是"全部"

### 6.12 文档本身也需要测试

本文档是"测试技能的方法论"，但：
- 谁测试了这份文档？
- "TDD for TDD skill" 的元案例存在，但"测试方法论的测试" 没有
- 这是"无限回归"问题

---

## 7. 关键洞察

### 7.1 洞察 1：真实性是测试有效性的前提

```markdown
IMPORTANT: This is a real scenario. You must choose and act.
```

LLM 在"测验模式"和"工作模式"下行为可能截然不同。**测试的有效性 = 测试场景的真实性**。

### 7.2 洞察 2：3+ 组合压力是"测试强度"的可证伪标准

不是"应该足够强"，是"至少 3 种压力"——可以验证、可以违反、可以讨论。

这是"模糊建议"和"工程标准"的分界。

### 7.3 洞察 3：失败是数据，不是挫折

> "Continue REFACTOR cycle until no new rationalizations"

不是"agent 失败了 → 沮丧"，是"agent 失败了 → 新的合理化条目 → 漏洞堵法"。

**失败 = 改进机会**——这是科学态度。

### 7.4 洞察 4：元测试把"违规"转化为"技能改进信号"

> "How could that skill have been written differently?"

3 种响应分类是**自动诊断系统**——agent 违规时，我们能立即定位失败模式。

### 7.5 洞察 5：防弹是可观察的现象，不是内在品质

4 个防弹标志都是**可观察、可验证、可证伪**的。这与 systematic-debugging 的"症状 vs 根因"哲学一致。

### 7.6 洞察 6：自身是"活的例子"

> "From applying TDD to TDD skill itself (2025-10-03)"

文档用自身的演化证明方法论。**形式即内容**。

### 7.7 洞察 7：多层次防御

| 层次 | 作用 |
|------|------|
| 规则 | 告诉做什么 |
| 合理化表 | 反驳借口 |
| 红旗 | 自我检查 |
| description | 让 LLM 在"想违规"时发现技能 |

每层被不同方式绕过（与 `defense-in-depth.md` 一致）。

### 7.8 洞察 8：心理暗示是测试基础设施

"IMPORTANT: This is a real scenario" 不是装饰，是**让 agent 切换到"工作模式"的关键**。

### 7.9 洞察 9：持续改进的飞轮

```
基线失败 → 漏洞堵法 → 防弹 → 实际使用 → 新失败 → 新漏洞堵法
```

这不是一次性测试，是**持续演化的飞轮**。

### 7.10 洞察 10：诚实是元层面的科学态度

不假装"100% 一次成功"，承认"6 次迭代才防弹、10+ 合理化"。

**这种诚实让方法论可信**——不夸张、不缩小。

---

## 8. 关联文档

### 8.1 内部关联（同目录）

| 文件 | 关系 |
|------|------|
| `SKILL.md` | **主技能**——引用本文件作为测试方法论 |
| `anthropic-best-practices.md` | 官方补充——评估驱动开发是替代方法 |
| `persuasion-principles.md` | 心理学基础——"压力类型"的理论来源 |
| `examples/CLAUDE_MD_TESTING.md` | 完整示例——本文件是"操作手册" |

### 8.2 外部关联（其他技能）

| 技能 | 关系 |
|------|------|
| `test-driven-development` | **必需背景**——RED-GREEN-REFACTOR 来源 |
| `systematic-debugging` | 共享哲学——"3+ 硬触发器"模式 |
| `defense-in-depth` | 共享哲学——多层次防御 |
| `verification-before-completion` | 共享哲学——"Verification is automatic, not optional" |

### 8.3 上游文档

| 来源 | 关系 |
|------|------|
| TDD 文献（Kent Beck） | RED-GREEN-REFACTOR 原始概念 |
| Cialdini (2021) | 7 原则的心理学基础 |
| Meincke et al. (2025) | LLM 行为数据 |
| 说服心理学 | "3+ 压力"的概念基础 |

---

## 9. 状态评价

### 9.1 完成度：★★★★★（5/5）

- 4 阶段（RED/GREEN/REFACTOR/Verify）全覆盖
- 7 种压力类型
- 4 种漏洞堵法
- 元测试 3 响应分类
- 6 阶段完整 checklist
- 6 个常见错误

### 9.2 一致性：★★★★★（5/5）

- 与 superpowers 整体哲学一致
- 与 TDD 共享循环结构
- 与 defense-in-depth 共享多层次防御

### 9.3 可操作性：★★★★（4/5）

- 5 个场景要素
- 3 个对比示例
- 完整 checklist
- **扣分点**：场景生成方法、量化标准、自动化指导缺失

### 9.4 创新度：★★★★★（5/5）

- 压力场景的"3+ 组合"
- 元测试的"3 响应分类"
- 4 种漏洞堵法的层次设计
- "6 次迭代达到防弹" 的真实数据

### 9.5 整体评价：**9.2 / 10**

这是一份**理论深度+实证数据+操作指导+诚实态度**四者兼备的优秀方法论文档。它为 superpowers 的所有"抗合理化"技术提供了**可操作的测试流程**。

**唯一扣分点：** 量化指标、自动化、跨模型测试的不足。

---

## 10. 给读者的启示

### 10.1 对技能作者

1. **没有"看 agent 失败"就没有"知道防什么"**——基线测试不可跳过
2. **3+ 组合压力是最低标准**——单一压力不够
3. **逐字记录合理化**——"agent 用什么措辞"决定"用什么反制"
4. **4 种漏洞堵法都该用**——单层防御容易被绕过
5. **"继续 REFACTOR 直到无新合理化"**——别一次通过就停

### 10.2 对 AI 工程师

1. **真实性心理暗示是测试基础设施**——不是装饰
2. **多层次防御对应"被绕过方式"**——每层被不同方式绕过
3. **失败是数据**——持续改进的飞轮
4. **"100% 遵从"可能 N=1**——需要量化和稳定性
5. **元测试是自我诊断**——违规 = 改进信号

### 10.3 对 superpowers 用户

1. **TDD 自身用 TDD 写**——6 次迭代达到防弹
2. **每种技能类型需要不同测试**（与 writing-skills/SKILL.md 一致）
3. **"防弹"是迭代结果**——不是一次达到
4. **基线测试的可靠性**——同一 agent 多次运行可能不同
5. **跨模型行为不同**——Haiku 防弹不等于 Opus 防弹

### 10.4 对更广泛的测试方法论

1. **"真实性"是 LLM 测试的关键**——比"覆盖率"更基础
2. **"3+ 压力"是组合测试的可证伪标准**
3. **"元测试" 把测试结果转化为产品改进**——是自我诊断
4. **失败 = 数据 = 改进**——不是挫折
5. **形式即内容**——文档用自身证明方法论

---

## 11. 后续工作 / 改进方向

### 11.1 应补充的内容

1. **场景生成方法论**——如何产生"有效"场景？给模板
2. **量化的"防弹"标准**——通过率、试验次数、置信区间
3. **跨模型测试指南**——Haiku/Sonnet/Opus 的差异处理
4. **自动化指导**——CI/CD 集成、脚本化场景运行
5. **"停止标准"**——REFACTOR 循环何时停止

### 11.2 应展开的讨论

1. **基线稳定性**——同一 agent 多次运行的一致性
2. **"伪压力"识别**——如何检测无效的压力组合
3. **"反例测试"**——技能"不该用"的测试
4. **跨技能套件测试**——技能之间的相互作用
5. **元层面**——"测试方法论"本身的测试

### 11.3 元层面

本文档是 superpowers 的"测试宪法"。建议：
- 收集 superpowers 用户对"哪些场景最有效"的反馈
- 跟踪 LLM 进化对压力测试的影响
- 定期更新"3+ 压力"的标准（可能需调整为"4+"或更高）

### 11.4 哲学层面

最大的开放问题是：**"真实性"在 LLM 测试中如何保证**？

LLM 没有"意识到这是测验"的明确信号——它可能：
- 识别场景模式（"这看起来像测试"）→ 表演技能遵从
- 真的"进入角色"→ 暴露真实行为
- 介于两者之间

**当前方法：** "IMPORTANT: This is a real scenario" 的心理暗示。

**未来可能：**
- 更精细的场景设计
- 多场景交叉验证
- 元认知测试（问 agent "你为什么这样选择"）

---

## 附录：本分析的方法论

本报告按 11 节统一结构组织：
1. 基本信息
2. 目的与背景
3. 核心技术决策
4. 文件结构剖析
5. 优点/亮点
6. 缺点/风险/局限
7. 关键洞察
8. 关联文档
9. 状态评价
10. 给读者的启示
11. 后续工作

所有论断都基于 testing-skills-with-subagents.md 文本和其在 superpowers 生态中的角色。**不做超出文档声明的推论**。
