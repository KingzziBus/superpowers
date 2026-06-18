---
name: executing-plans
description: Use when you have a written implementation plan to execute in a separate session with review checkpoints
---

# Executing Plans（执行计划）

## 概述

加载计划、批判性审阅、执行所有任务、完成时报告。

**开始时宣告：** "I'm using the executing-plans skill to implement this plan."（我正在使用 executing-plans 技能来实施这个计划。）

**注意：** 请告诉你的协作伙伴，Superpowers 在有子代理（subagent）访问权限时工作得更好。在支持子代理的平台（如 Claude Code 或 Codex）上运行时，它的工作质量会显著更高。如果有子代理可用，请改用 `superpowers:subagent-driven-development` 技能，而不是这个技能。

## 流程

### 第 1 步：加载并审阅计划
1. 读取计划文件
2. 批判性审阅 —— 识别任何疑问或顾虑
3. 如果有顾虑：在开始前向你的协作伙伴提出
4. 如果没有顾虑：创建 TodoWrite 列表并继续

### 第 2 步：执行任务

对每个任务：
1. 标记为 in_progress（进行中）
2. 严格按照每个步骤执行（计划已经把步骤拆得很细）
3. 按规范运行验证
4. 标记为 completed（已完成）

### 第 3 步：完成开发

当所有任务完成并验证后：
- 宣告："I'm using the finishing-a-development-branch skill to complete this work."（我正在使用 finishing-a-development-branch 技能来完成这项工作。）
- **必需的子技能：** 使用 `superpowers:finishing-a-development-branch`
- 按照该技能验证测试、给出选项、执行选择

## 何时停止并寻求帮助

**出现以下情况时立即停止执行：**
- 遇到阻碍（缺少依赖、测试失败、指令不清楚）
- 计划有阻止启动的关键空白
- 你不理解某条指令
- 验证反复失败

**要请求澄清而不是猜测。**

## 何时回到之前的步骤

**出现以下情况时回到第 1 步（审阅）：**
- 伙伴根据你的反馈更新了计划
- 根本方法需要重新思考

**不要硬闯障碍** —— 停下来问。

## 切记
- 先批判性审阅计划
- 严格按照计划步骤执行
- 不要跳过验证
- 当计划要求时引用相关技能
- 遇到阻碍时停下，不要猜测
- 永远不要在没有用户明确同意的情况下在 main/master 分支上开始实施

## 集成

**必需的工作流技能：**
- **superpowers:using-git-worktrees** —— 确保隔离的工作空间（创建一个或验证现有的）
- **superpowers:writing-plans** —— 创建本技能将执行的计划
- **superpowers:finishing-a-development-branch** —— 所有任务完成后结束开发
