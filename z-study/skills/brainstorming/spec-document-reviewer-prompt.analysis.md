# brainstorming/spec-document-reviewer-prompt.md 深度分析报告

> 本报告分析 superpowers 项目的规范文档审查者 prompt 模板。它是 `brainstorming/SKILL.md` 中"派遣规范审查者"步骤的具体实施工具。

---

## 1. 基本信息

| 项 | 值 |
|----|----|
| **文件名** | `spec-document-reviewer-prompt.md` |
| **总行数** | 50 行 |
| **文档定位** | 规范审查者子代理的 prompt 模板 |
| **派遣时机** | 规范文档已写入 docs/superpowers/specs/ 之后 |
| **核心功能** | 派遣 general-purpose 任务，验证规范是否可实施 |
| **核心方法** | "Approve unless there are serious gaps" 校准原则 |
| **关联技能** | `brainstorming`（派遣来源）、`subagent-driven-development`（子代理模式） |

---

## 2. 目的与背景

### 2.1 文档的"桥梁"角色

本文档在 superpowers 知识体系中扮演**桥梁**——把 brainstorming 流程与子代理模式连接起来：

```
brainstorming 流程
    ↓ 步骤 8 "User reviews spec"
spec-document-reviewer-prompt.md ← 本文档（派遣子代理审查）
    ↓
规范进入下一阶段
```

它是"用户审查"之前的**自动化质量门控**。

### 2.2 三大核心命题

1. **"Approve unless there are serious gaps that would lead to a flawed plan"** —— 校准原则
2. **"Only flag issues that would cause real problems during implementation planning"** —— 严格筛选
3. **"Issues block approval, Recommendations are advisory"** —— 二元裁决

### 2.3 解决的具体问题

| 问题 | 本文档提供的方案 |
|------|------------------|
| 不知道审查什么 | 5 维度检查表（Completeness/Consistency/Clarity/Scope/YAGNI） |
| 不知道如何校准 | "Only flag real problems" 标准 |
| 不知道如何输出 | 4 字段结构（Status/Issues/Recommendations） |

---

## 3. 核心技术决策

### 3.1 决策 1：5 维度检查表

| Category | What to Look For |
|----------|------------------|
| **Completeness** | TODOs, placeholders, "TBD", incomplete sections |
| **Consistency** | Internal contradictions, conflicting requirements |
| **Clarity** | Requirements ambiguous enough to cause someone to build the wrong thing |
| **Scope** | Focused enough for a single plan — not covering multiple independent subsystems |
| **YAGNI** | Unrequested features, over-engineering |

**这一设计的精妙：**
- 5 维度覆盖"实施规划前的关键质量属性"
- **YAGNI** 是 superpowers 的核心原则——防止过度设计
- **Scope** 与 brainstorming 的"范围评估"对齐

**这 5 维度是软件工程传统的精炼：**
- Completeness = 完整性
- Consistency = 一致性
- Clarity = 清晰性
- Scope = 范围
- YAGNI = 简洁性

### 3.2 决策 2：校准原则（Calibration）

> **"Only flag issues that would cause real problems during implementation planning."**

**核心二分：**
- ✅ "Missing section, contradiction, requirement ambiguous enough to interpret two ways" = **issue**
- ❌ "Minor wording improvements, stylistic preferences, sections less detailed than others" = **not issue**

**这一校准的价值：**
- 防止"完美主义"审查——不阻止微小的措辞改进
- 关注"实施影响"——只标记会真正导致问题的
- "Approve unless there are serious gaps that would lead to a flawed plan" —— **倾向于通过**

**与 brainstorming/SKILL.md 的"自审"哲学一致：**
- 自审：内联修复
- 文档审查：派遣子代理，**倾向于通过**

### 3.3 决策 3：Issues vs Recommendations 二元裁决

```
**Issues (if any):**
- [Section X]: [specific issue] - [why it matters for planning]

**Recommendations (advisory, do not block approval):**
- [suggestions for improvement]
```

