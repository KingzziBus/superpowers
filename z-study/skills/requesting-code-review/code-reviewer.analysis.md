# requesting-code-review/code-reviewer.md 分析报告

## 一、基本信息

- **文件路径**：`skills/requesting-code-review/code-reviewer.md`
- **文件大小**：169 行
- **文档类型**：子代理 Prompt 模板（任务分派用）
- **配套主文件**：`SKILL.md`（调度技能）
- **使用方式**：复制填充 4 个占位符后分派给子代理

## 二、目的

这是一个具体的"代码审查者"子代理 prompt 模板。`SKILL.md` 教会主代理"何时/为何/如何"请求审查，**本文件**则提供审查代理的**完整 prompt**。

两者的关系：
- `SKILL.md` —— 调度方视角（when / why / how）
- `code-reviewer.md` —— 审查方视角（具体的 prompt）

主代理拿到本文件后：

1. 填充 4 个占位符（`{DESCRIPTION}` / `{PLAN_OR_REQUIREMENTS}` / `{BASE_SHA}` / `{HEAD_SHA}`）
2. 包装到 `Task tool (general-purpose)` 调用中
3. 等待审查代理返回结构化结果

## 三、核心技术决策

### 3.1 角色定位：Senior Code Reviewer

> You are a Senior Code Reviewer with expertise in software architecture,
> design patterns, and best practices.

审查代理被定位为 **"高级代码审查者"** —— 这是一种**角色化**技术：

- LLM 在"我是 X"的角色下表现更稳定
- "Senior" 暗示**有判断力**而非机械执行
- 三大领域被显式声明：**架构 / 设计模式 / 最佳实践** —— 划定审查的"专业边界"

### 3.2 五大检查维度

代码审查被分解为 **5 大维度**：

| 维度 | 检查项数 | 关注点 |
|---|---|---|
| Plan alignment | 3 | 是否符合需求？偏离是否合理？ |
| Code quality | 5 | 关注点分离、错误处理、类型安全、DRY、边界 |
| Architecture | 4 | 设计决策、扩展性、安全性、集成 |
| Testing | 4 | 真测试 vs mock、边界、集成、全部通过 |
| Production readiness | 4 | 迁移策略、向后兼容、文档、无明显 bug |

**结构化分类的好处**：
- 审查代理不会"漏项"——按清单逐项检查
- 主代理能精准定位"哪个维度有问题"
- 反馈颗粒度细 → 后续修复精准

### 3.3 "Calibration" 校准段

> Categorize issues by actual severity. Not everything is Critical.
> Acknowledge what was done well before listing issues — accurate praise
> helps the implementer trust the rest of the feedback.

两个关键的反 LLM 偏差设计：

**(a) 反"过度标记 Critical"**
- LLM 倾向把每项都标成 Critical（"显得专业"）
- 显式否定："Not everything is Critical"
- 解决：审查输出有**三档分级**，让 Critical 真的只用于 Critical

**(b) 反"全盘否定"**
- LLM 倾向"挑错模式"——只列问题，不提优点
- 显式要求："Acknowledge what was done well **before** listing issues"
- **信任建设**："accurate praise helps the implementer trust the rest of the feedback"
- 这种**先肯定再否定**的设计，让主代理**愿意听反馈**而不是抵触

### 3.4 反馈分级系统（Critical / Important / Minor）

```
#### Critical (Must Fix)
   [Bugs, security issues, data loss risks, broken functionality]

#### Important (Should Fix)
   [Architecture problems, missing features, poor error handling, test gaps]

#### Minor (Nice to Have)
   [Code style, optimization opportunities, documentation polish]
```

**三档分级** 对应 **三种不同的紧迫度**：

- **Critical**（必须修）：不修不能合入 —— Bug / 安全 / 数据丢失 / 破坏性
- **Important**（应当修）：不修不阻塞合入，但留下隐患 —— 架构 / 缺失功能 / 测试缺口
- **Minor**（建议）：纯风格 / 优化 / 文档润色

