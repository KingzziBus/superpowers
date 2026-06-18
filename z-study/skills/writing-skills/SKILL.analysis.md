# writing-skills/SKILL.md 深度分析报告

> 本报告对 superpowers 项目的元技能（meta-skill）`writing-skills` 进行 11 维度深度分析。
> 它是关于"如何写技能"的方法论——是连接所有其他技能的"元层"技能。

---

## 1. 基本信息

| 项 | 值 |
|----|----|
| **文件名** | `SKILL.md` |
| **总行数** | 656 行 |
| **YAML 字段** | `name: writing-skills` + `description: Use when creating new skills, editing existing skills, or verifying skills work before deployment` |
| **核心定位** | 元技能 / 技能创建方法论 / TDD for 过程文档 |
| **依赖技能** | `superpowers:test-driven-development`（必需背景） |
| **配套文件** | `anthropic-best-practices.md`、`persuasion-principles.md`、`testing-skills-with-subagents.md`、`graphviz-conventions.dot`、`render-graphs.js`、`examples/CLAUDE_MD_TESTING.md` |
| **目录结构** | `skills/writing-skills/`（主目录 + examples/ 子目录） |

---

## 2. 目的与背景

### 2.1 为什么需要这个技能？

LLM 的"提示词/技能"是新型的可执行文档。它不像代码有编译器保护、不像 API 文档有运行时反馈。**技能的失败模式是：agent 根本不知道有这技能，或者 agent 在压力下绕过技能**。

`writing-skills` 解决的是 superpowers 生态系统的元问题：**如何让技能本身"防弹"**。

### 2.2 三个核心命题

1. **"Writing skills IS TDD applied to process documentation"**（编写技能就是 TDD 应用于过程文档）
2. **"If you didn't watch an agent fail without the skill, you don't know if the skill teaches the right thing"**（未观察失败 = 不知教的是否正确）
3. **"NO SKILL WITHOUT A FAILING TEST FIRST"**（铁律：没有失败测试就没有技能）

### 2.3 解决的具体问题

| 问题 | 本技能提供的方案 |
|------|------------------|
| 文档没人读 | CSO（Claude Search Optimization）让 description 可发现 |
| agent 压力下钻空子 | RED-GREEN-REFACTOR + 合理化表 + 红旗列表 |
| 技能互相冗余/冲突 | Skill 类型分类（技术/模式/参考）+ 文件组织指南 |
| Token 浪费 | Token 效率 4 级目标（150/200/500 字） |
| 编辑技能不测试 | 铁律覆盖新技能 AND 编辑 |

---

## 3. 核心技术决策

### 3.1 决策 1：将 TDD 范式迁移到文档

**为什么 TDD 适用于技能？**

| TDD 概念 | 技能创建映射 | 论证 |
|---------|--------------|------|
| 单元测试 | 压力场景 | "Agent 在 X 情境下会做什么"是可观测的 |
| 生产代码 | SKILL.md | 文档是 agent 的"接口" |
| RED | 观察基线失败 | 不知道失败模式 = 不知道要防什么 |
| GREEN | 写最小技能 | 不过度设计、只解决观测到的失败 |
| REFACTOR | 堵漏洞 | 关闭新发现的合理化 |

**优势：** 把抽象的"写好文档"转化为可观测、可度量、可迭代的工程问题。

**风险：** "压力场景"本身有方法论问题——什么算"足够压力"？技能在 6.7 节"Common Mistakes"中提到："单一压力的弱测试"。

### 3.2 决策 2：CSO（Claude Search Optimization）作为独立章节

**为什么不是简单写 description？**

- 测试证据：description 包含"代码审查"导致 agent 只做一次审查（即使流程图显示两次）
- 这是"提示注入式捷径"的反 LLM 失败模式
- **关键约束**：description = 触发条件，**不**概括工作流

**CSO 的 4 个子决策：**
1. **丰富描述**：用 "Use when..." + 具体症状/触发器
2. **关键词覆盖**：错误消息（"Hook timed out"）、症状（"flaky"）、同义词、工具名
3. **描述性命名**：动名词、动词优先（`creating-skills` > `skill-creation`）
4. **Token 效率**：入门 < 150 字、频繁加载 < 200 字、其他 < 500 字