**关键区分：**
- **Issues**：阻止批准的**严重问题**
- **Recommendations**：不阻止批准的**建议**

**这种区分的价值：**
- 防止"建议 = 阻止"的误用
- 区分"必须修改" vs "可以改进"
- 给 spec 作者清晰的"哪些必须改、哪些可选"

### 3.4 决策 4：Status 二元裁决

> **"Status:** Approved | Issues Found"

**两值设计：**
- **Approved** = 通过，可进入下一阶段
- **Issues Found** = 需修改，重新审查

**没有"Approved with Concerns"或"Conditional"等中间态**——这是 superpowers 的"二元裁决"风格（与 code-reviewer.md 的 Yes/No/With fixes 不同）。

### 3.5 决策 5：派遣为 general-purpose 任务

```
Task tool (general-purpose):
  description: "Review spec document"
  prompt: |
    ...
```

**这一设计的洞察：**
- 派遣"通用"任务而非"专门审查者"
- 描述简洁："Review spec document"
- prompt 是**完整可执行的指令**

**与 subagent-driven-development 的子代理模式一致**——审查是子代理任务之一。

### 3.6 决策 6：四字段输出格式

```
## Spec Review

**Status:** Approved | Issues Found

**Issues (if any):**
- [Section X]: [specific issue] - [why it matters for planning]

**Recommendations (advisory, do not block approval):**
- [suggestions for improvement]
```

**这 4 字段的设计：**
- **Status** = 决策
- **Issues** = 必须修复
- **Recommendations** = 可选改进
- **空字段表示无**（无需写"None"）

**价值：**
- 结构化输出便于解析
- 决策清晰
- "Advisory" 标签明确权重

### 3.7 决策 7：Issue 描述的三段式

> "[Section X]: [specific issue] - [why it matters for planning]"

**三段式结构：**
- **位置**：哪个章节
- **问题**：具体什么
- **影响**：为什么对规划重要

**这与 code-reviewer.md 的 4 字段（File:line / What's wrong / Why it matters / How to fix）几乎一致**，但少 "How to fix"——这是因为规范审查的"修复"在 spec 本身（作者决定如何修改）。

### 3.8 决策 8：与 brainstorming 自审的"双层门控"

| 层次 | 工具 | 目标 |
|------|------|------|
| 自审 | 内联（agent 自己） | 快速发现 |
| 文档审查 | 派遣子代理 | 上下文隔离的二次检查 |

**双层门控的价值：**
- 自审发现不了"作者的盲点"
- 子代理审查有**独立上下文**——不继承作者的偏见
- 与 subagent-driven-development 的"上下文隔离"哲学一致

---

## 4. 文件结构剖析

```
spec-document-reviewer-prompt.md (50 行)
├── 标题 + 概述 (5 行)
├── 派遣时机 (1 行)
├── Prompt 主体 (43 行)
│   ├── 角色定义 (1 行)
│   ├── Spec 路径占位符 (1 行)
│   ├── What to Check (8 行) - 5 维度表
│   ├── Calibration (7 行) - 严格筛选
│   ├── Output Format (12 行) - 4 字段
│   └── 内部格式 (4 行)
└── 审查者返回 (1 行)
```

**结构特点：**
1. **简洁**——50 行就定义了完整的派遣规范
2. **5 维度表**是核心方法论
3. **Calibration 段**是哲学宣言
4. **Output Format 段**是结构化输出模板

---

## 5. 优点 / 亮点

### 5.1 校准原则的"倾向于通过"

> "Approve unless there are serious gaps that would lead to a flawed plan"

**价值：**
- 防止"完美主义"审查
- 默认信任 spec 作者
- 只阻止**真正会出问题**的缺陷

这与 code-reviewer.md 的 "Approve unless there are serious gaps" 一致——superpowers 整体哲学。

### 5.2 5 维度检查的精炼

5 维度（Completeness/Consistency/Clarity/Scope/YAGNI）覆盖了"实施前必须验证"的关键质量属性。

