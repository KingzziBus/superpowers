# systematic-debugging/SKILL.md 分析报告

## 一、基本信息

- **文件路径**：`skills/systematic-debugging/SKILL.md`
- **文件大小**：297 行
- **文档类型**：技能主文档（带 YAML frontmatter）
- **配套文件**：
  - `CREATION-LOG.md`（创建日志）
  - `condition-based-waiting.md`（条件等待）
  - `defense-in-depth.md`（多层防御）
  - `root-cause-tracing.md`（根因追踪）
  - `test-academic.md`、`test-pressure-1/2/3.md`（压力测试案例）
  - `find-polluter.sh`（查找污染者脚本）
  - `condition-based-waiting-example.ts`（条件等待示例代码）

## 二、目的

该技能是 Superpowers 项目的"调试纪律核心"——它定义严格的 4 阶段调试流程：

1. **Phase 1: 根因调查**（READ → REPRODUCE → CHECK → GATHER → TRACE）
2. **Phase 2: 模式分析**（找到工作中的示例 → 对比 → 识别差异 → 理解依赖）
3. **Phase 3: 假设与测试**（科学方法）
4. **Phase 4: 实施**（创建失败测试 → 修复 → 验证）

它把"调试"从"凭感觉猜测" 提升为"系统化流程"。

## 三、核心技术决策

### 3.1 "铁律" + "Phase 1 必做"

```
NO FIXES WITHOUT ROOT CAUSE INVESTIGATION FIRST
```

**这是最强约束**——它用全大写 + 命令式语气声明：

- 没有根因调查 → 不能提修复
- 这是调试的**绝对铁律**

> If you haven't completed Phase 1, you cannot propose fixes.

**这一句把"铁律"和"Phase 1"绑定**——LLM 不能"跳过调查直接给修复"。

### 3.2 4 阶段严格分离

```
Phase 1: 根因调查    → 理解 WHAT 和 WHY
Phase 2: 模式分析    → 识别差异
Phase 3: 假设与测试  → 确认或新假设
Phase 4: 实施        → 创建测试、修复、验证
```

**4 阶段严格分离**的设计哲学：

- **必须完成每阶段才能进入下一阶段**
- **不能跳过**（即使是"简单" 问题）
- **不能并行**（不能在 Phase 1 阶段同时实施）

这种**严格分离**是 LLM 协作中反"跳过" 失败模式的关键。

### 3.3 "Use this ESPECIALLY when" 反压力段

> **Use this ESPECIALLY when:**
> - Under time pressure (emergencies make guessing tempting)
> - "Just one quick fix" seems obvious
> - You've already tried multiple fixes
> - Previous fix didn't work
> - You don't fully understand the issue

**5 个"特别使用" 场景** 精准覆盖 LLM 在压力下的典型失败：

- 时间压力 → 倾向猜测
- "快速修复" 明显 → 倾向偷懒
- 多次尝试 → 倾向再加一次
- 上次修复失败 → 倾向不重蹈覆辙
- 不完全理解 → 倾向猜

**5 个场景** 显式说"在这些情况下更要遵循流程"。

### 3.4 "Don't skip when" 反"理由化" 段

> **Don't skip when:**
> - Issue seems simple (simple bugs have root causes too)
> - You're in a hurry (rushing guarantees rework)
> - Manager wants it fixed NOW (systematic is faster than thrashing)

**3 个"不要跳过"场景** 精准打击 LLM 的 3 种"跳过" 理由：

- 简单问题 → 简单也有根因
- 赶时间 → 仓促保证返工
- 经理要现在修 → 系统化比胡乱尝试更快

**最后一句的"systematic is faster than thrashing"** 是关键洞察——**系统化不是慢，是快**。

### 3.5 Phase 1 的 5 步调查法

```
1. 仔细阅读错误信息
2. 一致地复现
3. 检查最近的更改
4. 在多组件系统中收集证据
5. 追踪数据流
```

**5 步调查法** 是**根因调查的完整流程**：

- 步骤 1-3：**从错误开始**（症状） → 复现 → 关联变更
- 步骤 4-5：**深入系统**（多组件证据） → 数据流追踪

特别是**步骤 4** 的多组件证据收集（CI → build → signing, API → service → database）——这是大型分布式系统调试的关键。

### 3.6 "Gather Evidence in Multi-Component Systems" 4 层记录法

