# code-quality-reviewer-prompt.md 分析报告

## 一、基本信息

- **文件路径**：`skills/subagent-driven-development/code-quality-reviewer-prompt.md`
- **文件大小**：26 行
- **文档类型**：子代理 Prompt 模板（**极简型**，非完整 prompt）
- **配套主文件**：`SKILL.md`（子代理驱动开发）
- **使用方式**：分派代码质量审查者时使用（**仅在规范符合性审查通过后**）
- **实际 prompt 来源**：复用 `requesting-code-review/code-reviewer.md` 模板

## 二、目的

这是子代理驱动开发中**两阶段审查的第二阶段**——代码质量审查。它的设计哲学是：

> **规范符合性** 解决"对不对"的问题；**代码质量** 解决"好不好"的问题。两者顺序不能颠倒。

本文件**不**复制完整的 prompt 模板，而是**引用** `requesting-code-review/code-reviewer.md` 模板，**只附加 4 项额外的代码质量检查项**。

这种"**复用 + 增强**"的设计哲学让：

- 代码质量审查的**基础结构**（Strengths/Issues/Assessment）由 `code-reviewer.md` 提供
- 子代理驱动开发的**专项检查**（文件结构/单元分解）由本文件提供

## 三、核心技术决策

### 3.1 "复用 + 增强" 模式

```
Task tool (general-purpose):
  Use template at requesting-code-review/code-reviewer.md
```

**关键设计**：本文件**不重新定义**代码审查 prompt，而是**复用** `requesting-code-review/code-reviewer.md` 的标准模板，**只附加 4 项专项检查**。

这种"复用 + 增强"的设计优势：

- **避免重复**：不复制粘贴相同的代码审查 prompt
- **一致性**：所有"代码审查"都使用同一基础模板
- **专业化**：本文件只在基础模板上附加**子代理驱动开发的特定关注点**

### 3.2 "Only dispatch after spec compliance passes" 强顺序约束

> **Only dispatch after spec compliance review passes.**

**这是整个 prompt 的"前置条件硬约束"**——它强制：

- 规范符合性审查未通过 → 不进入代码质量审查
- 即使代码质量看起来"好"，也不跳到代码质量审查

这是 SKILL.md 中"两阶段顺序"的**具体执行点**：

```
实施者完成 → [规范符合性审查 ✅] → [代码质量审查 ✅] → 任务完成
              ↑ 必须先过                ↑ 本文件
```

**反 LLM 失败模式**：

- LLM 倾向"同时看规范和质量"（一锅端）
- LLM 倾向"代码不错就放过"（质量补偿规范）
- 本 prompt 显式禁止这种"同时审查"

### 3.3 4 项专项检查

