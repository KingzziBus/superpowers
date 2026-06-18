# plan-document-reviewer-prompt.md 分析报告

## 一、基本信息

- **文件路径**：`skills/writing-plans/plan-document-reviewer-prompt.md`
- **文件大小**：50 行
- **文档类型**：技能配套 Prompt 模板
- **所属技能**：`writing-plans`
- **配套主文件**：`SKILL.md`（计划编写技能主规范）
- **使用时机**：计划写完后、分派实施者之前
- **调用方式**：分派 plan document reviewer 子代理（Task tool, general-purpose 类型）

## 二、目的

该 Prompt 模板用于分派"计划文档审查者"子代理，对刚刚写完的 `plan.md` 进行最后一轮质量校验，确保：

1. **Completeness（完整性）**：没有占位符、TODO、空任务、缺失步骤
2. **Spec Alignment（规范对齐）**：计划覆盖了 spec/设计文档的所有需求，没有重大的范围蔓延
3. **Task Decomposition（任务分解）**：任务边界清晰、步骤可执行
4. **Buildability（可构建性）**：一个工程师能否顺利按此计划执行而不卡壳

它是 `writing-plans` 技能的"质量门控"，但严格意义上**属于"审查者侧"的 prompt**，而非"实施者侧"。

## 三、核心技术决策

### 3.1 校准原则（Calibration）—— 最关键的设计

> **Only flag issues that would cause real problems during implementation.**
> An implementer building the wrong thing or getting stuck is an issue.
> Minor wording, stylistic preferences, and "nice to have" suggestions are not.

这是整个 Prompt 中最重要的反 LLM 陷阱设计：

| LLM 倾向 | 本 Prompt 的校准 |
|---|---|
| 过度审查（nitpicking） | "Minor wording... are not" |
| 讨好式审查（"looks good"） | 显式否定模糊好评 |
| 平均主义（每项都标 Critical） | 区分 "Issues"（阻塞） vs "Recommendations"（建议） |
| 误把风格当缺陷 | 显式区分 "real problems" 与 "nice to have" |

通过"校准原则"明确告诉审查代理：**不要做完美主义审查**——只标记实施时会真正造成问题的事项。

### 3.2 默认通过原则（Approve unless...）

> Approve unless there are serious gaps — missing requirements from the spec,
> contradictory steps, placeholder content, or tasks so vague they can't be acted on.

"默认通过"的设计理念是：

- LLM 倾向于"找问题"以求显式价值 → 必须反向校正
- "重大差距"被显式列举（spec 缺失、步骤矛盾、占位符内容、任务过于模糊）→ 让审查有据可循
- "Approve" 与 "Issues Found" 二元状态 → 倒逼审查者做出清晰裁决

### 3.3 Issues vs Recommendations 的二元区分

输出格式中显式分离：

- **Issues（阻塞通过）**：必须解决才能批准
- **Recommendations（建议，不阻塞）**：仅供改进参考

这一区分让"反馈"变成**两段式结构化数据**：

```
Issues:        [必须修复]    → 实施代理接收到后会立即处理
Recommendations:[可选改进]    → 可记录为 follow-up 而不阻塞主流程
```

### 3.4 校准 vs 软性建议的"软硬分层"

这个 Prompt 实际上定义了一个**反馈分级系统**：

| 层级 | 触发条件 | 行动 |
|---|---|---|
| 严重差距 | spec 缺失、步骤矛盾、占位符 | Issues 标记 → 阻塞 |
| 实施风险 | 任务过于模糊、不可操作 | Issues 标记 → 阻塞 |
| 真实工程问题 | 实施时会让工程师卡壳 | Issues 标记 |
| 风格、措辞、小建议 | "nice to have" | Recommendations（不阻塞） |

## 四、文件结构

文件结构非常紧凑，5 个区块：

```
1. 标题 + 用途说明
   - Plan Document Reviewer Prompt Template
   - Purpose: Verify the plan is complete, matches the spec, ...
   - Dispatch after: The complete plan is written.

2. Task tool 分派模板
   - description: "Review plan document"
   - prompt: 包含完整的多行 prompt

3. What to Check 表格
   - 4 项检查类别

4. Calibration 校准原则
   - 显式否定过度审查
   - 显式说明默认通过

5. Output Format 输出格式
   - ## Plan Review
   - Status: Approved | Issues Found
   - Issues: [Task X, Step Y]: [specific issue] - [why it matters]
   - Recommendations: [suggestions]
```

## 五、优点

### 5.1 极简而完整

