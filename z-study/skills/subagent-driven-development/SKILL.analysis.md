# subagent-driven-development/SKILL.md 分析报告

## 一、基本信息

- **文件路径**：`skills/subagent-driven-development/SKILL.md`
- **文件大小**：280 行
- **文档类型**：技能主文档（带 YAML frontmatter）
- **配套文件**：3 个 prompt 模板
  - `implementer-prompt.md`（实施者 prompt）
  - `spec-reviewer-prompt.md`（规范审查者 prompt）
  - `code-quality-reviewer-prompt.md`（代码质量审查者 prompt）

## 二、目的

该技能是 Superpowers 项目的"核心工作流引擎"，定义**同会话下的子代理驱动开发模式**：

- **每个任务一个全新的子代理** —— 避免上下文污染
- **两阶段审查** —— 规范符合性 → 代码质量
- **持续执行** —— 不打断 human

它把"如何拆解 + 实施 + 审查一个实施计划" 系统化为可重复的流程。

## 三、核心技术决策

### 3.1 "每个任务一个全新子代理" —— 上下文隔离原则

> Fresh subagent per task + two-stage review (spec then quality) = high quality, fast iteration

这是整个技能最核心的架构决策：

- **为什么"全新"？** 子代理**绝不继承**主代理的会话历史。每个子代理只拿到**主代理精心构建的精确上下文**（任务文本、场景设置、依赖）。
- **好处**：
  - 避免上下文污染（实施者看不到主代理的试错/妥协）
  - 每个子代理**专注**当前任务（不被过去任务干扰）
  - 主代理自己的上下文不被消耗（保持协调能力）

这是 LLM 协作中**最具原则性的设计**——它**严格区分"调度者"和"执行者"的角色**。

### 3.2 两阶段审查的"先规范后质量"

```dot
Spec compliance review → Code quality review
```

**两阶段顺序不能颠倒**：

1. **规范符合性审查**（先）：检查实施者**是否做了要求的事**（nothing more, nothing less）
2. **代码质量审查**（后）：在确认符合规范后，再检查**做得好不好**（clean, tested, maintainable）

**为何必须先规范？**

- 如果先看代码质量 → 实施者可能为了让代码"漂亮"而增加未请求的功能
- 规范符合性是**正确性的门控** → 先确认"对"再评"好"
- 反之：先评"好"再评"对" → 审查者可能容忍错误以维持"代码质量"

红旗中显式说："**Start code quality review before spec compliance is ✅** (wrong order)" —— 这是强约束。

### 3.3 持续执行（Continuous execution）

> Do not pause to check in with your human partner between tasks. Execute all tasks from the plan without stopping.

这是一个**反 LLM 焦虑**的设计：

- LLM 倾向"汇报进度" → 浪费 human 时间
- LLM 倾向"我应该继续吗？" → 制造中断
- 文档显式否定这种行为："'Should I continue?' prompts and progress summaries waste their time"

**停下来的唯一正当理由**：
1. BLOCKED 状态（你无法解决）
2. 真正阻碍进展的歧义
3. 所有任务完成

这种"反中断"原则让 LLM **保持工作流连贯**，避免 LLM 的"讨好式中断"。

### 3.4 实施者 4 状态系统

| 状态 | 含义 | 处理 |
|---|---|---|
| **DONE** | 任务完成 | 直接进入规范审查 |
| **DONE_WITH_CONCERNS** | 完成但有疑虑 | 主代理判断疑虑类型，区别处理 |
| **NEEDS_CONTEXT** | 缺少信息 | 主代理提供上下文后重派 |
| **BLOCKED** | 无法完成 | 4 种重试策略（按顺序） |

**设计亮点**：

- **DONE_WITH_CONCERNS** 是关键设计：LLM 倾向"假装完成"，这个状态让"完成但有疑虑"显式合法
- **BLOCKED** 的 4 种处理策略显式列举：上下文/能力/任务规模/计划错误 → 让主代理**有据可循**地选择

**红旗**：

