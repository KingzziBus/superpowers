# spec-reviewer-prompt.md 分析报告

## 一、基本信息

- **文件路径**：`skills/subagent-driven-development/spec-reviewer-prompt.md`
- **文件大小**：62 行
- **文档类型**：子代理 Prompt 模板
- **配套主文件**：`SKILL.md`（子代理驱动开发）
- **使用方式**：分派规范符合性审查者子代理时复制填充
- **使用时机**：实施者完成任务后、代码质量审查之前（**强顺序约束**）

## 二、目的

这是一个紧凑但**极其关键**的 prompt 模板，用于分派"规范符合性审查者"子代理。它的核心使命是**验证实施者是否做了"要求做的事"**——既不多也不少。

它是 `subagent-driven-development/SKILL.md` 中**两阶段审查的第一阶段**：

```
实施者完成 → [规范符合性审查 ✅] → [代码质量审查 ✅] → 任务完成
              ↑ 本文件
```

**为什么必须先做规范符合性？** 因为如果实施者**做错了事**（做少了或做多了），再去审查代码质量毫无意义。先确认"对不对"再评"好不好"。

## 三、核心技术决策

### 3.1 "Do Not Trust the Report" —— 反 LLM 信任失败

```
## CRITICAL: Do Not Trust the Report

The implementer finished suspiciously quickly. Their report may be incomplete,
inaccurate, or optimistic. You MUST verify everything independently.
```

**这是整个 prompt 中最反 LLM 失败的设计**——它**显式禁止审查者信任实施者的报告**。

LLM 审查的典型失败模式：

- 倾向"信任同事报告"（社会倾向）
- 倾向"快速一致"（避免冲突）
- 倾向"基于摘要评估"（偷懒）

本 prompt **精准打击**这 3 种失败：

> **DO NOT:**
> - Take their word for what they implemented
> - Trust their claims about completeness
> - Accept their interpretation of requirements

3 个 DO NOT 直接对应 3 种 LLM 失败模式。

> **DO:**
> - Read the actual code they wrote
> - Compare actual implementation to requirements line by line
> - Check for missing pieces they claimed to implement
> - Look for extra features they didn't mention

4 个 DO 给出**正确的审查方法**——**基于实际代码**而非报告。

**这种"显式禁止信任"是 LLM 协作中反"轻信"的强约束**。

### 3.2 "The implementer finished suspiciously quickly" —— 关键的怀疑论提示

> The implementer finished suspiciously quickly.

**这一句是 prompt 中的"心理暗示"** —— 它让审查者**主动怀疑**实施者的工作：

- LLM 倾向"乐观评价"（默认正面）
- 本句**预设怀疑**："如果它完成得太快，可能有问题"
- 这种**反 LLM 乐观偏置**的设计是 LLM 审查的稀缺设计

### 3.3 "Missing / Extra / Misunderstandings" 三维检查

```
**Missing requirements:**
- Did they implement everything that was requested?
- Are there requirements they skipped or missed?
- Did they claim something works but didn't actually implement it?

**Extra/unneeded work:**
- Did they build things that weren't requested?
- Did they over-engineer or add unnecessary features?
- Did they add "nice to haves" that weren't in spec?

**Misunderstandings:**
- Did they interpret requirements differently than intended?
- Did they solve the wrong problem?
- Did they implement the right feature but wrong way?
```

**3 维检查**精准覆盖"规范符合性失败"的 3 大类型：

| 维度 | 失败类型 | 反 LLM 失败 |
|---|---|---|
| Missing | 漏掉需求 | 反"假装做了" |
| Extra | 过度构建 | 反"自作主张" |
| Misunderstandings | 误解需求 | 反"理解错" |

**3 维检查**让审查者**有结构化清单可循**——避免 LLM 审查"挑自己看到的问题"。

**注意 Misunderstandings 维度的精妙**：

- "Did they solve the wrong problem?" —— 完全偏题
- "Did they implement the right feature but wrong way?" —— 实现方式不对

这两种失败的检查项分别覆盖"完全错"和"细节错"。

### 3.4 "Verify by reading code, not by trusting report."

> **Verify by reading code, not by trusting report.**

**这句作为整个段落的压轴——核心方法论**：

- 不是"读报告"
- 而是"读代码"
- **代码是真理之源**（code is the source of truth）