- 50 行内完成"分派 prompt + 校准原则 + 输出格式"三件事
- 没有任何冗余，**每一行都是约束**

### 5.2 抗 LLM 偏差的强约束

- 校准原则直接对抗"过度审查"
- 默认通过原则直接对抗"问题焦虑"
- Issues vs Recommendations 分离让审查有"软硬"两档

### 5.3 结构化的输出格式

- 强制要求 `[Task X, Step Y]` 引用 → 反馈必须可定位
- 强制要求 `why it matters for implementation` → 反馈必须说明影响
- 这种结构让**反馈可以直接喂给主代理执行修复**

### 5.4 与主技能 SKILL.md 形成闭环

`SKILL.md` 定义了**写计划**的规范，本文件定义了**审计划**的规范。两者形成：

- `SKILL.md` 写 → 本文件审 → 实施代理执行
- 计划本身的 writing-reviewing-acting 三段式

## 六、缺点 / 风险

### 6.1 缺少"双向校准"—— 没说清楚什么算"已批准"

"Approve unless there are serious gaps" 是单向校准，但**没有说"什么算严重差距"** 的具体判定标准。

虽然列举了"missing requirements, contradictory steps, placeholder content, vague tasks" 四项，但每项都依赖审查者的主观判断。例如：

- "步骤矛盾" → 多大程度的不一致算矛盾？
- "任务过于模糊" → 多少步骤算"可操作"？

### 6.2 缺乏对"scope creep" 的精确定义

> Spec Alignment: Plan covers spec requirements, no major scope creep

"Major scope creep" 缺少可量化指标：

- 多大比例的非 spec 任务算"major"？
- 写一个显然不在 spec 但又显然需要的小工具，是否算"scope creep"？

### 6.3 反馈粒度依赖实施代理的诚实

`[Task X, Step Y]` 的精确定位要求审查代理**真的精读计划**。但 LLM 审查代理有可能：

- 给出"模糊 step Y"的引用
- 列出"X-Y 步骤存在矛盾"但没说清是哪一处矛盾
- 反馈中"小问题"被打包进 Issues 逃避 Recommendations

这要求审查代理本身具备**纪律性**。如果审查代理不守纪律，这个模板的约束力会被削弱。

### 6.4 与 brainstorming 的 spec-document-reviewer-prompt 不对称

`brainstorming/spec-document-reviewer-prompt.md` 与本文的目录名类似（都是 `<document>-reviewer-prompt.md`），但两者的：

- 检查项不同（spec 4 项 vs plan 4 项）
- 校准措辞不同
- 输出格式不同

这意味着同一项目中**至少有 2 套"文档审查者"模式**。未来若引入第 3 套，需要做横向一致性。

### 6.5 缺少"实施前最后审"的强制约束

文档说 "Dispatch after: The complete plan is written"——但**没有强约束"主代理必须等待审查完成"**。

如果实施代理在审查代理未返回前就开始执行，反馈的 Issues 就无法阻塞流程。这需要主技能（`writing-plans/SKILL.md`）的纪律配合。

## 七、关键洞察

### 7.1 这是 Superpowers 项目的"二级质量门控"

整个项目有**多层质量门控**：

1. `using-superpowers/SKILL.md` —— 12 条红旗（决定是否调用技能）
2. `verification-before-completion/SKILL.md` —— 5 步门控函数（验证是否真完成）
3. **`writing-plans/plan-document-reviewer-prompt.md` —— 本文件（计划是否可执行）**
4. `requesting-code-review/code-reviewer.md` —— 代码审查（实施是否正确）
5. `subagent-driven-development/...reviewer-prompt.md` —— 规范/质量两阶段审查

**本文件是"计划侧"最后一道关卡**，决定了实施代理收到的计划是否真的可执行。

### 7.2 "软硬两档反馈"是 LLM 协作的关键设计

LLM 在"反馈"上有一个根本弱点：**倾向于把所有问题都标记为同等严重**。这个 Prompt 的核心创新在于：

- **Issues（硬）**：必须修复才能继续
- **Recommendations（软）**：仅作建议

通过结构化数据让"反馈"变成分级信号，而不是一锅粥。

### 7.3 "Approve unless..." 是反"问题焦虑"的处方

LLM 普遍存在"找问题焦虑"——即倾向于"找点问题才显得专业"。这导致：

- 审查时过度严苛
- 给出不必要的"小建议"
- 让"通过"变成罕见状态

"Approve unless there are serious gaps" 的反处方：

- 把"通过"作为基线
- 把"找差距"作为例外
- 显式列举哪些算"严重差距"让基线有据可循

