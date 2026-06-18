# requesting-code-review/SKILL.md 分析报告

## 一、基本信息

- **文件路径**：`skills/requesting-code-review/SKILL.md`
- **文件大小**：104 行
- **文档类型**：技能主文档（带 YAML frontmatter）
- **配套文件**：`code-reviewer.md`（审查者 prompt 模板）
- **所属项目**：Superpowers 技能库

## 二、目的

该技能教会主代理**如何**（when/why/how）请求代码审查者子代理对已完成的工作做一次独立审查。它本身**不是**审查 prompt，而是审查的"调度"技能。

核心价值：
1. **在错误级联前抓出** —— 早审查成本远低于晚审查
2. **审查者拿到独立上下文** —— 不会被主代理的"思考过程"污染
3. **三种工作流统一** —— 子代理驱动、执行计划、临时开发都覆盖

## 三、核心技术决策

### 3.1 审查时机分级（强制 vs 可选）

文档把"何时审查"分为两层：

| 类别 | 触发条件 |
|---|---|
| **强制（Mandatory）** | 子代理开发每个任务后、完成主要功能后、合入前 |
| **可选（Optional but valuable）** | 卡住时、重构前、修复复杂 bug 后 |

**设计哲学**：把"审查"内化为"流程节点"，而不是"主观选择"。

### 3.2 上下文隔离 —— 审查者不接触主代理历史

> The reviewer gets precisely crafted context for evaluation — never your session's history.

这一句是整个技能的核心架构决策：

- 主代理的会话历史包括：尝试、错误、推翻自己的想法、妥协、退路 → 审查代理**不应当被这些影响**
- 审查代理只看：**实施了什么**（git diff）+ **应该做什么**（plan/requirements）
- 效果：审查代理**更接近"零上下文"的真实用户视角**

这种"上下文隔离"在子代理协作中**非常重要**——它让"代码是否真满足需求"独立于"实现过程是否曲折"。

### 3.3 Git SHA 范围界定

```bash
BASE_SHA=$(git rev-parse HEAD~1)  # 或 origin/main
HEAD_SHA=$(git rev-parse HEAD)
```

用 **git SHAs 范围**而非"我刚改了什么"来界定审查范围：

- **精确可复现**：审查代理跑 `git diff BASE_SHA..HEAD_SHA` 就看到完整改动
- **避免主观筛选**：主代理不会无意/有意"漏给"审查代理某些改动
- **支持独立验证**：审查代理可以独立跑测试、复现

### 3.4 反馈分级（Critical / Important / Minor）

```yaml
- Critical: 立即修复
- Important: 继续前修复
- Minor: 记下稍后处理
```

**三层分级** 对应**三种不同紧迫度**的行动：

- Critical → 阻塞（必须修）
- Important → 阻塞（不能带病推进）
- Minor → 不阻塞（可延后）

这与 `writing-plans/plan-document-reviewer-prompt.md` 的 "Issues / Recommendations" 二档设计是**同一思想的不同表达**。

### 3.5 "反驳审查者"的合法化

> Push back if reviewer is wrong (with reasoning)
> ...
> If reviewer wrong:
> - Push back with technical reasoning
> - Show code/tests that prove it works
> - Request clarification

这一段**非常重要**——它把"反驳审查"合法化，避免主代理变成"审查者的传声筒"。

LLM 倾向讨好（"你都对"），文档显式给出**反驳的合法化路径**：

- 反驳必须带"技术论据"（不是"我觉得没问题"）
- 反驳必须"展示证据"（code/tests）
- 反驳必须"请求澄清"（而非无视）

这是 LLM 协作中的关键反"讨好"设计。

### 3.6 与三种工作流的集成点

文档显式区分三种工作流：

| 工作流 | 审查时机 | 节奏 |
|---|---|---|
| 子代理驱动开发 | 每个任务后 | 短周期、高频率 |
| 执行计划 | 每任务或自然检查点 | 中等 |
| 临时开发 | 合入前 | 单次低频 |

这种"集成点显式列举"让用户/主代理在选用技能时**清楚知道在哪一步停下来**。

## 四、文件结构

文件结构清晰紧凑，6 段：

```
1. YAML frontmatter
   - name: requesting-code-review
   - description: 触发条件

2. 引言段
   - 一句话说明 + 核心原则

3. 何时请求审查
   - Mandatory 强制时机
   - Optional 可选时机

4. 如何请求
   - 3 个步骤：获取 SHAs → 分派子代理 → 行动

5. 示例
   - 完整工作流示例

6. 与工作流的集成
   - 子代理驱动 / 执行计划 / 临时开发

7. 红旗
   - 绝对不要 + 审查者错了怎么办
```

## 五、优点

### 5.1 极简而完整（104 行覆盖完整流程）

