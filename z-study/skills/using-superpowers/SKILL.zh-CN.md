---
name: using-superpowers
description: Use when starting any conversation - establishes how to find and use skills, requiring Skill tool invocation before ANY response including clarifying questions
---

<SUBAGENT-STOP>
如果你作为子代理被分派去执行特定任务，跳过本技能。
</SUBAGENT-STOP>

<EXTREMELY-IMPORTANT>
如果你认为有哪怕 1% 的可能性某个技能可能适用于你正在做的事，你**绝对必须**调用该技能。

**如果某个技能适用于你的任务，你没有选择。你必须用它。**

这不可谈判。这不是可选的。你不能用合理化的方式回避它。
</EXTREMELY-IMPORTANT>

## 指令优先级

Superpowers 技能覆盖默认系统提示行为，但**用户指令始终优先**：

1. **用户的显式指令**（CLAUDE.md、GEMINI.md、AGENTS.md、直接请求）—— 最高优先级
2. **Superpowers 技能** —— 在冲突时覆盖默认系统行为
3. **默认系统提示** —— 最低优先级

如果 CLAUDE.md、GEMINI.md 或 AGENTS.md 说"不要用 TDD"而某个技能说"始终用 TDD"，遵从用户的指令。用户掌控一切。

## 如何访问技能

**在 Claude Code 中：** 使用 `Skill` 工具。当你调用一个技能，它的内容会被加载并展示给你 —— 直接遵循它。**永远不要**用 Read 工具读技能文件。

**在 Copilot CLI 中：** 使用 `skill` 工具。技能从已安装的插件自动发现。`skill` 工具的工作方式与 Claude Code 的 `Skill` 工具相同。

**在 Gemini CLI 中：** 技能通过 `activate_skill` 工具激活。Gemini 在会话开始时加载技能元数据，按需激活完整内容。

**在其他环境中：** 查阅你平台的文档了解如何加载技能。

## 平台适配

技能使用 Claude Code 的工具名。非 CC 平台：参见 `references/copilot-tools.md`（Copilot CLI）、`references/codex-tools.md`（Codex）了解工具等价物。Gemini CLI 用户通过 GEMINI.md 自动获得工具映射。

# 使用技能

## 规则

**在任何响应或行动之前调用相关或被请求的技能。** 哪怕只有 1% 的可能性某个技能可能适用，你都应该调用该技能来检查。如果调用的技能结果不适用于当前情况，你不需要用它。

```dot
digraph skill_flow {
    "User message received" [shape=doublecircle];
    "About to EnterPlanMode?" [shape=doublecircle];
    "Already brainstormed?" [shape=diamond];
    "Invoke brainstorming skill" [shape=box];
    "Might any skill apply?" [shape=diamond];
    "Invoke Skill tool" [shape=box];
    "Announce: 'Using [skill] to [purpose]'" [shape=box];
    "Has checklist?" [shape=diamond];
    "Create TodoWrite todo per item" [shape=box];
    "Follow skill exactly" [shape=box];
    "Respond (including clarifications)" [shape=doublecircle];

    "About to EnterPlanMode?" -> "Already brainstormed?";
    "Already brainstormed?" -> "Invoke brainstorming skill" [label="no"];
    "Already brainstormed?" -> "Might any skill apply?" [label="yes"];
    "Invoke brainstorming skill" -> "Might any skill apply?";

    "User message received" -> "Might any skill apply?";
    "Might any skill apply?" -> "Invoke Skill tool" [label="yes, even 1%"];
    "Might any skill apply?" -> "Respond (including clarifications)" [label="definitely not"];
    "Invoke Skill tool" -> "Announce: 'Using [skill] to [purpose]'";
    "Announce: 'Using [skill] to [purpose]'" -> "Has checklist?";
    "Has checklist?" -> "Create TodoWrite todo per item" [label="yes"];
    "Has checklist?" -> "Follow skill exactly" [label="no"];
    "Create TodoWrite todo per item" -> "Follow skill exactly";
}
```

## 红旗

这些想法意味着**停下** —— 你在合理化：

| 想法 | 真相 |
|------|------|
| "这只是一个简单的问题" | 问题就是任务。检查技能。 |
| "我需要先了解上下文" | 技能检查在澄清问题**之前**。 |
| "让我先探索代码库" | 技能告诉你**如何**探索。先检查。 |
| "我可以快速检查 git/文件" | 文件缺少会话上下文。检查技能。 |
| "让我先收集信息" | 技能告诉你**如何**收集信息。 |
| "这不需要正式技能" | 如果有技能存在，就用它。 |
| "我记得这个技能" | 技能会演化。读当前版本。 |
| "这不算任务" | 行动 = 任务。检查技能。 |
| "这个技能是过度" | 简单的事会变复杂。用它。 |
| "我就先做这一件事" | 在做任何事**之前**先检查。 |
| "这感觉有成效" | 无纪律的行动浪费时间。技能防止这个。 |
| "我知道那是什么意思" | 知道概念 ≠ 用技能。调用它。 |

## 技能优先级

当多个技能可能适用时，使用此顺序：

1. **流程技能优先**（brainstorming、debugging）—— 这些决定**如何**处理任务
2. **实施技能其次**（frontend-design、mcp-builder）—— 这些指导执行

"我们来做 X" → 先 brainstorming，然后实施技能。
"修复这个 bug" → 先 debugging，然后领域特定技能。

## 技能类型

**严格型**（TDD、debugging）：严格遵循。不要为了方便而削弱纪律。

**灵活型**（patterns）：根据上下文调整原则。

技能本身会告诉你它属于哪一类。

## 用户指令

指令说的是**做什么**，不是**怎么做**。"加 X"或"修 Y"不意味着跳过工作流。
