# implementer-prompt.md 分析报告

## 一、基本信息

- **文件路径**：`skills/subagent-driven-development/implementer-prompt.md`
- **文件大小**：114 行
- **文档类型**：子代理 Prompt 模板
- **配套主文件**：`SKILL.md`（子代理驱动开发主技能）
- **使用方式**：分派实施者子代理时复制填充
- **占位符**：`[task name]`、`[FULL TEXT ...]`、`[Scene-setting ...]`、`[directory]`

## 二、目的

这是"实施者"子代理的完整 prompt 模板。当主代理决定分派一个实施子代理做某个任务时，**直接复制本模板，填充占位符，分派即可**。

它是 `subagent-driven-development/SKILL.md` 中"流程图"中的 **"Dispatch implementer subagent"** 节点的**具体内容**。

## 三、核心技术决策

### 3.1 实施者任务的"6 步结构化流程"

```
1. Implement exactly what the task specifies   ← 严格按需求
2. Write tests (following TDD if task says to)  ← 测试驱动
3. Verify implementation works                  ← 自验证
4. Commit your work                             ← 提交
5. Self-review (see below)                      ← 自我审查
6. Report back                                  ← 报告
```

**6 步结构**是经过精心设计的：

- **步骤 1-2**：实施 + 测试（TDD 友好的并行）
- **步骤 3**：自验证（避免"我觉得能跑"的幻觉）
- **步骤 4**：**强制 git commit**（确保审查者有 git diff 可看）
- **步骤 5-6**：自审 + 报告（让 LLM 重新审视自己工作）

**关键设计**：步骤 4 强制 commit —— 没有 commit，**审查者无法看到 diff**，整个两阶段审查流程失效。

### 3.2 "Before You Begin" 段 —— 反 LLM 幻觉的关键

```
## Before You Begin

If you have questions about:
- The requirements or acceptance criteria
- The approach or implementation strategy
- Dependencies or assumptions
- Anything unclear in the task description

**Ask them now.** Raise any concerns before starting work.
```

**关键设计**：

- 4 类显式问题类别（需求/方法/依赖/其他）
- "Ask them now" —— 显式要求**工作开始前问**
- "Raise any concerns" —— 关注疑虑而非只关注事实

这是 LLM 协作中的**反幻觉关键**：

- LLM 倾向"先做再问" → 做完发现走错路
- 本 prompt 强制"先问再做" → 避免重做

**配合 SKILL.md 的"持续执行"原则**：任务开始前可以问（一次性提问），但开始后**不能反复打扰主代理**。

### 3.3 "While you work" 段 —— 二次澄清合法化

> **While you work:** If you encounter something unexpected or unclear, **ask questions**.
> It's always OK to pause and clarify. Don't guess or make assumptions.

这一段是"Before You Begin"的**延伸**：

- 工作前可以问
- 工作中**遇到意外时也可以问**
- 显式合法化"暂停澄清"

反 LLM 失败模式：

- LLM 倾向"猜" → 做出错误假设 → 走错路
- LLM 倾向"硬撑" → 错了也不说
- 本 prompt 显式否定这两种行为

**两段式"提问合法化"是 LLM 协作中的关键设计**——既不打断主代理（"持续执行"），又允许关键澄清（"问问题"）。

### 3.4 "Code Organization" 段 —— 反 LLM 重组倾向

> - Follow the file structure defined in the plan
> - Each file should have one clear responsibility with a well-defined interface
> - If a file you're creating is growing beyond the plan's intent, stop and report
>   it as DONE_WITH_CONCERNS — don't split files on your own without plan guidance
> - If an existing file you're modifying is already large or tangled, work carefully
>   and note it as a concern in your report
> - In existing codebases, follow established patterns. Improve code you're touching
>   the way a good developer would, but don't restructure things outside your task.

**5 条代码组织纪律**——每一条都对应一个 LLM 失败模式：

