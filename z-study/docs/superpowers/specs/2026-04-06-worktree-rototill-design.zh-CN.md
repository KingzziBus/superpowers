# Worktree 翻新：检测并让步

**日期：** 2026-04-06
**状态：** 草稿
**工单：** PRI-974
**合并：** PRI-823（Codex App 兼容性）

## 问题

Superpowers 对 worktree 管理有强烈意见——特定路径（`.worktrees/<branch>`）、特定命令（`git worktree add`）、特定清理（`git worktree remove`）。同时，Claude Code、Codex App、Gemini CLI 和 Cursor 都提供带有自己的路径、生命周期管理和清理的原生 worktree 支持。

这造成三种失败模式：

1. **重复** —— 在 Claude Code 上，技能做了 `EnterWorktree`/`ExitWorktree` 已经做的事
2. **冲突** —— 在 Codex App 上，技能试图在已托管的 worktree 内创建 worktree
3. **幻影状态** —— 技能在 `.worktrees/` 创建的 worktree 对宿主不可见；宿主在 `.claude/worktrees/` 创建的 worktree 对技能不可见

对于没有原生支持的宿主（Codex CLI、OpenCode、Copilot standalone），superpowers 填补了真正的空白。技能不应消失——当原生支持存在时，它应让出位置。

## 目标

1. 当存在原生宿主 worktree 系统时让步
2. 为缺乏原生 worktree 支持的宿主继续提供支持
3. 修复 finishing-a-development-branch 中三个已知 bug（#940、#999、#238）
4. 让 worktree 创建变成可选而非强制（#991）
5. 用平台中立语言替换硬编码的 `CLAUDE.md` 引用（#1049）

## 非目标

- 每个 worktree 的环境约定（`.worktree-env.sh`、端口偏移）—— Phase 4
- 路径强制的 PreToolUse 钩子 —— Phase 4
- 多仓库 worktree 文档 —— Phase 4
- 脑暴清单的 worktree 变更 —— Phase 4
- `.superpowers-session.json` 元数据跟踪（有趣的 PR #997 想法，v1 不需要）
- 钩子符号链接到 worktree（PR #965 想法，独立问题）

## 设计原则

### 检测状态，而非平台

使用 `GIT_DIR != GIT_COMMON` 来判断"我是否已处于 worktree 中"，而非通过嗅探环境变量来识别宿主。这是一个稳定的 git 原语（自 git 2.5，2015），跨所有宿主通用工作，且随新宿主出现需要零维护。

### 声明式意图，规范式回退

技能描述目标（"确保工作在隔离的工作区"）并在原生工具可用时让步。仅在缺乏原生 worktree 支持的宿主上规定具体 git 命令作为回退。Step 1a 在前且显式命名原生工具（`EnterWorktree`、`WorktreeCreate`、`/worktree`、`--worktree`）；Step 1b 在后作为 git 回退。原始 spec 让 Step 1a 保持抽象（"你了解自己的工具集"），但 TDD 证明当 Step 1a 太模糊时，智能体锚定在 Step 1b 的具体命令上。需要显式工具命名和同意授权桥接来让偏好可靠。

### 基于来源的所有权

谁创建 worktree 谁拥有其清理。如果宿主创建了它，superpowers 不碰它。如果 superpowers 通过 git 回退创建了它，superpowers 清理它。启发法：如果 worktree 位于 `.worktrees/` 或 `~/.config/superpowers/worktrees/` 下，superpowers 拥有它。其他任何位置（`.claude/worktrees/`、`~/.codex/worktrees/`、`.gemini/worktrees/`）属于宿主。

## 设计

### 1. `using-git-worktrees` SKILL.md 重写

技能在创建前获得三个新步骤，并简化创建流程。

#### Step 0: Detect Existing Isolation

```bash
GIT_DIR=$(cd "$(git rev-parse --git-dir)" 2>/dev/null && pwd -P)
GIT_COMMON=$(cd "$(git rev-parse --git-common-dir)" 2>/dev/null && pwd -P)
BRANCH=$(git branch --show-current)
```

三种结果：