这是 LLM 协作中的**关键方法论**——**所有验证必须基于产物**（代码/文档/配置），而非"中间人的描述"。

### 3.5 二元裁决（✅ / ❌）

```
Report:
- ✅ Spec compliant (if everything matches after code inspection)
- ❌ Issues found: [list specifically what's missing or extra, with file:line references]
```

**二元裁决**让审查结果**程序化可处理**：

- ✅ → 进入下一阶段（代码质量审查）
- ❌ → 实施者修复 → 重新规范审查

**强制要求"file:line references"** 让反馈**可定位**——避免"在某个地方" 的模糊反馈。

## 四、文件结构

文件结构紧凑，5 段：

```
1. 标题 + 用途
2. Task tool 分派模板
3. CRITICAL: Do Not Trust the Report（反 LLM 信任失败）
4. Your Job（3 维检查）
5. Report Format（二元裁决 + file:line 引用）
```

62 行覆盖了完整的"规范符合性审查" workflow。

## 五、优点

### 5.1 极简而完整（62 行）

5 段结构覆盖了完整的"规范符合性审查"：

- 输入（What Was Requested + Implementer's Claims）
- 反信任约束（Do Not Trust）
- 检查方法（3 维检查）
- 裁决（二元）

### 5.2 反 LLM 信任失败的强约束

> The implementer finished suspiciously quickly.

这一句**预设怀疑**的设计是 LLM 协作中**反轻信**的稀缺设计：

- 让审查者主动怀疑
- 不基于报告评估
- 必须读代码验证

### 5.3 3 维检查清单（Missing / Extra / Misunderstandings）

3 维检查精准覆盖"规范符合性失败"的 3 大类型：

- Missing → 漏掉
- Extra → 过度
- Misunderstandings → 误解

每种类型有 3 个具体问题，让审查者**有结构化清单**。

### 5.4 "Code is the source of truth" 方法论

> **Verify by reading code, not by trusting report.**

这是 LLM 协作中的**关键方法论**——**所有验证必须基于产物**：

- 不是"读报告"
- 是"读代码"
- 中间人的描述**不应当作为评估依据**

### 5.5 二元裁决 + file:line 引用

二元裁决（✅ / ❌）+ 强制 file:line 引用让**反馈程序化可处理**：

- ✅ → 下一阶段
- ❌ → 修复 → 重新审查
- file:line → 修复可定位

## 六、缺点 / 风险

### 6.1 缺乏"读取深度"的指导

"Read the actual code they wrote" 没有说明**读多深**：

- 只读 modified files？
- 还是要追到 callers / callees？
- 读相关 test files？
- 读依赖的库代码？

LLM 审查容易**只读 modified files** 而漏掉上下文。

可以补充：

- "Read modified files + 1 level of callers/callees"
- "Read test files to verify behavior claims"

### 6.2 缺乏"测试运行"的指导

代码审查是否应当**运行测试**？

- 实施者声称"5/5 tests passing" —— 审查者验证？
- 跑一遍 `npm test` 是审查代理能做的吗？
- 还是只验证"tests exist" + "tests look reasonable"？

不同 LLM 平台对"运行测试" 的支持不同。本 prompt **没说明**。

### 6.3 "suspiciously quickly" 的判定标准缺失

> The implementer finished suspiciously quickly.

"可疑地快" 缺乏判定标准：

- 多快算"快"？
- 5 分钟完成 1 小时的任务 = 可疑
- 30 分钟完成 1 小时的任务 = 正常
- 2 小时完成 1 小时的任务 = 慢（但仍可疑，因为可能没深入思考）

可以补充：

- "If a task that should take ~1hr completes in <10min, be extra thorough"
- "Quick completion is OK if evidence is solid (tests, docstrings)"

### 6.4 缺乏"需求模糊时" 的处理

如果需求本身就是模糊的（例如"实现一个好的错误处理"），审查者怎么判断"规范符合"？

- "Did they implement everything that was requested?" → 模糊需求时无法回答
- "Did they add 'nice to haves' that weren't in spec?" → 模糊需求时什么算 nice-to-have？

可以补充：

- "If requirements are ambiguous, flag it as Missing and ask for clarification"
- "Better to flag 'ambiguity in spec' than pass a fuzzy review"

### 6.5 缺乏"边界情况"的检查