| 纪律 | 反 LLM 失败 |
|---|---|
| Follow the file structure defined in the plan | 反"擅自重新组织" |
| Each file should have one clear responsibility | 反"上帝类" |
| Stop and report as DONE_WITH_CONCERNS | 反"自作主张拆分文件" |
| Note large/tangled existing files as concern | 反"假装没看到烂代码" |
| Don't restructure things outside your task | 反"重构一切" |

**这是 LLM 协作中"克制纪律" 的最完整设计**。

### 3.5 "When You're in Over Your Head" 段 —— 反 LLM 硬撑

```
It is always OK to stop and say "this is too hard for me." Bad work is worse than
no work. You will not be penalized for escalating.
```

**这是一个大胆的 LLM 协作设计**——显式合法化**"我做不了"**：

- "this is too hard for me" —— 让 LLM 敢于承认局限
- "Bad work is worse than no work" —— 反对 LLM 硬撑
- "You will not be penalized for escalating" —— 反对"惩罚诚实"的文化

**5 条 STOP 触发条件**：

```
- The task requires architectural decisions with multiple valid approaches
- You need to understand code beyond what was provided and can't find clarity
- You feel uncertain about whether your approach is correct
- The task involves restructuring existing code in ways the plan didn't anticipate
- You've been reading file after file trying to understand the system without progress
```

5 条具体可识别的"力不从心"信号，**比"你觉得难就停下" 更可操作**。

**升级路径**（3 条）：

- 主代理提供更多上下文
- 用更强模型重派
- 任务拆分为更小部分

### 3.6 "Self-Review" 4 维度

```
**Completeness:**
- Did I fully implement everything in the spec?
- Did I miss any requirements?
- Are there edge cases I didn't handle?

**Quality:**
- Is this my best work?
- Are names clear and accurate (match what things do, not how they work)?
- Is the code clean and maintainable?

**Discipline:**
- Did I avoid overbuilding (YAGNI)?
- Did I only build what was requested?
- Did I follow existing patterns in the codebase?

**Testing:**
- Do tests actually verify behavior (not just mock behavior)?
- Did I follow TDD if required?
- Are tests comprehensive?
```

**4 维度自审**比单维度的"做完了吗"更全面：

| 维度 | 关注点 | 反 LLM 失败 |
|---|---|---|
| Completeness | 完整性 | 反"漏掉需求" |
| Quality | 质量 | 反"代码草率" |
| Discipline | 纪律 | 反"过度构建/YAGNI 违反" |
| Testing | 测试 | 反"假测试/mocks" |

**YAGNI（You Aren't Gonna Need It）**作为显式纪律点出 —— LLM 倾向"未来会用到" 而增加未请求功能。

**关键的最后一句**：

> If you find issues during self-review, fix them now before reporting.

**自审发现问题 → 立即修复**——不要带着已知问题上报。

### 3.7 "Report Format" —— 4 状态 + 5 字段

```
- **Status:** DONE | DONE_WITH_CONCERNS | BLOCKED | NEEDS_CONTEXT
- What you implemented (or what you attempted, if blocked)
- What you tested and test results
- Files changed
- Self-review findings (if any)
- Any issues or concerns
```

**4 状态**（结构化反馈）：
- DONE
- DONE_WITH_CONCERNS
- BLOCKED
- NEEDS_CONTEXT

**5 字段**（具体内容）：
- 实施了/尝试了什么
- 测试了什么 + 结果
- 修改了哪些文件
- 自审发现
- 任何疑虑

**最后一句的关键约束**：

> Use DONE_WITH_CONCERNS if you completed the work but have doubts about correctness.
> Use BLOCKED if you cannot complete the task. Use NEEDS_CONTEXT if you need
> information that wasn't provided. **Never silently produce work you're unsure about.**

**"Never silently produce work you're unsure about"** 是**整个 prompt 中最关键的一句**——它**显式禁止 LLM 的"沉默错误"** 失败模式。

## 四、文件结构

文件结构清晰，9 段：

