# brainstorming/SKILL.md 深度分析报告

> 本报告分析 superpowers 项目的**入门级**技能 `brainstorming`。它通过 HARD-GATE 机制强制设计先于实现。

---

## 1. 基本信息

| 项 | 值 |
|----|----|
| **文件名** | `SKILL.md` |
| **总行数** | 165 行 |
| **YAML 字段** | `name: brainstorming` + `description: "You MUST use this before any creative work..."` |
| **核心定位** | 设计前的强制关卡（Hard Gate） |
| **强约束** | HARD-GATE：未批准设计前不能实现 |
| **核心创新** | Graphviz 流程图 + 视觉伴侣（HARD-GATE 与 V 可视化） |
| **终态** | 调用 writing-plans 技能 |
| **依赖技能** | `writing-plans`（终态）、`elements-of-style:writing-clearly-and-concisely`（文档） |

---

## 2. 目的与背景

### 2.1 文档的"强制关卡"角色

`brainstorming` 是 superpowers 流程的**第一个关卡**：

```
[任何创意工作]
    ↓
brainstorming ← 强制
    ↓
writing-plans
    ↓
executing-plans / subagent-driven-development
    ↓
[实现]
```

HARD-GATE 强制要求：
> "Do NOT invoke any implementation skill, write any code, scaffold any project, or take any implementation action until you have presented a design and the user has approved it."

### 2.2 三大核心命题

1. **"You MUST use this before any creative work"** —— 强制使用
2. **"Every project goes through this process"** —— 无例外
3. **"The terminal state is invoking writing-plans"** —— 单一终态

### 2.3 解决的具体问题

| 问题 | 本技能提供的方案 |
|------|------------------|
| agent 跳过设计直接实现 | HARD-GATE 强制关卡 |
| agent 不澄清用户需求 | 9 步清单 + 一次一个问题 |
| agent 设计不完整 | 4 项自审检查（占位符/一致性/范围/模糊性） |
| agent 范围过大 | 立即标记 + 分解为子项目 |
| agent 不使用工具辅助 | 视觉伴侣作为可选工具 |

---

## 3. 核心技术决策

### 3.1 决策 1：HARD-GATE 机制

```xml
<HARD-GATE>
Do NOT invoke any implementation skill, write any code, scaffold any project, 
or take any implementation action until you have presented a design and 
the user has approved it. This applies to EVERY project regardless of 
perceived simplicity.
</HARD-GATE>
```

**关键要素：**
- 大写 "HARD-GATE" 视觉强化
- "regardless of perceived simplicity" 切断"这太简单"的合理化
- 4 个动作禁令：invoke skill / write code / scaffold project / take action

**这一机制的洞察：**
- 与 TDD 的"NO PRODUCTION CODE WITHOUT A FAILING TEST FIRST" 一脉相承
- 把"设计"从"建议"变为"关卡"
- 与 `persuasion-principles.md` 的"权威"原则一致——绝对语言

### 3.2 决策 2：反模式"这太简单了不需要设计"

> "'Simple' projects are where unexamined assumptions cause the most wasted work."

**这一陈述的洞察：**
- 反直觉——简单项目被认为"不需要设计"
- 实际：简单项目的错误假设浪费最多工作
- "The design can be short (a few sentences for truly simple projects), but you MUST present it and get approval."

**这一设计的精妙：**
- 承认设计可以短（"a few sentences"）
- 但仍强制呈现和批准
- 既防止"过度设计"也防止"跳过设计"

### 3.3 决策 3：9 步清单的强制顺序

```
1. 探索项目上下文
2. 提供视觉伴侣（如适用）
3. 询问澄清问题
4. 提出 2-3 种方法
5. 呈现设计
6. 编写设计文档
7. 规范自审
8. 用户审查书面规范
9. 过渡到实现
```

**这一清单的精妙：**
- 9 步覆盖**从理解到实现的完整链路**
- 步骤 7（自审）+ 步骤 8（用户审查）= **双重质量门控**
- 步骤 9 明确指向 writing-plans ——**单一终态**

### 3.4 决策 4：Graphviz 流程图

```dot
digraph brainstorming {
    "Explore project context" [shape=box];
    "Visual questions ahead?" [shape=diamond];
    "Offer Visual Companion" [shape=box];
    "Ask clarifying questions" [shape=box];
    "Propose 2-3 approaches" [shape=box];
    "Present design sections" [shape=box];
    "User approves design?" [shape=diamond];
    "Write design doc" [shape=box];
    "Spec self-review" [shape=box];
    "User reviews spec?" [shape=diamond];
    "Invoke writing-plans skill" [shape=doublecircle];
    ...
}
```

