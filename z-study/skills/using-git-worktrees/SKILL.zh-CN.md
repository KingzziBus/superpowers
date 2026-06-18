---
name: using-git-worktrees
description: Use when starting feature work that needs isolation from current workspace or before executing implementation plans - ensures an isolated workspace exists via native tools or git worktree fallback
---

# Using Git Worktrees（使用 Git Worktrees）

## 概述

确保工作在隔离的工作空间中进行。**优先**使用你平台的原生 worktree 工具。**只有**当没有原生工具可用时才回退到手动 git worktrees。

**核心原则：** 先检测现有隔离。然后用原生工具。然后回退到 git。永远不要和 harness 作对。

**开始时宣告：** "I'm using the using-git-worktrees skill to set up an isolated workspace."（我正在使用 using-git-worktrees 技能来搭建一个隔离的工作空间。）

## 第 0 步：检测现有隔离

**在创建任何东西之前，先检查你是否已经处于隔离工作空间。**

```bash
GIT_DIR=$(cd "$(git rev-parse --git-dir)" 2>/dev/null && pwd -P)
GIT_COMMON=$(cd "$(git rev-parse --git-common-dir)" 2>/dev/null && pwd -P)
BRANCH=$(git branch --show-current)
```

**Submodule 防御：** `GIT_DIR != GIT_COMMON` 在 git submodule 内也成立。在得出"已经在 worktree 中"的结论之前，先验证你不是处于 submodule：

```bash
# 如果这个命令返回了一个路径，你就在 submodule 中，不在 worktree 中 —— 当作普通 repo 处理
git rev-parse --show-superproject-working-tree 2>/dev/null
```

**如果 `GIT_DIR != GIT_COMMON`（且不是 submodule）：** 你已经处于一个 linked worktree。跳到第 3 步（项目设置）。**不要**再创建一个 worktree。

报告分支状态：
- 在分支上："Already in isolated workspace at `<path>` on branch `<name>`."
- 分离 HEAD："Already in isolated workspace at `<path>` (detached HEAD, externally managed). Branch creation needed at finish time."

**如果 `GIT_DIR == GIT_COMMON`（或在 submodule 中）：** 你在普通 repo checkout 中。

用户是否已经在你的指令中表明过 worktree 偏好？如果没有，在创建 worktree 前先请求同意：

> "Would you like me to set up an isolated worktree? It protects your current branch from changes."

如果用户已经声明了偏好就尊重该偏好，不要再问。如果用户拒绝同意，原地工作并跳到第 3 步。

## 第 1 步：创建隔离工作空间

**你有两种机制。按此顺序尝试。**

### 1a. 原生 Worktree 工具（首选）

用户已经请求一个隔离工作空间（第 0 步同意）。你是否已经有创建 worktree 的方式？它的名字可能像 `EnterWorktree`、`WorktreeCreate`、`/worktree` 命令、或 `--worktree` 标志。如果有，用它并跳到第 3 步。

原生工具会自动处理目录放置、分支创建和清理。当你有原生工具时使用 `git worktree add` 会产生你的 harness 看不到或无法管理的"幽灵状态"。

只有当你没有原生 worktree 工具时才继续第 1b 步。

### 1b. Git Worktree 回退

**只有当第 1a 步不适用** —— 你没有原生 worktree 工具 —— 时才使用这个。手动用 git 创建 worktree。

#### 目录选择

按此优先级顺序。**显式的用户偏好**总是胜过观察到的文件系统状态。

1. **检查你的指令中是否声明了 worktree 目录偏好。** 如果用户已经指定，直接使用而不询问。

2. **检查是否存在项目级 worktree 目录：**
   ```bash
   ls -d .worktrees 2>/dev/null     # 首选（隐藏）
   ls -d worktrees 2>/dev/null      # 备选
   ```
   如果找到，用它。如果两者都存在，`.worktrees` 优先。

3. **检查是否存在全局目录：**
   ```bash
   project=$(basename "$(git rev-parse --show-toplevel)")
   ls -d ~/.config/superpowers/worktrees/$project 2>/dev/null
   ```
   如果找到，用它（向后兼容旧版全局路径）。

4. **如果没有任何其他指引**，默认为项目根目录的 `.worktrees/`。

#### 安全验证（仅项目级目录）

**在创建 worktree 之前必须验证目录被 ignore：**

