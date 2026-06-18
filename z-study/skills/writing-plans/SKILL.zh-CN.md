---
name: writing-plans
description: Use when you have a spec or requirements for a multi-step task, before touching code
---

# Writing Plans（编写实施计划）

## 概述

编写综合性的实施计划，假设工程师对我们代码库零上下文且品味可疑。记录他们需要知道的一切：每个任务要碰哪些文件、代码、测试、他们可能需要查阅的文档、如何测试。把整个计划作为可一口吃下的任务呈现给他们。DRY。YAGNI。TDD。频繁提交。

假设他们是熟练的开发者，但对我们工具集和问题域几乎一无所知。假设他们对好的测试设计不太了解。

**开始时宣告：** "I'm using the writing-plans skill to create the implementation plan."（我正在使用 writing-plans 技能来创建实施计划。）

**上下文：** 如果在隔离的 worktree 中工作，它应该已经在执行时通过 `superpowers:using-git-worktrees` 技能创建。

**计划保存到：** `docs/superpowers/plans/YYYY-MM-DD-<feature-name>.md`
- （用户对计划位置的偏好覆盖此默认值）

## 范围检查

如果规范涵盖多个独立子系统，它应该已经在头脑风暴阶段被分解为子项目规范。如果没分解，建议把这个分解为多个单独的计划 —— 每个子系统一个。每个计划应**单独**产出可工作、可测试的软件。

## 文件结构

在定义任务之前，绘制将要创建或修改的文件以及每个文件的职责。这是分解决策被锁定的地方。

- 用清晰的边界和明确定义的接口设计单元。每个文件应该有一个清晰的职责。
- 你对能一次性放入上下文的代码推理得最好，并且当文件聚焦时你的编辑更可靠。**优先**小而聚焦的文件，而不是做得太多的大文件。
- **一起改的文件应该放在一起。** 按职责拆分，而不是按技术层。
- 在现有代码库中，遵循既定模式。如果代码库使用大文件，不要单方面重构 —— 但如果你正在修改的文件已经变得笨重，在计划中包括一次拆分是合理的。

此结构告知任务分解。每个任务应产出**自包含**的、单独讲得通的改动。

## 一口大小的任务粒度

**每个步骤是一个动作（2-5 分钟）：**
- "Write the failing test" - 步骤
- "Run it to make sure it fails" - 步骤
- "Implement the minimal code to make the test pass" - 步骤
- "Run the tests and make sure they pass" - 步骤
- "Commit" - 步骤

## 计划文档头部

**每个计划必须以这个头部开始：**

```markdown
# [Feature Name] Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** [One sentence describing what this builds]

**Architecture:** [2-3 sentences about approach]

**Tech Stack:** [Key technologies/libraries]

---
```

## 任务结构

````markdown
### Task N: [Component Name]

**Files:**
- Create: `exact/path/to/file.py`
- Modify: `exact/path/to/existing.py:123-145`
- Test: `tests/exact/path/to/test.py`

- [ ] **Step 1: Write the failing test**

```python
def test_specific_behavior():
    result = function(input)
    assert result == expected
```

- [ ] **Step 2: Run test to verify it fails**

Run: `pytest tests/path/test.py::test_name -v`
Expected: FAIL with "function not defined"

- [ ] **Step 3: Write minimal implementation**

```python
def function(input):
    return expected
```

- [ ] **Step 4: Run test to verify it passes**

Run: `pytest tests/path/test.py::test_name -v`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add tests/path/test.py src/path/file.py
git commit -m "feat: add specific feature"
```
````

## 没有占位符

每个步骤必须包含工程师需要的实际内容。这些是**计划失败** —— 永远不要写：
- "TBD"、"TODO"、"implement later"、"fill in details"
- "Add appropriate error handling" / "add validation" / "handle edge cases"
- "Write tests for the above"（不带实际测试代码）
- "Similar to Task N"（重复代码 —— 工程师可能不按顺序读任务）
- 描述"做什么"但不展示"怎么做"的步骤（代码步骤需要代码块）
- 引用任何任务中未定义的类型、函数或方法

## 切记
- 始终是精确的文件路径
- 每个步骤的完整代码 —— 如果步骤改代码，展示代码
- 精确的命令及预期输出
- DRY、YAGNI、TDD、频繁提交

## 自审

写完完整计划后，用全新的眼光看规范并对照计划检查。这是你自己跑的清单 —— 不是分派子代理。

**1. 规范覆盖：** 略读规范中的每个章节/需求。你能指出实现它的任务吗？列出任何差距。

**2. 占位符扫描：** 在你的计划中搜索红旗 —— 上面"没有占位符"章节中的任何模式。修复它们。

**3. 类型一致性：** 你在后面任务中使用的类型、方法签名、属性名是否匹配你在前面任务中定义的？Task 3 中调用 `clearLayers()` 但 Task 7 中是 `clearFullLayers()` 是个 bug。

如果你发现问题，在原地修复。无需重新审查 —— 修复并继续。如果你发现一个规范需求没有对应任务，添加该任务。

## 执行交接

保存计划后，提供执行选择：

**"Plan complete and saved to `docs/superpowers/plans/<filename>.md`. Two execution options:**

**1. Subagent-Driven (recommended)** - I dispatch a fresh subagent per task, review between tasks, fast iteration

**2. Inline Execution** - Execute tasks in this session using executing-plans, batch execution with checkpoints

**Which approach?"**

**如果选择了 Subagent-Driven：**
- **必需的子技能：** 使用 `superpowers:subagent-driven-development`
- 每个任务一个全新的子代理 + 两阶段审查

**如果选择了 Inline Execution：**
- **必需的子技能：** 使用 `superpowers:executing-plans`
- 带审查检查点的批量执行