```
1. 标题 + 用途
2. Task tool 分派模板
3. Task Description（占位符）
4. Context（场景设置）
5. Before You Begin（工作前提问）
6. Your Job（6 步流程）
7. Code Organization（5 条纪律）
8. When You're in Over Your Head（升级路径）
9. Before Reporting Back: Self-Review（4 维度自审）
10. Report Format（4 状态 + 5 字段）
```

## 五、优点

### 5.1 "提问合法化" 的两段式设计

- **Before You Begin**：工作前问
- **While you work**：工作中遇到意外问

两段式设计让 LLM **既敢问、又不打扰**主代理。

### 5.2 "Code Organization" 5 条纪律的完整性

5 条纪律精准覆盖 LLM 在代码组织上的 5 大失败模式：

- 重新组织
- 上帝类
- 自作主张拆分
- 假装没看到烂代码
- 重构一切

这种**纪律清单**是 LLM 协作中反"放任 LLM 自由发挥"的关键。

### 5.3 "When You're in Over Your Head" 的诚实合法化

> It is always OK to stop and say "this is too hard for me."

**这是 LLM 协作中"反硬撑" 的关键设计**：

- LLM 倾向"假装能做"
- LLM 倾向"硬撑出半成品"
- 本 prompt 显式鼓励**诚实承认局限**

配合 5 条具体 STOP 信号，让"承认局限"有据可循。

### 5.4 "Self-Review" 4 维度 + 立即修复

4 维度自审 + "If you find issues during self-review, fix them now before reporting"：

- 自审不是"记录问题"——是"立即修复"
- 让 LLM **把自审作为"最后一次检查"** 而非"走过场"

### 5.5 "Report Format" 4 状态的结构化反馈

4 状态是**结构化反馈**——主代理可以**程序化**处理：

- DONE → 进入规范审查
- DONE_WITH_CONCERNS → 读疑虑再决定
- NEEDS_CONTEXT → 提供上下文
- BLOCKED → 选择升级路径

配合 SKILL.md 的"4 状态处理" 段，形成**完整的子代理反馈协议**。

### 5.6 "Never silently produce work you're unsure about"

最后一句是**整个 prompt 中最反 LLM 失败的一句**：

- 反"沉默错误"
- 反"假装能做"
- 反"硬撑出半成品"

这一句把"诚实"作为**硬约束**。

## 六、缺点 / 风险

### 6.1 缺乏"提问何时停止"的边界

"提问合法化"是好事，但**没有说"什么算足够问题"**：

- 多少问题算"过度提问"？
- 一个问题问几次算"卡住"？
- 提问的"成本"如何评估？

可以补充：

- "Aim to ask all questions in one batch, not drip-feed"
- "After 3 questions with no answer, escalate"

### 6.2 缺乏"测试密度"的指导

"Write tests (following TDD if task says to)" 缺乏测试密度的指导：

- 多少测试算"充分"？
- 单元测试 vs 集成测试比例？
- Mock vs 真测试的判断？

虽然 `test-driven-development` 技能有详细指导，但本 prompt **没重申**。

### 6.3 "Self-Review" 的局限性未说明

4 维度自审看似全面，但 LLM 自我审查有固有局限：

- 验证者偏差（自己发现不了自己的问题）
- LLM 倾向"对自己的工作评价偏高"
- 自审通过 ≠ 实际通过

应该补充：

- "Self-review is informational, not authoritative"
- "Always follow with actual review (spec + code quality)"

### 6.4 缺乏"任务规模 vs 实施者能力"的匹配

"5 条 STOP 信号" 没有显式说明：

- 任务太复杂怎么办？→ 应当拆分
- 任务对实施者模型来说太难怎么办？→ 应当升级模型
- 这两者如何在分派前评估？

虽然 SKILL.md 中提到"模型分级"，但本 prompt **没有**告诉实施者"如果你觉得你处理不了，应当主动报告"。

