# 文档审查系统实施计划

> **面向智能体（Agentic）工作者：** 必须：使用 `superpowers:subagent-driven-development`（如果支持子智能体）或 `superpowers:executing-plans` 技能来执行本计划。

**目标：** 为 brainstorming 和 writing-plans 两个技能添加 spec（规格说明）和 plan（计划）文档的审查循环。

**架构：** 在每个技能目录下创建审查提示词模板。修改技能文件，在文档创建后增加审查循环。通过 Task 工具配合通用目的子智能体（general-purpose subagent）来派发审查任务。

**技术栈：** Markdown 技能文件、通过 Task 工具派发子智能体

**Spec（规格）：** `docs/superpowers/specs/2026-01-22-document-review-system-design.md`

---

## Chunk 1：Spec 文档审查器

本 Chunk 为 brainstorming 技能添加 spec 文档审查器。

### Task 1：创建 Spec 文档审查器提示词模板

**文件：**
- 创建：`skills/brainstorming/spec-document-reviewer-prompt.md`

- [ ] **第 1 步：** 创建审查器提示词模板文件

```markdown
# Spec 文档审查器提示词模板

在派发 spec 文档审查子智能体时使用本模板。

**目的：** 验证 spec 是否完整、一致，并准备好进入实施规划阶段。

**派发时机：** Spec 文档写入 `docs/superpowers/specs/` 之后。

```
Task tool (general-purpose):
  description: "Review spec document"
  prompt: |
    You are a spec document reviewer. Verify this spec is complete and ready for planning.

    **Spec to review:** [SPEC_FILE_PATH]

    ## What to Check

    | Category | What to Look For |
    |----------|------------------|
    | Completeness | TODOs, placeholders, "TBD", incomplete sections |
    | Coverage | Missing error handling, edge cases, integration points |
    | Consistency | Internal contradictions, conflicting requirements |
    | Clarity | Ambiguous requirements |
    | YAGNI | Unrequested features, over-engineering |

    ## CRITICAL

    Look especially hard for:
    - Any TODO markers or placeholder text
    - Sections saying "to be defined later" or "will spec when X is done"
    - Sections noticeably less detailed than others

    ## Output Format

    ## Spec Review

    **Status:** ✅ Approved | ❌ Issues Found

    **Issues (if any):**
    - [Section X]: [specific issue] - [why it matters]

    **Recommendations (advisory):**
    - [suggestions that don't block approval]
```

**审查器返回：** 状态、问题（若有）、建议
```

- [ ] **第 2 步：** 验证文件已正确创建

执行：`cat skills/brainstorming/spec-document-reviewer-prompt.md | head -20`
预期：显示标题和目的小节

- [ ] **第 3 步：** 提交

```bash
git add skills/brainstorming/spec-document-reviewer-prompt.md
git commit -m "feat: add spec document reviewer prompt template"
```

---

### Task 2：为 Brainstorming 技能添加审查循环

**文件：**
- 修改：`skills/brainstorming/SKILL.md`

- [ ] **第 1 步：** 阅读当前的 brainstorming 技能

执行：`cat skills/brainstorming/SKILL.md`

- [ ] **第 2 步：** 在 "After the Design" 之后添加审查循环小节

定位 "After the Design" 小节，在文档化之后、实施之前，添加新的 "Spec Review Loop" 小节：

```markdown
**Spec Review Loop:**
After writing the spec document:
1. Dispatch spec-document-reviewer subagent (see spec-document-reviewer-prompt.md)
2. If ❌ Issues Found:
   - Fix the issues in the spec document
   - Re-dispatch reviewer
   - Repeat until ✅ Approved
3. If ✅ Approved: proceed to implementation setup

**Review loop guidance:**
- Same agent that wrote the spec fixes it (preserves context)
- If loop exceeds 5 iterations, surface to human for guidance
- Reviewers are advisory - explain disagreements if you believe feedback is incorrect
```

- [ ] **第 3 步：** 验证修改

执行：`grep -A 15 "Spec Review Loop" skills/brainstorming/SKILL.md`
预期：显示新增的审查循环小节

- [ ] **第 4 步：** 提交

```bash
git add skills/brainstorming/SKILL.md
git commit -m "feat: add spec review loop to brainstorming skill"
```

---

## Chunk 2：Plan 文档审查器

本 Chunk 为 writing-plans 技能添加 plan 文档审查器。

### Task 3：创建 Plan 文档审查器提示词模板

**文件：**
- 创建：`skills/writing-plans/plan-document-reviewer-prompt.md`

- [ ] **第 1 步：** 创建审查器提示词模板文件

```markdown
# Plan 文档审查器提示词模板

在派发 plan 文档审查子智能体时使用本模板。

**目的：** 验证 plan 块（chunk）是否完整、与 spec 匹配，并具备合理的任务分解。

**派发时机：** 每个 plan 块编写完成之后。

```
Task tool (general-purpose):
  description: "Review plan chunk N"
  prompt: |
    You are a plan document reviewer. Verify this plan chunk is complete and ready for implementation.

    **Plan chunk to review:** [PLAN_FILE_PATH] - Chunk N only
    **Spec for reference:** [SPEC_FILE_PATH]

    ## What to Check

    | Category | What to Look For |
    |----------|------------------|
    | Completeness | TODOs, placeholders, incomplete tasks, missing steps |
    | Spec Alignment | Chunk covers relevant spec requirements, no scope creep |
    | Task Decomposition | Tasks atomic, clear boundaries, steps actionable |
    | Task Syntax | Checkbox syntax (`- [ ]`) on tasks and steps |
    | Chunk Size | Each chunk under 1000 lines |

    ## CRITICAL

    Look especially hard for:
    - Any TODO markers or placeholder text
    - Steps that say "similar to X" without actual content
    - Incomplete task definitions
    - Missing verification steps or expected outputs

    ## Output Format

    ## Plan Review - Chunk N

    **Status:** ✅ Approved | ❌ Issues Found

    **Issues (if any):**
    - [Task X, Step Y]: [specific issue] - [why it matters]

    **Recommendations (advisory):**
    - [suggestions that don't block approval]
```

**审查器返回：** 状态、问题（若有）、建议
```

- [ ] **第 2 步：** 验证文件已创建

执行：`cat skills/writing-plans/plan-document-reviewer-prompt.md | head -20`
预期：显示标题和目的小节

- [ ] **第 3 步：** 提交

```bash
git add skills/writing-plans/plan-document-reviewer-prompt.md
git commit -m "feat: add plan document reviewer prompt template"
```

---

### Task 4：为 Writing-Plans 技能添加审查循环

**文件：**
- 修改：`skills/writing-plans/SKILL.md`

- [ ] **第 1 步：** 阅读当前技能文件

执行：`cat skills/writing-plans/SKILL.md`

- [ ] **第 2 步：** 添加按块（chunk）审查小节

在 "Execution Handoff" 小节之前添加：

```markdown
## Plan Review Loop

After completing each chunk of the plan:

1. Dispatch plan-document-reviewer subagent for the current chunk
   - Provide: chunk content, path to spec document
2. If ❌ Issues Found:
   - Fix the issues in the chunk
   - Re-dispatch reviewer for that chunk
   - Repeat until ✅ Approved
3. If ✅ Approved: proceed to next chunk (or execution handoff if last chunk)

**Chunk boundaries:** Use `## Chunk N: <name>` headings to delimit chunks. Each chunk should be ≤1000 lines and logically self-contained.
```

- [ ] **第 3 步：** 更新任务语法示例，使用复选框形式

将 "Task Structure" 小节中的任务语法改为复选框形式：

```markdown
### Task N: [Component Name]

- [ ] **Step 1:** Write the failing test
  - File: `tests/path/test.py`
  ...
```

- [ ] **第 4 步：** 验证审查循环小节已添加

执行：`grep -A 15 "Plan Review Loop" skills/writing-plans/SKILL.md`
预期：显示新增的审查循环小节

- [ ] **第 5 步：** 验证任务语法示例已更新

执行：`grep -A 5 "Task N:" skills/writing-plans/SKILL.md`
预期：显示复选框语法 `### Task N:`

- [ ] **第 6 步：** 提交

```bash
git add skills/writing-plans/SKILL.md
git commit -m "feat: add plan review loop and checkbox syntax to writing-plans skill"
```

---

## Chunk 3：更新 Plan 文档头部

本 Chunk 更新 plan 文档头部模板，以引用新的复选框语法要求。

### Task 5：在 Writing-Plans 技能中更新 Plan 头部模板

**文件：**
- 修改：`skills/writing-plans/SKILL.md`

- [ ] **第 1 步：** 阅读当前的 plan 头部模板

执行：`grep -A 20 "Plan Document Header" skills/writing-plans/SKILL.md`

- [ ] **第 2 步：** 更新头部模板，引用复选框语法

plan 头部应注明任务和步骤使用复选框语法。更新头部注释：

```markdown
> **For agentic workers:** REQUIRED: Use superpowers:subagent-driven-development (if subagents available) or superpowers:executing-plans to implement this plan. Tasks and steps use checkbox (`- [ ]`) syntax for tracking.
```

- [ ] **第 3 步：** 验证修改

执行：`grep -A 5 "For agentic workers:" skills/writing-plans/SKILL.md`
预期：显示更新后的头部，提到复选框语法

- [ ] **第 4 步：** 提交

```bash
git add skills/writing-plans/SKILL.md
git commit -m "docs: update plan header to reference checkbox syntax"
```