```
For EACH component boundary:
  - Log what data enters component
  - Log what data exits component
  - Verify environment/config propagation
  - Check state at each layer
```

**4 层记录法** 让 LLM 在多组件系统中**精确隔离失败点**：

- 数据进入 → 数据退出 → 环境/配置 → 状态

**4 层 + 3 步骤"先收集，再分析，再调查"** 是 LLM 协作中**反"盲目猜测"的关键**。

### 3.7 "Trace Data Flow" 反向追踪

> - Where does bad value originate?
> - What called this with bad value?
> - Keep tracing up until you find the source
> - **Fix at source, not at symptom**

**4 步反向追踪法**：

- 找起源
- 找调用方
- 持续向上
- **在源头修复**

**最后一句是核心**——**修复在源头，而非症状**。这是 LLM 调试中的"治本 vs 治标" 原则。

### 3.8 Phase 2 "Find Working Examples" 反"闭门造车"

> 1. **Find Working Examples**
>    - Locate similar working code in same codebase
>    - What works that's similar to what's broken?

**这一条精准打击 LLM 的"凭感觉写代码" 失败**：

- 在同一代码库中找到类似工作代码
- 对比工作的和坏掉的

**"Find Working Examples" 是 LLM 调试中的关键方法论**——LLM 倾向"凭直觉"，但**对比工作示例**是更可靠的方法。

### 3.9 Phase 3 "Form Single Hypothesis" 科学方法

> 1. **Form Single Hypothesis**
>    - State clearly: "I think X is the root cause because Y"
>    - Write it down
>    - Be specific, not vague

**"Form Single Hypothesis" 精准打击 LLM 的"多重假设并行测试" 失败**：

- LLM 倾向"试试 X、试试 Y、试试 Z"
- 文档强制**单一假设** + 写下来 + 具体

**"Don't fix multiple things at once"** 也是关键——**一次一个变量**。

### 3.10 Phase 4 "3+ 失败 → 质疑架构"

> 4. **If Fix Doesn't Work**
>    - STOP
>    - Count: How many fixes have you tried?
>    - If < 3: Return to Phase 1, re-analyze with new information
>    - **If ≥ 3: STOP and question the architecture (step 5 below)**
>    - DON'T attempt Fix #4 without architectural discussion

**这是整个 Phase 4 中最关键的设计**——**3+ 修复失败的硬触发器**：

- < 3 次 → 返回 Phase 1 重新分析
- ≥ 3 次 → 质疑架构（不是再加修复）
- 严禁第 4 次修复无架构讨论

**"3 次" 阈值**是经验值——避免 LLM 在"再试一次" 循环中浪费。

### 3.11 "If 3+ Fixes Failed: Question Architecture" 模式

> **Pattern indicating architectural problem:**
> - Each fix reveals new shared state/coupling/problem in different place
> - Fixes require "massive refactoring" to implement
> - Each fix creates new symptoms elsewhere

**3 个"架构问题" 模式** 让 LLM 识别何时应停止修复：

- 每个修复揭示新问题（**打地鼠** 模式）
- 修复需要大规模重构
- 修复产生新症状

**最后的关键洞察**：

> This is NOT a failed hypothesis - this is a wrong architecture.

**"不是失败的假设，是错误的架构"** —— 这是一个哲学级的修正：

- LLM 倾向"假设错了" → 再试
- 实际是"架构错了" → 重构

### 3.12 "Red Flags" 11 条

> - "Quick fix for now, investigate later"
> - "Just try changing X and see if it works"
> - "Add multiple changes, run tests"
> - "Skip the test, I'll manually verify"
> - "It's probably X, let me fix that"
> - "I don't fully understand but this might work"
> - "Pattern says X but I'll adapt it differently"
> - "Here are the main problems: [lists fixes without investigation]"
> - Proposing solutions before tracing data flow
> - **"One more fix attempt" (when already tried 2+)**
> - **Each fix reveals new problem in different place**

**11 条红旗**精准打击 LLM 在调试中的 11 种失败模式：

- "快速修复" → 反偷懒
- "试试 X" → 反盲目
- "添加多个变更" → 反并行
- "跳过测试" → 反 TDD
- "可能是 X" → 反猜测
- "不完全理解" → 反假装
- "适应模式" → 反规则
- "列出修复未调查" → 反无根因
- "追踪前提出方案" → 反顺序
- **"再来一次"（2+ 后）** → 反无止境
- **"每个修复揭示新问题"** → 反打地鼠