每一档都有**清晰的例子**（brackets 内的描述），降低审查代理的**误分类**风险。

### 3.5 反馈的具体性（File:line / Why / How to fix）

> For each issue:
> - File:line reference
> - What's wrong
> - Why it matters
> - How to fix (if not obvious)

强制要求每条反馈包含 4 个字段：

| 字段 | 目的 |
|---|---|
| File:line reference | 精确定位（避免"在某个地方" 的模糊反馈） |
| What's wrong | 问题描述（具体行为） |
| Why it matters | 影响（为什么这是个问题） |
| How to fix | 修复建议（如果非显然） |

**这是 LLM 协作中反"模糊反馈"的关键设计**——大多数 LLM 默认产出 "improve error handling" 这种**不可执行的**反馈。

### 3.6 明确裁决（Ready to merge?）

```
### Assessment
**Ready to merge?** [Yes | No | With fixes]
**Reasoning:** [1-2 sentence technical assessment]
```

**强制要求审查者给出明确裁决**——三选一：

- **Yes** —— 没问题，合入
- **No** —— 有 Critical，不能合入
- **With fixes** —— 修完重要问题可合入

LLM 倾向**"两边都对"** 的模糊裁决。本文件：

1. 强制三选一
2. 强制附 **1-2 句技术理由**（Reasoning）

这让**主代理能程序化地决定是否合入**，不用再做主观判断。

### 3.7 "Critical Rules" —— DO/DON'T 列表

```
**DO:**
- Categorize by actual severity
- Be specific (file:line, not vague)
- Explain WHY each issue matters
- Acknowledge strengths
- Give a clear verdict

**DON'T:**
- Say "looks good" without checking
- Mark nitpicks as Critical
- Give feedback on code you didn't actually read
- Be vague ("improve error handling")
- Avoid giving a clear verdict
```

**5 DO + 5 DON'T** 形成正反对照的强约束：

- **5 DO**：5 项审查纪律
- **5 DON'T**：5 项审查禁忌（显式禁止的 LLM 偏差）

特别是：

- **DON'T: "Say 'looks good' without checking"** —— 反"廉价好评"
- **DON'T: "Mark nitpicks as Critical"** —— 反"过度标记"
- **DON'T: "Give feedback on code you didn't actually read"** —— 反"幻觉反馈"
- **DON'T: "Be vague ('improve error handling')"** —— 反"模糊建议"
- **DON'T: "Avoid giving a clear verdict"** —— 反"模糊裁决"

这 5 条 DON'T 是**精准打击 LLM 审查的 5 大典型失败模式**。

## 四、文件结构

文件结构清晰，5 段：

```
1. 标题 + 用途 + 目的说明

2. Task tool 分派模板（大段 prompt）
   - 角色定义
   - What Was Implemented
   - Requirements / Plan
   - Git Range to Review
   - What to Check（5 大维度）
   - Calibration 校准
   - Output Format 输出格式
   - Critical Rules 铁律

3. 占位符说明（4 个）

4. 审查者返回字段说明

5. 示例输出
   - 完整的 Strengths / Issues / Recommendations / Assessment
```

## 五、优点

### 5.1 极致的结构化

- 5 大检查维度（Plan / Code / Architecture / Testing / Production）
- 3 档反馈分级（Critical / Important / Minor）
- 4 字段反馈格式（File:line / What / Why / How）
- 三选一裁决（Yes / No / With fixes）

**每个维度都对应一个结构化约束**，审查代理几乎不可能"漏项"。

### 5.2 反 LLM 偏好的强约束

- 反"廉价好评"：5 DON'T 第一条
- 反"过度标记"：5 DON'T 第二条 + Calibration
- 反"幻觉反馈"：5 DON'T 第三条
- 反"模糊建议"：5 DON'T 第四条
- 反"模糊裁决"：5 DON'T 第五条