> **In addition to standard code quality concerns, the reviewer should check:**
> - Does each file have one clear responsibility with a well-defined interface?
> - Are units decomposed so they can be understood and tested independently?
> - Is the implementation following the file structure from the plan?
> - Did this implementation create new files that are already large, or significantly grow existing files? (Don't flag pre-existing file sizes — focus on what this change contributed.)

**4 项专项检查**对应**子代理驱动开发的特定关注点**：

| 检查项 | 关注点 | 反 LLM 失败 |
|---|---|---|
| One clear responsibility per file | 关注点分离 | 反"上帝类" |
| Units decomposed for independent understanding/testing | 单元可独立测试 | 反"耦合代码" |
| Following the file structure from the plan | 计划合规 | 反"擅自重组" |
| New large files or significantly grown files | 增量大小 | 反"巨文件" |

**第 4 项的精妙**：

> (Don't flag pre-existing file sizes — focus on what this change contributed.)

**只关注本次变更的贡献，不标记先前就存在的大小**——这避免了 LLM 审查代理把"老问题" 算在"新任务" 头上。

### 3.4 4 项专项与 `implementer-prompt.md` 的纪律呼应

4 项专项检查**与 `implementer-prompt.md` 中的"Code Organization" 5 条纪律精确呼应**：

| `code-quality-reviewer-prompt.md` 检查项 | `implementer-prompt.md` 对应纪律 |
|---|---|
| One clear responsibility per file | "Each file should have one clear responsibility" |
| Units decomposed for independent testing | "your edits are more reliable when files are focused" |
| Following the file structure from the plan | "Follow the file structure defined in the plan" |
| New large files or significantly grown files | "If a file you're creating is growing beyond the plan's intent" |

**这种"实施纪律 + 审查纪律"的精确呼应**是子代理驱动开发的关键设计：

- 实施 prompt 告诉实施者**"不要做 X"**
- 审查 prompt 告诉审查者**"检查是否做了 X"**
- 两者形成**纪律闭环**

### 3.5 复用 `code-reviewer.md` 的 5 大维度

> Use template at requesting-code-review/code-reviewer.md

`code-reviewer.md` 中的 5 大维度（Plan alignment / Code quality / Architecture / Testing / Production readiness）被全部继承。本文件**只附加 4 项专项检查**。

**Plan alignment 维度的特别价值**：在子代理驱动开发中，**"是否对齐计划"是质量的一部分**——这与 SKILL.md 中的"两阶段审查"中"先规范后质量" 的顺序看似重复，**但关注点不同**：

- **规范符合性审查**：是否做了要求的事（nothing more, nothing less）
- **代码质量审查的 Plan alignment**：是否按计划的方式做了（architecture / file structure）

**两者是"内容合规"与"形式合规"的区别**。

## 四、文件结构

文件结构极致紧凑，5 段：

```
1. 标题 + 用途
2. 目的 + 强顺序约束
3. Task tool 分派模板（4 个占位符）
4. 4 项专项检查
5. 审查者返回字段
```

26 行覆盖了完整的"代码质量审查" workflow。

## 五、优点

### 5.1 极简而精确（26 行）

5 段结构覆盖了"代码质量审查"的关键内容：

- 使用时机（仅在规范符合性通过后）
- 基础模板引用
- 4 项专项检查
- 返回字段

### 5.2 "复用 + 增强" 模式的一致性

不重新定义代码审查 prompt，而是**复用** `code-reviewer.md` 标准模板，**只附加专项检查**：

- 避免重复
- 保证一致性
- 专业化

### 5.3 强顺序约束的显式化

> **Only dispatch after spec compliance review passes.**

**这一句是整个 prompt 的"前置条件"**——它**显式禁止"先代码质量后规范"** 的失败模式。

### 5.4 4 项专项检查的精妙设计

4 项检查**与实施 prompt 的纪律精确呼应**——形成"实施纪律 + 审查纪律"的闭环。

特别是第 4 项的"不要标记先前就存在的大小"——避免 LLM 把老问题算在新任务头上。

### 5.5 "Don't flag pre-existing file sizes" 的边界清晰

> (Don't flag pre-existing file sizes — focus on what this change contributed.)

**这一句定义了审查的边界**——审查者**只看本次变更的贡献**：

- 如果文件已经很大，本次没有让它更大 → 不标记
- 如果文件已经很大，本次让它更大 → 标记

这种**"增量审查"而非"绝对审查"** 是 LLM 协作中的清晰边界设计。

## 六、缺点 / 风险

### 6.1 缺乏"复用" 的版本控制机制

本文件**引用** `code-reviewer.md` 模板，但**没说如果该模板改了怎么办**：

- `code-reviewer.md` 增加了新维度 → 本文件是否自动受益？
- `code-reviewer.md` 修改了输出格式 → 本文件的预期输出是否同步？
- 是否有版本对应关系？

可以补充：

- "If requesting-code-review/code-reviewer.md changes, this template inherits the changes"
- 显式声明这种"复用依赖"

### 6.2 4 项专项检查的"深度"未说明

"Does each file have one clear responsibility" 没有说审查者应当**如何判断"清晰的职责"**：

- 文件行数上限？
- 函数数量上限？
- 类的字段数量上限？
- 还是基于"语义内聚"判断？

LLM 审查代理可能机械地用"行数 < 200" 这种简单标准。

可以补充：

- "A file's responsibility is unclear if it requires 'and' to describe"
- "Look for multiple distinct concepts, not just line count"

### 6.3 缺乏"未读代码" 的诊断信号

本文件没有像 `spec-reviewer-prompt.md` 那样显式说"Do Not Trust the Report"：

- 审查者是否应当独立读代码？
- 还是可以基于实施者报告 + git diff 就评估？
- 实施者声称"clean code" 是否值得相信？

`code-reviewer.md` 中隐含了"读代码" 的要求，但本文件**没重申**。

可以补充：

- "Verify by reading code, not by trusting report"

### 6.4 "Following the file structure from the plan" 需要 plan 上下文

这一项要求审查者**知道计划的文件结构**——但 prompt 模板没说如何提供：

- 主代理需要在分派时把 plan 的文件结构附在 prompt 中？
- 还是审查者自己去读 plan file？
- 还是有标准化的"file structure 摘要"格式？

SKILL.md 中提到"主代理提供完整文本"，所以应当是**主代理把计划的 file structure 段附在 prompt 中**——但本文件**没显式说明**。

### 6.5 缺乏"质量问题的紧急度分级"的指导

虽然 `code-reviewer.md` 有 Critical/Important/Minor 三档，但本文件的 4 项专项检查**没说明**这些检查项**应当对应哪一档**：

- "One clear responsibility" 不符合 → Critical / Important / Minor？
- "Following the file structure" 不符合 → Critical / Important / Minor？

`code-reviewer.md` 的 5 大维度中有"Architecture problems" → "Important"，但本文件的 4 项没明确归类。

可以补充：

- "These 4 checks typically map to Important (architectural concerns)"

### 6.6 缺乏"审查质量本身"的元检查

本文件**没说**审查者**应当如何自评自己的审查质量**：

- 是否真的读了代码？
- 是否理解了上下文？
- 反馈是否 actionable？

可以补充：

- "Self-check: Did I read the code, or just the report?"
- "Self-check: Is each finding actionable?"

## 七、关键洞察

### 7.1 这是 Superpowers 项目的"代码质量门控"

整个项目有 4 个核心子代理 prompt：

- `implementer-prompt.md` —— 实施者
- `spec-reviewer-prompt.md` —— 规范符合性审查
- **`code-quality-reviewer-prompt.md`（本文件）** —— 代码质量审查
- `code-reviewer.md`（requesting-code-review 目录下） —— 最终代码审查

**本文件位于"两阶段审查"的第二阶段**——它是**质量的门控**。

### 7.2 "复用 + 增强" 是 LLM 协作的工程化模式

本文件**不复制定义** code review prompt，而是**复用** `code-reviewer.md` 模板，**只附加 4 项专项检查**。

这种"**复用 + 增强**" 是 LLM 协作的工程化模式：

- **避免重复**：不复制粘贴相同的审查 prompt
- **一致性**：所有代码审查都使用同一基础模板
- **专业化**：本文件只附加**子代理驱动开发的特定关注点**

可以推广到其他 LLM 协作场景。

### 7.3 "Only dispatch after spec compliance passes" 是反"质量补偿规范"的关键

> **Only dispatch after spec compliance review passes.**

**这一句是反"质量补偿规范"的关键**——LLM 审查倾向"代码不错就放过"，本 prompt 强制**两阶段严格分离**。

可以推广到所有审查场景：**先把"对不对"判断完，再评"好不好"**。

### 7.4 "Don't flag pre-existing file sizes" 的边界设计

> (Don't flag pre-existing file sizes — focus on what this change contributed.)

**这一句定义了审查的"边界"**——审查者**只看本次变更的贡献**：

- 老问题不归新任务
- 增量审查而非绝对审查

这种**"边界清晰的审查"** 是 LLM 协作中的清晰设计。

### 7.5 4 项专项与实施纪律的精确呼应

4 项专项检查**与 `implementer-prompt.md` 中的"Code Organization" 5 条纪律精确呼应**——形成"实施纪律 + 审查纪律" 的闭环：

- 实施 prompt 告诉实施者**"不要做 X"**
- 审查 prompt 告诉审查者**"检查是否做了 X"**
- 两者形成**纪律闭环**

## 八、关联文档

### 8.1 同目录

- `SKILL.md`（[subagent-driven-development/SKILL.md](file:///e:/WorkStation/MyProject/superpowers/skills/subagent-driven-development/SKILL.md)）—— 调度方主技能
- `implementer-prompt.md`（[subagent-driven-development/implementer-prompt.md](file:///e:/WorkStation/MyProject/superpowers/skills/subagent-driven-development/implementer-prompt.md)）—— 实施者 prompt
- `spec-reviewer-prompt.md`（[subagent-driven-development/spec-reviewer-prompt.md](file:///e:/WorkStation/MyProject/superpowers/skills/subagent-driven-development/spec-reviewer-prompt.md)）—— 规范符合性审查

### 8.2 引用 / 被引用

- `requesting-code-review/code-reviewer.md` —— **被本文件引用**为基础模板

### 8.3 配套技能

- `writing-plans/SKILL.md` —— 计划来源（提供文件结构）
- `using-git-worktrees/SKILL.md` —— 工作区隔离

## 九、状态评价

**成熟度**：✅ **生产可用 + 关键门控**

- 文件极简但精准
- 复用模式工程化
- 是子代理驱动开发两阶段审查的关键节点

**完备度**：⭐⭐⭐ (3.5/5)

- 核心机制完整
- 缺少"复用依赖"的版本控制
- 4 项检查的"深度" 未说明
- 缺乏"未读代码"的反 LLM 信任约束

**风险等级**：🟢 低

- 主要风险是审查者"机械应用检查清单"
- 但 `code-reviewer.md` 的 5 大维度已经覆盖大部分质量关注
- 4 项专项只是"补充检查" 而非"主审查"

## 十、给读者的启示

### 10.1 启示 1：复用 + 增强是 LLM 协作的工程化模式

不要为每种场景**重新定义**完整的 prompt。**复用基础模板 + 附加专项检查** 是 LLM 协作的工程化模式：

- 避免重复
- 保证一致性
- 专业化

### 10.2 启示 2：强顺序约束必须显式声明

> **Only dispatch after spec compliance review passes.**

**强顺序约束必须显式声明**——不要让 LLM "自由选择" 审查顺序：

- 显式说"先 X 后 Y"
- 显式禁止"先 Y 后 X"
- 显式声明"违反顺序的代价"

### 10.3 启示 3：实施纪律 + 审查纪律必须呼应

4 项专项检查与 `implementer-prompt.md` 中的纪律精确呼应——形成"实施纪律 + 审查纪律"的闭环：

- 实施 prompt 告诉实施者**"不要做 X"**
- 审查 prompt 告诉审查者**"检查是否做了 X"**
- **两者必须呼应**才能形成纪律闭环

### 10.4 启示 4：审查的边界必须清晰

> (Don't flag pre-existing file sizes — focus on what this change contributed.)

**审查的边界必须清晰**——审查者**只看本次变更的贡献**：

- 老问题不归新任务
- 增量审查而非绝对审查
- 避免 LLM 把"老问题"算在"新任务"头上

### 10.5 启示 5：极简是设计能力的体现

本文件只有 26 行——但**覆盖了完整的"代码质量审查" workflow**：

- 使用时机
- 基础模板
- 4 项专项
- 返回字段

**极简是设计能力的体现**——能用 26 行说清楚的事，不要用 260 行。

## 十一、后续工作

### 11.1 可能的改进

1. **添加"复用依赖"的版本控制**
   - 显式声明"复用 `code-reviewer.md`"
   - 显式说明"如果该模板改了，本文件自动继承"

2. **添加 4 项检查的"深度"指导**
   - "A file's responsibility is unclear if it requires 'and' to describe"
   - "Look for multiple distinct concepts, not just line count"

3. **添加"Do Not Trust the Report" 约束**
   - 复用 spec-reviewer 的"读代码而非信任报告"约束

4. **添加"plan file structure" 的传递机制**
   - 显式说明主代理需要把 plan 的 file structure 段附在 prompt 中

5. **添加 4 项检查的"紧急度映射"**
   - 显式说明 4 项检查通常对应 Important（architectural concerns）

6. **添加"审查质量自评"**
   - "Self-check: Did I read the code?"
   - "Self-check: Is each finding actionable?"

### 11.2 监控点

- 4 项检查的 issue 分布（异常多 → 实施质量问题）
- "Don't flag pre-existing file sizes" 实际触发频率
- 复用 `code-reviewer.md` 后两者输出格式的一致性

### 11.3 长期演进

- 与 `verification-before-completion` 串联形成"自审→外审→验证"
- 引入"自动代码质量门控"（如 lint、type check）作为 LLM 审查的预筛
- 引入"代码质量趋势分析"（基于历史审查数据）