**最后一句"ALL of these mean: STOP. Return to Phase 1."** 是统一指令。

### 3.13 "human partner's Signals" 5 个红旗

> - "Is that not happening?" - You assumed without verifying
> - "Will it show us...?" - You should have added evidence gathering
> - "Stop guessing" - You're proposing fixes without understanding
> - "Ultrathink this" - Question fundamentals, not just symptoms
> - "We're stuck?" (frustrated) - Your approach isn't working

**5 个 human partner 的话**是 LLM 协作中的"反馈信号"——

- 当 LLM 听到 human 说这些 → 立即返回 Phase 1
- 这种"自然语言信号" 让 LLM **像与 human 协作一样**进行自我检查

**5 个信号** 精准覆盖调试中的 5 大失败模式：

- "Is that not happening" → 反假设
- "Will it show us" → 反缺证据
- "Stop guessing" → 反猜测
- "Ultrathink" → 反浅思考
- "We're stuck" → 反方法错误

### 3.14 "Common Rationalizations" 8 条

| 借口 | 现实 |
|--------|---------|
| "Issue is simple, don't need process" | Simple issues have root causes too. Process is fast for simple bugs. |
| "Emergency, no time for process" | Systematic debugging is FASTER than guess-and-check thrashing. |
| "Just try this first, then investigate" | First fix sets the pattern. Do it right from the start. |
| "I'll write test after confirming fix works" | Untested fixes don't stick. Test first proves it. |
| "Multiple fixes at once saves time" | Can't isolate what worked. Causes new bugs. |
| "Reference too long, I'll adapt the pattern" | Partial understanding guarantees bugs. Read it completely. |
| "I see the problem, let me fix it" | Seeing symptoms ≠ understanding root cause. |
| "One more fix attempt" (after 2+ failures) | 3+ failures = architectural problem. Question pattern, don't fix again. |

**8 条借口-现实对照表** 精准打击 LLM 在调试中的 8 种合理化：

- 简单 → 反驳：简单也有根因
- 紧急 → 反驳：系统化更快
- 先试再查 → 反驳：第一个修复设定模式
- 先手动验证 → 反驳：未测试的修复不持久
- 一次多个 → 反驳：无法隔离
- 跳过参考 → 反驳：部分理解保证 bug
- 看到就修 → 反驳：看到症状 ≠ 理解根因
- 再来一次（2+ 后） → 反驳：3+ = 架构问题

**每条反驳都直接攻击借口的核心**。

### 3.15 "When Process Reveals 'No Root Cause'" 5% 警告

> 1. You've completed the process
> 2. Document what you investigated
> 3. Implement appropriate handling (retry, timeout, error message)
> 4. Add monitoring/logging for future investigation
>
> **But:** 95% of "no root cause" cases are incomplete investigation.

**这一段是反 LLM "放弃" 的关键**：

- LLM 倾向"找不到根因 → 处理一下得了"
- 文档显式说"95% 的'没有根因' 案例是不完整的调查"
- 显式说**记录调查** + **实施处理** + **添加监控**

**这种"95% 警告" 是 LLM 调试中的反合理化硬约束**。

### 3.16 "Real-World Impact" 现实影响数据

> - Systematic approach: 15-30 minutes to fix
> - Random fixes approach: 2-3 hours of thrashing
> - First-time fix rate: 95% vs 40%
> - New bugs introduced: Near zero vs common

**4 个对比数据**精准量化"系统化 vs 随机" 的差异：

- 修复时间：15-30 min vs 2-3 hr（**4-12 倍差**）
- 首次修复率：95% vs 40%（**2.4 倍差**）
- 新 bug：近零 vs 常见

**数据让 LLM 看到"流程的价值"** —— 不只是"应该"，而是"显著更快"。

## 四、文件结构

文件结构清晰，14 段：

