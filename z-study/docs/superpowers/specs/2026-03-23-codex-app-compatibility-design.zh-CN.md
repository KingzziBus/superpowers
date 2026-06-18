# Codex App 兼容性：Worktree 与 Finishing 技能适配

让 superpowers 技能在 Codex App 的沙盒化 worktree 环境中工作，且不破坏现有 Claude Code 或 Codex CLI 行为。

**工单：** PRI-823

## 动机

Codex App 在它管理的 git worktree 内运行智能体——分离头指针（detached HEAD），位于 `$CODEX_HOME/worktrees/`，并使用 Seatbelt 沙盒阻止 `git checkout -b`、`git push` 和网络访问。三个 superpowers 技能假设无限制的 git 访问：`using-git-worktrees` 用命名分支创建手动 worktree，`finishing-a-development-branch` 按分支名合并/推送/PR，`subagent-driven-development` 同时依赖两者。

Codex CLI（开源终端工具）**没有**这个冲突——它没有内建的 worktree 管理。我们的手动 worktree 方法在那里填补了隔离空白。问题特定于 Codex App。

## 经验性发现

2026-03-23 在 Codex App 中测试：

| 操作 | workspace-write 沙盒 | 完全访问沙盒 |
|---|---|---|
| `git add` | 可用 | 可用 |
| `git commit` | 可用 | 可用 |
| `git checkout -b` | **被阻止**（无法写入 `.git/refs/heads/`） | 可用 |
| `git push` | **被阻止**（网络 + `.git/refs/remotes/`） | 可用 |
| `gh pr create` | **被阻止**（网络） | 可用 |
| `git status/diff/log` | 可用 | 可用 |

其他发现：
- `spawn_agent` 子智能体**共享**父线程的文件系统（通过标记文件测试确认）
- "Create branch" 按钮出现在 App header 中，无论 worktree 从哪个分支启动
- App 的原生完成流程：Create branch → Commit 模态框 → Commit and push / Commit and create PR
- macOS 上 `network_access = true` 配置静默失效（issue #10390）

## 设计：只读环境检测

三条只读 git 命令检测环境且无副作用：

```bash
GIT_DIR=$(cd "$(git rev-parse --git-dir)" 2>/dev/null && pwd -P)
GIT_COMMON=$(cd "$(git rev-parse --git-common-dir)" 2>/dev/null && pwd -P)
BRANCH=$(git branch --show-current)
```

派生出两个信号：

- **IN_LINKED_WORKTREE（已处于关联 worktree）：** `GIT_DIR != GIT_COMMON` —— 智能体处于由其他方创建的 worktree 中（Codex App、Claude Code Agent 工具、之前运行的 skill、用户）
- **ON_DETACHED_HEAD（分离头指针）：** `BRANCH` 为空 —— 没有命名分支

为何用 `git-dir != git-common-dir` 而非检查 `show-toplevel`：
- 在普通仓库中，两者解析为同一 `.git` 目录
- 在关联 worktree 中，`git-dir` 为 `.git/worktrees/<name>` 而 `git-common-dir` 为 `.git`
- 在子模块中，两者相等——避免 `show-toplevel` 会产生的误报
- 通过 `cd && pwd -P` 解析可处理相对路径问题（`git-common-dir` 在普通仓库中返回相对的 `.git` 而在 worktree 中是绝对的）和符号链接（macOS 上 `/tmp` → `/private/tmp`）

### 决策矩阵

| 处于关联 Worktree？ | 分离头指针？ | 环境 | 动作 |
|---|---|---|---|
| 否 | 否 | Claude Code / Codex CLI / 普通 git | 完整的技能行为（不变） |
| 是 | 是 | Codex App worktree（workspace-write） | 跳过 worktree 创建；完成时发出交接载荷 |
| 是 | 否 | Codex App（完全访问）或手动 worktree | 跳过 worktree 创建；完整完成流程 |
| 否 | 是 | 异常（手动分离头指针） | 正常创建 worktree；完成时警告 |

## 变更内容

### 1. `using-git-worktrees/SKILL.md` —— 增加 Step 0（约 12 行）

在 "Overview" 和 "Directory Selection Process" 之间新增小节：

**Step 0: Check if Already in an Isolated Workspace**

运行检测命令。若 `GIT_DIR != GIT_COMMON`，**完全跳过** worktree 创建。改为：
1. 跳到 Creation Steps 下的 "Run Project Setup" 子小节 —— `npm install` 等是幂等的，为安全起见值得运行
2. 然后是 "Verify Clean Baseline" —— 运行测试
3. 按分支状态报告：
   - 在分支上："Already in an isolated workspace at `<path>` on branch `<name>`. Tests passing. Ready to implement."
   - 分离头指针："Already in an isolated workspace at `<path>` (detached HEAD, externally managed). Tests passing. Note: branch creation needed at finish time. Ready to implement."