| 条件 | 含义 | 动作 |
|-----------|---------|--------|
| `GIT_DIR == GIT_COMMON` | 普通仓库检出 | 继续到 Step 0.5 |
| `GIT_DIR != GIT_COMMON`，命名分支 | 已处于关联 worktree | 跳到 Step 3（项目设置）。报告："Already in isolated workspace at `<path>` on branch `<name>`." |
| `GIT_DIR != GIT_COMMON`，分离头指针 | 外部管理的 worktree（如 Codex App 沙盒） | 跳到 Step 3。报告："Already in isolated workspace at `<path>` (detached HEAD, externally managed)." |

Step 0 不关心谁创建了 worktree 或哪个宿主在运行。worktree 就是 worktree，无论来源。

**子模块守卫：** `GIT_DIR != GIT_COMMON` 在 git 子模块内也为真。在得出"已在 worktree 中"结论之前，检查我们不在子模块中：

```bash
# 如果此命令返回路径，你在子模块中而非 worktree 中
git rev-parse --show-superproject-working-tree 2>/dev/null
```

如果在子模块中，按 `GIT_DIR == GIT_COMMON` 处理（继续到 Step 0.5）。

#### Step 0.5: Consent

当 Step 0 未发现已有隔离（`GIT_DIR == GIT_COMMON`）时，在创建前询问：

> "Would you like me to set up an isolated worktree? This protects your current branch from changes. (y/n)"

若回答是，继续到 Step 1。若否，原地工作——直接跳到 Step 3，不创建 worktree。

当 Step 0 检测到已有隔离时（询问已存在的内容无意义），此步骤被完全跳过。

#### Step 1a: Native Tools（首选）

> 用户已请求隔离工作区（Step 0 同意）。检查你可用的工具 —— 你是否有 `EnterWorktree`、`WorktreeCreate`、`/worktree` 命令或 `--worktree` 标志？若是：用户对创建 worktree 的同意就是你使用它的授权。现在使用它并跳到 Step 3。

使用原生工具后，跳到 Step 3（项目设置）。

**设计注 —— TDD 修订：** 原始 spec 使用了故意简短、抽象的 Step 1a（"你了解自己的工具集 —— 技能不需要命名具体工具"）。TDD 验证推翻了这一点：智能体锚定在 Step 1b 的具体 git 命令上，忽略抽象指导（2/6 通过率）。三个变更修复了它（GREEN 和 PRESSURE 测试中 50/50 通过率）：

1. **显式工具命名** —— 通过名字列出 `EnterWorktree`、`WorktreeCreate`、`/worktree`、`--worktree` 将决策从解释（"我有原生工具吗？"）转为事实查找（"`EnterWorktree` 在我的工具列表中吗？"）。在没有这些工具的平台上的智能体只需检查、找不到，然后回到 Step 1b。未观察到误报。
2. **同意桥接** —— "用户对创建 worktree 的同意就是你使用它的授权"直接解决了 `EnterWorktree` 的工具级护栏（"仅在用户明确要求时"）。工具描述覆盖技能指令（Claude Code #29950），所以技能必须将用户同意框定为工具所需的授权。
3. **Red Flag 条目** —— 在 Red Flags 小节中命名具体反模式（"当你有原生 worktree 工具时使用 `git worktree add` —— 这是 #1 错误"）。

文件分离（Step 1b 在单独的技能中）经测试证明是不必要的。锚定问题通过 Step 1a 文本的质量解决，而非物理分离 git 命令。完整 240 行技能（所有 git 命令可见）的对照测试通过 20/20。

#### Step 1b: Git Worktree Fallback

当没有原生工具可用时，手动创建 worktree。

**目录选择**（优先级顺序）：
1. 检查是否已存在 `.worktrees/` 或 `worktrees/` 目录 —— 如果存在则使用。若两者都存在，`.worktrees/` 胜出。
2. 检查是否已存在 `~/.config/superpowers/worktrees/<project>/` 目录 —— 如果存在则使用（向后兼容旧全局路径）。
3. 检查项目的代理指令文件（CLAUDE.md、GEMINI.md、AGENTS.md、.cursorrules 或等效文件）中的 worktree 目录偏好。
4. 默认为 `.worktrees/`。

无交互式目录选择提示。全局路径（`~/.config/superpowers/worktrees/`）不再作为新用户的选择提供，但该位置的现有 worktree 会被检测和使用以实现向后兼容。

**安全验证**（仅项目本地目录）：

```bash
git check-ignore -q .worktrees 2>/dev/null
```

若未忽略，添加到 `.gitignore` 并在继续前提交。

**创建：**