**这一决策的价值：**
- 视觉化呈现 9 步流程
- `shape=doublecircle` 标识终态
- `shape=diamond` 标识决策点
- "User approves design?" 是关键**循环节点**——失败则回到 "Present design sections"

**与 writing-skills/SKILL.md 的"何时使用流程图"一致：**
- 决策非显而易见
- 过程循环可能过早停止
- "何时用 A vs B" 的决策

### 3.5 决策 5：范围评估在"询问详细问题之前"

> "Before asking detailed questions, assess scope: if the request describes multiple independent subsystems..."

**这一决策的洞察：**
- 防止"在错误的问题上浪费时间"
- "Build a platform with chat, file storage, billing, and analytics"——明显太大
- 立即标记 = 节省时间

**分解为子项目：**
- 独立的部分是什么？
- 它们如何关联？
- 应该按什么顺序构建？

**这与 systematic-debugging 的"3+ 修复失败 = 质疑架构" 一致**——发现规模问题要立即重新规划。

### 3.6 决策 6：增量验证（Incremental Validation）

> "Ask after each section whether it looks right so far"

**这一决策的洞察：**
- 不是"一次性呈现所有设计"
- 而是"每节一确认"
- "Be ready to go back and clarify if something doesn't make sense"

**价值：**
- 减少后期返工
- 让用户参与设计过程
- "Be flexible" —— 关键原则

### 3.7 决策 7：4 项自审（Spec Self-Review）

| 检查 | 目标 |
|------|------|
| **占位符扫描** | "TBD"、"TODO"、不完整的章节 |
| **内部一致性** | 章节不矛盾 |
| **范围检查** | 足够聚焦 |
| **模糊性检查** | 不可二义性解释 |

**这一决策的精妙：**
- 不依赖外部审查（节省时间）
- 但有明确标准
- "Fix any issues inline. No need to re-review — just fix and move on."

### 3.8 决策 8：用户审查关卡（User Review Gate）

> "Wait for the user's response. If they request changes, make them and re-run the spec review loop. Only proceed once the user approves."

**这一决策的洞察：**
- 自审 + 用户审 = **双重门控**
- 等待响应（不是"假设批准"）
- 反馈循环明确：changes → re-review → approved

### 3.9 决策 9：单一终态的强约束

> "The terminal state is invoking writing-plans. Do NOT invoke frontend-design, mcp-builder, or any other implementation skill."

**这一决策的洞察：**
- 防止 agent "创造性"地选择其他技能
- 与 brainstorming 配套的 writing-plans 是**唯一**的下一步
- 这种"单一路径"是纪律的体现

### 3.10 决策 10：视觉伴侣作为"工具"而非"模式"

> "Available as a tool — not a mode."

**关键区分：**
- 工具：agent **按问题**决定使用
- 模式：每个问题都强制使用

**"per-question decision"：**
- 即使用户接受，每次都重新决定
- 测试："would the user understand this better by seeing it than reading it?"

**这种"工具而非模式"的设计避免：**
- 强制使用造成 token 浪费
- 视觉化不适合的概念问题

---

## 4. 文件结构剖析

```
SKILL.md (165 行)
├── YAML frontmatter (3 行) - "MUST use this before any creative work"
├── 标题 + 概述 (4 行)
├── HARD-GATE 段 (5 行) - 4 个动作禁令
├── 反模式 (3 行) - "这太简单"
├── 清单 (12 行) - 9 步骤
├── 流程图 (29 行) - Graphviz 11 节点
├── 过程 (32 行)
│   ├── 理解想法 (8 行)
│   ├── 探索方法 (4 行)
│   ├── 呈现设计 (8 行)
│   ├── 为隔离和清晰而设计 (5 行)
│   └── 在现有代码库中工作 (5 行)
├── 设计之后 (28 行)
│   ├── 文档 (5 行)
│   ├── 规范自审 (8 行)
│   ├── 用户审查关卡 (5 行)
│   └── 实现 (3 行)
├── 关键原则 (8 行) - 6 条原则
└── 视觉伴侣 (19 行)
```