```bash
git check-ignore -q .worktrees 2>/dev/null || git check-ignore -q worktrees 2>/dev/null
```

**如果 NOT ignored：** 添加到 .gitignore，提交该改动，然后继续。

**为什么至关重要：** 防止意外将 worktree 内容提交到仓库。

全局目录（`~/.config/superpowers/worktrees/`）不需要验证。

#### 创建 Worktree

```bash
project=$(basename "$(git rev-parse --show-toplevel)")

# 根据选择的目录决定路径
# 项目级: path="$LOCATION/$BRANCH_NAME"
# 全局级: path="~/.config/superpowers/worktrees/$project/$BRANCH_NAME"

git worktree add "$path" -b "$BRANCH_NAME"
cd "$path"
```

**沙箱回退：** 如果 `git worktree add` 因权限错误失败（沙箱拒绝），告诉用户沙箱阻止了 worktree 创建，而你改为在当前目录工作。然后在原地运行设置和基线测试。

## 第 3 步：项目设置

自动检测并运行相应的设置：

```bash
# Node.js
if [ -f package.json ]; then npm install; fi

# Rust
if [ -f Cargo.toml ]; then cargo build; fi

# Python
if [ -f requirements.txt ]; then pip install -r requirements.txt; fi
if [ -f pyproject.toml ]; then poetry install; fi

# Go
if [ -f go.mod ]; then go mod download; fi
```

## 第 4 步：验证干净基线

跑测试确保工作空间起点干净：

```bash
# 使用项目相应的命令
npm test / cargo test / pytest / go test ./...
```

**如果测试失败：** 报告失败，询问是继续还是调查。

**如果测试通过：** 报告就绪。

### 报告

```
Worktree ready at <full-path>
Tests passing (<N> tests, 0 failures)
Ready to implement <feature-name>
```

## 快速参考

| 情况 | 动作 |
|------|------|
| 已在 linked worktree | 跳过创建（第 0 步） |
| 在 submodule 中 | 当作普通 repo（第 0 步防御） |
| 有原生 worktree 工具 | 用它（第 1a 步） |
| 没有原生工具 | Git worktree 回退（第 1b 步） |
| `.worktrees/` 存在 | 用它（验证被 ignore） |
| `worktrees/` 存在 | 用它（验证被 ignore） |
| 两者都存在 | 用 `.worktrees/` |
| 两者都不存在 | 检查指令文件，然后默认 `.worktrees/` |
| 全局路径存在 | 用它（向后兼容） |
| 目录未被 ignore | 添加到 .gitignore + 提交 |
| 创建时权限错误 | 沙箱回退，原地工作 |
| 基线测试失败 | 报告失败 + 询问 |
| 无 package.json/Cargo.toml | 跳过依赖安装 |

## 常见错误

### 与 harness 作对

- **问题：** 平台已经提供隔离时使用 `git worktree add`
- **修复：** 第 0 步检测现有隔离。第 1a 步让原生工具优先。

### 跳过检测

- **问题：** 在已有 worktree 内创建嵌套 worktree
- **修复：** 创建任何东西前始终跑第 0 步

### 跳过 ignore 验证

- **问题：** Worktree 内容被跟踪，污染 git status
- **修复：** 创建项目级 worktree 前始终用 `git check-ignore`

### 假设目录位置

- **问题：** 产生不一致，违反项目约定
- **修复：** 按优先级：现有 > 全局旧版 > 指令文件 > 默认

### 带失败测试继续

- **问题：** 无法区分新 bug 和已存在的问题
- **修复：** 报告失败，获取明确的继续许可

## 红旗

**永远不要：**
- 在第 0 步检测到现有隔离时创建 worktree
- 当你有原生 worktree 工具（如 `EnterWorktree`）时使用 `git worktree add`。**这是 #1 错误** —— 如果你有，就用它。
- 跳过第 1a 步直接用第 1b 步的 git 命令
- 创建 worktree 时不验证它被 ignore（项目级）
- 跳过基线测试验证
- 未经询问就带失败测试继续

**始终：**
- 先跑第 0 步检测
- 优先用原生工具而不是 git 回退
- 遵循目录优先级：现有 > 全局旧版 > 指令文件 > 默认
- 项目级目录验证被 ignore
- 自动检测并运行项目设置
- 验证干净的测试基线
