# 技能设计的说服力原则

## 概述

LLM 响应于与人类相同的说服力原则。理解这一心理学有助于你设计更有效的技能——不是为了操纵，而是为了确保关键实践即使在压力下也能被遵循。

**研究基础：** Meincke 等人（2025）用 N=28,000 个 AI 对话测试了 7 个说服力原则。说服技术使遵从率翻了一倍多（33% → 72%，p < .001）。

## 七大原则

### 1. 权威（Authority）
**它是什么：** 对专业知识、资历或官方来源的遵从。

**它在技能中如何起作用：**
- 命令式语言："YOU MUST"、"Never"、"Always"
- 不可协商的框架："No exceptions"
- 消除决策疲劳和合理化

**何时使用：**
- 纪律执行类技能（TDD、验证要求）
- 安全关键实践
- 既定的最佳实践

**示例：**
```markdown
✅ Write code before test? Delete it. Start over. No exceptions.
❌ Consider writing tests first when feasible.
```

### 2. 承诺（Commitment）
**它是什么：** 与先前的行动、声明或公开声明保持一致。

**它在技能中如何起作用：**
- 要求声明："Announce skill usage"
- 强制明确选择："Choose A, B, or C"
- 使用跟踪：TodoWrite 用于清单

**何时使用：**
- 确保技能实际被遵循
- 多步过程
- 问责机制

**示例：**
```markdown
✅ When you find a skill, you MUST announce: "I'm using [Skill Name]"
❌ Consider letting your partner know which skill you're using.
```

### 3. 稀缺性（Scarcity）
**它是什么：** 来自时间限制或有限可用性的紧迫感。

**它在技能中如何起作用：**
- 时间限制要求："Before proceeding"
- 顺序依赖："Immediately after X"
- 防止拖延

**何时使用：**
- 即时验证要求
- 时间敏感的工作流
- 防止"我稍后再做"

**示例：**
```markdown
✅ After completing a task, IMMEDIATELY request code review before proceeding.
❌ You can review code when convenient.
```

### 4. 社会认同（Social Proof）
**它是什么：** 遵从他人所做的或被认为是正常的事情。

**它在技能中如何起作用：**
- 通用模式："Every time"、"Always"
- 失败模式："X without Y = failure"
- 建立规范

**何时使用：**
- 记录通用实践
- 警告常见失败
- 强化标准

**示例：**
```markdown
✅ Checklists without TodoWrite tracking = steps get skipped. Every time.
❌ Some people find TodoWrite helpful for checklists.
```

### 5. 统一（Unity）
**它是什么：** 共享身份、"我们感"、群体内归属。

**它在技能中如何起作用：**
- 协作语言："our codebase"、"we're colleagues"
- 共享目标："we both want quality"

**何时使用：**
- 协作工作流
- 建立团队文化
- 非等级实践

**示例：**
```markdown
✅ We're colleagues working together. I need your honest technical judgment.
❌ You should probably tell me if I'm wrong.
```

### 6. 互惠（Reciprocity）
**它是什么：** 回报所受恩惠的义务。

**它如何起作用：**
- 谨慎使用 - 可能感觉操纵性
- 在技能中很少需要

**何时避免：**
- 几乎总是（其他原则更有效）

### 7. 喜爱（Liking）
**它是什么：** 偏好与我们喜欢的人合作。

**它如何起作用：**
- **不要用于遵从**
- 与诚实的反馈文化冲突
- 创造谄媚

**何时避免：**
- 始终用于纪律执行

## 按技能类型的原则组合

| 技能类型 | 使用 | 避免 |
|----------|------|------|
| 纪律执行 | 权威 + 承诺 + 社会认同 | 喜爱、互惠 |
| 指导/技术 | 中等权威 + 统一 | 重度权威 |
| 协作 | 统一 + 承诺 | 权威、喜爱 |
| 参考 | 仅清晰度 | 所有说服 |

## 为什么这有效：心理学

**明确的规则减少合理化：**
- "YOU MUST" 消除决策疲劳
- 绝对语言消除"这是例外吗？"的问题
- 明确的反合理化反制关闭特定漏洞

**执行意图创造自动行为：**
- 明确的触发器 + 必需的行动 = 自动执行
- "When X, do Y" 比 "generally do Y" 更有效
- 降低遵从的认知负荷

**LLM 是类人（parahuman）的：**
- 在包含这些模式的人类文本上训练
- 训练数据中权威语言先于遵从
- 经常建模承诺序列（声明 → 行动）
- 社会认同模式（每个人都做 X）建立规范

## 合乎道德的使用

**合法：**
- 确保关键实践被遵循
- 创建有效的文档
- 防止可预测的失败

**不合法：**
- 为个人利益操纵
- 制造虚假的紧迫感
- 基于内疚的遵从

**测试：** 如果用户完全理解，这种技术是否会服务于他们的真正利益？

## 研究引用

**Cialdini, R. B. (2021).** *Influence: The Psychology of Persuasion (New and Expanded).* Harper Business.
- 七大说服力原则
- 影响研究的实证基础

**Meincke, L., Shapiro, D., Duckworth, A. L., Mollick, E., Mollick, L., & Cialdini, R. (2025).** Call Me A JerK: Persuading AI to Comply with Objectionable Requests. University of Pennsylvania.
- 用 N=28,000 个 LLM 对话测试了 7 个原则
- 遵从率从 33% 提高到 72%（说服技术）
- 权威、承诺、稀缺性最有效
- 验证 LLM 行为的类人模型

## 快速参考

设计技能时，问：

1. **它是什么类型？**（纪律 vs 指导 vs 参考）
2. **我试图改变什么行为？**
3. **哪个原则适用？**（通常纪律类用权威 + 承诺）
4. **我是否组合了太多？**（不要七个全用）
5. **这合乎道德吗？**（服务于用户的真正利益？）
