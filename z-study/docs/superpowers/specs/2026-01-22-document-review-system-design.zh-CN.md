# 文档审查系统设计

## 概述

为 superpowers 工作流增加两个新的审查阶段：

1. **Spec 文档审查** —— 在 brainstorming 之后、writing-plans 之前
2. **Plan 文档审查** —— 在 writing-plans 之后、实施之前

两者都遵循实施审查所用的迭代循环模式。

## Spec 文档审查器

**目的：** 验证 spec 是否完整、一致，并准备好进入实施规划阶段。

**位置：** `skills/brainstorming/spec-document-reviewer-prompt.md`

**检查项：**

| 类别 | 检查内容 |
|----------|------------------|
| 完整性（Completeness） | TODO 占位符、"TBD"、未完成小节 |
| 覆盖度（Coverage） | 缺失的错误处理、边缘情况、集成点 |
| 一致性（Consistency） | 内部矛盾、冲突的需求 |
| 清晰度（Clarity） | 模糊的需求 |
| YAGNI | 未要求的功能、过度工程 |

**输出格式：**
```
## Spec Review

**Status:** Approved | Issues Found

**Issues (if any):**
- [Section X]: [issue] - [why it matters]

**Recommendations (advisory):**
- [suggestions that don't block approval]
```

**审查循环：** 发现问题 → brainstorming 智能体修复 → 重新审查 → 重复直到通过

**派发机制：** 使用 Task 工具，`subagent_type: general-purpose`。审查器提示词模板提供完整 prompt。brainstorming 技能的控制器负责派发审查器。

## Plan 文档审查器

**目的：** 验证 plan 是否完整、与 spec 匹配，并具备合理的任务分解。

**位置：** `skills/writing-plans/plan-document-reviewer-prompt.md`

**检查项：**

| 类别 | 检查内容 |
|----------|------------------|
| 完整性（Completeness） | TODO 占位符、未完成的任务 |
| Spec 对齐（Spec Alignment） | 计划覆盖 spec 需求，无范围蔓延 |
| 任务分解（Task Decomposition） | 任务原子化、边界清晰 |
| 任务语法（Task Syntax） | 任务和步骤使用复选框语法 |
| Chunk 大小（Chunk Size） | 每个 chunk 少于 1000 行 |

**Chunk 定义：** Chunk 是 plan 文档内由 `## Chunk N: <name>` 标题分隔的逻辑任务分组。writing-plans 技能基于逻辑阶段（如 "Foundation"、"Core Features"、"Integration"）创建这些边界。每个 chunk 应足够独立以便单独审查。

**Spec 对齐验证：** 审查器接收两样东西：
1. plan 文档（或当前 chunk）
2. spec 文档的路径作为参考

审查器阅读两者并比较需求覆盖度。

**输出格式：** 与 spec 审查器相同，但范围限定在当前 chunk。

**审查流程（按 chunk）：**
1. writing-plans 创建 chunk N
2. 控制器派发 plan-document-reviewer，传入 chunk N 内容和 spec 路径
3. 审查器阅读 chunk 和 spec，返回裁决
4. 若有问题：writing-plans 智能体修复 chunk N，回到步骤 2
5. 若通过：进入 chunk N+1
6. 重复直到所有 chunk 通过

**派发机制：** 与 spec 审查器相同——使用 Task 工具，`subagent_type: general-purpose`。

## 更新的工作流

```
brainstorming -> spec -> SPEC REVIEW LOOP -> writing-plans -> plan -> PLAN REVIEW LOOP -> implementation
```

**Spec 审查循环：**
1. Spec 完成
2. 派发审查器
3. 若有问题：修复 → 回到步骤 2
4. 若通过：继续

**Plan 审查循环：**
1. Chunk N 完成
2. 为 chunk N 派发审查器
3. 若有问题：修复 → 回到步骤 2
4. 若通过：进入下一个 chunk 或实施

## Markdown 任务语法

任务和步骤使用复选框语法：

```markdown
- [ ] ### Task 1: Name

- [ ] **Step 1:** Description
  - File: path
  - Command: cmd
```

## 错误处理

**审查循环终止：**
- 无硬性迭代上限——循环持续到审查器通过
- 若循环超过 5 轮，控制器应主动上抛给人类请求指导
- 人类可选择：继续迭代、带着已知问题通过、或中止

**分歧处理：**
- 审查器是顾问性的——它们标记问题但不阻塞
- 若智能体认为审查器反馈不正确，应在修复中说明原因
- 若同一问题在 3 轮后仍存在分歧，主动上抛给人类

**审查器输出格式错误：**
- 控制器应验证审查器输出包含必需字段（Status、Issues 如果适用）
- 若格式错误，重新派发审查器并附上期望格式的说明
- 连续 2 次格式错误后，主动上抛给人类

## 待修改的文件

**新增文件：**
- `skills/brainstorming/spec-document-reviewer-prompt.md`
- `skills/writing-plans/plan-document-reviewer-prompt.md`

**修改文件：**
- `skills/brainstorming/SKILL.md` —— 在 spec 写入后增加审查循环
- `skills/writing-plans/SKILL.md` —— 增加按 chunk 的审查循环，更新任务语法示例