> **Never** ignore an escalation or force the same model to retry without changes. If the implementer said it's stuck, something needs to change.

这一条直接对抗 LLM 的"重试循环"失败模式。

### 3.5 "模型分级使用" —— 成本与能力平衡

> Use the least powerful model that can handle each role to conserve cost and increase speed.

三类任务 → 三种模型：

| 任务类型 | 模型选择 | 理由 |
|---|---|---|
| 机械实现 | 便宜/快 | 1-2 文件、清晰规范，LLM 推理负担小 |
| 集成/判断 | 标准 | 多文件协调、模式匹配、调试 |
| 架构/设计/审查 | 最强 | 需要深度理解和判断 |

**这种"模型分级"是工程上的关键优化**——不是所有任务都需要 GPT-4/Claude Opus。

### 3.6 决策图（Graphviz）让选择流程可视化

文档用了 2 个 Graphviz 图：

**(a) when_to_use**：决定用 `subagent-driven-development` 还是 `executing-plans` 还是手动

**(b) process**：每个任务的完整流程（实施→规范审查→质量审查→循环）

Graphviz 选择的理由（同 `using-superpowers`）：

- **比纯文本更清晰**：决策路径一眼可见
- **比表格更可操作**：箭头带 label
- **LLM 解析友好**：图结构化比段落更易推理

### 3.7 红旗清单（12 条绝对不要）

```
- Start implementation on main/master branch without explicit user consent
- Skip reviews (spec compliance OR code quality)
- Proceed with unfixed issues
- Dispatch multiple implementation subagents in parallel (conflicts)
- Make subagent read plan file (provide full text instead)
- Skip scene-setting context (subagent needs to understand where task fits)
- Ignore subagent questions (answer before letting them proceed)
- Accept "close enough" on spec compliance
- Skip review loops
- Let implementer self-review replace actual review
- Start code quality review before spec compliance is ✅
- Move to next task while either review has open issues
```

**12 条红旗**是 LLM 协作中**最危险的失败模式清单**：

- 第 1 条：保护 main 分支
- 第 4 条：避免并行冲突
- 第 5 条：避免子代理读文件浪费 token
- 第 6 条：避免子代理缺上下文
- 第 9-10 条：避免审查循环失效
- 第 11 条：避免审查顺序错误
- 第 12 条：避免带病推进

每条都是 LLM 协作中**真实发生过的失败**。

## 四、文件结构

文件结构紧凑，9 段：

```
1. YAML frontmatter
2. 引言 + 核心原则 + 持续执行原则
3. 何时使用（含 Graphviz when_to_use 图）
4. 流程（含 Graphviz process 图，cluster_per_task）
5. 模型选择（3 类任务 → 3 种模型）
6. 处理实施者状态（4 状态处理）
7. Prompt 模板（指向 3 个 .md）
8. 示例工作流（Task 1/2 + 完整流程）
9. 优势（4 维度对比）
10. 红旗（12 条绝对不要）
11. 集成（4 个必需技能 + 1 个子代理技能 + 1 个备选）
```

## 五、优点

### 5.1 极致的上下文隔离纪律

整个技能严格遵循"子代理不继承主代理历史"的原则：

- "Fresh subagent per task" 在核心原则中显式声明
- "Never inherit your session's context" 在引言中显式说明
- "Make subagent read plan file" 列入红旗（让主代理传完整文本）

这种**纪律性**让子代理协作真正有效。

### 5.2 严格的"先规范后质量"审查顺序

两阶段审查的顺序是**架构级**的硬约束：

- 规范符合性 = 正确性门控
- 代码质量 = 质量门控
- 顺序不能颠倒（红旗中显式列出）

这种**先后顺序的硬约束**是 LLM 协作中**反 LLM 主观判断的关键**。

### 5.3 完整的 4 状态处理系统

DONE / DONE_WITH_CONCERNS / NEEDS_CONTEXT / BLOCKED 形成**完整的子代理反馈协议**：