### 3.3 决策 3：4 种技能类型 + 不同测试方法

| 技能类型 | 示例 | 测试方法 | 成功标准 |
|----------|------|----------|----------|
| 纪律执行（TDD、verification） | TDD、verification-before-completion | 学术问题 + 压力场景 + 组合压力 | 压力下遵从 |
| 技术（how-to） | condition-based-waiting、root-cause-tracing | 应用 + 变化 + 缺失信息 | 正确应用 |
| 模式（心智模型） | reducing-complexity | 识别 + 应用 + 反例 | 正确识别 |
| 参考（文档/API） | API 文档 | 检索 + 应用 + 缺口 | 找到并正确应用 |

**为什么需要不同测试？** 因为它们的失败模式不同。纪律类失败 = 钻空子；技术类失败 = 不会用；模式类失败 = 不会识别；参考类失败 = 找不到。

### 3.4 决策 4：抗合理化的 5 种技术（5 子节）

| 技术 | 作用 | 例子 |
|------|------|------|
| 明确关闭每个漏洞 | 禁止特定变通 | "Don't keep as reference" |
| 解决"精神 vs 文字"争论 | 切断"我遵循精神"借口 | "Violating letter is violating spirit" |
| 构建合理化表 | 攻击所有观测到的借口 | "Tests after = what does this do?" |
| 创建红旗列表 | 自我检查触发器 | "Code before test" |
| 更新 CSO 描述 | 把违规症状加入触发器 | description 中加 violation symptoms |

**洞察：** 这是把"反 LLM 钻空子"这一软性议题转化为"具体可操作的技术清单"。

### 3.5 决策 5：STOP 关卡——禁止批量创建

> "After writing ANY skill, you MUST STOP and complete the deployment process."

**为什么要硬性 STOP？** 因为批量创建的反模式是"测试节省成本"——agent 会合理化"批处理更高效"但实际上批量 = 未测试 = 违反纪律。STOP 强制每个技能走完完整 RED-GREEN-REFACTOR。

### 3.6 决策 6：交叉引用而非 @-引用

**禁用 @-引用：**
> "@ syntax force-loads files immediately, consuming 200k+ context before you need them"

**使用方式：**
- ✅ `**REQUIRED SUB-SKILL:** Use superpowers:test-driven-development`
- ❌ `@skills/testing/test-driven-development/SKILL.md`

**为什么？** LLM 的文件系统模型是"按需读取"。@-引用违反这一模型，导致 token 浪费。

### 3.7 决策 7：Graphviz 决策图作为可视化锚点

仅一个流程图（"何时使用流程图"），但意义重大：
- 展示 **decision 节点** 用 `shape=diamond`
- 展示 **action 节点** 用 `shape=box`
- 流程图用于"非显而易见的决策"，不用于参考材料

---

## 4. 文件结构剖析

```
SKILL.md (656 行)
├── YAML frontmatter (3 行)
├── Overview (12 行) - "TDD for documentation" 命题
├── What is a Skill? (6 行)
├── TDD Mapping for Skills (15 行) - 7 字段映射表
├── When to Create a Skill (13 行) - 4 创建条件 + 4 不创建条件
├── Skill Types (10 行) - 技术/模式/参考
├── Directory Structure (21 行) - 扁平命名空间 + 拆分条件
├── SKILL.md Structure (45 行) - 字段约束 + 模板
├── Claude Search Optimization (CSO) (146 行) - 4 子节
│   ├── Rich Description Field (52 行) - CSO 最关键部分
│   ├── Keyword Coverage (7 行)
│   ├── Descriptive Naming (3 行)
│   ├── Token Efficiency (74 行)
│   └── Cross-Referencing Other Skills (10 行)
├── Flowchart Usage (32 行) - 决策图 + 何时用/不用
├── Code Examples (22 行) - 1 个好例子 > 多个平庸例子
├── File Organization (28 行) - 3 种组织模式
├── The Iron Law (19 行) - 铁律 + 5 条无例外
├── Testing All Skill Types (49 行) - 4 类型 × 3 测试方法
├── Common Rationalizations for Skipping Testing (14 行) - 8 条借口
├── Bulletproofing Skills Against Rationalization (66 行) - 5 技术
├── RED-GREEN-REFACTOR for Skills (28 行) - 3 阶段详述
├── Anti-Patterns (20 行) - 4 个反模式
├── STOP: Before Moving to Next Skill (11 行) - 硬性停止
├── Skill Creation Checklist (TDD Adapted) (37 行) - 4 阶段清单
├── Discovery Workflow (10 行) - 6 步用户旅程
└── The Bottom Line (9 行) - 总结
```