### 6.5 "Work from: [directory]" 占位符不明确

主代理在填充占位符时，可能给出模糊的工作目录：

- 是 git 仓库根目录？
- 是 worktree？
- 是特定子目录？

可以补充：

- "If working in a worktree, [directory] = absolute path to worktree root"

### 6.6 "DONE_WITH_CONCERNS" 的疑虑可能过于宽泛

"DONE_WITH_CONCERNS" 是好的设计，但**没说什么算"concern"**：

- 大的疑虑（架构问题）→ 升级
- 小的疑虑（"文件变大了"）→ 记录即可
- 中间地带（"我不确定测试覆盖是否足够"）→ 模糊

可以补充疑虑分级。

### 6.7 缺乏"提交粒度"的指导

"Commit your work" 没说怎么提交：

- 整个任务一次提交？
- 多个原子提交？
- 提交信息应当包含什么？

可以补充：

- "Commit with descriptive message including task reference"
- "Prefer atomic commits per logical change"

## 七、关键洞察

### 7.1 这是 Superpowers 项目的"实施者侧" Prompt

整个项目有 4 个核心子代理 prompt：

- **`implementer-prompt.md`（本文件）** —— 实施者
- `spec-reviewer-prompt.md` —— 规范审查
- `code-quality-reviewer-prompt.md` —— 代码质量审查
- `code-reviewer.md`（requesting-code-review 目录下） —— 最终代码审查

**本文件是"实施侧"的入口**——它定义了 LLM 子代理在"做事"时的行为准则。

### 7.2 "诚实承认局限"是 LLM 协作的关键

> It is always OK to stop and say "this is too hard for me."

**这是 LLM 协作中最稀缺也最重要的设计**：

- LLM 倾向"假装能做"
- LLM 倾向"硬撑出半成品"
- 本 prompt 显式合法化"承认局限"

配合 `SKILL.md` 的 4 状态处理（特别是 BLOCKED 的 4 种升级路径），形成**"诚实→升级→解决"**的完整闭环。

### 7.3 "Never silently produce work you're unsure about"

**这是整个 prompt 中最反 LLM 失败的一句**——它把"诚实" 作为**硬约束**：

- 反"沉默错误"
- 反"假装能做"
- 反"硬撑出半成品"

可以推广到所有 LLM 协作 prompt：**显式禁止"沉默的不确定"**。

### 7.4 "Code Organization" 5 条纪律的反 LLM 自由倾向

LLM 倾向"自由发挥"——自作主张拆分文件、重构一切、创造上帝类。

5 条纪律精准打击这种倾向：

- Follow the plan's file structure
- One responsibility per file
- Don't split without plan guidance
- Note concerns about existing code
- Don't restructure outside task

这种**纪律清单**是 LLM 协作中"反放任"的关键。

### 7.5 "Self-Review" 4 维度的全面性

4 维度（Completeness/Quality/Discipline/Testing）比单维度的"做完了吗"更全面：

- 不仅看"做没做"
- 还看"做得好不好"
- 还看"做得克制不克制"
- 还看"测得真不真"

**4 维度形成"自审的 4 象限"**——任何一个象限缺失都意味着工作未完成。

## 八、关联文档

### 8.1 同目录

