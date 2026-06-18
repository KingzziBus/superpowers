# Codex App 兼容性实施计划 - 分析报告

> **对应原文**：`docs/superpowers/plans/2026-03-23-codex-app-compatibility.md`
> **翻译文件**：`z-study/docs/superpowers/plans/2026-03-23-codex-app-compatibility.zh-CN.md`
> **报告时间**：2026-06-03

---

## 一、基本信息

| 项目 | 内容 |
|---|---|
| 文档类型 | 实施计划（Implementation Plan） |
| Chunk 数量 | 无显式 Chunk（8 个 Task） |
| 总行数 | 565 行 |
| 关联 Spec | `docs/superpowers/specs/2026-03-23-codex-app-compatibility-design.md` |
| 关联工单 | PRI-823 |
| 核心机制 | `git-dir` vs `git-common-dir` 对比检测 |
| 涉及技能 | 5 个：`using-git-worktrees`、`finishing-a-development-branch`、`subagent-driven-development`、`executing-plans`、`using-superpowers/references/codex-tools.md` |
| 平台 | Codex App（OpenAI 的 AI Coding 沙盒环境） |
| 新增测试 | 1 个 shell 脚本（6 个测试用例） |

---

## 二、目的

让 Superpowers 项目的核心技能（worktree 创建、branch 完成、计划执行）在 **Codex App 的沙盒化 worktree 环境**中正常工作，且不破坏其他环境的既有行为。

### 痛点

Codex App（OpenAI 推出的 AI Coding 工具）运行在**沙盒环境**中，对工作目录和 git 操作有特殊限制：

1. **强制 worktree 化**：Codex App 自动把用户代码放在一个**外部管理的 worktree** 中（用户看不到创建过程）
2. **Detached HEAD**：由于沙盒限制，agent 不能直接 `git checkout -b` 或 `git push`
3. **没有清理权限**：沙盒 worktree 由 Codex App 拥有，agent 不能 `git worktree remove`

如果 Superpowers 技能仍按"普通环境"假设运行，会出现：
- 试图在已有 worktree 内嵌套创建 worktree → 报错
- 试图 `git checkout -b` → 失败（无权限）
- 试图清理沙盒 worktree → 留下"幻影状态"

### 解决方案

**检测 + 适配**：在技能开头运行只读 git 命令，根据环境状态走不同分支。

| 环境状态 | 动作 |
|---|---|
| 普通仓库（`GIT_DIR == GIT_COMMON`） | 走原流程 |
| 关联 worktree 且有分支 | 跳过创建，走完成流程 |
| 关联 worktree 且 detached HEAD | 发出"交接载荷"给用户用 Codex App 原生控件完成 |

---

## 三、核心技术决策

### 1. 只读环境检测（不修改任何 git 状态）

```bash
GIT_DIR=$(cd "$(git rev-parse --git-dir)" 2>/dev/null && pwd -P)
GIT_COMMON=$(cd "$(git rev-parse --git-common-dir)" 2>/dev/null && pwd -P)
BRANCH=$(git branch --show-current)
```

**为什么用 `git-common-dir`**：
- 在关联 worktree 中，`--git-dir` 指向 worktree 自己的 `.git/worktrees/<name>/`
- `--git-common-dir` 始终指向主仓库的 `.git/`
- **两者不同 = 当前在关联 worktree 中**

**只读的好处**：
- 检测是安全的（不会改变任何状态）
- 可以在任何时候调用
- 即使环境判断错了也不会留下副作用

### 2. 沙盒回退（late-detection fallback）

如果在创建 worktree 时遇到权限错误（"Operation not permitted"），**回退**到"已存在 worktree"行为。

```bash
git worktree add -b <branch> <path>
# 失败时：fall back to in-place execution
```

**价值**：应对"Codex App 增加了新的沙盒限制"等未来场景——不被早期检测漏过的情况困住。

### 3. 交接载荷（Handoff Payload）替代菜单