- 状态本身是结构化数据
- 每种状态的处理是显式的
- 主代理**有据可循**地决定下一步

### 5.4 显式的"持续执行"反中断原则

> Do not pause to check in with your human partner between tasks.

这一段直击 LLM 协作中的"讨好式中断"问题。**让 LLM 持续工作是工作流的关键**。

### 5.5 完整的工作流集成图

`Integration` 段把本技能放到 Superpowers 项目的整体工作流中：

- 上游：writing-plans（创建计划）
- 下游：finishing-a-development-branch（完成任务）
- 横向：using-git-worktrees（隔离工作区）
- 子代理侧：test-driven-development（TDD）
- 备选：executing-plans（并行会话）

**这种显式集成让用户清楚本技能的"上下游"**。

## 六、缺点 / 风险

### 6.1 缺乏"两阶段审查循环终止条件"

文档说"重复直到通过"，但**没有终止条件**：

- 如果规范审查总是失败？→ 任务可能根本不该由该实施者做
- 如果代码质量审查循环 5 次仍不通过？→ 应当升级到 human
- 循环的最大次数/总成本上限？

可以补充：

- "If review loops exceed N, escalate to human"
- "If spec compliance fails twice, re-think task design"

### 6.2 缺乏"并行 vs 串行"的明确决策

文档说"绝不并行分派多个实施子代理" —— 但**没有说"哪些任务可以并行"**：

- 完全独立的两个文件？→ 可以并行
- 共享一个公共接口？→ 不能并行
- 同一个文件的两个函数？→ 不能并行

可以补充并行决策图。

### 6.3 "持续执行"原则的风险

> Do not pause to check in with your human partner between tasks.

这条原则在 LLM 协作中**有效但有风险**：

- 如果计划本身错了 → LLM 持续推进但全部做错
- 如果任务间有依赖但没显式说明 → 后续任务会基于错误前提
- 如果 LLM 反复重试失败任务 → 浪费 token

可以补充"硬中断条件"：

- "Stop if 3+ tasks in a row fail"
- "Stop if the same BLOCKED status repeats"
- "Stop if you find the plan has fundamental issues"

### 6.4 "实施者子代理提问"的可能过多

文档鼓励"实施者提问"（before and during work）：

- 优势：澄清需求
- 风险：每个任务都问 → 主代理变成"人工应答器"
- 风险：问的问题不必要 → 浪费 token

可以补充"该不该问"的判断标准。

### 6.5 "Self-review" 的有效性假设

实施者 prompt 中要求"自我审查"（Completeness/Quality/Discipline/Testing）：

- LLM 自我审查**容易发现不了自己的问题**（验证者偏差）
- 自我审查的输出**应当不被信任**作为最终结果
- 但主代理可能误把"自审通过"当作"实施通过"

可以补充：

- "Self-review is informational, not authoritative"
- "Always follow with actual review, never skip"

### 6.6 缺乏"实施者自审失败的处理"

如果实施者自审发现"漏掉了 --force 标志"——文档**没说什么**：

- 实施者已经修了 → 仍然需要外部审查吗？
- 实施者自审 vs 外部审查的成本对比？
- 是不是要分派"修复子代理"而非"原实施者修复"？

可以补充"自审 + 修复后"的处理流程。

### 6.7 模型分级缺乏"具体模型名"

文档说"便宜/标准/最强"——但**没给具体建议**：

- 在 OpenAI 生态：gpt-4o-mini / gpt-4o / o1？
- 在 Anthropic 生态：haiku / sonnet / opus？
- 在开源生态：哪个本地模型？

可以补充**平台特定的模型映射**（类似 `using-superpowers/references/codex-tools.md` 的设计）。

### 6.8 缺乏"任务粒度"的指导

文档没有说明**任务应该多细**：

- 1 个函数 = 1 个任务？
- 1 个完整功能 = 1 个任务？
- 1 个 PR = 1 个任务？

虽然 `writing-plans/SKILL.md` 给出了"2-5 分钟粒度"的指导，但本技能**没重申**。