- `SKILL.md`（[subagent-driven-development/SKILL.md](file:///e:/WorkStation/MyProject/superpowers/skills/subagent-driven-development/SKILL.md)）—— 调度方主技能
- `spec-reviewer-prompt.md`（[subagent-driven-development/spec-reviewer-prompt.md](file:///e:/WorkStation/MyProject/superpowers/skills/subagent-driven-development/spec-reviewer-prompt.md)）—— 规范审查
- `code-quality-reviewer-prompt.md`（[subagent-driven-development/code-quality-reviewer-prompt.md](file:///e:/WorkStation/MyProject/superpowers/skills/subagent-driven-development/code-quality-reviewer-prompt.md)）—— 代码质量审查

### 8.2 配套技能

- `test-driven-development/SKILL.md` —— TDD 规范
- `writing-plans/SKILL.md` —— 计划来源
- `using-git-worktrees/SKILL.md` —— 工作区隔离

### 8.3 母技能

- `using-superpowers/SKILL.md` —— 12 条红旗

## 九、状态评价

**成熟度**：✅ **生产可用 + 核心模板**

- 结构清晰，纪律强
- 反 LLM 失败设计成熟
- 已经在项目中被实际使用

**完备度**：⭐⭐⭐⭐ (4/5)

- 核心机制完整
- 缺少"提问何时停止" 的边界
- 缺少"Self-Review 局限性" 的说明
- 缺少"提交粒度" 的指导

**风险等级**：🟡 中

- 主要风险是 LLM 实施者"硬撑"或"沉默错误"
- 4 状态反馈 + 升级路径 部分缓解
- 但仍需要主代理具备"识别异常报告"的能力

## 十、给读者的启示

### 10.1 启示 1：提问合法化要"两段式"

LLM 协作中的提问应当"两段式"：

- **工作前**：批量提问（一次性问完）
- **工作中**：遇到意外再问（少量、关键）

**两段式设计让 LLM 既敢问、又不打扰**。

### 10.2 启示 2：诚实承认局限要显式合法化

LLM 协作中最稀缺的是**诚实承认局限**。显式写：

- "It is always OK to stop and say 'this is too hard for me.'"
- "Bad work is worse than no work."
- "You will not be penalized for escalating."

让 LLM **敢于承认**自己做不到，比"让它硬撑" 更有效。

### 10.3 启示 3：代码组织纪律要清单化

LLM 协作中"反放任 LLM 自由发挥"需要**纪律清单**：

- Follow the plan
- One responsibility per file
- Don't split without guidance
- Note concerns
- Don't restructure outside task

每一条都对应 LLM 的具体失败模式。

### 10.4 启示 4：自审要"立即修复"而非"记录问题"

LLM 自审的设计要点是**发现就修**：

- "If you find issues during self-review, fix them now before reporting."
- 不要让"自审" 变成"问题清单"——要让它变成"修复完成"

### 10.5 启示 5：4 状态反馈是 LLM 协作的标准协议

LLM 子代理的反馈应当结构化为 4 状态：

- DONE
- DONE_WITH_CONCERNS
- NEEDS_CONTEXT
- BLOCKED

**主代理根据状态选择不同处理路径**。这是 LLM 协作中的**反馈协议标准**。

### 10.6 启示 6："Never silently produce work" 是 LLM 协作的硬约束

> Never silently produce work you're unsure about.

这一句应当**显式出现在所有 LLM 实施 prompt 中**——它是**反 LLM 沉默错误**的关键硬约束。

## 十一、后续工作

### 11.1 可能的改进

1. **添加"提问何时停止"的边界**
   - 一次性批量提问
   - 3 个问题无答案后升级

2. **添加"测试密度"指导**
   - 重申 TDD 的具体要求
   - 测试覆盖目标（如 80%+ line coverage）

3. **添加"Self-Review 局限性"说明**
   - 显式声明自审是信息性
   - 仍然需要外部审查

4. **添加"提交粒度"指导**
   - 原子提交原则
   - 提交信息规范

5. **添加"疑虑分级"**
   - 大的疑虑（架构问题）→ 升级
   - 小的疑虑（"文件变大了"）→ 记录

### 11.2 监控点

- 4 状态的实际分布（BLOCKED 异常多 → 计划问题）
- 提问频率（异常多 → 主代理上下文构建有问题）
- Self-review 发现的频率（异常多 → 实施质量问题）
- "Never silently produce work" 实际触发频率

### 11.3 长期演进

- 引入"实施者能力画像"（基于历史任务成功率）
- 引入"任务复杂度预测"（让主代理提前选择合适模型）
- 与 `verification-before-completion` 串联形成"自审→外审→验证"三道关卡