这 5 条与 `using-superpowers/SKILL.md` 的"12 条红旗"是同源思想：**把 LLM 失败模式显式列入"绝对不要"**。

### 5.3 "Plan alignment" 作为第一维度

5 大维度中，**Plan alignment 排在最前** —— 这与 Superpowers 项目"spec/plan 优先" 的核心哲学一致：

- 计划是真理之源（plan is the source of truth）
- 实施必须对齐计划
- 偏离计划必须显式标记

### 5.4 "Production readiness" 作为最后维度

5 大维度中，**Production readiness 排在最后** —— 强调"代码不只是能跑，还要能上生产"：

- 迁移策略
- 向后兼容
- 文档完整

**这一维度在大多数代码审查 prompt 中缺失**，是本文件的差异化亮点。

### 5.5 "先肯定再否定" 的设计

> Acknowledge what was done well before listing issues

这种"先认可后建议"的设计是**协作心理学**层面的考量：

- 让主代理**愿意听反馈**（避免抵触）
- 让"否定"基于具体贡献（不是全面否定）
- 提升整个反馈循环的**效率**

### 5.6 示例输出的真实可参考

示例输出（135-168 行）展示了**完整的、带具体 file:line 引用的、带 reasoning 的**审查输出。这让主代理**第一次使用本模板时就有清晰的输出预期**。

## 六、缺点 / 风险

### 6.1 缺乏对"审查深度"的控制

文档没有告诉审查代理**审查应该多深**：

- 是只过一遍 diff？
- 还是要追上下文、读相关模块？
- 是只审 prompt 中指定的范围，还是可以扩展到相关代码？

**LLM 审查代理的常见问题**：

- 浅层过 diff → 漏掉集成问题
- 深度过整个项目 → 浪费 token、超时
- 读但理解错 → 给出错误反馈

可以补充：

- "Read enough surrounding code to understand context, no more"
- "Trace one level up to caller, one level down to callee"

### 6.2 缺乏"审查代理能力局限"的应对

LLM 审查代理**有一些固有限制**：

- 不能运行测试（除非专门工具）
- 不能查数据库/网络
- 不能感知运行时性能
- 跨语言能力不一致

文档没有告诉审查代理"什么时候承认自己看不到"——可以补充：

- "If you cannot evaluate [aspect], explicitly state so"
- "Mark uncertain assessments as 'verify manually'"

### 6.3 反馈粒度可能过细，导致"列表爆炸"

强制 4 字段格式（File:line / What / Why / How）可能导致：

- 100 条"Minor" 反馈淹没主代理
- 反馈数量 > 实际可处理量
- 主代理放弃修复大量 Minor → 反馈循环失效

可以补充：

- "Aggregate similar minor issues"
- "Cap Minor to top 5 most impactful"
- "Group file:line references if they share a root cause"

### 6.4 缺乏对"重复审查"的处理

如果同一段代码被多次审查：

- 第 1 次：Critical A, B, C
- 修复后：Critical 0, but Minor 出现新 D, E
- 修复后：Minor 0, but 出现新 Minor F, G

文档没有说明"什么算'审查通过'"。可以补充：

- "If Critical and Important are 0 (or all fixed), merge approved"
- "Minor is informational, can be addressed in follow-up"

### 6.5 "Acknowledge strengths" 缺乏边界

> Acknowledge what was done well before listing issues

LLM 可能：

- 过度赞美（"Great job!"、"Excellent work!"）→ 廉价
- 找到不存在的优点（"Implemented caching" 但实际没有） → 幻觉
- 列出非相关的优点（"Used vim"）→ 噪音

可以补充：

- "Be specific about strengths, not generic praise"
- "Only acknowledge concrete contributions, not process"

### 6.6 "If you find issues with the plan itself rather than the implementation, say so" —— 风险

这一条让审查代理**有合法身份去批评 plan**。这是好事，但**可能导致**：