## 七、关键洞察

### 7.1 这是 Superpowers 项目的"工作流核心"

整个项目有多个"开发"相关技能：

- `writing-plans` —— 写计划
- **`subagent-driven-development` —— 本文件（执行计划）**
- `executing-plans` —— 并行执行
- `finishing-a-development-branch` —— 完成开发

**本技能是"计划→实施"的桥梁**，是 Superpowers 项目工作流的最核心节点。

### 7.2 "上下文隔离"是 LLM 协作的第一原则

整个技能的所有设计都围绕"上下文隔离"展开：

- 子代理不继承主代理历史
- 主代理提供完整任务文本
- 实施者有独立的"提问"机会
- 审查者只看 git diff，不看主代理的思考

这种**"调度者—执行者"严格分离**是 LLM 协作成功的关键。

### 7.3 "两阶段审查"是反 LLM 偏好的关键

为什么必须"先规范后质量"？

- 如果同时审 → 审查者倾向"代码不错就放过"（质量补偿规范）
- 如果先审质量 → 实施者会为了"漂亮"而增加未请求功能
- 如果先审规范 → 实施者被迫"严格按需求做" → 再看质量才有意义

这种**审查顺序**是 LLM 协作中的**反偏差关键设计**。

### 7.4 "持续执行"是反讨好关键

LLM 倾向"汇报、询问、确认"——这些"讨好式行为"浪费 human 时间。

> Do not pause to check in with your human partner between tasks.

**这条原则显式对抗 LLM 的讨好行为**。它把"持续工作"作为工作流节点的默认状态。

### 7.5 决策图（Graphviz）作为复杂流程的呈现方式

本文件用了 2 个 Graphviz 图：

- `when_to_use` —— 选择本技能 vs 替代技能
- `process` —— 完整的子代理驱动流程

这种**图形化流程** 比纯文本段落**更易推理**（LLM 和 human 都受益）。

### 7.6 "模型分级"是 LLM 协作的工程优化

LLM 调用成本是真实问题。文档显式说"用最弱模型能处理的角色"：

- 机械实现 → 便宜模型
- 集成判断 → 标准模型
- 架构审查 → 最强模型

这种**任务-模型匹配**是 Superpowers 项目的工程优化亮点。

## 八、关联文档

### 8.1 同目录（3 个 prompt 模板）