这是 LLM 时代文档治理的关键模式：**先建立"默认通过"，再划出"必须不通过"的硬边界**。

## 八、关联文档

### 8.1 同一目录

- `SKILL.md`（[writing-plans/SKILL.md](file:///e:/WorkStation/MyProject/superpowers/skills/writing-plans/SKILL.md)）—— 计划编写主技能，与本文件形成"写-审"对偶

### 8.2 同类 Prompt 模板

- `brainstorming/spec-document-reviewer-prompt.md` —— spec 文档审查模板（更上游的"质量门控"）
- `subagent-driven-development/spec-reviewer-prompt.md` —— 子代理实施前的 spec 复核
- `subagent-driven-development/code-quality-reviewer-prompt.md` —— 子代理实施后的代码质量复核
- `requesting-code-review/code-reviewer.md` —— 更下游的"代码审查"模板

### 8.3 测试 / 示例

- `tests/subagent-driven-dev/` 下的 plan.md 文件可作为本 prompt 的实际审查目标
- `tests/explicit-skill-requests/prompts/after-planning-flow.txt` 等 prompt 测试可能涉及本 prompt 触发链

## 九、状态评价

**成熟度**：✅ **生产可用**

- 模板结构清晰、约束明确
- 与上游 SKILL.md 形成闭环
- 校准原则与默认通过原则是 LLM 时代的强需求
- 已经在项目的子代理驱动开发流程中被实际使用

**完备度**：⭐⭐⭐⭐ (4/5)

- 核心机制完整
- 缺少"判定严重差距"的可量化标准
- 缺少"审查代理不守纪律时的降级"机制

**风险等级**：🟢 低

- 即便审查代理偶尔过度审查，Recommendations 不阻塞的影响有限
- 真正可能失效的场景是审查代理漏检 → 但这是审查质量问题，不是 prompt 设计问题

## 十、给读者的启示

### 10.1 启示 1：把"通过"作为基线

学习本文件最重要的设计是 **"Approve unless..."**。在自己设计 LLM 协作流程时：

- 不要让审查/校验变成"默认找问题"
- 让"通过"是容易发生的，反向列举"哪些情况不通过"
- 通过结构化输出（Issues / Recommendations 二元）让反馈变成分级信号

### 10.2 启示 2：审查 prompt 的"软硬两档"模式

LLM 协作中，反馈必须有"必须修复"与"可选建议"的两档区分。可以参考本文件的：

```
Issues:        [File:line] - [Problem] - [Why it matters] - [How to fix]
Recommendations: [advisory, do not block]
```

### 10.3 启示 3：审查 prompt 应该包含"校准"段

几乎所有 LLM 审查类 prompt 都应该有 "Calibration" 段，**显式否定过度审查**：

- "Only flag issues that would cause real problems during implementation."
- "Minor wording, stylistic preferences, and 'nice to have' suggestions are not."

不要让审查代理凭"感觉"判断。

### 10.4 启示 4：审查 prompt 的输出格式应当可解析

本文件的输出格式：

```
## Plan Review
**Status:** Approved | Issues Found
**Issues (if any):** - [Task X, Step Y]: [specific issue] - [why it matters for implementation]
**Recommendations (advisory, do not block approval):** - [suggestions for improvement]
```

这种格式让主代理可以**程序化地**决定是否阻塞实施。

## 十一、后续工作

### 11.1 可能的改进

1. **添加"严重差距"判定标准清单**
   - 显式列举每种"严重差距"的具体判定条件
   - 例如："步骤矛盾 = 步骤 A 要求删除文件 X 但步骤 B 依赖文件 X"

2. **添加"scope creep" 检测启发式**
   - 自动比对 plan 任务与 spec 需求
   - 不在 spec 但又被计划执行的任务需要标注为"未在 spec 中，是否应加入 spec？"

3. **横向统一 4 套"审查者 prompt"**
   - spec / plan / code-quality / code-review
   - 提取共用骨架：校准原则 + 输出格式 + 占位符规范
   - 降低维护成本

### 11.2 监控点

- 审查代理是否真的在**精读**计划？（监控 `[Task X, Step Y]` 引用的具体性）
- Issues vs Recommendations 的比例是否合理？（若 Issues 过多则过度严苛）
- 审查通过率是否异常？（若通过率过低则审查严苛或计划质量下降）

### 11.3 长期演进

如果未来引入"自动验证 plan 是否真可执行"的机制（例如沙箱中按 plan 跑一遍），本文件的角色可能从"LLM 审查"演化为"机器验证的入口"—— 但这需要 LLM 计划质量的进一步稳定。