- 审查代理把"plan 不好" 当作 "implementation 不好" 的借口
- 主代理不知道该修 plan 还是 implementation
- 修复循环被破坏

可以补充：

- "Distinguish clearly: 'plan issue' (X) vs 'implementation issue' (Y)"
- "Plan issues are routed to planning step, not implementation step"

## 七、关键洞察

### 7.1 这是 Superpowers 项目的"代码侧" 主审查模板

整个项目有多个"X 审查者 prompt"：

- `brainstorming/spec-document-reviewer-prompt.md` —— spec 审查
- `writing-plans/plan-document-reviewer-prompt.md` —— plan 审查
- **`requesting-code-review/code-reviewer.md` —— 代码审查（本文件）**
- `subagent-driven-development/spec-reviewer-prompt.md` —— 实施前 spec 复核
- `subagent-driven-development/code-quality-reviewer-prompt.md` —— 实施后代码质量复核

**本文件位于流程的"代码侧下游"**——它审的是"已经实施完的代码是否符合 spec/plan"。

### 7.2 5 DON'T 是 LLM 审查的"五大失败模式"

```
- Say "looks good" without checking          → 反"廉价好评"
- Mark nitpicks as Critical                  → 反"过度标记"
- Give feedback on code you didn't read      → 反"幻觉反馈"
- Be vague ("improve error handling")        → 反"模糊建议"
- Avoid giving a clear verdict               → 反"模糊裁决"
```

这 5 条精准打击 LLM 审查的 5 大失败模式。**每个 LLM 审查 prompt 都应当有类似的"禁忌清单"**。

### 7.3 反馈粒度的"4 字段"是 LLM 协作的黄金模板

```
For each issue:
- File:line reference
- What's wrong
- Why it matters
- How to fix (if not obvious)
```

这种**4 字段反馈格式** 是 LLM 协作中**反"模糊反馈"的黄金模板**：

- **File:line** —— 精确定位
- **What's wrong** —— 客观描述
- **Why it matters** —— 影响说明（避免"挑刺"）
- **How to fix** —— 可执行修复（不只是问题）

可以推广到所有 LLM 审查场景。

### 7.4 "Production readiness" 维度的差异化

5 大维度中，"Production readiness" 是**本文件区别于其他代码审查 prompt 的最大特色**：

- 迁移策略
- 向后兼容
- 文档完整

**大多数 LLM 审查 prompt 只关注"功能正确性"和"代码质量"，但忽略了"生产准备度"**。本文件填补了这一点。

### 7.5 反馈的"软硬两档"扩展为"硬硬软三档"

`writing-plans/plan-document-reviewer-prompt.md` 是"Issues / Recommendations" 二档。

本文件升级为"Critical / Important / Minor" 三档，对应**三种紧迫度**。

这种**反馈粒度的演进**反映了 Superpowers 项目对"反馈分级"的不断细化：

```
v1: pass/fail 二元
v2: Issues/Recommendations 二档
v3: Critical/Important/Minor 三档（本文件）
```

未来可能进一步演化为"5 档"或"按维度分别定档"。

## 八、关联文档

### 8.1 同目录

