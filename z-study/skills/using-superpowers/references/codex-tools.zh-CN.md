# Codex Tool Mapping（Codex 工具映射）

技能使用 Claude Code 的工具名。当你遇到这些出现在某个技能中时，使用你平台的等价物：

| 技能引用 | Codex 等价物 |
|----------|--------------|
| `Task` 工具（分派子代理） | `spawn_agent`（参见 [Subagent dispatch requires multi-agent support](#subagent-dispatch-requires-multi-agent-support)） |
| 多个 `Task` 调用（并行） | 多个 `spawn_agent` 调用 |
| Task 返回结果 | `wait_agent` |
| Task 自动完成 | `close_agent` 释放槽位 |
| `TodoWrite`（任务跟踪） | `update_plan` |
| `Skill` 工具（调用技能） | 技能原生加载 —— 直接遵循说明即可 |
| `Read`、`Write`、`Edit`（文件） | 使用你的原生文件工具 |
| `Bash`（运行命令） | 使用你的原生 shell 工具 |

## Subagent dispatch requires multi-agent support

子代理分派需要多代理支持

添加到你的 Codex 配置（`~/.codex/config.toml`）：

```toml
[features]
multi_agent = true
```

这为 `dispatching-parallel-agents` 和 `subagent-driven-development` 等技能启用了 `spawn_agent`、`wait_agent` 和 `close_agent`。

旧版说明：`rust-v0.115.0` 之前的 Codex 构建把分派代理的等待暴露为 `wait`。当前 Codex 对分派代理使用 `wait_agent`。`wait` 名称现在属于 code-mode 的 `exec/wait`，它通过 `cell_id` 恢复一个 yield 的 exec 单元；它不是分派代理的结果工具。

## Environment Detection（环境检测）

创建 worktree 或完成分支的技能在继续之前应该用只读 git 命令检测它们的环境：

```bash
GIT_DIR=$(cd "$(git rev-parse --git-dir)" 2>/dev/null && pwd -P)
GIT_COMMON=$(cd "$(git rev-parse --git-common-dir)" 2>/dev/null && pwd -P)
BRANCH=$(git branch --show-current)
```

- `GIT_DIR != GIT_COMMON` → 已在 linked worktree（跳过创建）
- `BRANCH` 为空 → 分离 HEAD（不能从沙箱中 branch/push/PR）

参见 `using-git-worktrees` 第 0 步和 `finishing-a-development-branch` 第 1 步了解每个技能如何使用这些信号。

## Codex App Finishing（Codex 应用完成）

当沙箱阻止 branch/push 操作（外部管理的 worktree 中分离 HEAD）时，代理提交所有工作并通知用户使用 App 的原生控件：

- **"Create branch"** —— 命名分支，然后通过 App UI 进行 commit/push/PR
- **"Hand off to local"** —— 把工作转交给用户的本地 checkout

代理仍然可以跑测试、暂存文件、输出建议的分支名、commit message 和 PR 描述供用户复制。