```
1. YAML frontmatter
2. Overview（核心原则）
3. The Iron Law（最强约束）
4. When to Use（含"特别使用"和"不要跳过"）
5. The Four Phases（4 阶段总览）
   - Phase 1: 根因调查（5 步）
   - Phase 2: 模式分析（4 步）
   - Phase 3: 假设与测试（4 步）
   - Phase 4: 实施（5 步）
6. Red Flags - STOP and Follow Process（11 条）
7. human partner's Signals（5 条）
8. Common Rationalizations（8 条）
9. Quick Reference（4 阶段对照表）
10. When Process Reveals "No Root Cause"（95% 警告）
11. Supporting Techniques（指向配套文件）
12. Real-World Impact（数据对比）
```

## 五、优点

### 5.1 4 阶段严格分离

4 阶段（根因 / 模式 / 假设 / 实施）严格分离，每阶段有明确进入条件：

- **必须完成前一阶段**
- **不能跳过**
- **不能并行**

这种**严格分离**是 LLM 协作中反"跳过"失败模式的关键。

### 5.2 "3+ 失败 → 质疑架构" 的硬触发器

> If ≥ 3: STOP and question the architecture (step 5 below)
> DON'T attempt Fix #4 without architectural discussion

**这是整个 Phase 4 中最关键的设计**——**3+ 修复失败的硬触发器**：

- < 3 次 → 重新分析
- ≥ 3 次 → 质疑架构
- 严禁第 4 次修复无架构讨论

**"3 次"阈值**是经验值——避免 LLM 在"再试一次"循环中浪费。

### 5.3 "5 步红旗" 让 LLM 听到 human 的"停止"信号

> "Is that not happening?" - You assumed without verifying
> "Stop guessing" - You're proposing fixes without understanding
> "We're stuck?" (frustrated) - Your approach isn't working

**5 个 human partner 的话**是 LLM 协作中的"反馈信号"——

- 当 LLM 听到 human 说这些 → 立即返回 Phase 1
- 这种"自然语言信号" 让 LLM **像与 human 协作一样**进行自我检查

### 5.4 "8 条借口-现实" 对照表

**8 条借口-现实对照表** 精准打击 LLM 的 8 种合理化：

- 简单 → 反驳：简单也有根因
- 紧急 → 反驳：系统化更快
- 看到就修 → 反驳：看到症状 ≠ 理解根因
- 再来一次（2+ 后） → 反驳：3+ = 架构问题

### 5.5 "95% 没有根因" 警告

> **But:** 95% of "no root cause" cases are incomplete investigation.

**这一句是反 LLM "放弃" 的关键**——LLM 倾向"找不到根因 → 处理一下得了"，文档显式说"95% 是不完整调查"。

### 5.6 配套技术文件齐全

本目录提供 4 个配套技术文件：

- `root-cause-tracing.md`（反向追踪）
- `defense-in-depth.md`（多层防御）
- `condition-based-waiting.md`（条件等待）
- `test-academic.md` + 3 个 pressure test（压力测试案例）

**这种"主技能 + 配套技术 + 测试案例" 的三层结构**让学习路径完整。

## 六、缺点 / 风险

### 6.1 4 阶段串行可能在某些场景下过慢

4 阶段严格串行在某些场景下可能过慢：

- 简单 bug（如拼写错误）不需要 Phase 2
- 重现问题（Phase 1.2）可能耗时很长
- 假设测试（Phase 3）可能需要多轮

虽然文档说"对简单 bug 流程是快的"，但 LLM 仍可能感觉"4 阶段都走一遍" 浪费时间。

可以补充"什么情况可跳过 Phase 2" 的指导。

### 6.2 缺乏"何时重读错误信息"的指导

Phase 1.1 说"Read Error Messages Carefully"——但 LLM 可能在 Phase 1 后就忘记再读。

可以补充"在每个 Phase 入口都重读错误信息" 的建议。

### 6.3 "Gather Evidence" 4 层法可能不适用所有系统

4 层记录法（数据进入、数据退出、环境/配置、状态）适用于**有明确组件边界**的系统（如多进程服务）。

但**单进程脚本**或**纯函数**没有"组件边界"—— LLM 可能机械应用 4 层法但无所获。

可以补充"单进程系统" 的替代方法。

### 6.4 "Find Working Examples" 的局限性

"Find Working Examples" 在小型代码库或独特功能下可能找不到示例：

- 全新功能没有类似代码
- 重构旧代码时没有"工作版"
- LLM 可能强行找不相关的"工作示例"

可以补充"找不到工作示例时" 的处理。