3 维检查（Missing / Extra / Misunderstandings）没有显式覆盖"边界情况"：

- 实施者实现了正常路径但漏掉错误处理 → 算 Missing？
- 实施者实现了功能但漏掉空值检查 → 算 Misunderstandings？

可以补充第 4 维：**Edge Cases**（边界情况）。

### 6.6 "Extra/unneeded work" 的"度" 没有量化

"过度的额外功能" 难以量化：

- 加一个 `--verbose` flag = 过度？
- 加一个 `--json` flag = 过度？
- 加一个 README 更新 = 必要 vs 过度？

本 prompt 没给"什么算合理范围" 的指南。可以补充：

- "Small DX improvements (e.g., better error messages) are OK"
- "New features, new flags, new APIs = Extra"

### 6.7 缺乏"审查失败" 的回退

如果审查代理**自己**也不确定（"我觉得 80% 符合"），文档没给**回退路径**：

- 是"通过"还是"不通过"？
- 还是"标记为 NEEDS_CONTEXT"？
- 还是"标记为 BLOCKED"？

可以补充：

- "If uncertain, default to flagging issue with rationale"
- "Better to be over-cautious than miss a real issue"

## 七、关键洞察

### 7.1 这是 Superpowers 项目的"规范符合性审查"模板

整个项目有 4 个核心子代理 prompt：

- `implementer-prompt.md` —— 实施者
- **`spec-reviewer-prompt.md`（本文件）** —— 规范符合性审查
- `code-quality-reviewer-prompt.md` —— 代码质量审查
- `code-reviewer.md`（requesting-code-review 目录下） —— 最终代码审查

**本文件位于"实施"与"质量"之间**——它是**正确性的门控**。

### 7.2 "Do Not Trust the Report" 是 LLM 协作中最稀缺的设计

LLM 协作中的"信任链" 是**单点失败**的高风险路径：

- 实施者报告 → 审查者 → 主代理
- 如果审查者**信任**实施者报告 → 整条链失效

**本 prompt 显式禁止信任**——它强制审查者**独立验证**。

> **Verify by reading code, not by trusting report.**

**这是 LLM 协作中"反单点失败"的关键设计**。

### 7.3 "suspiciously quickly" 是反 LLM 乐观偏置的关键

LLM 倾向：

- 相信同事的快速完成
- 倾向"乐观"评估
- 避免冲突

**"suspiciously quickly" 这一句**：

- 预设怀疑
- 让审查者**主动找问题**而非"找亮点"
- 反 LLM 偏置

**这种"心理暗示"设计是 LLM 协作中的稀缺设计**。

### 7.4 "Code is the source of truth" 是 LLM 协作的方法论

> **Verify by reading code, not by trusting report.**

**这是 LLM 协作的方法论基石**：

- **代码是真理之源**（code is the source of truth）
- **报告是辅助**（reports are auxiliary）
- **验证必须基于产物**（verification must be artifact-based）

可以推广到所有 LLM 协作场景：

- 实施报告 → 验证必须基于产物（代码/文档/配置）
- 测试结果 → 验证必须基于测试运行
- 性能声称 → 验证必须基于 benchmark

### 7.5 3 维检查形成"规范符合性"的完整画面

3 维（Missing / Extra / Misunderstandings）形成**规范符合性的完整三象限**：

- **Missing（漏）**：实施者遗漏了需求
- **Extra（多）**：实施者添加了未请求内容
- **Misunderstandings（错）**：实施者理解错了需求

任何一维有 issue → 不通过。**3 维都 ✅ → 规范符合**。

## 八、关联文档

### 8.1 同目录