**与软件工程传统的对应：**
- Completeness = 完整性
- Consistency = 一致性
- Clarity = 清晰性
- Scope = 范围聚焦
- YAGNI = 简洁性

### 5.3 Issues vs Recommendations 的二元区分

**关键洞察：**
- "Issues" 阻止批准
- "Recommendations" 不阻止

这种区分防止了"建议 = 阻止"的滥用——给 spec 作者清晰的修改优先级。

### 5.4 上下文隔离的子代理模式

派遣子代理做审查，**不继承作者偏见**。这是 subagent-driven-development 的核心优势。

### 5.5 简洁的 50 行

不需要冗长的说明——50 行定义了完整的派遣规范。**形式即内容**——它本身就是"清晰文档"的范例。

### 5.6 Issue 描述的三段式

> "[Section X]: [specific issue] - [why it matters for planning]"

**位置 + 问题 + 影响**——给出"问题在哪、是什么、为什么重要"。

### 5.7 Status 二元裁决

只有 "Approved" 或 "Issues Found"——**没有中间态**。这种二元设计减少歧义。

### 5.8 与 brainstorming 流程的无缝衔接

派遣时机明确："Spec document is written to docs/superpowers/specs/"——与 brainstorming 步骤 7（自审）和步骤 8（用户审查）形成完整流程。

### 5.9 与 superpowers 整体哲学的一致性

- 与 code-reviewer.md 共享校准原则
- 与 subagent-driven-development 共享子代理模式
- 与 brainstorming 共享"设计先于实现"

### 5.10 YAGNI 维度的精妙

把 YAGNI 作为独立维度——**防止审查者"宽容过度设计"**。

---

## 6. 缺点 / 风险 / 局限

### 6.1 5 维度检查的"完整性"问题

**问题：**
- 5 维度是否覆盖了所有重要检查？
- 缺少"测试性"（Testability）维度
- 缺少"可观测性"（Observability）维度
- 缺少"演进性"（Evolution）维度

**风险：** 重要的质量问题可能被忽略。

### 6.2 "倾向于通过" 的潜在风险

> "Approve unless there are serious gaps"

**风险：**
- 审查者可能"过于宽松"
- "Serious gaps" 的标准模糊
- 可能在某些场景下应更严格

**改进空间：** 提供"严格 vs 宽松"的选择。

### 6.3 Issues 描述缺少 "How to fix"

**问题：**
- code-reviewer.md 有 "How to fix" 字段
- spec-document-reviewer 没有
- spec 作者可能不知道如何修复

**改进空间：** 至少建议 "How to fix" 字段。

### 6.4 派遣时机的"必须" vs "可选"

> "Dispatch after: Spec document is written to docs/superpowers/specs/"

**问题：**
- "after" 是顺序还是强制？
- 文档没说"是否必须派遣"
- 缺少"派遣 vs 不派遣"的标准

### 6.5 与 brainstorming 自审的"重复"问题

**问题：**
- brainstorming 步骤 7 是"自审"
- spec-document-reviewer 是"文档审查"
- 两者可能重复劳动

**改进空间：** 明确两者的差异（如"自审查占位符，文档审查查矛盾"）。

### 6.6 缺少"分级严重性"

**问题：**
- 所有 Issues 都是同一级别
- 没有 "Critical / Important / Minor"
- spec 作者不知道哪些 Issues 优先

**对比 code-reviewer.md**：
- code-reviewer 用 3 级（Critical/Important/Minor）
- spec-document-reviewer 只有 1 级（block approval）

### 6.7 推荐"不阻止批准"的副作用

> "Recommendations (advisory, do not block approval)"

**风险：**
- 审查者倾向于把"严重问题"降级为"建议"
- 减少"必须修改"的数量
- 可能让 spec 质量下降

### 6.8 缺少"复杂度"或"工作量"评估

**问题：**
- 文档没说"这个 spec 实现的复杂度"
- 缺少"实施风险"评估
- 缺少"工作量估算"