### 6.5 "Form Single Hypothesis" 可能在多因素问题时过严

"Form Single Hypothesis" 精准打击 LLM 的"多重假设并行测试" 失败，但**真正的 bug 可能是多因素**：

- A + B 一起导致 bug
- 但单独的 A 和 B 都不触发

强制"单一假设" 可能让 LLM 反复回到 Phase 1。

可以补充"多因素问题" 的处理（先拆分为单因素）。

### 6.6 "3+ 失败 → 质疑架构" 的执行依赖 human

> Discuss with your human partner before attempting more fixes

**这一步依赖 human 协作**——但 LLM 协作场景中未必有 human partner。

可以补充"没有 human partner" 的回退策略。

### 6.7 缺乏"跨会话调试"的指导

LLM 调试常跨多个会话（如 Phase 1 一次会话，Phase 3 另一次）：

- 上次调查的"证据"如何保存？
- 假设的"写下来" 在哪保存？
- 跨会话的状态如何保持？

可以补充"跨会话状态" 的管理。

## 七、关键洞察

### 7.1 这是 Superpowers 项目的"调试纪律核心"

整个项目有多个技能直接关联调试：

- `test-driven-development` —— TDD 中的"Bug 修复" 集成
- `verification-before-completion` —— 完成前验证
- **`systematic-debugging`（本文件）** —— 调试纪律核心
- `using-superpowers` —— 12 条红旗（母技能）

**本技能位于"调试" 工作流的最核心** —— 它是 Superpowers 项目对"调试" 的方法论贡献。

### 7.2 "3+ 失败 = 架构问题" 是关键哲学

> This is NOT a failed hypothesis - this is a wrong architecture.

**"不是失败的假设，是错误的架构"** 是哲学级的修正：

- LLM 倾向"假设错了" → 再试
- 实际是"架构错了" → 重构

**3+ 阈值**是经验值——避免 LLM 在"再试一次"循环中浪费。

### 7.3 "在源头修复，而非症状" 是治本原则

> **Fix at source, not at symptom**

**这一句是 LLM 调试中的"治本 vs 治标"原则** —— 可以推广到：

- 治 bug 在源头，不在表现
- 治问题在设计，不在补丁
- 治失败在架构，不在症状

### 7.4 "Find Working Examples" 是 LLM 调试的关键方法论

LLM 倾向"凭直觉"——但**对比工作示例**是更可靠的方法。

**"在同一代码库中找到类似工作代码"** 是 LLM 协作中的方法论基石：

- 工作的代码 = 已知正确的样本
- 对比差异 = 找到问题点
- **这种方法论可以推广到所有 LLM 协作场景**

### 7.5 "Gather Evidence Before Proposing Fixes" 是反 LLM 关键

Phase 1 的 5 步调查法 + Phase 1.4 的 4 层记录法 = **"先收集证据再提修复"**：

- LLM 倾向"先猜再验证"
- 文档强制"先收集再分析再调查"
- **这种方法论是 LLM 协作中的反"凭空猜测"关键**

### 7.6 "95% 没有根因" 警告是反 LLM 放弃关键

> **But:** 95% of "no root cause" cases are incomplete investigation.

**这一句是反 LLM "放弃" 的关键**——LLM 倾向"找不到根因 → 处理一下得了"，文档显式说"95% 是不完整调查"。

## 八、关联文档

### 8.1 同目录