```bash
git worktree add "$path" -b "$BRANCH_NAME"
cd "$path"
```

**钩子感知：** Git worktree 不继承父仓库的 hooks 目录。在通过 1b 创建 worktree 后，若主仓库存在 hooks 目录，则将其符号链接过来：

```bash
if [ -d "$MAIN_ROOT/.git/hooks" ]; then
    ln -sf "$MAIN_ROOT/.git/hooks" "$path/.git/hooks"
fi
```

这防止 pre-commit 检查、linter 和其他钩子在工作转移到 worktree 时静默停止。（来自 PR #965 的想法。）

**沙盒回退：** 若 `git worktree add` 因权限错误失败，视为受限环境。跳过创建，在当前目录工作，继续到 Step 3。

**步骤编号注：** 当前技能有 Steps 1-4 作为扁平列表。此重新设计使用 0、0.5、1a、1b、3、4。没有 Step 2 —— 它是旧的单体"创建隔离工作区"，现已拆分为 1a/1b 结构。实施应干净地重新编号（例如 0 → "Step 0: Detect"，0.5 → Step 0 流程内，1a/1b → "Step 1"，3 → "Step 2"，4 → "Step 3"）或保留当前编号并加注释。由实施者决定。

#### Steps 3-4：项目设置和基线测试（不变）

无论哪条路径创建了工作区（Step 0 检测已有、Step 1a 原生工具、Step 1b git 回退，或根本没有 worktree），执行汇聚：

- **Step 3：** 自动检测并运行项目设置（`npm install`、`cargo build`、`pip install`、`go mod download` 等）
- **Step 4：** 运行测试套件。若测试失败，报告失败并询问是否继续。

### 2. `finishing-a-development-branch` SKILL.md 重写

完成技能获得环境检测并修复三个 bug。

#### Step 1: Verify Tests（不变）

运行项目的测试套件。若测试失败，停止。不提供完成选项。

#### Step 1.5: Detect Environment（新增）

重新运行与创建时 Step 0 相同的检测：

```bash
GIT_DIR=$(cd "$(git rev-parse --git-dir)" 2>/dev/null && pwd -P)
GIT_COMMON=$(cd "$(git rev-parse --git-common-dir)" 2>/dev/null && pwd -P)
```

三种路径：

| 状态 | 菜单 | 清理 |
|-------|------|---------|
| `GIT_DIR == GIT_COMMON`（普通仓库） | 标准 4 选项 | 无 worktree 可清理 |
| `GIT_DIR != GIT_COMMON`，命名分支 | 标准 4 选项 | 基于来源（见 Step 5） |
| `GIT_DIR != GIT_COMMON`，分离头指针 | 精简菜单：推送为新分支 + PR、保留原样、丢弃 | 无合并选项（无法从分离头指针合并） |

#### Step 2: Determine Base Branch（不变）

#### Step 3: Present Options

**普通仓库和命名分支 worktree：**

1. 本地合并回 `<base-branch>`
2. 推送并创建 Pull Request
3. 保留分支原样（稍后处理）
4. 丢弃此工作

**分离头指针：**

1. 推送为新分支并创建 Pull Request
2. 保留原样（稍后处理）
3. 丢弃此工作

#### Step 4: Execute Choice

**Option 1（本地合并）：**

```bash
# 获取主仓库根以保证 CWD 安全（Bug #238 修复）
MAIN_ROOT=$(git -C "$(git rev-parse --git-common-dir)/.." rev-parse --show-toplevel)
cd "$MAIN_ROOT"

# 先合并，成功后再移除任何东西
git checkout <base-branch>
git pull
git merge <feature-branch>
<运行测试>

# 仅在合并成功后：移除 worktree，然后删除分支（Bug #999 修复）
git worktree remove "$WORKTREE_PATH"  # 仅当 superpowers 拥有它时
git branch -d <feature-branch>
```

顺序至关重要：合并 → 验证 → 移除 worktree → 删除分支。旧技能在移除 worktree 之前删除了分支（失败，因为 worktree 仍引用分支）。先移除 worktree 的天真修复也是错的 —— 如果合并失败，工作目录会消失，更改会丢失。

**Option 2（创建 PR）：**

推送分支，创建 PR。**不要**清理 worktree —— 用户需要它来迭代 PR。（Bug #940 修复：移除矛盾的"Then: Cleanup worktree"措辞。）