若 `GIT_DIR == GIT_COMMON`，按原流程进行 worktree 创建（不变）。

Step 0 触发时跳过安全验证（.gitignore 检查）—— 对外部创建的 worktree 无关紧要。

更新 Integration 小节的 "Called by" 条目。将每个条目上的描述从上下文特定文本改为："Ensures isolated workspace (creates one or verifies existing)"。例如，`subagent-driven-development` 条目从 "REQUIRED: Set up isolated workspace before starting" 变为 "REQUIRED: Ensures isolated workspace (creates one or verifies existing)"。

**沙盒回退：** 若 `GIT_DIR == GIT_COMMON` 且技能继续到 Creation Steps，但 `git worktree add -b` 因权限错误失败（例如 Seatbelt 沙盒拒绝），将其视为**延迟检测到的受限环境**。回退到 Step 0 "已在 worktree 中" 行为 —— 跳过创建，在当前目录运行设置和基线测试，相应地报告。

Step 0 报告后，停止。不要继续 Directory Selection 或 Creation Steps。

**其余不变：** Directory Selection、Safety Verification、Creation Steps、Project Setup、Baseline Tests、Quick Reference、Common Mistakes、Red Flags。

### 2. `finishing-a-development-branch/SKILL.md` —— 增加 Step 1.5 + 清理守卫（约 20 行）

**Step 1.5: Detect Environment**（在 Step 1 "Verify Tests" 之后，Step 2 "Determine Base Branch" 之前）

运行检测命令。三种路径：

- **Path A** 完全跳过 Steps 2 和 3（不需要基础分支或选项）。
- **Paths B 和 C** 按正常流程经过 Step 2（Determine Base Branch）和 Step 3（Present Options）。

**Path A — 外部管理的 worktree + 分离头指针**（`GIT_DIR != GIT_COMMON` **且** `BRANCH` 为空）：

首先，确保所有工作已暂存并提交（`git add` + `git commit`）。Codex App 的完成控件作用于已提交的工作。

然后向用户呈现此内容（**不要**呈现 4 选项菜单）：

```
Implementation complete. All tests passing.
Current HEAD: <full-commit-sha>

This workspace is externally managed (detached HEAD).
I cannot create branches, push, or open PRs from here.

⚠ These commits are on a detached HEAD. If you do not create a branch,
they may be lost when this workspace is cleaned up.

If your host application provides these controls:
- "Create branch" — to name a branch, then commit/push/PR
- "Hand off to local" — to move changes to your local checkout

Suggested branch name: <ticket-id/short-description>
Suggested commit message: <summary-of-work>
```

分支名推导：若有工单 ID 则使用（如 `pri-823/codex-compat`），否则将 plan 标题的前 5 个词 slugify，否则省略建议。避免在分支名中包含敏感内容（漏洞描述、客户名）。

跳到 Step 5（对外部管理的 worktree，清理是无操作）。

**Path B — 外部管理的 worktree + 命名分支**（`GIT_DIR != GIT_COMMON` **且** `BRANCH` 存在）：

按正常流程呈现 4 选项菜单。（Step 5 清理守卫将独立重新检测外部管理状态。）

**Path C — 普通环境**（`GIT_DIR == GIT_COMMON`）：

按今天的方式呈现 4 选项菜单（不变）。

**Step 5 清理守卫：**

在清理时重新运行 `GIT_DIR` 与 `GIT_COMMON` 检测（不要依赖之前技能输出 —— finishing 技能可能在不同会话运行）。若 `GIT_DIR != GIT_COMMON`，跳过 `git worktree remove` —— 宿主环境拥有此 worktree。

否则，按今天的方式检查和移除。注意：现有 Step 5 文本说 "For Options 1, 2, 4" 但 Quick Reference 表格和 Common Mistakes 小节说 "Options 1 & 4 only."。新守卫添加在现有逻辑之前，不改变哪些选项触发清理。

**其余不变：** Options 1-4 逻辑、Quick Reference、Common Mistakes、Red Flags。

### 3. `subagent-driven-development/SKILL.md` 和 `executing-plans/SKILL.md` —— 每个 1 行编辑

两个技能都有相同的 Integration 小节行。从：

```
- superpowers:using-git-worktrees - REQUIRED: Set up isolated workspace before starting
```

改为：

```
- superpowers:using-git-worktrees - REQUIRED: Ensures isolated workspace (creates one or verifies existing)
```

**其余不变：** Dispatch/review loop、prompt 模板、模型选择、状态处理、red flags。

### 4. `codex-tools.md` —— 增加环境检测文档（约 15 行）

末尾两个新小节：

**Environment Detection：**