- `CREATION-LOG.md`（[systematic-debugging/CREATION-LOG.md](file:///e:/WorkStation/MyProject/superpowers/skills/systematic-debugging/CREATION-LOG.md)）—— 创建日志
- `root-cause-tracing.md`（[systematic-debugging/root-cause-tracing.md](file:///e:/WorkStation/MyProject/superpowers/skills/systematic-debugging/root-cause-tracing.md)）—— 根因追踪技术
- `defense-in-depth.md`（[systematic-debugging/defense-in-depth.md](file:///e:/WorkStation/MyProject/superpowers/skills/systematic-debugging/defense-in-depth.md)）—— 多层防御
- `condition-based-waiting.md`（[systematic-debugging/condition-based-waiting.md](file:///e:/WorkStation/MyProject/superpowers/skills/systematic-debugging/condition-based-waiting.md)）—— 条件等待
- `test-academic.md`、`test-pressure-1/2/3.md` —— 压力测试案例
- `find-polluter.sh` —— 查找污染者脚本
- `condition-based-waiting-example.ts` —— 条件等待示例代码

### 8.2 配套技能

- `test-driven-development/SKILL.md` —— Phase 4.1 创建失败测试
- `verification-before-completion/SKILL.md` —— 完成前验证

### 8.3 母技能

- `using-superpowers/SKILL.md` —— 12 条红旗

## 九、状态评价

**成熟度**：✅ **生产可用 + 核心纪律**

- 文档结构清晰、纪律强
- 反合理化设计成熟
- 4 阶段流程 + 3+ 失败硬触发器
- 已经在项目中被实际使用

**完备度**：⭐⭐⭐⭐ (4/5)

- 核心机制完整
- 缺少"跨会话调试"指导
- 4 阶段串行可能过慢
- "3+ 失败 → 质疑架构" 依赖 human

**风险等级**：🟡 中

- 主要风险是 LLM 跳过 Phase 1
- 11 条红旗 + 8 条合理化大部分缓解
- 但仍需要主代理具备"识别跳过" 的能力

## 十、给读者的启示

### 10.1 启示 1：4 阶段严格分离是流程设计核心

LLM 协作中的复杂流程应当**严格分离**：

- 4 阶段：调查 → 模式 → 假设 → 实施
- 不能跳过
- 不能并行

**这种"阶段分离"是 LLM 流程设计的核心模式**。

### 10.2 启示 2：硬触发器（3+ 失败）是反循环关键

> If ≥ 3: STOP and question the architecture

**"3+ 失败 = 质疑架构"是反"再试一次"循环的硬触发器**：

- 阈值（3）是经验值
- 超过 → 改变方向（不要"再来一次"）
- **这种"硬触发器"是 LLM 协作中的关键设计**

### 10.3 启示 3：自然语言信号让 LLM 听到 human 的"停止"

> "Is that not happening?"
> "Stop guessing"
> "We're stuck?"

**5 个 human partner 的话是 LLM 协作中的"自然语言信号"** ——

- 当 LLM 听到 human 说这些 → 立即返回 Phase 1
- **这种"自然语言信号"让 LLM 像与 human 协作一样进行自我检查**

### 10.4 启示 4：在源头修复是治本原则

> **Fix at source, not at symptom**

**"在源头修复"是 LLM 调试中的"治本 vs 治标"原则** —— 可以推广到所有 LLM 协作场景。

### 10.5 启示 5：先收集证据再提修复是反 LLM 凭空猜测关键

> Gather Evidence Before Proposing Fixes

**"先收集证据再提修复"** 是 LLM 调试中的方法论基石：

- LLM 倾向"先猜再验证"
- 文档强制"先收集再分析再调查"
- **这种方法论可以推广到所有 LLM 协作场景**

### 10.6 启示 6：95% 警告是反 LLM 放弃关键

> **But:** 95% of "no root cause" cases are incomplete investigation.

**这一句是反 LLM "放弃" 的关键**——LLM 倾向"找不到根因 → 处理一下得了"，文档显式说"95% 是不完整调查"。

## 十一、后续工作

### 11.1 可能的改进

1. **添加"什么情况可跳过 Phase 2" 的指导**
   - 简单 bug 直接走 Phase 1 → 3 → 4
   - 全新功能没有"工作示例"

2. **添加"在每个 Phase 入口重读错误信息" 的建议**

3. **添加"单进程系统" 的替代方法**
   - 没有组件边界时的证据收集
   - 纯函数调试的简化流程

4. **添加"多因素问题" 的处理**
   - 先拆分为单因素
   - 再用"单一假设" 流程

5. **添加"没有 human partner" 的回退策略**
   - 完全自动化场景下的"3+ 失败" 处理
   - 自评架构 vs 找 human

6. **添加"跨会话调试" 的状态管理**
   - 调查证据的保存
   - 假设的记录

### 11.2 监控点

- 4 阶段的实际使用频率
- 3+ 失败硬触发器的触发频率
- "Quick fix" 红旗下 LLM 的反应
- 跨会话调试的"状态丢失"频率

### 11.3 长期演进

- 引入"自动证据收集"（基于调试 instrumentation）
- 引入"自动假设生成"（基于 LLM 推理）
- 与 `verification-before-completion` 集成形成"自审→外审→验证"