**Option 3（保留原样）：** 无动作。

**Option 4（丢弃）：** 要求输入"discard"确认。然后移除 worktree（若 superpowers 拥有它），强制删除分支。

#### Step 5: Cleanup（已更新）

```
if GIT_DIR == GIT_COMMON:
    # 普通仓库，无 worktree 可清理
    done

if worktree path is under .worktrees/ or ~/.config/superpowers/worktrees/:
    # Superpowers 创建了它 —— 我们拥有清理权
    cd to main repo root       # Bug #238 修复
    git worktree remove <path>

else:
    # 宿主创建了它 —— 不要碰
    # 若平台提供 workspace-exit 工具，使用它
    # 否则，让 worktree 保留原位
```

清理仅对 Options 1 和 4 运行。Options 2 和 3 始终保留 worktree。（Bug #940 修复。）

**陈旧 worktree 剪除：** 在任何 `git worktree remove` 之后，运行 `git worktree prune` 作为自愈步骤。worktree 目录可能被带外删除（例如被宿主清理、手动 `rm` 或 `.claude/` 清理），留下导致混乱错误的陈旧注册。一行代码，防止静默腐烂。（来自 PR #1072 的想法。）

### 3. 集成更新

#### `subagent-driven-development` 和 `executing-plans`

两者当前在其 integration 小节中将 `using-git-worktrees` 列为 REQUIRED。改为：

> `using-git-worktrees` — Ensures isolated workspace (creates one or verifies existing)

技能本身现在处理同意（Step 0.5）和检测（Step 0），所以调用技能不需要门控或提示。

#### `writing-plans`

移除陈旧声明"should be run in a dedicated worktree (created by brainstorming skill)"。Brainstorming 是一个设计技能，不创建 worktree。worktree 提示在执行时通过 `using-git-worktrees` 出现。

### 4. 平台中立指令文件引用

worktree 相关技能中所有硬编码 `CLAUDE.md` 实例被替换为：

> "your project's agent instruction file (CLAUDE.md, GEMINI.md, AGENTS.md, .cursorrules, or equivalent)"

这适用于 Step 1b 中的目录偏好检查。

## Bug 修复（捆绑）

| Bug | 问题 | 修复 | 位置 |
|-----|---------|-----|----------|
| #940 | Option 2 措辞说"Then: Cleanup worktree (Step 5)"但快速参考说要保留。Step 5 说"For Options 1, 2, 4"但 Common Mistakes 说"Options 1 and 4 only." | 从 Option 2 移除清理。Step 5 仅适用于 Options 1 和 4。 | finishing SKILL.md |
| #999 | Option 1 在移除 worktree 之前删除分支。`git branch -d` 可能失败，因为 worktree 仍引用分支。 | 重新排序为：合并 → 验证测试 → 移除 worktree → 删除分支。必须合并成功才能移除任何东西。 | finishing SKILL.md |
| #238 | 若 CWD 在被移除的 worktree 内，`git worktree remove` 静默失败。 | 添加 CWD 守卫：在 `git worktree remove` 之前 `cd` 到主仓库根。 | finishing SKILL.md |

## 已解决问题

| Issue | 解决方案 |
|-------|-----------|
| #940 | 直接修复（Bug #940） |
| #991 | Step 0.5 中的同意机制 |
| #918 | Step 0 检测 + Step 1.5 完成检测 |
| #1009 | 通过 Step 1a 解决 —— 智能体使用原生工具（如 `EnterWorktree`）在宿主原生路径上创建。取决于 Step 1a 工作；见风险。 |
| #999 | 直接修复（Bug #999） |
| #238 | 直接修复（Bug #238） |
| #1049 | 平台中立指令文件引用 |
| #279 | 通过检测-让步解决 —— 原生路径被尊重因为我们不覆盖它们 |
| #574 | **延期。** 此 spec 中没有任何内容触及该 bug 所在的 brainstorming 技能。完整修复（向 brainstorming 清单添加 worktree 步骤）是 Phase 4。 |

## 风险

### Step 1a 是承重假设 —— 已解决

Step 1a —— 智能体优先选择原生 worktree 工具而非 git 回退 —— 是整个设计的基础。如果智能体忽略 Step 1a 并在有原生支持的宿主上回到 Step 1b，detect-and-defer 完全失败。