- `SKILL.md`（[requesting-code-review/SKILL.md](file:///e:/WorkStation/MyProject/superpowers/skills/requesting-code-review/SKILL.md)）—— 调度方主技能

### 8.2 同类 Prompt 模板

- `brainstorming/spec-document-reviewer-prompt.md` —— spec 审查模板
- `writing-plans/plan-document-reviewer-prompt.md` —— plan 审查模板
- `subagent-driven-development/spec-reviewer-prompt.md` —— 实施前 spec 复核
- `subagent-driven-development/code-quality-reviewer-prompt.md` —— 实施后代码质量复核

### 8.3 关联技能

- `receiving-code-review/SKILL.md` —— 接收代码审查（被审查方视角）
- `verification-before-completion/SKILL.md` —— 完成前验证
- `using-superpowers/SKILL.md` —— 母技能

## 九、状态评价

**成熟度**：✅ **生产可用**

- 模板结构清晰、约束强
- 反 LLM 偏差设计成熟
- 已经在项目的子代理协作流程中被实际使用

**完备度**：⭐⭐⭐⭐ (4/5)

- 核心机制完整
- 缺少"审查深度"控制
- 缺少"重复审查"终止条件
- 反馈粒度可能爆炸的风险

**风险等级**：🟡 中

- 主要风险是 LLM 审查代理对大型 diff 的审查质量不稳定
- "5 DON'T" 大部分缓解了 LLM 失败模式
- 但仍需要主代理具备"评估审查质量"的能力

## 十、给读者的启示

### 10.1 启示 1：审查 prompt 必须有"Critical Rules" 段

任何 LLM 审查 prompt 都应当有 "DO/DON'T" 列表。DON'T 部分尤其重要：

- 显式禁止 LLM 的典型失败模式
- 比"如何做好"更有效的是"绝对不要做 X"

### 10.2 启示 2：反馈必须 4 字段化（File/What/Why/How）

LLM 协作中的反馈应当强制 4 字段：

- **File:line** —— 可定位
- **What** —— 可观察
- **Why** —— 可理解
- **How** —— 可执行

**没有这 4 字段的反馈都是模糊反馈**。

### 10.3 启示 3：反馈必须分级（推荐 3 档：Critical/Important/Minor）

LLM 协作中的反馈必须有"紧迫度"：

- Critical —— 不修不能继续
- Important —— 不修不阻塞
- Minor —— 建议级别

**无分级的反馈是"反馈过载" 的根源**。

### 10.4 启示 4：审查必须有"明确裁决"（Yes/No/With fixes）

LLM 审查必须强制 3 选 1 的裁决：

- Yes —— 立即合入
- No —— 不合入
- With fixes —— 修完合入

**不强制裁决的审查 = 主代理无法自动化决定**。

### 10.5 启示 5：先肯定再否定（建立信任）

> Acknowledge what was done well before listing issues

这是协作心理学的关键：**先认可具体贡献**让被审查方愿意听反馈。

不限于代码审查，可以推广到：

- 文档审查
- 设计审查
- 计划审查

### 10.6 启示 6：审查维度应当包含"Production readiness"

5 大维度中，"Production readiness" 是**本文件区别于其他 LLM 审查 prompt 的最大亮点**：

- 迁移策略
- 向后兼容
- 文档完整

**LLM 审查应当不止于"代码质量"，还应包括"生产准备度"**。

## 十一、后续工作

### 11.1 可能的改进

1. **添加"审查深度"控制**
   - 显式说明审查的"上下文边界"
   - 例如："Read enough surrounding code to understand context, no more"

2. **添加"审查代理能力局限"应对**
   - 显式说明 LLM 审查的"盲区"
   - 让审查代理在不确定时显式声明 "verify manually"

3. **添加"反馈粒度上限"**
   - 限制 Minor 反馈数量（如最多 5 条）
   - 鼓励聚合相似问题

4. **添加"重复审查终止条件"**
   - 明确"什么算审查通过"
   - 避免 Minor 永远修不完的循环

5. **添加"plan issue" 与 "implementation issue" 的路由**
   - 显式区分批评对象
   - 路由到不同修复路径

### 11.2 监控点

- Critical/Important/Minor 的实际分布（应符合预期比例）
- "Ready to merge" 裁决的 Yes/No/With fixes 比例
- 审查代理"幻觉反馈" 频率（未读代码的反馈）
- "Acknowledge strengths" 的具体性（避免廉价好评）

### 11.3 长期演进

- 与 `verification-before-completion` 串联：审查 → 验证 → 完成
- 引入"自动代码审查"作为 LLM 审查的预筛
- 引入"审查代理质量评估"（基于历史反馈准确率）
- 横向统一 5 套审查 prompt 的"分级体系"
