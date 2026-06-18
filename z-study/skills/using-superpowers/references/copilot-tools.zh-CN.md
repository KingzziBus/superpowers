# Copilot CLI Tool Mapping（Copilot CLI 工具映射）

技能使用 Claude Code 的工具名。当你遇到这些出现在某个技能中时，使用你平台的等价物：

| 技能引用 | Copilot CLI 等价物 |
|----------|---------------------|
| `Read`（文件读取） | `view` |
| `Write`（文件创建） | `create` |
| `Edit`（文件编辑） | `edit` |
| `Bash`（运行命令） | `bash` |
| `Grep`（搜索文件内容） | `grep` |
| `Glob`（按名搜索文件） | `glob` |
| `Skill` 工具（调用技能） | `skill` |
| `WebFetch` | `web_fetch` |
| `Task` 工具（分派子代理） | `task` 带 `agent_type: "general-purpose"` 或 `"explore"` |
| 多个 `Task` 调用（并行） | 多个 `task` 调用 |
| Task 状态/输出 | `read_agent`、`list_agents` |
| `TodoWrite`（任务跟踪） | `sql` 带内建 `todos` 表 |
| `WebSearch` | 无等价物 —— 使用 `web_fetch` 加搜索引擎 URL |
| `EnterPlanMode` / `ExitPlanMode` | 无等价物 —— 保持在主会话中 |

## 异步 shell 会话

Copilot CLI 支持持久化的异步 shell 会话，没有直接的 Claude Code 等价物：

| 工具 | 用途 |
|------|------|
| `bash` 带 `async: true` | 在后台启动长时间运行的命令 |
| `write_bash` | 向正在运行的异步会话发送输入 |
| `read_bash` | 从异步会话读取输出 |
| `stop_bash` | 终止异步会话 |
| `list_bash` | 列出所有活跃的 shell 会话 |

## 其他 Copilot CLI 工具

| 工具 | 用途 |
|------|------|
| `store_memory` | 为未来会话持久化关于代码库的事实 |
| `report_intent` | 用当前意图更新 UI 状态行 |
| `sql` | 查询会话的 SQLite 数据库（todos、元数据） |
| `fetch_copilot_cli_documentation` | 查阅 Copilot CLI 文档 |
| GitHub MCP 工具（`github-mcp-server-*`） | 原生 GitHub API 访问（issue、PR、代码搜索） |