- `implementer-prompt.md`（[subagent-driven-development/implementer-prompt.md](file:///e:/WorkStation/MyProject/superpowers/skills/subagent-driven-development/implementer-prompt.md)）—— 实施者 prompt
- `spec-reviewer-prompt.md`（[subagent-driven-development/spec-reviewer-prompt.md](file:///e:/WorkStation/MyProject/superpowers/skills/subagent-driven-development/spec-reviewer-prompt.md)）—— 规范审查者 prompt
- `code-quality-reviewer-prompt.md`（[subagent-driven-development/code-quality-reviewer-prompt.md](file:///e:/WorkStation/MyProject/superpowers/skills/subagent-driven-development/code-quality-reviewer-prompt.md)）—— 代码质量审查者 prompt

### 8.2 上游 / 下游技能

- `writing-plans/SKILL.md` —— 创建计划（本技能执行）
- `using-git-worktrees/SKILL.md` —— 隔离工作区
- `requesting-code-review/SKILL.md` —— 最终代码审查
- `finishing-a-development-branch/SKILL.md` —— 完成开发
- `test-driven-development/SKILL.md` —— TDD（子代理应当遵循）
- `executing-plans/SKILL.md` —— 备选工作流

### 8.3 母技能

- `using-superpowers/SKILL.md` —— 12 条红旗

### 8.4 测试

- `tests/subagent-driven-development.sh` —— 集成测试
- `tests/subagent-driven-dev/` —— 真实项目测试（go-fractals, svelte-todo）
- `tests/explicit-skill-requests/prompts/subagent-driven-development-please.txt`

## 九、状态评价

**成熟度**：✅ **生产可用 + 核心引擎**

- 文档结构清晰、约束强
- 反 LLM 偏好设计成熟
- 已经在项目中被实际使用且有完整测试覆盖
- 是 Superpowers 项目工作流的最核心节点

**完备度**：⭐⭐⭐⭐ (4/5)

- 核心机制完整
- 缺少"审查循环终止条件"
- 缺少"持续执行的硬中断条件"
- 缺少"模型分级的平台特定映射"
- 缺少"任务粒度"的明确指导

**风险等级**：🟡 中

- 主要风险是"持续执行"原则的滥用
- LLM 可能基于错误前提持续推进
- 需要 human 监督作为"安全网"

## 十、给读者的启示

### 10.1 启示 1：上下文隔离是 LLM 协作的第一原则

**子代理绝不继承主代理历史**。这是 LLM 协作的核心原则：

- 主代理：调度 + 协调
- 子代理：执行具体任务
- 中间的"上下文"由主代理**精确构建**

不遵守这条原则 → LLM 协作必然失败。

### 10.2 启示 2：审查必须有"先 X 后 Y" 的顺序

LLM 审查的"两阶段顺序"是反偏差的关键：

- 规范符合性 → 正确性门控
- 代码质量 → 质量门控
- 顺序不能颠倒

可以推广到所有"审查"场景：**先把"对不对"判断完，再评"好不好"**。

### 10.3 启示 3：LLM 反馈应当结构化（4 状态）

子代理的反馈应当是**结构化状态**：

- DONE
- DONE_WITH_CONCERNS
- NEEDS_CONTEXT
- BLOCKED

每种状态对应不同的处理路径。**不要让 LLM 自由文本报告状态**——主代理无法程序化处理。

### 10.4 启示 4："持续执行"是反讨好关键

LLM 倾向"汇报 + 询问 + 确认"——这些都是讨好行为。

**显式写明"不要在任务之间暂停"** 是反讨好关键：

- 让 LLM 持续工作
- 中断只用于 BLOCKED 或 ambiguity
- 不需要"Should I continue?" 提示

### 10.5 启示 5：模型分级是 LLM 工程的成本优化

**不是所有任务都需要最强模型**：

- 机械实现 → 便宜模型
- 集成判断 → 标准模型
- 架构审查 → 最强模型

任务-模型匹配是 LLM 协作中的工程优化基础。

### 10.6 启示 6：Graphviz 是 LLM 友好的复杂流程呈现

复杂的 LLM 协作流程应当用**图形**（Graphviz）而非纯文本：

- 决策路径清晰
- LLM 解析友好
- human 阅读友好

## 十一、后续工作

### 11.1 可能的改进

1. **添加"审查循环终止条件"**
   - 显式规定最大循环次数
   - 超出后升级到 human

2. **添加"持续执行的硬中断条件"**
   - 连续 N 个任务失败 → 停止
   - 同一 BLOCKED 重复出现 → 停止
   - 发现计划有根本问题 → 停止

3. **添加"平台特定模型映射"**
   - OpenAI / Anthropic / 开源 各自的模型分级
   - 类似 `using-superpowers/references/codex-tools.md` 的设计

4. **添加"任务粒度"指导**
   - 重申 writing-plans 的"2-5 分钟粒度"
   - 显式说明"任务太细/太粗"的判断

5. **添加"实施者自审的局限性"说明**
   - Self-review 是信息性，不是权威
   - 仍然需要外部审查

### 11.2 监控点

- 审查循环的平均次数（异常多 → 实施质量下降）
- BLOCKED 状态频率（异常多 → 计划/任务设计问题）
- DONE_WITH_CONCERNS 的疑虑类型分布
- "持续执行"中断频率（异常多 → 中断条件过于宽松）

### 11.3 长期演进

- 引入"自动 LLM 实施者能力评估"（基于历史任务成功率）
- 引入"任务复杂度预测"（让主代理提前选择合适模型）
- 与 `brainstorming`、`writing-plans` 形成完整的"想法→计划→实施" 串联
