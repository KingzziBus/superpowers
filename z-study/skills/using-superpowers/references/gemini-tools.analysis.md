# gemini-tools.md 分析报告

## 基本信息

| 项目 | 内容 |
| --- | --- |
| 文档路径 | `skills/using-superpowers/references/gemini-tools.md` |
| 字数规模 | 约 52 行 / 2.9KB |
| 文档类型 | 平台适配参考 / 工具映射表 |
| 主题 | Claude Code 工具名 → Gemini CLI 等价物的映射 |
| 核心定位 | Superpowers 在 Gemini CLI 上的"运行时翻译层" |
| 核心特性 | `@` 语法子代理、Prompt 填充模式 |
| 面向读者 | 在 Gemini CLI 上运行 Superpowers 技能的 AI 代理 |

---

## 一、文档目的

解决 Superpowers 在 Google Gemini CLI 上的**跨平台运行**问题。

具体目标：
1. **建立工具名映射**：11 个 Claude Code 工具 → Gemini CLI 等价物
2. **揭示子代理支持机制**：`@` 语法 vs Claude Code 的 `Task` 工具
3. **Prompt 填充模式**：技能 prompt 模板如何与 `@generalist` 配合
4. **并行分派规则**：什么时候并行、什么时候串行

定位上属于 **"Gemini CLI 平台运行时翻译层"**——是 `using-superpowers` 母技能的多平台扩展。

---

## 二、核心技术决策

### 决策 1：`@` 语法作为子代理调用方式

```text
Task 工具（分派子代理）→ @agent-name
```

**设计意图**：
- `@` 是聊天工具的常见语法（Slack、Discord、邮件等）
- Gemini CLI 用 `@` 触发子代理——**自然、直观**
- 不需要额外的 `Task` 工具调用

**这是"聊天式编程"的设计**。

### 决策 2：`@generalist` 作为通用子代理

```text
@generalist with implementer-prompt.md template
```

**关键洞察**：
- Gemini CLI **不强制** "implementer"、"spec-reviewer" 等具名代理
- 一个 `@generalist` + 填好的 prompt 模板 = 等价
- **降低** 平台维护成本（不需要预定义多种代理）

**这是"通用化代理"的设计**。

### 决策 3：具名子代理的可选支持

```text
@code-reviewer（捆绑的代理）或 @generalist with review prompt
```

**设计意图**：
- 提供**两种选项**——预捆绑的 `@code-reviewer`（开箱即用）或 `@generalist`（灵活）
- 用户**选择**便利性 vs 灵活性

**这是"灵活性与便利性并重"**。

### 决策 4：Prompt 填充模式

```text
技能提供带占位符的 prompt 模板
填充所有占位符并将完整 prompt 作为消息传递给 @generalist
```

**精妙之处**：
- 技能**自带** prompt 模板
- 调用方**只负责**填充占位符
- `@generalist` 执行模板的指令

**这是"模板 + 填充"模式**。

### 决策 5：并行分派的明确规则

```text
在同一 prompt 中一起请求所有这些 @generalist 或具名子代理任务
保持依赖任务顺序
但不要为了保留更简单的历史而把独立子代理任务串行化
```

**关键洞察**：
- 独立任务 → **并行**
- 依赖任务 → **串行**
- 不要"为了简单"而串行化独立任务

**这是"并行优化的明确规则"**。

### 决策 6：5 个 Gemini CLI 特有工具

| 工具 | 用途 |
|------|------|
| `list_directory` | 列出文件/目录 |
| `save_memory` | 持久化到 GEMINI.md 跨会话 |
| `ask_user` | 结构化输入 |
| `tracker_create_task` | 富任务管理 |
| `enter_plan_mode` / `exit_plan_mode` | 只读研究模式 |

**设计意图**：
- 主动暴露**平台特有能力**
- 特别 `enter_plan_mode`——对应 Claude Code 的 EnterPlanMode
- `save_memory`——类似 Claude Code 的 memory 机制

**这是"平台能力暴露"**。

### 决策 7：完整工具映射

11 个工具全部映射——**没有"无等价物"**。

**对比**：
- copilot-tools 有 2 个无等价物
- codex-tools 没有无等价物（但只映射 9 个）

**Gemini CLI 工具覆盖最全**。

---

## 三、文件结构

| 章节 | 内容 | 工程价值 |
| --- | --- | --- |
| 工具映射表 | 11 个工具 | 核心 |
| 子代理支持 | 详细机制 | 关键特性 |
| Prompt 填充 | 模板模式 | 关键模式 |
| 并行分派 | 规则 | 性能 |
| 其他工具 | 5 个特有 | 平台特有 |