**结构特点：**
1. **HARD-GATE + 反模式占 5%**（8 行）—— 不可绕过的约束
2. **清单 + 流程图占 25%**（41 行）—— 操作规程
3. **过程占 19%**（32 行）—— 具体方法
4. **设计之后占 17%**（28 行）—— 后续动作
5. **关键原则 + 视觉伴侣占 16%**（27 行）—— 补充信息

---

## 5. 优点 / 亮点

### 5.1 HARD-GATE 范式

XML 标签 + 大写命名 + "regardless of perceived simplicity" 是**对 LLM 钻空子的预防性约束**。

这是 superpowers 整套体系的标志性"硬关卡"模式。

### 5.2 反模式的反向论证

> "'Simple' projects are where unexamined assumptions cause the most wasted work"

不直接说"你必须"，而是论证"为什么不"——让合理化更难。

### 5.3 9 步清单的完整性

从"探索上下文"到"调用 writing-plans"覆盖完整链路，且**每步都有具体目标**。

### 5.4 Graphviz 流程图的双重作用

- 流程图作为**视觉检查工具**——人看
- 流程图作为**LLM 提示**——决策点用 `diamond`，终态用 `doublecircle`

### 5.5 范围评估在前置位置

> "Before asking detailed questions, assess scope"

防止"在错误的问题上浪费时间"——这是元层面的效率。

### 5.6 4 项自审的"轻量级质量保证"

不依赖外部审查（节省时间），但有 4 个明确标准。

**与 spec-document-reviewer-prompt.md 的区别：**
- 自审：内联、agent 自我
- 文档审查：派遣子代理、上下文隔离

两者互补形成**双层质量保证**。

### 5.7 视觉伴侣的"工具"哲学

"工具而非模式"——按需使用，避免 token 浪费。

**"per-question decision"**——避免"开启就要一直用"的沉没成本。

### 5.8 "be flexible"的元原则

> "Be ready to go back and clarify when something doesn't make sense"

**洞察：** 流程图是"指导"不是"法律"——必要时可以回退。

### 5.9 单一终态的强约束

> "Do NOT invoke frontend-design, mcp-builder, or any other implementation skill. The ONLY skill you invoke after brainstorming is writing-plans."

**价值：** 防止 agent 创造性选择导致的流程破坏。

### 5.10 形式即内容

文档自身是"形式即内容"的好例子：
- 9 步清单的清单化呈现
- 流程图用 Graphviz dot
- 视觉伴侣用 XML 标签（`<HARD-GATE>`）

**它**用文档本身的形式**展示了**自己教授的内容**。

---

## 6. 缺点 / 风险 / 局限

### 6.1 HARD-GATE 可能过度严格

> "This applies to EVERY project regardless of perceived simplicity"

**风险：**
- 紧急 bug 修复可能不需要完整 brainstorming
- 探索性编程（spike）可能不需要完整设计
- 文档没说"什么算'创意工作'"

**改进空间：** 区分"生产实现" vs "探索性实验"。

### 6.2 "1 个问题 1 条消息"在某些场景下效率低

> "Only one question per message"

**风险：**
- 用户需要等待多轮才能回答 5 个问题
- 异步场景（如非交互式 CI）下不可行
- 紧急情况下可能不可接受

**改进空间：** 区分"实时对话" vs "异步场景"。

### 6.3 9 步清单的"过度结构化"风险

**问题：**
- 9 步是强结构
- 但某些简单项目可能不需要"提出 2-3 种方法"
- 文档承认"设计可以短"但没承认"步骤可以少"

**风险：** 形式主义而非实质。

### 6.4 自审的"主观性"

> "Could any requirement be interpreted two different ways?"

**问题：**
- "二义性"是 agent 的主观判断
- 没有"二义性检测工具"
- 文档没说"如何处理边界情况"

### 6.5 视觉伴侣的"token 密集型"成本

> "This feature is still new and can be token-intensive"

**风险：**
- token 成本可能超过价值
- 没有量化的"何时值得使用"
- 依赖用户对成本的主观判断

### 6.6 流程图与文字的潜在不一致

**风险：**
- 流程图说 11 节点
- 清单说 9 步骤
- 二者可能不完全对应（步骤 2 包含两个流程图节点："Offer Visual Companion" 是独立消息）
- 这种"流程图 ⊃ 清单"的关系可能让 LLM 困惑

### 6.7 范围评估的"主观性"

