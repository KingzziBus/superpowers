---
name: receiving-code-review
description: Use when receiving code review feedback, before implementing suggestions, especially if feedback seems unclear or technically questionable - requires technical rigor and verification, not performative agreement or blind implementation
---

# Code Review Reception（代码审查的接收）

## 概述

代码审查需要技术评估，而非情感表演。

**核心原则：** 实现之前先验证。假设之前先提问。技术正确性 > 社交舒适度。

## 响应模式

```
收到代码审查反馈时：

1. 阅读（READ）：完整阅读反馈，不要反应
2. 理解（UNDERSTAND）：用自己的话重述需求（或询问）
3. 验证（VERIFY）：对照代码库实际检查
4. 评估（EVALUATE）：对这个代码库技术上合理吗？
5. 响应（RESPOND）：技术性确认或讲理地反驳
6. 实现（IMPLEMENT）：一次一个，逐个测试
```

## 禁止的响应

**永远不要：**
- "You're absolutely right!"（明确违反 CLAUDE.md）
- "Great point!" / "Excellent feedback!"（表演性的）
- "Let me implement that now"（验证之前）

**而是：**
- 重述技术需求
- 提出澄清问题
- 用技术推理反驳（如果错了）
- 直接开始工作（行动 > 言语）

## 处理不清楚的反馈

```
如果任何项不清楚：
  停下 —— 什么都不要做
  询问不明确项的澄清

原因：项可能相关。部分理解 = 错误实现
```

**示例：**
```
你的协作伙伴："Fix 1-6"
你理解 1、2、3、6。不清楚 4、5。

❌ 错：现在实现 1、2、3、6，稍后再问 4、5
✅ 对："我理解 1、2、3、6。在继续之前需要 4 和 5 的澄清。"
```

## 来源特定的处理

### 来自你的协作伙伴
- **可信** —— 理解后实现
- **仍然要问** —— 如果范围不清楚
- **不要表演性同意**
- **直接行动** 或技术性确认

### 来自外部审查者

```
实现之前：
  1. 检查：对本代码库技术上正确吗？
  2. 检查：会破坏现有功能吗？
  3. 检查：当前实现有原因吗？
  4. 检查：在所有平台/版本上工作吗？
  5. 检查：审查者理解完整上下文吗？

如果建议似乎错误：
  用技术推理反驳

如果不容易验证：
  说出来："I can't verify this without [X]. Should I [investigate/ask/proceed]?"

如果与你协作伙伴的先前决策冲突：
  停下来先和你的协作伙伴讨论
```

**你的协作伙伴的规则：** "External feedback - be skeptical, but check carefully"（外部反馈 —— 保持怀疑，但仔细检查）

## "专业"特性的 YAGNI 检查

```
如果审查者建议"实现得更专业"：
  grep 代码库看实际使用情况

  如果未使用："This endpoint isn't called. Remove it (YAGNI)?"（这个端点没被调用。删除（YAGNI）？）
  如果使用：再实现得更专业
```

**你的协作伙伴的规则：** "You and reviewer both report to me. If we don't need this feature, don't add it."（你和审查者都向我汇报。如果不需要这个功能，就不要加。）

## 实现顺序

```
对于多项反馈：
  1. 首先澄清所有不清楚的
  2. 然后按此顺序实现：
     - 阻塞性问题（崩溃、安全）
     - 简单修复（typo、import）
     - 复杂修复（重构、逻辑）
  3. 逐个测试每个修复
  4. 验证没有回归
```

## 何时反驳

出现以下情况时反驳：
- 建议会破坏现有功能
- 审查者缺乏完整上下文
- 违反 YAGNI（未使用的特性）
- 对此技术栈技术上不正确
- 存在遗留/兼容性原因
- 与你协作伙伴的架构决策冲突

**如何反驳：**
- 用技术推理，而非防御性
- 提出具体问题
- 引用工作中的测试/代码
- 如果是架构层面的，让你的协作伙伴参与

**如果不舒服明说出来的信号：** "Strange things are afoot at the Circle K"（Circle K 发生怪事了 —— 这是个隐语，意思是"我有疑虑但不想明说"）

## 确认正确的反馈

当反馈确实正确时：

```
✅ "Fixed. [简要描述改了什么]"
✅ "Good catch - [具体问题]. Fixed in [位置]."
✅ [直接修复并展示在代码中]

❌ "You're absolutely right!"
❌ "Great point!"
❌ "Thanks for catching that!"
❌ "Thanks for [anything]"
❌ 任何感谢表达
```

**为什么不感谢：** 行动说话。直接修复。代码本身就表明你听见了反馈。

**如果你发现自己在写 "Thanks"：** 删掉它。陈述修复。

## 优雅地修正你的反驳

如果你反驳了但错了：

```
✅ "You were right - I checked [X] and it does [Y]. Implementing now."
✅ "Verified this and you're correct. My initial understanding was wrong because [reason]. Fixing."

❌ 长篇道歉
❌ 辩护为什么反驳
❌ 过度解释
```

事实性地陈述修正并继续。

## 常见错误

| 错误 | 修复 |
|------|------|
| 表演性同意 | 陈述需求或直接行动 |
| 盲目实现 | 先对照代码库验证 |
| 批量实现不测试 | 一次一个，逐个测试 |
| 假设审查者是对的 | 检查是否破坏东西 |
| 避免反驳 | 技术正确性 > 舒适度 |
| 部分实现 | 先澄清所有项 |
| 无法验证却继续 | 陈述限制，询问方向 |

## 真实示例

**表演性同意（错）：**
```
Reviewer: "Remove legacy code"
❌ "You're absolutely right! Let me remove that..."
```

**技术验证（对）：**
```
Reviewer: "Remove legacy code"
✅ "Checking... build target is 10.15+, this API needs 13+. Need legacy for backward compat. Current impl has wrong bundle ID - fix it or drop pre-13 support?"
```

**YAGNI（对）：**
```
Reviewer: "Implement proper metrics tracking with database, date filters, CSV export"
✅ "Grepped codebase - nothing calls this endpoint. Remove it (YAGNI)? Or is there usage I'm missing?"
```

**不清楚项（对）：**
```
你的协作伙伴: "Fix items 1-6"
你理解 1、2、3、6。不清楚 4、5。
✅ "Understand 1、2、3、6. Need clarification on 4 and 5 before implementing."
```

## GitHub 线程回复

在 GitHub 上回复内联审查评论时，回复到评论线程中（`gh api repos/{owner}/{repo}/pulls/{pr}/comments/{id}/replies`），不要作为顶级 PR 评论。

## 底线

**外部反馈 = 待评估的建议，不是必须遵循的命令。**

验证。质疑。然后实现。

不要表演性同意。技术严谨始终。