**结构特点**：**核心表 + 详细子代理章节 + 平台特有**——比 copilot-tools 更详细。

---

## 四、优点

### 1. `@` 语法直观

聊天式编程——比 Claude Code 的 `Task` 工具调用更**自然**。

### 2. `@generalist` 通用化代理

不需要预定义多种代理——一个通用代理 + 模板 = 完成。

### 3. 完整工具映射（11 个）

没有"无等价物"——Gemini CLI 工具覆盖最全。

### 4. Prompt 填充模式清晰

技能自带模板，调用方只填占位符——**关注点分离**。

### 5. 并行分派规则明确

独立并行、依赖串行——性能优化的清晰规则。

### 6. `enter_plan_mode` 对应 Claude Code 的 EnterPlanMode

**关键洞察**：
- Gemini CLI **支持** PlanMode
- 通过 `enter_plan_mode` 触发
- 这是 Superpowers 决策图中 PlanMode 检查的基础

---

## 五、缺点与风险

### 1. `@generalist` 不如具名代理结构化

**问题**：
- 所有子代理都是 `@generalist`——没有专门优化
- 技能 prompt 模板的所有 burden 在调用方

**风险**：prompt 模板质量决定代理质量。

### 2. 缺 `@` 语法边界

**问题**：
- `@generalist` 触发的范围是什么？
- 同一消息多个 `@` 算几个独立代理？

**缺失**：`@` 语法的边界规则。

### 3. 缺 prompt 模板示例

**问题**：
- "Fill all placeholders" —— 但**没给示例**
- 用户可能填错

**缺失**：具体 prompt 模板示例。

### 4. 并行分派的"消息格式"没说

**问题**：
- "同一 prompt 中一起请求"——具体什么格式？
- 多个 `@` 怎么排版？

**缺失**：具体格式示例。

### 5. 缺 `@code-reviewer` 的细节

**问题**：
- 提到"bundled agent"——但**没说什么能力**
- 和 `@generalist` with review prompt 的**区别**是什么？

**缺失**：具名代理的能力说明。

### 6. 缺性能/成本说明

**问题**：
- `@generalist` 消耗多少 token？
- 并行分派的成本？

**缺失**：性能/成本基准。

---

## 六、关键洞察

### 洞察 1：`@` 语法是"聊天式编程"的设计

**关键洞察**：
- `@` 是**聊天工具**的常见语法
- Gemini CLI 把子代理调用**伪装成聊天提及**
- 用户体验**自然**——"像 @ 同事一样"

**这是"用户体验优先"的设计**。

### 洞察 2：`@generalist` 是"通用化代理"

**对比**：
- Claude Code 的 `Task` 工具 → 可指定 agent_type
- Gemini CLI 的 `@generalist` → **没有类型**

**设计权衡**：
- 通用化 = **平台简单**（少定义）
- 类型化 = **技能可控**（多优化）

**Gemini CLI 选择前者**——保持平台简单。

### 洞察 3：Prompt 填充模式是"关注点分离"

**关键洞察**：
- 技能 = **模板**（知道要做什么）
- 调用方 = **填充**（知道具体情况）
- `@generalist` = **执行**（遵循模板）

**这是"职责分离"模式**。

### 洞察 4：并行分派规则是"性能优化的明确化"

**关键洞察**：
- 显式说"不要为了简单而串行"
- 显式说"独立任务并行"
- **不依赖** AI 自己的"判断"

**这是"性能优化从软文化变硬规则"**。

### 洞察 5：`enter_plan_mode` 是 Superpowers 决策图的基础

**关键洞察**：
- Gemini CLI **原生支持** PlanMode
- `using-superpowers` 决策图中的 `About to EnterPlanMode?` 节点**依赖** 这个工具
- 平台特性 = 技能的基础设施

**这是"平台能力支撑上层技能"**。

### 洞察 6：5 个平台特有工具是"差异化优势"

| 工具 | 价值 |
|------|------|
| `list_directory` | 比 Claude Code 的 Glob 更直接 |
| `save_memory` | 跨会话持久化 |
| `ask_user` | 结构化用户输入 |
| `tracker_create_task` | 富任务管理 |
| `enter_plan_mode` | PlanMode 集成 |

**关键洞察**：
- Gemini CLI 不只做"等价映射"
- 主动**暴露独有能力**
- 让技能作者**利用**这些能力