**结构特点：**
1. **CSO 章节占 22%**（146 行）—— 反映"可发现性"是技能创建的第一性问题
2. **"反合理化"占 24%**（合理化表 14 行 + 反合理化技术 66 行 + 借口表 8 行）—— 这是纪律执行类技能的"刚需"
3. **清单占 6%**（37 行）—— 落地动作
4. **哲学/原则占 10%**（铁律、TDD 映射、The Bottom Line）—— 提供支撑

---

## 5. 优点 / 亮点

### 5.1 范式转移：将"写文档"重新定义为"工程任务"

> "Writing skills IS Test-Driven Development applied to process documentation."

这是该技能最深层的价值。它把模糊的"文档写得好"问题转化为：
- **可观测**：观察 agent 失败
- **可度量**：基线 vs 改进
- **可迭代**：REFACTOR 循环
- **可验证**：压力测试 + 元测试

### 5.2 "反 LLM 钻空子"的具体化

把抽象的"agent 会偷懒"转化为 5 个具体技术、8 条借口、4 类技能 × 3 测试方法。这种具体化是 superpowers 整套体系的标志性价值。

### 5.3 CSO 的可发现性优先

> "Critical for discovery: Future Claude needs to FIND your skill"

- description = 触发条件，**不**总结工作流（带反例证据）
- 关键词覆盖：errors、symptoms、synonyms、tools
- 命名：active voice、gerund form、what-you-DO

这种"为 agent 优化而非为人类优化"的思路是 LLM 时代的文档新范式。

### 5.4 铁律的绝对化

```
NO SKILL WITHOUT A FAILING TEST FIRST
This applies to NEW skills AND EDITS to existing skills.
```

> "Not for 'simple additions', Not for 'just adding a section', Not for 'documentation updates'"

**价值**：消除"小修改可以例外"的合理化。TDD 文化最怕的就是"我这次破例"。

### 5.5 Token 经济性目标

明确的字数目标（150/200/500 字）让"简洁"从抽象原则变为可验证的指标。

### 5.6 STOP 机制

> "After writing ANY skill, you MUST STOP and complete the deployment process."

防止"批量创建"的反模式。明确告诉 agent：**批处理不是效率，是偷工减料**。

### 5.7 元测试（meta-test）方法论

```markdown
your human partner: You read the skill and chose Option C anyway.
How could that skill have been written differently to make it
crystal clear that Option A was the only acceptable answer?
```

3 种响应类型（清晰但忽略 / 文档问题 / 组织问题）—— 把"agent 违规"转化为"技能本身的可改进信号"。这是持续改进的飞轮。

### 5.8 跨子代理协作哲学的体现

> "Use @ 禁用"反映对 LLM 文件系统模型的理解——LLM 不会"已经加载所有内容"，按需读取才是正确范式。

---

## 6. 缺点 / 风险 / 局限

### 6.1 压力场景的方法论不严密

技能推荐"3+ 组合压力"但：
- 没有给出如何**生成**有效压力场景的方法
- 容易陷入"造一个压力 = 自圆其说"——无法证伪
- 哪些是"有效压力"？清单给出 7 种但没说"哪种压力对应哪种违规"

**风险：** agent 可能"伪压力"——写一个不真正让 agent 想去违规的场景，然后写一个空技能通过"压力测试"。

