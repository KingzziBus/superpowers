---
name: finishing-a-development-branch
description: Use when implementation is complete, all tests pass, and you need to decide how to integrate the work - guides completion of development work by presenting structured options for merge, PR, or cleanup
---

# Finishing a Development Branch（完成一个开发分支）

## 概述

通过呈现清晰的选项并处理选定的工作流，来指导开发工作的完成。

**核心原则：** 验证测试 → 检测环境 → 呈现选项 → 执行选择 → 清理。

**开始时宣告：** "I'm using the finishing-a-development-branch skill to complete this work."（我正在使用 finishing-a-development-branch 技能来完成这项工作。）

## 流程

### 第 1 步：验证测试

**在呈现选项之前，验证测试通过：**

```bash
# 跑项目的测试套件
npm test / cargo test / pytest / go test ./...
```

**如果测试失败：**
```
Tests failing (<N> failures). Must fix before completing:

[Show failures]

Cannot proceed with merge/PR until tests pass.
```

停下。不要进入第 2 步。

**如果测试通过：** 继续第 2 步。

### 第 2 步：检测环境

**在呈现选项之前确定工作空间状态：**

```bash
GIT_DIR=$(cd "$(git rev-parse --git-dir)" 2>/dev/null && pwd -P)
GIT_COMMON=$(cd "$(git rev-parse --git-common-dir)" 2>/dev/null && pwd -P)
```

这决定了显示哪个菜单以及如何清理：

| 状态 | 菜单 | 清理 |
|------|------|------|
| `GIT_DIR == GIT_COMMON`（普通 repo） | 标准 4 选项 | 无 worktree 可清理 |
| `GIT_DIR != GIT_COMMON`，命名分支 | 标准 4 选项 | 基于出处（见第 6 步） |
| `GIT_DIR != GIT_COMMON`，分离 HEAD | 缩减 3 选项（无 merge） | 无清理（外部管理） |

### 第 3 步：确定基础分支

```bash
# 尝试常见的基础分支
git merge-base HEAD main 2>/dev/null || git merge-base HEAD master 2>/dev/null
```

或询问："This branch split from main - is that correct?"

### 第 4 步：呈现选项

**普通 repo 和命名分支 worktree —— 严格呈现这 4 个选项：**

```
Implementation complete. What would you like to do?

1. Merge back to <base-branch> locally
2. Push and create a Pull Request
3. Keep the branch as-is (I'll handle it later)
4. Discard this work

Which option?
```

**分离 HEAD —— 严格呈现这 3 个选项：**

```
Implementation complete. You're on a detached HEAD (externally managed workspace).

1. Push as new branch and create a Pull Request
2. Keep as-is (I'll handle it later)
3. Discard this work

Which option?
```

**不要添加解释** —— 保持选项简洁。

### 第 5 步：执行选择

#### 选项 1：本地 Merge

```bash
# 获取主 repo 根目录以确保 CWD 安全
MAIN_ROOT=$(git -C "$(git rev-parse --git-common-dir)/.." rev-parse --show-toplevel)
cd "$MAIN_ROOT"

# 先 merge —— 在删除任何东西之前验证成功
git checkout <base-branch>
git pull
git merge <feature-branch>

# 在 merge 结果上验证测试
<test command>

# 只有在 merge 成功后：清理 worktree（第 6 步），然后删除分支
```

然后：清理 worktree（第 6 步），然后删除分支：

```bash
git branch -d <feature-branch>
```

#### 选项 2：Push 并创建 PR

```bash
# push 分支
git push -u origin <feature-branch>

# 创建 PR
gh pr create --title "<title>" --body "$(cat <<'EOF'
## Summary
<2-3 bullets of what changed>

## Test Plan
- [ ] <verification steps>
EOF
)"
```

**不要清理 worktree** —— 用户需要它活着以便对 PR 反馈进行迭代。

#### 选项 3：保持原样

报告："Keeping branch <name>. Worktree preserved at <path>."

**不要清理 worktree。**

#### 选项 4：丢弃