### 6.9 5 维度的"具体阈值"缺失

**问题：**
- "Focused enough for a single plan" 的标准是什么？
- "Internal contradictions" 如何量化？
- 缺少"何时算 issue" 的客观标准

### 6.10 文档本身的元审查缺失

**问题：**
- 谁审查这个 prompt 模板？
- 派遣规范本身的质量如何保证？
- 缺少"prompt 模板的有效性"评估

---

## 7. 关键洞察

### 7.1 洞察 1：校准原则是"反对完美主义"

> "Approve unless there are serious gaps"

不追求"完美规范"——追求"足够好"。这种**实用主义**与 superpowers 的"够好哲学"一致。

### 7.2 洞察 2：5 维度 = 4 传统 + 1 YAGNI

Completeness/Consistency/Clarity/Scope 是软件工程传统，**YAGNI 是 superpowers 的特殊贡献**——把"不必要功能"作为独立审查维度。

### 7.3 洞察 3：Issues vs Recommendations 是"阻止 vs 不阻止"

这是**影响力的二元化**——一个阻止批准，一个不阻止。这种区分让 spec 作者知道优先级。

### 7.4 洞察 4：上下文隔离的二次审查

派遣子代理做审查，**不继承作者偏见**。这是 superpowers 的"独立视角"模式。

### 7.5 洞察 5：简洁的 50 行

不需要冗长说明——50 行定义完整规范。**形式即内容**——文档本身是"清晰简洁"的范例。

### 7.6 洞察 6：与 code-reviewer.md 的微妙差异

| 字段 | code-reviewer | spec-document-reviewer |
|------|---------------|------------------------|
| Status | Yes/No/With fixes | Approved/Issues Found |
| Issue 级别 | Critical/Important/Minor | 单一级别（block approval） |
| "How to fix" | 4 字段之一 | 缺失 |

**为什么不同？** code 是可执行的（"how to fix"是必需的），spec 是设计文档（"how to fix"由 spec 作者决定）。

### 7.7 洞察 7：派遣的"一般性"

派遣为 general-purpose 任务——不创建专门"spec-reviewer"角色。**这是 superpowers 的"轻量级"哲学**——避免过度专业化。

### 7.8 洞察 8：与 brainstorming 自审的"双层"

| 层次 | 工具 | 上下文 |
|------|------|--------|
| 自审 | agent 自己 | 继承上下文 |
| 文档审查 | 子代理 | 独立上下文 |

**两道门**比"一道"更可靠。

### 7.9 洞察 9：YAGNI 作为独立维度

把 YAGNI 作为独立维度——**防止审查者"宽容过度设计"**。这种"显式列出反模式"是 superpowers 的特征。

### 7.10 洞察 10：倾向于通过 = 信任 + 校准

> "Approve unless there are serious gaps"

**不是"无脑通过"也不是"无脑拒绝"**——是"信任但校准"。这种平衡是质量审查的成熟态度。

---

## 8. 关联文档

### 8.1 内部关联（同目录）

| 文件 | 关系 |
|------|------|
| `SKILL.md` | 派遣来源——步骤 7 自审后派遣审查者 |
| `visual-companion.md` | 独立工具——本文档不涉及视觉化 |

### 8.2 外部关联（其他技能）

| 技能 | 关系 |
|------|------|
| `subagent-driven-development` | **核心模式**——子代理派遣 |
| `requesting-code-review` | 类似模式——code-reviewer.md 是代码审查的同款 |
| `using-superpowers` | 元技能——本文档是其实施细节 |
| `test-driven-development` | 共享哲学——审查前的"先失败后通过" |

### 8.3 上游文档

| 来源 | 关系 |
|------|------|
| 软件工程传统 | 5 维度检查是经典质量属性 |
| superpowers:requesting-code-review | 校准原则的同款哲学 |
| `persuasion-principles.md` | "Approve unless" 是温和权威 |

---

## 9. 状态评价

### 9.1 完成度：★★★★★（5/5）