```markdown
## Environment Detection

Skills that create worktrees or finish branches should detect their
environment with read-only git commands before proceeding:

\```bash
GIT_DIR=$(cd "$(git rev-parse --git-dir)" 2>/dev/null && pwd -P)
GIT_COMMON=$(cd "$(git rev-parse --git-common-dir)" 2>/dev/null && pwd -P)
BRANCH=$(git branch --show-current)
\```

- `GIT_DIR != GIT_COMMON` → already in a linked worktree (skip creation)
- `BRANCH` empty → detached HEAD (cannot branch/push/PR from sandbox)

See `using-git-worktrees` Step 0 and `finishing-a-development-branch`
Step 1.5 for how each skill uses these signals.
```

**Codex App Finishing：**

```markdown
## Codex App Finishing

When the sandbox blocks branch/push operations (detached HEAD in an
externally managed worktree), the agent commits all work and informs
the user to use the App's native controls:

- **"Create branch"** — names the branch, then commit/push/PR via App UI
- **"Hand off to local"** — transfers work to the user's local checkout

The agent can still run tests, stage files, and output suggested branch
names, commit messages, and PR descriptions for the user to copy.
```

## 什么**不**变

- `implementer-prompt.md`、`spec-reviewer-prompt.md`、`code-quality-reviewer-prompt.md` —— 子智能体 prompt 不动
- `executing-plans/SKILL.md` —— 仅 1 行 Integration 描述变更（与 `subagent-driven-development` 相同）；所有运行时行为不变
- `dispatching-parallel-agents/SKILL.md` —— 无 worktree 或完成操作
- `.codex/INSTALL.md` —— 安装流程不变
- 4 选项完成菜单 —— 在 Claude Code 和 Codex CLI 上保持原样
- 完整 worktree 创建流程 —— 在非 worktree 环境中保持原样
- 子智能体 dispatch/review/iterate 循环 —— 不变（已确认文件系统共享）

## 范围汇总

| 文件 | 变更 |
|---|---|
| `skills/using-git-worktrees/SKILL.md` | +12 行（Step 0） |
| `skills/finishing-a-development-branch/SKILL.md` | +20 行（Step 1.5 + 清理守卫） |
| `skills/subagent-driven-development/SKILL.md` | 1 行编辑 |
| `skills/executing-plans/SKILL.md` | 1 行编辑 |
| `skills/using-superpowers/references/codex-tools.md` | +15 行 |

5 个文件中共计约 50 行新增/修改。零新增文件。零破坏性变更。

## 未来考虑

若第三个技能需要相同的检测模式，将其提取到共享的 `references/environment-detection.md` 文件（Approach B）。目前不需要 —— 只有 2 个技能使用它。

## 测试计划

### 自动化（在 Claude Code 中实施后运行）

1. 普通仓库检测 —— 断言 IN_LINKED_WORKTREE=false
2. 关联 worktree 检测 —— `git worktree add` 测试 worktree，断言 IN_LINKED_WORKTREE=true
3. 分离头指针检测 —— `git checkout --detach`，断言 ON_DETACHED_HEAD=true
4. Finishing 技能交接输出 —— 验证受限环境下交接消息（而非 4 选项菜单）
5. **Step 5 清理守卫** —— 创建关联 worktree（`git worktree add /tmp/test-cleanup -b test-cleanup`），`cd` 进入，运行 Step 5 清理检测（`GIT_DIR` vs `GIT_COMMON`），断言它**不会**调用 `git worktree remove`。然后 `cd` 回到主仓库，运行相同检测，断言它**会**调用 `git worktree remove`。之后清理测试 worktree。

### 手动 Codex App 测试（5 项测试）

1. Worktree 线程中的检测（workspace-write）—— 验证 GIT_DIR != GIT_COMMON，分支为空
2. Worktree 线程中的检测（完全访问）—— 相同检测，不同沙盒行为
3. Finishing 技能交接格式 —— 验证智能体发出交接载荷而非 4 选项菜单
4. 完整生命周期 —— 检测 → 提交 → 完成检测 → 正确行为 → 清理
5. **本地线程中的沙盒回退** —— 启动 Codex App **本地线程**（workspace-write 沙盒）。提示："使用 superpowers 技能 `using-git-worktrees` 设置隔离的工作区来实现小变更。"预检查：`git checkout -b test-sandbox-check` 应以 `Operation not permitted` 失败。预期：技能检测 `GIT_DIR == GIT_COMMON`（普通仓库），尝试 `git worktree add -b`，遇到 Seatbelt 拒绝，回退到 Step 0 "已在工作区" 行为 —— 运行设置、基线测试，从当前目录报告就绪。通过：智能体优雅恢复，无神秘错误消息。失败：智能体打印原始 Seatbelt 错误、重试，或以混乱输出放弃。

### 回归

- 现有 Claude Code 技能触发测试仍通过
- 现有 subagent-driven-development 集成测试仍通过
- 普通 Claude Code 会话：完整 worktree 创建 + 4 选项完成仍工作