### 6.2 缺乏量化的"技能质量"指标

成功标准都是定性的：
- "Agent follows rule under maximum pressure"
- "Agent correctly identifies when/how to apply pattern"
- "Agent finds and correctly applies reference information"

没有：
- 通过率目标（如 ≥ 90%）
- 压力场景数下限（"3+" 太模糊）
- 元测试通过的具体标准

### 6.3 CSO 的 description 约束可能反噬

> "NEVER summarize the skill's process or workflow"

**风险：** 极度压缩 description 后，agent 可能因为 description 信息不足而**不触发**该技能。CSO 的"丰富描述"和"不总结工作流"之间存在张力。

**例证：** 文档中的"❌ BAD: Use for TDD - write test first..."表明 description 不能含过程——但 description 还要"含具体触发器/症状"，这两者如何平衡？文档没有给出权重。

### 6.4 Token 效率 vs 完整性 的矛盾

> "frequently-loaded skills: <200 words total"

但 SKILL.md 本身有 656 行 ≈ 数千字。这是合理的（因为 writing-skills 不是频繁加载）但**会让 agent 担心"我写的技能太长"**，导致过度压缩、技能内容不完整。

**缺少：** "如何判断一个技能是否需要拆分"的决策规则。

### 6.5 "停止在下一个技能之前"的硬性约束可能产生副作用

> "Do NOT: Create multiple skills in batch without testing each"

**风险：** 这会让真正需要批量创建技能的工作流变得低效（例如为新项目初始化 5 个相关技能）。文档没有区分"独立技能"和"相互依赖的技能套件"。

### 6.6 缺少"技能退役/合并"机制

只有"创建"和"编辑"，没有：
- 何时退役技能？
- 何时合并两个相似技能？
- 技能版本管理？

**风险：** superpowers 这种生态系统的技能数量会指数增长，缺少生命周期管理会导致"技能腐化"。

### 6.7 缺乏"可调试性"

技能创建失败时：
- 是 RED 阶段没做好？
- 是 GREEN 阶段写错了？
- 是 REFACTOR 循环没走完？

文档没有"调试技能创建"的方法——当 agent 写出"无效技能"时怎么办？

### 6.8 "理性化表"的过拟合风险

> "Every excuse agents make goes in the table"

**风险：** 表会无限膨胀，最终变成"列出所有可能的借口"——这会让技能变得防御性过强、可读性下降。文档没有给出"什么时候停止扩展表"的标准。

### 6.9 对"agent 模型"的依赖

整个 CSO 模型假设 agent 的 description 触发是稳定的。但不同模型（Haiku/Sonnet/Opus）行为不同。技能中提到"Test with all models you plan to use"——但**没有给出测试方法**。

### 6.10 与 anthropic-best-practices.md 的关系微妙

文档明确说"For Anthropic's official skill authoring best practices, see anthropic-best-practices.md. This document provides additional patterns and guidelines that complement the TDD-focused approach in this skill."

但两者在某些方面**有冲突**：
- anthropic-best-practices 强调"Concise is key"（Token budget 500 行）
- writing-skills 强调"丰富 description + 关键词覆盖 + 跨引用"
- anthropic-best-practices 强调"Freedom to Claude"（高低自由度）
- writing-skills 强调"close loopholes, no exceptions"（低自由度）

**风险：** 用户可能不知道该听谁的。文档没有给"什么时候遵循哪个"的优先级。

---

## 7. 关键洞察

### 7.1 洞察 1：技能 = 文档作为可执行接口

`writing-skills` 的核心洞察：**LLM 时代的"文档"不是给人看的，是给 agent 加载的接口**。这意味着文档的优化目标从"易读"变成"易加载+易触发+易遵从"。

CSO、token 效率、description 优化——这些都源于这一重新定义。

### 7.2 洞察 2：抗合理化是纪律类技能的核心

TDD、verification、systematic-debugging 都是"纪律类技能"。它们的失败不是"agent 不会"，而是"agent 不想"。