> "if the request describes multiple independent subsystems (e.g., 'build a platform with chat, file storage, billing, and analytics')"

**问题：**
- "独立子系统"由谁定义？
- "build a chat app" 算独立子系统还是单个项目？
- 边界模糊

### 6.8 缺少"快速模式"或"简略模式"

文档只有一个流程，没有"快速版"或"最小版"。对于**显然简单**的项目（如调整一个配置），9 步流程可能太重。

**改进空间：** 提供"1-2-3 步快速模式"作为补充。

### 6.9 "用户审查关卡" 的循环风险

> "If they request changes, make them and re-run the spec review loop. Only proceed once the user approves."

**风险：**
- 用户可能反复要求修改
- 没有"停止迭代"的标准
- 可能陷入"完美主义陷阱"

### 6.10 与其他技能的"硬衔接"

> "The ONLY skill you invoke after brainstorming is writing-plans"

**问题：**
- 如果用户实际想要的是"先原型"呢？
- 缺少"快速原型"技能作为 brainstorming 的替代
- 文档没承认有些项目**不需要**完整 plan

---

## 7. 关键洞察

### 7.1 洞察 1：HARD-GATE 是"关卡思维"的体现

不是"应该"做，而是"必须"做。这种"关卡"是软件工程传统（如 CI/CD 门控）的 LLM 适配。

### 7.2 洞察 2：反模式的反向论证

> "'Simple' projects are where unexamined assumptions cause the most wasted work"

不直接说"你必须"——而是论证"为什么不"。**这种反向论证**让合理化更难。

### 7.3 洞察 3：9 步清单是"过程脚手架"

清单不是"必须严格遵守"，而是"防止遗忘"——**过程脚手架**而非"法律"。

### 7.4 洞察 4：流程图的"LLM 友好"设计

- `shape=doublecircle` 标识终态
- `shape=diamond` 标识决策点
- 决策标签说明"yes/no"路径

这种"图形语义"是 LLM 友好的——agent 可以"看见"流程。

### 7.5 洞察 5：范围评估的前置性

> "Before asking detailed questions, assess scope"

防止"在错误的问题上浪费时间"——**元层面的效率**。

### 7.6 洞察 6：自审 + 用户审 = 双层质量门控

| 层次 | 检查者 | 目标 |
|------|--------|------|
| 自审 | agent 自己 | 内联修复 |
| 用户审 | 人类 | 最终批准 |

**两道门控**比"一道"更可靠。

### 7.7 洞察 7：视觉伴侣是"按需工具"

> "Per-question decision" + "tool not mode"

避免 token 浪费——只在视觉化真正提升理解时使用。

### 7.8 洞察 8：单一终态的纪律

> "The ONLY skill you invoke after brainstorming is writing-plans"

防止"创造性"——**单一路径**是纪律。

### 7.9 洞察 9："be flexible"的元原则

流程图是"指导"不是"法律"——必要时可以回退。**形式与实质的平衡**。

### 7.10 洞察 10：形式即内容

文档自身用：
- HARD-GATE 用 XML 标签
- 9 步用清单
- 流程用 Graphviz
- 视觉伴侣提供 XML 示例

**形式即内容**——它**用文档本身的形式**展示了**自己教授的内容**。

---

## 8. 关联文档

### 8.1 内部关联（同目录）

| 文件 | 关系 |
|------|------|
| `visual-companion.md` | **配套指南**——SKILL.md 在视觉伴侣部分引用 |
| `spec-document-reviewer-prompt.md` | **子代理工具**——派遣规范审查者 |

### 8.2 外部关联（其他技能）

| 技能 | 关系 |
|------|------|
| `writing-plans` | **终态**——brainstorming 后的唯一技能 |
| `using-superpowers` | 元技能——brainstorming 是 using-superpowers 推荐流程的第一步 |
| `test-driven-development` | 共享哲学——"先设计后实现"是 TDD 的扩展 |
| `elements-of-style:writing-clearly-and-concisely` | 文档写作——规范文档的写作建议 |

### 8.3 上游文档

| 来源 | 关系 |
|------|------|
| 软件工程传统 | CI/CD 门控、HARD-GATE 模式 |
| 系统化设计方法 | "先设计后实现" 的传统 |
| `persuasion-principles.md` | 权威原则——"You MUST"、"No exceptions" |

---

## 9. 状态评价

### 9.1 完成度：★★★★★（5/5）