- `SKILL.md`（[subagent-driven-development/SKILL.md](file:///e:/WorkStation/MyProject/superpowers/skills/subagent-driven-development/SKILL.md)）—— 调度方主技能
- `implementer-prompt.md`（[subagent-driven-development/implementer-prompt.md](file:///e:/WorkStation/MyProject/superpowers/skills/subagent-driven-development/implementer-prompt.md)）—— 实施者 prompt
- `code-quality-reviewer-prompt.md`（[subagent-driven-development/code-quality-reviewer-prompt.md](file:///e:/WorkStation/MyProject/superpowers/skills/subagent-driven-development/code-quality-reviewer-prompt.md)）—— 代码质量审查

### 8.2 同类 Prompt 模板

- `writing-plans/plan-document-reviewer-prompt.md` —— plan 审查（更上游）
- `requesting-code-review/code-reviewer.md` —— 最终代码审查（更下游）

### 8.3 配套技能

- `writing-plans/SKILL.md` —— 计划来源
- `using-git-worktrees/SKILL.md` —— 工作区隔离

## 九、状态评价

**成熟度**：✅ **生产可用 + 关键门控**

- 文档极简但精准
- 反 LLM 失败设计成熟
- 是 Superpowers 项目工作流的关键节点

**完备度**：⭐⭐⭐ (3.5/5)

- 核心机制完整
- 缺少"读取深度" 的指导
- 缺少"测试运行" 的处理
- "suspiciously quickly" 缺乏量化标准

**风险等级**：🟡 中

- 主要风险是审查者**未读代码**就裁决
- "Do Not Trust the Report" 部分缓解
- 但仍需要主代理具备"识别审查质量"的能力

## 十、给读者的启示

### 10.1 启示 1：审查必须基于产物，而非报告

> **Verify by reading code, not by trusting report.**

这是 LLM 协作中**最稀缺也最重要的方法论**——**所有验证必须基于产物**（代码/文档/配置），而非"中间人的描述"。

可以推广到所有 LLM 协作场景：

- 实施报告 → 验证必须基于代码
- 测试结果 → 验证必须基于测试运行
- 性能声称 → 验证必须基于 benchmark

### 10.2 启示 2："Do Not Trust" 必须显式声明

LLM 审查 agent 倾向"信任同事报告"——这是社会倾向的 LLM 表现。

**显式写"Do Not Trust the Report"** 是 LLM 协作中**反轻信**的强约束：

- 让审查 agent 主动怀疑
- 强制独立验证
- 反 LLM 信任失败

### 10.3 启示 3："suspiciously quickly" 是反 LLM 乐观偏置的关键

LLM 倾向"乐观评价"——本 prompt 用"可疑地快" 预设怀疑：

- 让审查者主动找问题
- 不基于"快速完成" 给好评
- 反 LLM 乐观偏置

**这种"心理暗示"设计**可以推广到所有审查 prompt。

### 10.4 启示 4：3 维检查形成"规范符合性"的完整画面

3 维（Missing / Extra / Misunderstandings）形成**完整三象限**：

- Missing（漏）
- Extra（多）
- Misunderstandings（错）

**3 维让审查者有结构化清单可循**——避免"挑自己看到的问题"。

### 10.5 启示 5：二元裁决让审查结果程序化可处理

✅ / ❌ 二元裁决让**反馈程序化可处理**：

- ✅ → 下一阶段
- ❌ → 修复 → 重新审查

**不强制裁决的审查 = 主代理无法自动化决定**。

## 十一、后续工作

### 11.1 可能的改进

1. **添加"读取深度"指导**
   - 读 modified files + 1 level of callers/callees
   - 读 test files 验证 behavior claims

2. **添加"测试运行"处理**
   - 显式说明审查者是否应当运行测试
   - 平台特定的测试运行能力

3. **量化"suspiciously quickly"**
   - "如果任务预计 1hr 但 <10min 完成，额外严格"
   - "Quick completion is OK if evidence is solid (tests, docstrings)"

4. **添加"边界情况"作为第 4 维**
   - Edge cases 检查
   - 错误处理检查
   - 空值/极端输入检查

5. **添加"过度构建"的量化标准**
   - 显式说明什么算合理范围
   - "Small DX improvements OK, new features = Extra"

6. **添加"审查失败"回退**
   - 不确定时默认标记 issue
   - 优于放过

### 11.2 监控点

- ✅ / ❌ 裁决比例（异常高失败率 → 实施质量问题）
- 实施者报告 vs 实际代码的差异（异常大 → 实施者报告不可信）
- 3 维检查的 issue 分布（Missing 多 → 需求不清；Extra 多 → 实施者过度构建）
- 审查代理"未读代码" 的频率

### 11.3 长期演进

- 引入"自动代码质量门控"（如 lint、type check）作为 LLM 审查的预筛
- 引入"需求清晰度评估"（让主代理在分派前评估 spec 清晰度）
- 与 `verification-before-completion` 串联形成"自审→外审→验证"