- 何时（when）
- 如何（how）
- 反馈如何处理（act on feedback）
- 三种工作流集成
- 红旗清单

### 5.2 反 LLM 偏好的强约束

- 显式列出"绝对不要"（never skip review because "it's simple"）
- 显式给出"反驳审查"路径（避免讨好审查者）
- 强制要求"修复 Important 后才能继续"（避免带病推进）

### 5.3 工作流集成的显式列举

不只说"什么时候审查"，而是**显式**说"在 X 工作流的 Y 步骤后审查"——降低主代理的决策负担。

### 5.4 反馈分级化（Critical / Important / Minor）

让"反馈"变成可程序化处理的**分级信号**，而非一锅粥。

### 5.5 示例真实可执行

示例不是抽象描述，而是**带具体 SHA、带具体命令、带具体响应**的真实工作流。

## 六、缺点 / 风险

### 6.1 缺乏对"小改动"的跳过判断

"绝对不要：因为'很简单'就跳过审查" 是铁律，但没有给"什么算简单"的可操作判定。

LLM 主代理在面对"修改一行" 任务时可能机械地照搬"每个任务后都审查"，浪费审查资源。

可以补充：

- 改动行数 < N 行的可以省审查
- 测试覆盖完整 + 改动纯局部的可以省
- 修改 string/常量值/注释的不需要审查

### 6.2 缺乏"审查者被污染"的诊断

文档说"审查者拿到独立上下文"，但**没有诊断信号**让主代理判断审查代理是否真的没被污染。

例如：

- 审查代理回信中提到"我看到你之前试了 X 方法" → 上下文污染
- 审查代理回信中提到"你提到的 Y 困难" → 上下文污染

可以补充"审查代理回信合规性检查"段。

### 6.3 缺乏"审查代理本身能力限制"的说明

文档假设审查代理"有专业能力"识别 Critical/Important/Minor 问题，但**没有说明审查代理的局限**：

- 大型 diff（> 1000 行）审查质量会下降
- 跨语言/跨框架代码审查质量不稳定
- 缺乏运行时环境的审查可能漏掉集成问题

### 6.4 缺乏"审查失败"的回退

如果审查代理超时、崩溃、返回无效结果，文档没有给出**回退机制**：

- 重试一次？
- 退回到"自行 review"？
- 暂停流程等待用户？

### 6.5 "Mandatory" 时机的执行缺乏强约束

"在子代理驱动开发中，每个任务之后" 是强制，但**没有"主代理必须等审查完成才能继续任务 N+1"** 的强约束。如果主代理分派审查代理后直接继续任务 N+1，审查反馈就来不及"修复"。

可以补充"分派审查代理后必须等待" 的强约束。

### 6.6 "BASE_SHA 来自 HEAD~1" 在多任务场景不准确

示例中：

```bash
BASE_SHA=$(git rev-parse HEAD~1)
```

但子代理驱动开发是"任务 1 完成后审查，任务 2 完成后审查"... 每个任务审查的 BASE_SHA 应当是**上一次任务结束后的 HEAD**，而不是机械的 HEAD~1。

- 任务 1：BASE = main, HEAD = task1
- 任务 2：BASE = task1, HEAD = task2（不是 HEAD~1！）

文档示例是简化的，但**容易误导**。应当显式说明 BASE_SHA 的计算规则。

## 七、关键洞察

### 7.1 这是 Superpowers 项目的"反馈收口点"

整个项目有多个"审查"技能：

- `brainstorming/spec-document-reviewer-prompt.md` —— spec 审查
- `writing-plans/plan-document-reviewer-prompt.md` —— plan 审查
- **`requesting-code-review/code-reviewer.md` —— 代码审查（本技能配套）**
- `subagent-driven-development/spec-reviewer-prompt.md` —— 实施前 spec 复核
- `subagent-driven-development/code-quality-reviewer-prompt.md` —— 实施后代码质量复核

**本技能位于流程的最下游**——它审的是"已经实施完的代码"。

### 7.2 上下文隔离是 LLM 协作的关键架构原则

文档第 8 行的关键论断：

> The reviewer gets precisely crafted context for evaluation — never your session's history.

这是 LLM 协作中**通用架构原则**：

- **隔离原则**：让审查代理只看"产物"，不看"过程"
- **专注原则**：让审查代理不被主代理的妥协/试错污染
- **可复现原则**：用 git SHA 让审查可精确复现

这个原则可以推广到所有 LLM 协作场景：**审查者只应当看"做了什么"，不应当看"怎么做的"**。

### 7.3 "反驳审查"合法化是反 LLM 讨好的关键

LLM 倾向：

- 同意审查者（避免冲突）
- 不质疑权威
- 即使错了也接受

文档显式给出反驳的合法路径：

1. 技术论据
2. 代码/测试证据
3. 请求澄清

