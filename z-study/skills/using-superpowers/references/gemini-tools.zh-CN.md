# Gemini CLI Tool Mapping（Gemini CLI 工具映射）

技能使用 Claude Code 的工具名。当你遇到这些出现在某个技能中时，使用你平台的等价物：

| 技能引用 | Gemini CLI 等价物 |
|----------|---------------------|
| `Read`（文件读取） | `read_file` |
| `Write`（文件创建） | `write_file` |
| `Edit`（文件编辑） | `replace` |
| `Bash`（运行命令） | `run_shell_command` |
| `Grep`（搜索文件内容） | `grep_search` |
| `Glob`（按名搜索文件） | `glob` |
| `TodoWrite`（任务跟踪） | `write_todos` |
| `Skill` 工具（调用技能） | `activate_skill` |
| `WebSearch` | `google_web_search` |
| `WebFetch` | `web_fetch` |
| `Task` 工具（分派子代理） | `@agent-name`（参见 [Subagent support](#subagent-support)） |

## Subagent support（子代理支持）

Gemini CLI 通过 `@` 语法原生支持子代理。使用内建的 `@generalist` 代理来分派任何任务 —— 它可以访问所有工具并遵循你提供的 prompt。

当某个技能说要分派一个具名的代理类型时，使用 `@generalist` 加上技能 prompt 模板中的完整 prompt：

| 技能指令 | Gemini CLI 等价物 |
|----------|---------------------|
| `Task tool (superpowers:implementer)` | `@generalist` 加上填好的 `implementer-prompt.md` 模板 |
| `Task tool (superpowers:spec-reviewer)` | `@generalist` 加上填好的 `spec-reviewer-prompt.md` 模板 |
| `Task tool (superpowers:code-reviewer)` | `@code-reviewer`（捆绑的代理）或 `@generalist` 加上填好的审查 prompt |
| `Task tool (superpowers:code-quality-reviewer)` | `@generalist` 加上填好的 `code-quality-reviewer-prompt.md` 模板 |
| `Task tool (general-purpose)` 带内联 prompt | `@generalist` 加上你的内联 prompt |

### Prompt 填充

技能提供带占位符的 prompt 模板，如 `{WHAT_WAS_IMPLEMENTED}` 或 `[FULL TEXT of task]`。填充所有占位符并将完整 prompt 作为消息传递给 `@generalist`。Prompt 模板本身包含代理的角色、审查标准和预期输出格式 —— `@generalist` 会遵循它。

### 并行分派

Gemini CLI 支持并行子代理分派。当某个技能要求你并行分派多个独立的子代理任务时，在同一 prompt 中一起请求所有这些 `@generalist` 或具名子代理任务。保持依赖任务顺序，但不要为了保留更简单的历史而把独立子代理任务串行化。

## 其他 Gemini CLI 工具

这些工具在 Gemini CLI 中可用但没有 Claude Code 等价物：

| 工具 | 用途 |
|------|------|
| `list_directory` | 列出文件和子目录 |
| `save_memory` | 把事实持久化到 GEMINI.md 跨会话保留 |
| `ask_user` | 请求用户的结构化输入 |
| `tracker_create_task` | 富任务管理（创建、更新、列表、可视化） |
| `enter_plan_mode` / `exit_plan_mode` | 在做改动前切换到只读研究模式 |