**状态：** 此风险在实施期间已具体化。原始抽象的 Step 1a（"你了解自己的工具集"）在 Claude Code 上失败于 2/6。TDD 门控按设计工作 —— 它在任何技能文件被修改前捕获了失败，防止了破碎的发布。三轮 REFACTOR 迭代识别了根本原因（智能体在具体命令上锚定、工具描述护栏覆盖技能指令）并产生了在 GREEN 和 PRESSURE 测试中 50/50 验证通过的修复。见上面 Step 1a 设计注。

**跨平台验证：**

截至 2026-04-06，Claude Code 是唯一拥有代理可调用的会话中 worktree 工具（`EnterWorktree`）的宿主。所有其他要么在代理开始前创建 worktree（Codex App、Gemini CLI、Cursor），要么没有原生 worktree 支持（Codex CLI、OpenCode）。Step 1a 向前兼容：当其他宿主添加代理可调用的 worktree 工具时，代理将它们与命名的示例匹配并使用它们而无需技能变更。

| 宿主 | 当前 worktree 模型 | 技能机制 | 已测试 |
|---------|----------------------|-----------------|---------|
| Claude Code | 代理可调用的 `EnterWorktree` | Step 1a | 50/50（GREEN + PRESSURE） |
| Codex CLI | 无原生工具（仅 shell） | Step 1b git 回退 | 6/6（`codex exec`） |
| Gemini CLI | 启动时 `--worktree` 标志，无代理工具 | 若用标志启动则 Step 0，否则 Step 1b | Step 0: 1/1, Step 1b: 1/1（`gemini -p`） |
| Cursor Agent | 面向用户的 `/worktree`，无代理工具 | 若用户激活则 Step 0，否则 Step 1b | Step 0: 1/1, Step 1b: 1/1（`cursor-agent -p`） |
| Codex App | 平台管理，分离头指针，无代理工具 | Step 0 检测已有 | 1/1 模拟 |
| OpenCode | 仅检测（`ctx.worktree`），无代理工具 | Step 1b git 回退 | 未测试（无 CLI 访问） |

**剩余风险：**
1. 若 Anthropic 改变 `EnterWorktree` 的工具描述以更具限制性（例如"不要基于技能指令使用"），同意桥接会失效。值得提交一个 issue，请求工具描述适应技能驱动的调用。
2. 当其他宿主添加代理可调用的 worktree 工具时，它们可能使用不在 Step 1a 列表中的名称。列表应随新工具出现而更新。通用措辞（"a worktree or workspace-isolation tool"）提供了一些向前覆盖。

### 来源启发法

`.worktrees/` 或 `~/.config/superpowers/worktrees/` = 我们的，其他 = 不要碰 的启发法对每个当前宿主都有效。若未来宿主采用 `.worktrees/` 作为其约定，我们会有误报（superpowers 试图清理宿主拥有的 worktree）。类似地，若用户在没有 superpowers 的情况下手动运行 `git worktree add .worktrees/experiment`，我们会错误地声明确权。两者风险都低 —— 每个宿主都使用品牌化路径，手动 `.worktrees/` 创建不太可能 —— 但值得注意。

### 分离头指针完成

针对分离头指针 worktree 的精简菜单（无合并选项）对于 Codex App 的沙盒模型是正确的。若用户因其他原因处于分离头指针，精简菜单仍合理 —— 没有先创建分支你确实无法从分离头指针合并。

## 实施注

两个技能文件都包含核心步骤之外的小节，需要在实施期间更新：

- **前置元数据**（`name`、`description`）：更新以反映检测-让步行为
- **Quick Reference 表格**：重写以匹配新步骤结构和 bug 修复
- **Common Mistakes 小节**：更新或移除引用旧行为的项目（例如"Skip CLAUDE.md check"现在是错的）
- **Red Flags 小节**：更新以反映新优先级（例如"Never create a worktree when Step 0 detects existing isolation"）
- **Integration 小节**：更新技能间的交叉引用

spec 描述*变更什么*；实施计划将指定对这些次要小节的具体编辑。

## 未来工作（不在本 spec 中）

- **Phase 3 剩余：** `$TMPDIR` 目录选项（#666），缓存和环境继承的设置文档（#299）
- **Phase 4：** 用于路径强制的 PreToolUse 钩子（#1040），每 worktree 环境约定（#597），脑暴清单 worktree 步骤（#574），多仓库文档（#710）