当处于 detached HEAD 状态时，**不要**给用户传统的 4 选项菜单（merge/PR/keep/discard），而是：

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
```

**关键设计点**：
- ✅ 包含 commit SHA（用户引用方便）
- ✅ 包含数据丢失警告（防止用户误操作丢失工作）
- ✅ 引导用户使用宿主应用原生控件（Codex App UI）
- ✅ 给出建议的分支名/提交信息（可复制粘贴）
- ❌ 不给 4 选项菜单（因为沙盒无法执行合并/PR/删除）

### 4. 三种路径的清晰分支

```
Step 1.5: Detect Environment
├── Path A: 外部 worktree + detached HEAD → 交接载荷 + 跳到 Step 5
├── Path B: 外部 worktree + 有分支名 → 4 选项菜单
└── Path C: 普通仓库 → 4 选项菜单
```

**对称性**：Path B 和 C 行为完全相同（都用 4 选项菜单）；Path A 是特殊路径。

### 5. 集成行的语义更新

**之前**：
```
- **superpowers:using-git-worktrees** - REQUIRED: Set up isolated workspace before starting
```

**之后**：
```
- **superpowers:using-git-worktrees** - REQUIRED: Ensures isolated workspace (creates one or verifies existing)
```

**变化**：
- "Set up" → "Ensures"（创建 or 验证存在）
- 明确表达 skill 是"幂等"的——可重复调用

### 6. 修复 Step 5 的既存不一致

原 Step 5 写 "For Options 1, 2, 4"，但 Quick Reference 表格和 Common Mistakes 写 "Options 1 & 4 only"。

**本计划顺手修复**——清理守卫的同时修正文档不一致。

### 7. 测试设计：模拟 4 种环境状态

测试脚本创建临时 git 仓库，逐步操作验证 4 种状态：

| 状态 | 操作 | 期望检测结果 |
|---|---|---|
| 普通仓库 | `git init` + `git commit --allow-empty` | normal |
| 关联 worktree | `git worktree add ... -b test-branch` | linked |
| detached HEAD | `git checkout --detach HEAD` | branch 为空 |
| 关联 worktree + detached HEAD | 上述两者组合 | Codex App 模拟 |

**测试 5-6 验证清理守卫**：在同一仓库内切换 `cd` 路径，确认检测函数能正确识别当前所在 worktree。

---

## 四、文件结构

### 修改（5 个）
| 文件 | 修改内容 | 行数变化 |
|---|---|---|
| `using-git-worktrees/SKILL.md` | 增加 Step 0（环境检测 + 沙盒回退） | +60 行 |
| `using-git-worktrees/SKILL.md` | 更新 Integration 描述（Task 2） | ~5 行 |
| `finishing-a-development-branch/SKILL.md` | 增加 Step 1.5（三路径） | +50 行 |
| `finishing-a-development-branch/SKILL.md` | Step 5 增加清理守卫（Task 4） | +20 行 |
| `subagent-driven-development/SKILL.md` | 第 268 行单行修改 | 1 行 |
| `executing-plans/SKILL.md` | 第 68 行单行修改 | 1 行 |
| `using-superpowers/references/codex-tools.md` | 末尾追加两个小节 | +30 行 |

### 新建（1 个）
- `tests/codex-app-compat/test-environment-detection.sh`（80+ 行）

**总计**：6 个文件修改 + 1 个测试新增

---

## 五、Task 划分逻辑

| Task | 关注点 | 风险等级 |
|---|---|---|
| **Task 1** | using-git-worktrees Step 0 检测 | 中（核心功能） |
| **Task 2** | using-git-worktrees Integration 更新 | 低（文档） |
| **Task 3** | finishing-a-development-branch Step 1.5 检测 | 中（多路径分支） |
| **Task 4** | finishing-a-development-branch Step 5 清理守卫 | 中（修改既有逻辑） |
| **Task 5** | SDD 和 executing-plans 的 Integration 行 | 低（1 行） |
| **Task 6** | codex-tools.md 文档增加 | 低（新增） |
| **Task 7** | 自动化测试 | 中（可量化） |
| **Task 8** | 最终验证 | 低（汇总） |

**设计模式**：
- Task 1-4 = 功能（行为变更）
- Task 5-6 = 同步（文档描述）
- Task 7 = 验证（自动化）
- Task 8 = 收尾（手动）

---

## 六、优点

| 维度 | 具体表现 |
|---|---|
| **零破坏性** | 原 4 选项菜单在普通环境下完全保留；只在 detached HEAD 走特殊路径 |
| **只读检测** | `git rev-parse` 是只读命令，不会污染环境 |
| **清晰的失败模式** | 沙盒权限错误不是崩溃，而是优雅回退 |
| **用户友好** | 交接载荷包含 commit SHA、分支建议、数据丢失警告——用户能直接复制使用 |
| **平台中立** | 检测模式适用于任何沙盒环境（Codex App 之外还有 Claude Code Agent tool、CircleCI 等） |
| **可测性** | 4 种状态都可在 shell 脚本中模拟，无需真实沙盒 |
| **文档与代码同步** | codex-tools.md 同步更新，提供"为什么这样"的解释 |
| **修复历史债务** | 顺手修正 Step 5 的"Options 1, 2, 4" vs "Options 1, 4" 不一致 |

---

## 七、缺点 / 风险

| 风险点 | 影响 | 缓解 |
|---|---|---|
| **检测状态可能滞后** | 创建 worktree 后 `.git/worktrees/<name>` 可能未立即出现 | 使用 `cd "$(git rev-parse --git-dir)" && pwd -P` 强制解析 |
| **未来沙盒策略变化** | Codex App 可能限制 `git rev-parse` 的输出 | 检测是只读的，理论上不受限 |
| **Path A 用户认知负担** | 用户需要理解 "external management" 等术语 | 交接载荷用自然语言解释，配合 emoji 警告 |
| **建议分支名不一定合适** | 模板"前 5 个词的 slugify"可能产生尴尬名称 | 文档明确"omittable"作为兜底 |
| **Step 5 修复是"附加"变更** | 可能引入未预期的回归 | 6 个测试覆盖 4 种状态 |
| **测试脚本的 Windows 兼容性** | `mktemp`、`trap` 是 Unix 习惯，Windows Git Bash 部分支持 | 项目可能需要 WSL 跑测试 |
| **跨平台差异** | macOS、Linux、Windows 的 worktree 行为可能略不同 | 检测逻辑跨平台一致 |
| **下游技能需配套更新** | 其他调用 using-git-worktrees 的技能可能假设"必然创建" | Task 5 同步更新两处 Integration 行 |

---

## 八、关键洞察

### 洞察 1：环境检测是"跨平台兼容性"的标准模式

```
本地 Mac     → 普通环境 → 原流程
Codex App   → 沙盒环境 → 交接流程
CI/CD       → 受限环境 → 跳过流程
SSH 远程    → 关联 worktree → 跳过创建
```

**任何能"自我感知环境"的系统都比"假设环境一致"更鲁棒**。本计划是这一原则在 git 工作流中的体现。

### 洞察 2：交接载荷的"工程化"是亮点

很多 AI 工具在沙盒失败时只会说"无法完成"——本计划更进一步：

```
⚠ These commits are on a detached HEAD. If you do not create a branch,
they may be lost when this workspace is cleaned up.
```

**它承担了"数据安全告知"的责任**——这才是合格的工程交付。

### 洞察 3：检测应在"最廉价的位置"做

```bash
GIT_DIR=$(cd "$(git rev-parse --git-dir)" 2>/dev/null && pwd -P)
GIT_COMMON=$(cd "$(git rev-parse --git-common-dir)" 2>/dev/null && pwd -P)
```

- 三个命令 < 100ms
- 无副作用
- 无网络请求
- 失败时返回空（`|| true` 隐含）

**廉价 + 准确 + 安全** = 完美的环境探针。

### 洞察 4：让宿主环境"接管"是优雅的退让

Path A 不试图"硬上"，而是承认沙盒限制，主动让用户用 Codex App UI 完成剩余动作。

**这是 AI Agent 的成熟表现**——不与宿主环境对抗，而是协作。

### 洞察 5：测试覆盖"4 种状态"是状态完备性

普通仓库、关联 worktree、detached HEAD、组合状态——4 种状态穷举了所有可能。

**状态机的完备测试 = 不留死角**。

### 洞察 6：顺手修复历史债务

Task 4 在增加清理守卫的同时，把原 "Options 1, 2, 4" 修正为 "Options 1, 4"——与 Quick Reference 一致。

**这种"在主任务中顺手清理"的做法**是健康的工程文化（虽然有时会被批评为"scope creep"）。

---

## 九、关联文档

### 上游 / Spec
- [`2026-03-23-codex-app-compatibility-design.md`](../specs/2026-03-23-codex-app-compatibility-design.md) — 设计文档（本计划的依据）

### 下游 / 受影响
- 所有调用 `using-git-worktrees` 的技能：`brainstorming`、`subagent-driven-development`、`executing-plans`
- Codex App 平台用户的使用体验

### 横向关联
- [`2026-04-06-worktree-rototill.md`](./2026-04-06-worktree-rototill.md) — 紧随其后的"worktree 翻新"计划，扩展了本计划的检测思路
- [`2025-11-22-opencode-support-implementation.md`](../../plans/2025-11-22-opencode-support-implementation.md) — 多平台支持
- `skills/using-git-worktrees/SKILL.md` — 接受 Task 1-2 的修改
- `skills/finishing-a-development-branch/SKILL.md` — 接受 Task 3-4 的修改
- `skills/using-superpowers/references/codex-tools.md` — 接受 Task 6 的追加

### 历史
- Codex App 推出后，Superpowers 用户陆续报告"worktree 技能在沙盒中失败"
- 本计划（PRI-823）是该问题的一次性修复

---

## 十、状态评价

**计划成熟度：高** ✅

- 8 个 Task 全部对应 commit
- 自动化测试 6 个用例
- 最终验证步骤包含 `git diff --stat HEAD~7` 量化变更范围
- 描述了"如不可用则手动"的回退方案

**实施复杂度：中** ⚖️

- 涉及多个技能文件，但每个文件的修改都很局部
- 测试需要 shell + Git 知识
- 最棘手的是 Path A 的"交接载荷"文案需精心设计

**适用性**：✅ 强烈推荐
- 解决了真实问题
- 对其他平台（CircleCI、GitHub Codespaces）也有借鉴价值
- 改动局部，不破坏现有用户

---

## 十一、给读者的启示

### 启示 1：环境检测优于"假设环境一致"
任何依赖外部环境的工具（CI、容器、沙盒）都应**先检测再行动**。3 行 bash 命令的检测几乎零成本。

### 启示 2：让宿主环境"接管"是退让也是进化
当 agent 自身无法完成时，主动告知用户用宿主 UI 继续——这是**协作**而非**对抗**。

### 启示 3：交接载荷是"用户工程的最后一公里"
包含 commit SHA、分支建议、数据警告的载荷，让用户"复制粘贴即可继续"——**好的工程交付 = 把用户成功路径做到最短**。

### 启示 4：测试状态完备性
4 种环境状态（普通/关联/detached/组合）的穷举测试，让未来重构有安全网。**状态机问题用"状态穷举测试"覆盖**。

### 启示 5：跨平台兼容性的"检测+分支"模式
任何"在多个环境运行的工具"都应：
1. 检测当前环境
2. 为每种环境定义分支
3. 失败时优雅回退
这是 **"platform-aware code"** 的标准范式。

### 启示 6：主任务顺手清理债务
"在主任务里修一个相关的不一致"是健康的工程文化——前提是**有测试覆盖**。

---

## 十二、后续工作（推测）

1. **更多沙盒平台支持**：CircleCI、GitHub Codespaces、Gitpod 各自的 worktree 行为可能略有不同
2. **检测脚本提取为库**：把 `detect_worktree` 提取为独立 shell 库供更多技能复用
3. **交接载荷的 i18n**：当前是英文，未来可翻译
4. **Codex App API 集成**：未来若 Codex App 提供"创建分支"API，可直接调用而非让用户手动
5. **`GIT_DIR != GIT_COMMON` 的边界情况**：submodule（子模块）的检测会"误报"为 worktree——[2026-04-06 计划](./2026-04-06-worktree-rototill.md) 中已加入 submodule guard
6. **递归 worktree 警告**：用户在 worktree 内再次尝试创建 worktree 的更友好提示
7. **Codex App 版本兼容矩阵**：不同版本的 Codex App 沙盒策略可能不同，需维护兼容矩阵
8. **路径统一化**：让所有平台都通过 `${CLAUDE_PLUGIN_ROOT}` 等占位符引用资源