`writing-skills` 提供 5 个具体技术（关闭漏洞、精神 vs 文字、合理化表、红旗列表、CSO 描述）来对抗这种"不想"。这本质上是**把说服心理学武器化用于防 LLM 钻空子**。

### 7.3 洞察 3：停止 = 元纪律

> "After writing ANY skill, you MUST STOP and complete the deployment process."

这条**不是为了该技能本身**，而是为了**防止"批量 = 未测试"** 的合理化。它是元层面的纪律——关于"如何应用纪律"的纪律。

### 7.4 洞察 4：元测试是飞轮

> "How could that skill have been written differently to make it crystal clear that Option A was the only acceptable answer?"

3 种响应类型（清晰但忽略 / 文档问题 / 组织问题）把违规转化为改进信号。**这是 PDCA 循环的 LLM 版本**。

### 7.5 洞察 5：@-引用的禁用反映对 LLM 文件系统模型的理解

> "@ syntax force-loads files immediately, consuming 200k+ context before you need them"

这反映对 Claude 等 LLM 的**实际工作机制**的深入理解：按需加载优于预加载。

### 7.6 洞察 6：技能类型的"测试方法"映射是 LLM 文档学的元方法

| 类型 | 测试方法 | 失败模式 |
|------|----------|----------|
| 纪律 | 压力 + 学术 | 钻空子 |
| 技术 | 应用 + 变化 | 不会用 |
| 模式 | 识别 + 反例 | 不会识别 |
| 参考 | 检索 + 缺口 | 找不到 |

**这个映射表是文档科学的方法论**——它把"测试"重新定义为"针对失败模式的针对性检查"。

### 7.7 洞察 7：SKILL.md 本身是该技能的"活的例子"

`<Bad>` vs `<Good>` 对比、合理化表、红旗列表——SKILL.md 示范了它自己教授的内容。**形式即内容**。

### 7.8 洞察 8：CSO 体现"为机器优化"的文档哲学

传统的"为人写的文档"关注可读性、教育性。`writing-skills` 关注可发现性、可触发性、可遵从性——这是**为 LLM 优化的文档学**。

---

## 8. 关联文档

### 8.1 内部关联（同目录）

| 文件 | 关系 |
|------|------|
| `anthropic-best-practices.md` | 官方补充——本技能明确说"complement the TDD-focused approach in this skill" |
| `persuasion-principles.md` | 心理学基础——7 大原则（Cialdini 2021）+ LLM 实验（Meincke 2025） |
| `testing-skills-with-subagents.md` | 技能测试的方法论——RED-GREEN-REFACTOR 在技能领域的具体操作 |
| `graphviz-conventions.dot` | Graphviz 风格规则——用于流程图 |
| `render-graphs.js` | 流程图渲染工具——把 dot 转 SVG |
| `examples/CLAUDE_MD_TESTING.md` | 完整测试用例——CLAUDE.md 文档变体的测试活动 |

### 8.2 外部关联（其他技能）

| 技能 | 关系 |
|------|------|
| `test-driven-development` | **必需背景**——TDD 基础被复用为技能创建基础 |
| `using-superpowers` | 元技能——如何发现和使用技能 |
| `systematic-debugging` | 反合理化模式——"3+ 修复失败 = 质疑架构" 的同款哲学 |
| `verification-before-completion` | 纪律类技能——与 TDD 共享"无测试不前进" |
| `subagent-driven-development` | 测试基础设施——用子代理运行压力场景 |

### 8.3 上游文档

| 来源 | 关系 |
|------|------|
| `persuasion-principles.md` | Cialdini (2021) + Meincke et al. (2025) 实验数据 |
| TDD 文献（Kent Beck） | RED-GREEN-REFACTOR 原始概念 |
| Anthropic Agent Skills 文档 | CSO 规范来源（agentskills.io/specification） |

---

## 9. 状态评价

### 9.1 完成度：★★★★★（5/5）

- 概念框架完整（CSO、TDD 映射、4 类型、抗合理化）
- 操作清单完整（4 阶段 × 多子项）
- 反模式列表完整
- 自示范完整（文档本身是活的例子）