- 5 维度检查完整
- 校准原则明确
- 输出格式结构化
- 派遣时机清晰

### 9.2 一致性：★★★★★（5/5）

- 与 code-reviewer.md 共享校准
- 与 subagent-driven-development 共享模式
- 与 brainstorming 共享"设计先于实现"

### 9.3 可操作性：★★★★★（5/5）

- 50 行直接可用
- 派遣语法完整
- 输出模板清晰

### 9.4 创新度：★★★★（4/5）

- YAGNI 作为独立维度
- Issues vs Recommendations 区分
- **扣分点**：5 维度本身是软件工程传统的精炼

### 9.5 整体评价：**9.0 / 10**

这是一份**简洁、实用、哲学正确**的规范审查者 prompt 模板。它把抽象的"派遣子代理审查"转化为 50 行的可执行规范。

**唯一扣分点：** 缺少 "How to fix"、分级严重性、具体阈值。

---

## 10. 给读者的启示

### 10.1 对技能作者

1. **校准原则**是质量审查的成熟态度——"倾向于通过"但有底线
2. **5 维度**是软件工程传统的精炼——不发明新概念
3. **YAGNI 独立维度**是 superpowers 的贡献——防止过度设计
4. **Issues vs Recommendations**区分"必须 vs 可选"
5. **简洁 50 行**是"形式即内容"的范例

### 10.2 对 AI 工程师

1. **派遣子代理**是"独立视角"的关键
2. **校准原则**防止"完美主义"审查
3. **YAGNI 作为维度**——把"反模式"转化为"审查点"
4. **简洁模板**——50 行可以定义完整的派遣规范
5. **"Approve unless"**是质量审查的成熟哲学

### 10.3 对 superpowers 用户

1. **本文档**是 brainstorming 流程的子代理审查步骤
2. **5 维度**是派遣前的质量检查清单
3. **Issues vs Recommendations**——spec 作者知道哪些必须改
4. **Status 二元裁决**——清晰的下一步指示
5. **与 code-reviewer.md 对照**——理解代码审查和规范审查的异同

### 10.4 对更广泛的提示工程

1. **子代理派遣**是 LLM 时代的新模式
2. **校准原则**是质量审查的通用智慧
3. **YAGNI 显式化**是反过度设计
4. **简洁模板**——复杂概念可以用 50 行表达
5. **"Approve unless"** 是质量与效率的平衡

---

## 11. 后续工作 / 改进方向

### 11.1 应补充的内容

1. **"How to fix" 字段**——给 spec 作者具体的修复建议
2. **分级严重性**——Critical/Important/Minor
3. **具体阈值**——"Focused enough" 的量化标准
4. **测试性、演进性维度**——补充 5 维度
5. **元审查**——prompt 模板本身的有效性评估

### 11.2 应展开的讨论

1. **派遣的"必须" vs "可选"**——什么时候必须派遣审查者
2. **"倾向于通过"的校准**——不同场景下的校准调整
3. **与自审的边界**——什么时候用自审 vs 派遣
4. **审查成本 vs 价值**——何时跳过审查
5. **多轮审查的协调**——如果 spec 修改后再审查

### 11.3 元层面

本文档是 superpowers 子代理模式的"轻量级"应用。建议：
- 收集"Issues vs Recommendations"区分的反馈
- 跟踪"倾向于通过"是否导致质量问题
- 定期更新 5 维度清单

### 11.4 哲学层面

最大的开放问题是：**"倾向于通过"与"严格审查"如何平衡**？

当前立场：倾向于通过，但有问题阻止。
**问题：**
- 严格 vs 宽松的不同场景
- "可接受的 spec" 与 "完美的 spec" 的区别
- 审查成本的优化

**未来可能：**
- 多级校准（快速/标准/严格）
- 按项目类型调整（关键项目严格、实验项目宽松）
- 校准的可配置化

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

所有论断都基于 spec-document-reviewer-prompt.md 文本和其在 superpowers 生态中的角色。**不做超出文档声明的推论**。
