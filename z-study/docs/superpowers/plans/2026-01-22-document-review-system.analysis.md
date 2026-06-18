# 文档审查系统实施计划 - 分析报告

> **对应原文**：`docs/superpowers/plans/2026-01-22-document-review-system.md`
> **翻译文件**：`z-study/docs/superpowers/plans/2026-01-22-document-review-system.zh-CN.md`
> **报告时间**：2026-06-03

---

## 一、基本信息

| 项目 | 内容 |
|---|---|
| 文档类型 | 实施计划（Implementation Plan） |
| Chunk 数量 | 3 个 Chunk，共 5 个 Task |
| 总行数 | 302 行 |
| 关联 Spec | `docs/superpowers/specs/2026-01-22-document-review-system-design.md` |
| 目标技能 | `brainstorming`、`writing-plans` |
| 创建产物 | 2 个新模板 + 2 个技能文件修改 + 1 个头部更新 |
| 核心机制 | Task 工具派发通用目的子智能体做文档审查 |
| 适用平台 | Claude Code（支持子智能体派发） |

---

## 二、目的

为 Superpowers 项目的"规格说明→计划"工作流增加**文档质量门控**机制。当前工作流存在以下痛点：

1. **缺乏反馈循环**：brainstorming 产出的 spec 文档直接进入计划阶段，作者无客观校验
2. **TODO 残留风险**：作者倾向于"先写完，再补 TODO"——这些 TODO 经常被遗忘
3. **spec 与 plan 脱节**：计划阶段重新阅读 spec 时才发现理解偏差
4. **任务粒度失控**：复杂 plan 中常出现"步骤类似 X"这类偷懒表述

本计划引入**两层审查循环**：
- **Spec 审查器**：brainstorming 写完 spec 后触发，验证完整性、覆盖度、一致性、YAGNI
- **Plan 审查器**：writing-plans 每写完一个 Chunk 后触发，验证 spec 对齐、任务分解、复选框语法

---

## 三、核心技术决策

### 1. 复用子智能体机制（不是新增 Skill 工具）

不引入新的"审查器"工具，而是**通过 Task 工具派发 general-purpose 子智能体**承担审查角色。这样做的好处：
- 零基础设施投入（不需要新工具开发）
- 子智能体是隔离的上下文，不会污染主对话
- 审查 prompt 是 markdown 模板，可被版本控制和迭代优化

### 2. 审查循环的终止条件

- ✅ `Approved` → 进入下一步
- ❌ `Issues Found` → 修复后重新派发
- **5 轮上限**：超过 5 轮仍未 Approved，主动上抛给人类（避免无限循环）
- 审查器"是顾问性的"（advisory），可被否决——保留作者最终决定权

### 3. 任务语法强制升级到复选框（`### Task N:` + `- [ ]`）

将计划文档从叙述性升级为可勾选式：
- 进度可视化
- 与 `using-git-worktrees` 等技能中的现有 step 语法保持一致
- 便于后续工具（如 TDD 跟踪）解析

### 4. 复选框语法贯穿三层

- **任务标题**：`### Task N: [Component Name]`
- **步骤标题**：`- [ ] **Step 1:** ...`
- **文档头部注释**：明确声明使用复选框

---

## 四、文件结构

```
skills/
├── brainstorming/
│   ├── SKILL.md                       # 修改：添加 "Spec Review Loop" 小节
│   └── spec-document-reviewer-prompt.md  # 新建：spec 审查器模板
└── writing-plans/
    ├── SKILL.md                       # 修改：添加 "Plan Review Loop" 小节 + 复选框示例
    └── plan-document-reviewer-prompt.md  # 新建：plan 审查器模板
```

每个 Chunk 1-3 个 Task，逐步引入能力：
- Chunk 1 → Spec 审查能力上线
- Chunk 2 → Plan 审查能力上线
- Chunk 3 → 文档头部声明统一

---

## 五、优点

| 维度 | 具体表现 |
|---|---|
| **设计哲学** | "在文档流动的关键节点插入质量门控"——对的过程产生对的结果 |
| **可验证性** | 每个 Task 都有明确的验证命令（`grep`/`cat`/commit） |
| **解耦性** | 审查器作为独立模板文件，不与 SKILL.md 主体强耦合 |
| **可演进性** | 审查 prompt 可独立迭代优化，无需触动技能主体 |
| **失败兜底** | 5 轮上限 + 顾问性（advisory）双重保险，不会被审查器绑架 |
| **复用性** | 同一套 Task 派发模式可推广到其他需要审查的场景（如代码审查） |

---

## 六、缺点 / 风险