**首先确认：**
```
This will permanently delete:
- Branch <name>
- All commits: <commit-list>
- Worktree at <path>

Type 'discard' to confirm.
```

等待精确的确认。

如果确认：
```bash
MAIN_ROOT=$(git -C "$(git rev-parse --git-common-dir)/.." rev-parse --show-toplevel)
cd "$MAIN_ROOT"
```

然后：清理 worktree（第 6 步），然后强制删除分支：

```bash
git branch -D <feature-branch>
```

### 第 6 步：清理工作空间

**仅对选项 1 和 4 运行。** 选项 2 和 3 始终保留 worktree。

```bash
GIT_DIR=$(cd "$(git rev-parse --git-dir)" 2>/dev/null && pwd -P)
GIT_COMMON=$(cd "$(git rev-parse --git-common-dir)" 2>/dev/null && pwd -P)
WORKTREE_PATH=$(git rev-parse --show-toplevel)
```

**如果 `GIT_DIR == GIT_COMMON`：** 普通 repo，无 worktree 可清理。完成。

**如果 worktree 路径在 `.worktrees/`、`worktrees/` 或 `~/.config/superpowers/worktrees/` 下：** Superpowers 创建了这个 worktree —— 我们负责清理。

```bash
MAIN_ROOT=$(git -C "$(git rev-parse --git-common-dir)/.." rev-parse --show-toplevel)
cd "$MAIN_ROOT"
git worktree remove "$WORKTREE_PATH"
git worktree prune  # 自愈：清理任何过时的注册
```

**否则：** 宿主环境（harness）拥有这个工作空间。**不要**删除它。如果你的平台提供 workspace-exit 工具，用它。否则，让工作空间保持原位。

## 快速参考

| 选项 | Merge | Push | 保留 Worktree | 清理分支 |
|------|-------|------|---------------|----------|
| 1. 本地 merge | 是 | - | - | 是 |
| 2. 创建 PR | - | 是 | 是 | - |
| 3. 保持原样 | - | - | 是 | - |
| 4. 丢弃 | - | - | - | 是（强制） |

## 常见错误

**跳过测试验证**
- **问题：** Merge 损坏的代码，创建失败的 PR
- **修复：** 提供选项之前始终验证测试

**开放式问题**
- **问题：** "What should I do next?" 是模糊的
- **修复：** 严格呈现 4 个结构化选项（分离 HEAD 时 3 个）

**为选项 2 清理 worktree**
- **问题：** 移除用户对 PR 迭代需要的 worktree
- **修复：** 只对选项 1 和 4 进行清理

**在移除 worktree 之前删除分支**
- **问题：** `git branch -d` 失败，因为 worktree 仍引用该分支
- **修复：** 先 merge，移除 worktree，然后删除分支

**从 worktree 内部跑 `git worktree remove`**
- **问题：** 当 CWD 在被删除的 worktree 内部时命令静默失败
- **修复：** 在 `git worktree remove` 之前始终 `cd` 到主 repo 根目录

**清理 harness 拥有的 worktrees**
- **问题：** 移除 harness 创建的 worktree 会导致幽灵状态
- **修复：** 只清理 `.worktrees/`、`worktrees/` 或 `~/.config/superpowers/worktrees/` 下的 worktree

**没有对丢弃的确认**
- **问题：** 意外删除工作
- **修复：** 要求输入 "discard" 确认

## 红旗

**永远不要：**
- 测试失败时继续
- 不在结果上验证测试就 merge
- 没有确认就删除工作
- 未经明确请求就 force-push
- 在确认 merge 成功之前移除 worktree
- 清理你没创建的 worktree（出处检查）
- 从 worktree 内部跑 `git worktree remove`

**始终：**
- 在提供选项之前验证测试
- 在呈现菜单之前检测环境
- 严格呈现 4 个选项（分离 HEAD 时 3 个）
- 为选项 4 获取输入确认
- 仅对选项 1 和 4 清理 worktree
- 在移除 worktree 之前 `cd` 到主 repo 根目录
- 移除后跑 `git worktree prune`