**这种"反驳合法化" 是 LLM 协作中的稀缺设计**。大多数 LLM 工具默认"审查者说的都对"，本技能显式打破这个假设。

### 7.4 反馈分级（Critical/Important/Minor）= 行动的优先级

文档把反馈分成三档，对应三种行动：

| 级别 | 行动 | 含义 |
|---|---|---|
| Critical | 立即修 | 不修不能继续 |
| Important | 继续前修 | 不修不能进下一阶段 |
| Minor | 记下 | 不阻塞 |

这种**反馈分级 = 行动优先级**的设计，让审查输出直接可被主代理程序化处理。

## 八、关联文档

### 8.1 同目录

- `code-reviewer.md`（[requesting-code-review/code-reviewer.md](file:///e:/WorkStation/MyProject/superpowers/skills/requesting-code-review/code-reviewer.md)）—— 配套的审查者 prompt 模板

### 8.2 上游 / 下游技能

- `writing-plans/SKILL.md` —— 计划编写（本技能审的是按计划实施的代码）
- `writing-plans/plan-document-reviewer-prompt.md` —— 计划审查
- `subagent-driven-development/SKILL.md` —— 子代理驱动开发
- `subagent-driven-development/code-quality-reviewer-prompt.md` —— 实施后代码质量复核
- `receiving-code-review/SKILL.md` —— 接收代码审查（如何处理被审查）
- `verification-before-completion/SKILL.md` —— 完成前验证

### 8.3 测试

- `tests/claude-code/test-requesting-code-review.sh` —— 集成测试
- `tests/skill-triggering/prompts/requesting-code-review.txt` —— 触发测试

## 九、状态评价

**成熟度**：✅ **生产可用**

- 文档结构清晰，约束明确
- 反 LLM 偏好设计（反驳合法化、强制时机）成熟
- 配套的 code-reviewer.md prompt 模板已成型
- 已经在项目中被实际使用

**完备度**：⭐⭐⭐⭐ (4/5)

- 核心机制完整
- 缺少"小改动跳过审查"的明确边界
- 缺少"审查失败"的回退机制
- 缺少"上下文污染"的诊断信号

**风险等级**：🟡 中

- 主要风险是"审查者质量不稳定"
- LLM 审查代理对大型 diff 容易粗放
- 但本技能的"早审查常审查"原则能部分缓解这个问题

## 十、给读者的启示

### 10.1 启示 1：审查代理应当拿到独立上下文

> 审查者只应当看"做了什么"，不应当看"怎么做的"

这是 LLM 协作中的**通用架构原则**。在所有需要"独立评估"的场景：

- 代码审查
- 文档审查
- 计划审查
- 设计审查

都应当**只给审查代理看产物，不给过程**。

### 10.2 启示 2：反馈必须有"硬软两档"（或更多档）

LLM 协作中的反馈必须分级：

- 必须修（Critical）
- 应当修（Important）
- 建议（Minor）

无分级的反馈让主代理**不知道哪些阻塞、哪些不阻塞**，容易要么"全修浪费时间"、要么"挑着修遗漏重要"。

### 10.3 启示 3：反驳审查应当合法化

LLM 主代理**默认倾向"审查者都对"**，这导致大量错误反馈被接受。**在 LLM 工具中显式写"反驳合法化"** 是反讨好设计的关键。

### 10.4 启示 4：审查时机应当内化为流程节点

不要让"何时审查"成为主观判断。把它写进流程：

- 任务完成后（强制）
- 主要功能后（强制）
- 合入前（强制）
- 卡住时（可选）
- 重构前（可选）

让主代理**不需要决策**审查时机。

## 十一、后续工作

### 11.1 可能的改进

1. **明确"小改动跳过审查"边界**
   - 改动行数 < N
   - 改动类型：常量、注释、字符串
   - 测试覆盖完整

2. **添加"审查代理上下文污染诊断"**
   - 列出审查代理回信中可能出现的"污染信号"
   - 触发后让主代理重做审查

3. **添加"审查失败回退"机制**
   - 审查超时 → 重试一次
   - 审查代理崩溃 → 退回到"自行 review"
   - 审查返回空 → 标记为 Critical（视为未审查）

4. **修正示例中的 BASE_SHA 计算**
   - 在多任务场景显式说明 BASE_SHA 来自"上次任务完成后的 HEAD"

### 11.2 监控点

- 审查通过率（异常低 → 审查严苛或实施质量下降）
- 反馈分级分布（Critical:Important:Minor 比例）
- 审查者回信中的"上下文污染"频率
- BASE_SHA/HEAD_SHA 的实际使用与文档一致度

### 11.3 长期演进

- 与 `verification-before-completion` 整合：审查 → 验证 → 完成 形成串联
- 引入"自动代码审查"作为 LLM 审查的预筛
- 引入"审查代理质量评估"（基于历史反馈准确率）