| 风险点 | 影响 | 缓解 |
|---|---|---|
| **依赖子智能体可用性** | 非 Claude Code 平台（如某些纯 IDE 插件）可能不支持 Task 工具 | 计划开头注明"if subagents available"作为回退条件 |
| **审查者模型质量** | 通用子智能体未必熟悉 spec/plan 评审约定 | 通过详细的"What to Check"表格约束 + "CRITICAL"重点提示引导 |
| **审查循环开销** | 每个 spec 走 1-2 轮审查 = 多 1-2 次子智能体派发 = 时间成本 | 顾问性原则避免无谓争论；5 轮上限避免无限循环 |
| **"advisory"原则易被滥用** | 作者可能无理由否决一切反馈 | 文档明确要求"explain disagreements"——必须给出理由 |
| **TODO 拦截是事后行为** | 作者仍可能在审查前来不及发现 | 这是设计权衡：无法在写之前拦截，所以走"写完即查"路线 |

---

## 七、关键洞察

### 洞察 1：审查是软件工程的"防御性编程"扩展

该计划把"代码层的 Linter"理念上推到"文档层的 Reviewer"——任何产出都值得被另一双眼睛看一眼。在 AI 写作时代，这种"自检-他检"双轨尤其重要。

### 洞察 2：审查模板本身就是知识资产

`spec-document-reviewer-prompt.md` 和 `plan-document-reviewer-prompt.md` 不只是"提示词"，更是**项目对"什么是好 spec / 好 plan"的具象定义**。它们的可读性、可教学性比其技术功能更重要——任何团队成员阅读这两个文件就能学会评审标准。

### 洞察 3："5 轮上限"是元规则

它把"何时停止"的判断从工程权衡提升为流程规则。这体现了一个更深的设计原则：**为元层级（meta-level）的活动也设定终止条件**，避免"为了完美而无限循环"。

### 洞察 4：Chunk 3 看似很小，意义重大

更新头部注释看起来是"一行改动"，但它建立了**整个项目文档格式的契约**——所有未来的 plan 文档将自动声明使用复选框语法。这是元层级的标准化，杠杆效应巨大。

---

## 八、关联文档

### 上游 / Spec
- [`2026-01-22-document-review-system-design.md`](../specs/2026-01-22-document-review-system-design.md) — 设计文档（本计划的依据）

### 下游 / 受影响
- `skills/brainstorming/SKILL.md` — 接受 Chunk 1 的修改
- `skills/writing-plans/SKILL.md` — 接受 Chunk 2、3 的修改
- 所有未来编写的 spec 文档 — 将被新审查流程评审
- 所有未来编写的 plan 文档 — 将使用复选框语法

### 横向关联
- [`2025-11-28-skills-improvements-from-user-feedback.md`](../../plans/2025-11-28-skills-improvements-from-user-feedback.md) — 用户反馈驱动的技能改进（同样在迭代技能质量）
- [`2026-01-17-visual-brainstorming.md`](../../plans/2026-01-17-visual-brainstorming.md) — 视觉化 brainstorming（脑暴技能本身的演进）

---

## 九、状态评价

**计划成熟度：高** ✅

- 每个 Task 都有可执行的步骤
- 验证命令明确（`grep`/`cat`/commit）
- commit message 预写好
- 与上游 spec 文档对应

**实施复杂度：低** ⬇️

- 不涉及新工具开发
- 仅修改 2 个技能 + 创建 2 个模板 + 1 个头部
- 5 个 Task，每个 Task 3-6 步

**适用性**：✅ 推荐实施
- 投入产出比高
- 解决了真实痛点
- 不破坏现有工作流

---

## 十、给读者的启示

### 启示 1：质量门控可以"无侵入"
不一定需要新工具、新流程。**把现有资源（Task 工具 + 通用子智能体）组合到现有工作流的关键节点**，就能搭建出高效的质量门控。

### 启示 2：模板化是知识沉淀的最佳载体
评审标准写在 `prompt.md` 里，比写在脑海或 wiki 中更可执行、更可演进。**让流程标准以可执行文件形式存在**，是组织能力建设的重要手段。

### 启示 3：设定元层级的终止条件
"5 轮上限"看似不起眼，实则把"何时停止"提升为显性规则。**给任何循环结构都加上最大迭代次数**，是工程文化成熟的标志。

### 启示 4：审查与创作是互补的
本计划不是"用审查取代创作"，而是"创作 + 审查 + 修复"的闭环。**让 agent 自己既是作者也是审稿人**——这是 AI 协作工作流的核心模式。

---

## 十一、后续工作（推测）

1. **审查器效果评估**：上线 1-2 个月后统计"审查发现的真实问题 vs 无效抱怨"的比例
2. **审查 prompt 模板化复用**：将"文档审查"模式推广到 `requesting-code-review`、`writing-plans` 的 spec 阶段
3. **审查报告归档**：将每次审查的 issues/recommendations 写入 git commit message 或独立报告文件，便于复盘
4. **审查者模型选择**：未来可考虑用专用"评审者"模型（reviewer-only model），与主创作模型解耦
5. **自动评分机制**：在 5 轮循环结束后自动记录 "📊 Spec 审查平均轮数"，作为技能成熟度指标