**这是"做加法"原则**。

### 洞察 7：与 codex/copilot 对比

| 维度 | codex | copilot | gemini |
|------|-------|---------|--------|
| 工具数 | 9 | 14 | 11 |
| 平台特有 | multi_agent | 异步 shell, GitHub MCP | @ 语法, PlanMode |
| 子代理机制 | spawn_agent | task with agent_type | @agent-name |
| Legacy 说明 | 有 | 无 | 无 |

**启示**：3 个平台的"特殊点"完全不同——**没有统一模板**。

---

## 七、关联文档

| 文档 | 关系 |
| --- | --- |
| `skills/using-superpowers/SKILL.md` | 母技能 |
| `skills/using-superpowers/references/codex-tools.md` | 平行：另一个平台映射 |
| `skills/using-superpowers/references/copilot-tools.md` | 平行：另一个平台映射 |
| `skills/subagent-driven-development/SKILL.md` | 下游：使用 `@generalist` 模式 |
| `skills/brainstorming/SKILL.md` | 下游：使用 `enter_plan_mode` |
| Gemini CLI 官方文档 | 上游 |

---

## 八、状态评价

| 维度 | 评分 | 说明 |
| --- | --- | --- |
| **翻译完整性** | ⭐⭐⭐⭐ (4/5) | 11 个工具，无"无等价物" |
| **平台特有暴露** | ⭐⭐⭐⭐⭐ (5/5) | `@` 语法、PlanMode、save_memory |
| **子代理机制** | ⭐⭐⭐⭐ (4/5) | `@generalist` 通用化好，但缺能力细节 |
| **Prompt 模式** | ⭐⭐⭐⭐ (4/5) | 模板 + 填充模式好，缺示例 |
| **并行规则** | ⭐⭐⭐⭐ (4/5) | 规则明确，缺消息格式 |
| **诚实性** | ⭐⭐⭐⭐⭐ (5/5) | 没有"无等价物"——Gemini CLI 工具覆盖最全 |

**总体评价**：**Gemini CLI 平台映射的完整参考**。11 个工具映射 + `@` 语法详细说明 + Prompt 填充模式 + 并行规则构成了对 Gemini 编程工具的完整支持。**`@` 语法、PlanMode 支持、save_memory** 是亮点。**`@generalist` 能力细节、Prompt 示例、消息格式** 是不足。

---

## 九、给读者的启示

### 如果你在 Gemini CLI 上用 Superpowers
- **遇到 `Read`/`Write`/`Bash`**：用 `read_file`/`write_file`/`run_shell_command`
- **遇到 `Task`**：用 `@agent-name`（如 `@generalist`）
- **遇到 `EnterPlanMode`**：用 `enter_plan_mode`
- **想用 PlanMode**：触发 `enter_plan_mode` → Superpowers 决策图接管
- **想并行分派**：在同一 prompt 多个 `@`
- **想跨会话记忆**：用 `save_memory`

### 如果你写新技能
- **始终用 Claude Code 工具名**——不要写 Gemini CLI 特定工具
- 利用**平台特有能力**：`@` 语法、PlanMode、save_memory

### 如果你维护 Superpowers
- **改进建议 1**：增加 prompt 模板的具体示例
- **改进建议 2**：增加 `@` 语法的边界规则
- **改进建议 3**：增加 `@code-reviewer` 等具名代理的能力说明
- **改进建议 4**：增加并行分派的消息格式示例
- **改进建议 5**：增加 Gemini CLI 性能/成本基准

---

## 十、后续工作

按重要性排序：

1. **中优先级**：增加 prompt 模板的具体示例
2. **中优先级**：增加 `@` 语法的边界规则
3. **中优先级**：增加并行分派的消息格式示例
4. **低优先级**：增加 Gemini CLI 性能/成本基准
5. **低优先级**：增加 `@generalist` 能力的详细说明

---

**总结**：这个 52 行的文件是 **"Gemini CLI 平台映射的完整参考"**。11 个工具映射 + `@` 语法详细说明 + Prompt 填充模式 + 并行规则构成了完整支持。**`@` 语法、PlanMode 集成、save_memory** 是亮点。**Prompt 示例、消息格式、能力细节** 是不足。

对在 Gemini CLI 上使用 Superpowers 的任何人来说，**这是必读必用的参考**。其"`@` 语法 + 通用化代理 + PlanMode 集成"的设计哲学值得所有跨平台工具学习。