- HARD-GATE 明确
- 9 步清单完整
- 流程图视觉化
- 4 项自审
- 用户审查关卡
- 视觉伴侣指南

### 9.2 一致性：★★★★★（5/5）

- 与 superpowers 整体哲学一致
- HARD-GATE 模式与 TDD 铁律一致
- 视觉伴侣的"工具而非模式"与 CSO 哲学一致

### 9.3 可操作性：★★★★★（5/5）

- 9 步清单可直接执行
- 流程图可直接遵循
- 自审 4 项可直接检查

### 9.4 创新度：★★★★（4/5）

- HARD-GATE 是 superpowers 的标志性创新
- 视觉伴侣的"按需工具"哲学
- **扣分点**：基础方法论（设计前实现）与软件工程传统一致

### 9.5 整体评价：**9.3 / 10**

这是一份**纪律严明、流程清晰、形式即内容**的设计关卡技能。它是 superpowers 整套体系的"入口"，承担"防止 agent 跳过设计"的核心责任。

**唯一扣分点：** 缺少"快速模式"、过度严格可能导致低效。

---

## 10. 给读者的启示

### 10.1 对技能作者

1. **HARD-GATE 是反合理化的核武器**——绝对语言 + 无例外
2. **反模式的反向论证**——论证"为什么不"比直接说"必须"更有效
3. **清单 + 流程图双形式**——文字与视觉互补
4. **自审 + 用户审 = 双层门控**——比单层更可靠
5. **形式即内容**——文档自身示范自己教授的内容

### 10.2 对 AI 工程师

1. **"设计关卡"是 LLM 时代的新模式**——软件工程传统的 LLM 适配
2. **"工具而非模式"是资源节约的智慧**——避免 token 浪费
3. **"per-question decision"是细粒度决策**——避免沉没成本
4. **范围评估前置**是元层面效率——防止在错误问题上花时间
5. **单一终态的纪律**防止流程破坏

### 10.3 对 superpowers 用户

1. **不要跳过 brainstorming**——即使简单项目也要走流程
2. **9 步清单是过程脚手架**——不是法律
3. **HARD-GATE 是真约束**——不是建议
4. **自审后用户审**——双层门控不可省
5. **终态是 writing-plans**——不要跳到实现

### 10.4 对更广泛的提示工程

1. **HARD-GATE 模式**可推广到其他纪律性需求
2. **反模式 + 反向论证**比直接说"必须"更有效
3. **Graphviz 流程图**是 LLM 友好的可视化工具
4. **自审 + 用户审**是质量保证的双层门控
5. **"工具而非模式"**是 token 经济学的应用

---

## 11. 后续工作 / 改进方向

### 11.1 应补充的内容

1. **"快速模式"**——对于显然简单的项目（如配置调整）
2. **"探索性模式"**——spike、原型等不需要完整设计
3. **量化的"何时值得"标准**——视觉伴侣的成本/价值判断
4. **停止迭代的标准**——用户审查关卡的"完美主义陷阱"
5. **与其他技能的衔接说明**——如果用户想做"先原型"怎么办

### 11.2 应展开的讨论

1. **范围评估的客观标准**——"独立子系统"如何定义
2. **自审工具化**——是否有自动检测占位符/矛盾的脚本
3. **HARD-GATE 的例外**——紧急 bug 修复怎么办
4. **跨技能冲突**——用户想做的不在 writing-plans 之后
5. **异步场景**——非交互式 CI/CD 下的 brainstorming

### 11.3 元层面

本文档是 superpowers 的"入口关卡"。建议：
- 收集"用户觉得 9 步太多"的反馈
- 跟踪"HARD-GATE 被绕过"的场景
- 定期更新"何时使用 vs 不使用"标准

### 11.4 哲学层面

最大的开放问题是：**"设计 vs 探索"的边界在哪里**？

当前立场：所有"创意工作"都要设计。
**问题：**
- 探索性编程（spike）是创意工作吗？
- "先尝试再设计"在某些场景下更高效
- HARD-GATE 可能压制探索精神

**未来可能：**
- "探索模式"作为 brainstorming 的轻量级替代
- 区分"实验性" vs "生产性"项目
- 让用户在 HARD-GATE 与"快速原型"之间选择

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

所有论断都基于 SKILL.md 文本和其在 superpowers 生态中的角色。**不做超出文档声明的推论**。