### 9.2 一致性：★★★★★（5/5）

- 内部自洽：CSO、TDD、anti-rationalization 互相强化
- 形式即内容：文档展示了它教授的技能
- 与配套文件无矛盾

### 9.3 可操作性：★★★★（4/5）

- 清单明确
- 模板齐全
- **扣分点**：压力场景生成方法、量化指标、技能退役机制缺失

### 9.4 创新度：★★★★★（5/5）

- "TDD for documentation" 是范式创新
- CSO 是 LLM 时代文档学的新概念
- 元测试（meta-test）的 3 响应分类
- 抗合理化的 5 技术武器化

### 9.5 整体评价：**9.0 / 10**

这是一份**元技能层级的杰作**。它不仅教"如何写技能"，更重新定义了"如何让文档成为可执行接口"。**形式即内容、自示范、自指涉**——是 superpowers 整套体系的"宪法级别"技能。

---

## 10. 给读者的启示

### 10.1 对技能作者

1. **永远先 RED**：在写技能之前先观察 agent 失败
2. **CSO 是首要问题**：写不写得出来比写得好不好更重要
3. **抗合理化是纪律类技能的核心**：TDD/verification 类技能必须 5 个技术都用上
4. **STOP 关卡**：不要批量创建，逐个走完 RED-GREEN-REFACTOR
5. **元测试**：agent 违规时问"我该怎么改进技能本身"

### 10.2 对 AI 工程师

1. **LLM 时代文档学的根本转变**：从"给人读"到"给 agent 加载"
2. **CSO 是新 SEO**：description、关键词、命名 = LLM 发现机制
3. **token 经济学**：每个加载的技能都是成本，需要量化目标
4. **@-引用的危险**：强制加载违反按需读取范式
5. **形式即内容**：文档本身应该是"活样板"

### 10.3 对 superpowers 用户

1. **不要批量写技能**：违反 RED-GREEN-REFACTOR
2. **跨引用用名称而非 @**：保护上下文
3. **违反 symptoms 加入 description**：让 LLM 在"想违规"时触发技能
4. **测试场景 = 压力**：单一压力不够，要 3+ 组合
5. **元测试是改进飞轮**：agent 违规时转化为技能改进信号

### 10.4 对更广泛的提示工程领域

`writing-skills` 提出的"文档作为可执行接口"概念可推广到：
- 系统提示（system prompt）的设计
- 工具描述（tool description）的设计
- API 文档的"agent-first"重写
- 智能体（agent）的"自我介绍"设计

---

## 11. 后续工作 / 改进方向

### 11.1 应补充的内容

1. **压力场景的生成方法**：如何产生"有效"压力？给模板
2. **量化的技能质量指标**：通过率、压力场景数、meta-test 通过标准
3. **技能生命周期管理**：退役、合并、版本化
4. **CSO 冲突解决**：description 丰富度 vs 不总结工作流——给权重规则
5. **技能可调试性**：技能失败时如何诊断 RED/GREEN/REFACTOR 哪个阶段出问题

### 11.2 应展开的讨论

1. **writing-skills 与 anthropic-best-practices 的优先级**：何时遵循哪个？
2. **token 效率 vs 完整性的具体平衡点**：每个类型的目标字数应不同吗？
3. **"相互依赖的技能套件"**：何时允许批量创建？
4. **"合理化表"的边界**：何时停止扩展？

### 11.3 元层面

这个技能本身也应该是"测试驱动"的产物。建议在 production 使用一段时间后：
- 收集"agent 写技能时仍会犯的错误"作为新的合理化条目
- 收集"用户觉得含糊不清的章节"作为 CSO 改进信号
- 收集"被频繁引用的章节"vs"被跳过的章节"作为结构优化信号

### 11.4 哲学层面

最大的未解决问题是：**"agent-first 文档"会如何改变软件开发文化**？当文档从"给人读"变成"给 agent 加载"，文档的设计目标、可读性、教育价值会发生什么变化？这是值得长期跟踪的开放问题。

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
