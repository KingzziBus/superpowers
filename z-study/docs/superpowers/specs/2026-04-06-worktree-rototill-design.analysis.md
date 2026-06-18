# Worktree 翻新设计 - 分析报告

> **对应原文**：`docs/superpowers/specs/2026-04-06-worktree-rototill-design.md`
> **翻译文件**：`z-study/docs/superpowers/specs/2026-04-06-worktree-rototill-design.zh-CN.md`
> **报告时间**：2026-06-03

---

## 一、基本信息

| 项目 | 内容 |
|---|---|
| 文档类型 | 设计 Spec（合并型） |
| 总行数 | 343 行（项目最大 spec） |
| 关联计划 | `docs/superpowers/plans/2026-04-06-worktree-rototill.md` |
| 关联工单 | PRI-974（+ subsumes PRI-823） |
| 状态 | 草稿 |
| 风险已解决 | 1 个（Step 1a） |
| 已解决 issue | 8 个（#940, #991, #918, #1009, #999, #238, #1049, #279） |
| 延期 issue | 1 个（#574 → Phase 4） |
| 跨平台测试 | 6 个宿主 |

---

## 二、目的

这是 worktree 工作流的**全面翻新**——既要"与宿主平台协作"（detect-and-defer），又要"修复 8 个历史问题"，还要"建立 spec 合并模式"。

### 三大目标

1. **detect-and-defer**：让 superpowers 与宿主原生 worktree 系统协作
2. **修复历史债务**：8 个累积 issue 一次性解决
3. **可移植性**：平台中立语言取代 Claude Code 硬编码

### 失败的 3 种模式

1. **重复** —— Claude Code 上 EnterWorktree 已做的事
2. **冲突** —— Codex App 上试图在托管 worktree 内创建
3. **幻影状态** —— `.worktrees/` 对宿主不可见

---

## 三、核心技术决策

### 1. 检测状态而非平台

```
环境状态（git 原语） = 通用、跨平台、稳定
平台识别（环境变量） = 平台特定、需维护
```

**`GIT_DIR != GIT_COMMON`** 自 git 2.5（2015）以来稳定。

**优势**：
- 跨所有宿主通用
- 零维护
- 未来新宿主自动支持

### 2. 声明式意图 + 规范式回退

| 步骤 | 性质 |
|---|---|
| Step 1a | 声明式（"如果有原生工具就用它"） |
| Step 1b | 规范式（"用 git worktree add"） |

**为何这样排序**：声明式在前 = 优先使用高级能力；规范式在后 = 回退。

### 3. 基于来源的所有权

```
.worktrees/                 → superpowers 拥有 → superpowers 清理
~/.config/superpowers/...   → superpowers 拥有 → superpowers 清理
.claude/worktrees/          → Claude Code 拥有 → 不要碰
~/.codex/worktrees/         → Codex App 拥有   → 不要碰
```

**清理责任 = 创建责任**。

### 4. 子模块守卫（submodule guard）

`GIT_DIR != GIT_COMMON` 在 git 子模块内也为真——**会产生误报**。

**修复**：
```bash
git rev-parse --show-superproject-working-tree 2>/dev/null
```

如果返回路径 → 在子模块中 → 视为普通仓库。

**这是从 [2026-03-23 spec](./2026-03-23-codex-app-compatibility-design.md) 继承的边界 case 修复**。

### 5. Step 1a 的 TDD 修订

**原始抽象版本**：
> "You know your own toolkit — the skill does not need to name specific tools."

**测试结果**：2/6 通过率（智能体锚定在 Step 1b 的具体命令）。

**修订后版本**（50/50 通过率）：

1. **显式工具命名**：
   ```
   EnterWorktree, WorktreeCreate, /worktree, --worktree
   ```
   把决策从"我有没有原生工具？"转为"`EnterWorktree` 在我工具列表里吗？"

2. **同意桥接**：
   > "The user's consent to create a worktree is your authorization to use it."
   
   直接解决 `EnterWorktree` 的工具级护栏（"仅在用户明确要求时"）。

3. **Red Flag 条目**：
   > "Use `git worktree add` when you have a native worktree tool — this is the #1 mistake"

**核心教训**：**抽象指导被智能体忽略；显式命名 + 同意桥接 + Red Flag 三重保障才有效**。

### 6. 文件分离 vs 文本质量的实验

**假设**：把 Step 1b 拆到独立文件可解决锚定问题
**结论**：**不需要文件分离**——控制测试证明 240 行完整技能也能 20/20 通过

**洞见**：**问题的根源是文本质量，不是物理分离**。

### 7. 三大设计原则

| 原则 | 体现 |
|---|---|
| 检测状态，非平台 | `GIT_DIR != GIT_COMMON` |
| 声明意图 + 规范回退 | Step 1a vs 1b |
| 基于来源的所有权 | 路径前缀判断 |

**这是 spec 顶层抽象的核心**——三个原则可推广。

### 8. 顺序的隐式契约（Bug #999 修复）

```bash
# 错误的顺序
git worktree remove <path>  # 失败：worktree 仍引用分支
git branch -d <branch>      # 失败：branch 仍被引用

# 正确的顺序
git merge <feature-branch>  # 1. 先合并
<run tests>                  # 2. 验证
git worktree remove <path>   # 3. 再清理
git branch -d <branch>       # 4. 最后删除分支
```

> "The naive fix of removing the worktree first is also wrong — if the merge then fails, the working directory is gone and changes are lost."

**工程教训**：**操作顺序的细节里藏着智慧**。

### 9. CWD 守卫（Bug #238 修复）

```bash
# 错误：在 worktree 内移除自己（静默失败）
cd /path/to/worktree
git worktree remove /path/to/worktree

# 正确：先 cd 到主仓库根
MAIN_ROOT=$(git -C "$(git rev-parse --git-common-dir)/.." rev-parse --show-toplevel)
cd "$MAIN_ROOT"
git worktree remove /path/to/worktree
```

**服从工具的语义**比"试图绕过"更可靠。

### 10. 自愈性 prune

```bash
git worktree remove "$WORKTREE_PATH"
git worktree prune  # 清理任何残留注册
```

**`git worktree prune` 是"自愈性"操作**——清理因崩溃而残留的注册信息。

**"One line, prevents silent rot"** —— 一行代码的鲁棒性投资。

---

## 四、文件结构

### 重写（2 个完整文件）
| 文件 | 变化 |
|---|---|
| `using-git-worktrees/SKILL.md` | 完整重写 |
| `finishing-a-development-branch/SKILL.md` | 完整重写 |

### 集成更新（3 个 1 行修改）
| 文件 | 修改 |
|---|---|
| `executing-plans/SKILL.md` | "Set up" → "Ensures" |
| `subagent-driven-development/SKILL.md` | "Set up" → "Ensures" |
| `writing-plans/SKILL.md` | 修正误导性"created by brainstorming" |

### 修复历史问题（8 个）
| Issue | 解决方案 |
|---|---|
| #940 | Option 2 不清理 + Step 5 限定 Options 1/4 |
| #991 | Step 0.5 同意机制 |
| #918 | Step 0 + Step 1.5 检测 |
| #1009 | Step 1a 原生工具偏好 |
| #999 | 合并→验证→移除→删除的顺序 |
| #238 | CWD 守卫 |
| #1049 | 平台中立指令文件引用 |
| #279 | detect-and-defer 不覆盖原生路径 |

---

## 五、跨平台测试矩阵

| 宿主 | 当前 worktree 模型 | 技能机制 | 测试结果 |
|---|---|---|---|
| Claude Code | 代理可调用的 `EnterWorktree` | Step 1a | 50/50（GREEN + PRESSURE） |
| Codex CLI | 无原生工具 | Step 1b | 6/6（`codex exec`） |
| Gemini CLI | 启动时 `--worktree` 标志 | Step 0 / 1b | 1/1 + 1/1（`gemini -p`） |
| Cursor Agent | 面向用户的 `/worktree` | Step 0 / 1b | 1/1 + 1/1（`cursor-agent -p`） |
| Codex App | 平台管理，分离头指针 | Step 0 检测 | 1/1 模拟 |
| OpenCode | 仅检测 | Step 1b | 未测试（无 CLI 访问） |

**价值**：**6 个宿主中 5 个有真实测试数据**——罕见的跨平台验证密度。

---

## 六、优点

| 维度 | 具体表现 |
|---|---|
| **detect-and-defer 哲学** | 三大设计原则贯穿始终 |
| **TDD 修订** | Step 1a 通过实证迭代找到最佳文本 |
| **修复历史债务** | 8 个 issue 一次性解决 |
| **跨平台验证** | 5/6 宿主有真实测试数据 |
| **明确风险** | "剩余风险"小节列具体担忧 |
| **延期明确** | #574 明确延期到 Phase 4 |
| **基于来源所有权** | 避免误删宿主资源 |
| **自愈性** | `git worktree prune` 防静默腐烂 |
| **平台中立** | `${CLAUDE_PLUGIN_ROOT}` 占位符 |
| **经验性数据** | 6 个宿主 × 2 种沙盒的实测表 |

---

## 七、缺点 / 风险

| 风险点 | 影响 | 缓解 |
|---|---|---|
| **`.worktrees/` 路径冲突** | 未来宿主若用同名路径 | 风险已识别并记录 |
| **手动 `.worktrees/` 误判** | 用户手动创建会被误认为 superpowers 拥有 | "不太可能" |
| **EnterWorktree 描述变更** | 同意桥接依赖工具描述 | 提交 issue 请求 |
| **新宿主新工具名** | Step 1a 列表可能不包含 | 通用措辞 "a worktree or workspace-isolation tool" |
| **OpenCode 未测试** | 无 CLI 访问 | "Untested" 标注 |
| **submodule 嵌套** | submodule 内的 worktree 仍可能误报 | 渐进式修复 |
| **Step 5 假设宿主不清理** | 宿主清理时机不可控 | 自愈性 prune |

---

## 八、关键洞察

### 洞察 1：TDD 让"假设可被验证"

Step 1a 原始抽象版本在 Claude Code 上失败（2/6）——**TDD 在投入资源前捕获了失败**。

> "TDD gate worked as designed — it caught the failure before any skill files were modified, preventing a broken release."

**这是 TDD 范式应用于 Markdown 技能的最大胜利**。

### 洞察 2：抽象指导对智能体无效

> "You know your own toolkit — the skill does not need to name specific tools."

智能体锚定在 Step 1b 的具体命令，忽略抽象指导。**显式命名 + 同意桥接 + Red Flag 三重保障才有效**。

**AI 系统的 prompt 工程不是"少说"——是"明确说"**。

### 洞察 3：工具描述覆盖技能指令

> "Tool descriptions override skill instructions (Claude Code #29950)."

工具的"仅在用户明确要求时"护栏会**覆盖**技能的"如果有就用"指导。

**技能必须将用户同意框定为工具所需的授权**——这是工具/技能层级关系的根本。

### 洞察 4：问题根源是文本质量，不是文件分离

假设 Step 1b 拆到独立文件能解决锚定问题。**结论：不需要**——控制测试证明 240 行完整技能也能 20/20 通过。

**这避免了不必要的架构复杂性**。

### 洞察 5：检测状态 > 平台识别

```
环境状态（git 原语） = 通用、跨平台、稳定
平台识别（环境变量） = 平台特定、需维护
```

**这是"鲁棒设计"的金科玉律**——选择**基础原语**而非**应用层假设**。

### 洞察 6：基于来源的所有权

清理责任 = 创建责任。**这是"权限最小化"在资源清理中的体现**。

### 洞察 7：操作顺序藏着智慧

> "The naive fix of removing the worktree first is also wrong — if the merge then fails, the working directory is gone and changes are lost."

**正确的顺序是：合并 → 验证 → 清理 → 删除**。任何顺序错误都可能导致数据丢失。

### 洞察 8：spec 合并（subsume）模式

```
PRI-974 subsumes PRI-823
```

**当新 spec 覆盖旧 spec 时，明确标注 "subsumes"**——保留历史记录。

### 洞察 9：延期是工程决策

#574 延期到 Phase 4：

> "Nothing in this spec touches the brainstorming skill where the bug lives."

**明确说明"为什么不修"**——比"忽略"更负责。

### 洞察 10：跨平台测试矩阵

6 个宿主中 5 个有真实测试数据——**罕见的跨平台验证密度**。

**当系统声称支持多平台时，必须有测试矩阵**。

---

## 九、关联文档

### 下游 / 实施
- [`2026-04-06-worktree-rototill.md`](../plans/2026-04-06-worktree-rototill.md) — 实施计划

### 上游 / 被合并
- [`2026-03-23-codex-app-compatibility-design.md`](./2026-03-23-codex-app-compatibility-design.md) — 本 spec **subsumes** 它

### 横向关联
- [`2026-01-22-document-review-system-design.md`](./2026-01-22-document-review-system-design.md)
- [`2026-02-19-visual-brainstorming-refactor-design.md`](./2026-02-19-visual-brainstorming-refactor-design.md)
- [`2026-03-11-zero-dep-brainstorm-server-design.md`](./2026-03-11-zero-dep-brainstorm-server-design.md)

### 受影响
- 2 个核心 SKILL.md 完整重写
- 3 个集成更新
- 8 个历史 issue 修复

### 历史
- 8 个 issue 来自过去半年的累积
- 本 spec 是 worktree 工作流的最大单次重构
- TDD 修订是项目方法论的关键突破

---

## 十、状态评价

**设计成熟度：极高** ⭐⭐⭐

- 三大设计原则贯穿
- TDD 修订有具体数据（2/6 → 50/50）
- 8 个 issue 解决路径明确
- 跨平台测试矩阵完整
- 风险识别清晰

**实施复杂度：高** ⚠️

- 2 个完整重写
- 3 个集成更新
- TDD 框架需建立
- 失败兜底（GATE）需纪律

**适用性**：✅ 强烈推荐
- 解决大量历史债务
- 引入 Markdown TDD 元方法论
- detect-and-defer 可推广

---

## 十一、给读者的启示

### 启示 1：TDD 让假设可验证
**任何可被验证的东西都可用 TDD**——包括 Markdown 技能、配置文件、设计文档。

### 启示 2：抽象指导对 AI 系统无效
**"少说"对智能体无效**——需要显式命名 + 同意桥接 + Red Flag 三重保障。

### 启示 3：工具描述覆盖技能指令
**理解工具/技能层级关系**——技能必须将用户同意框定为工具所需的授权。

### 启示 4：检测状态 > 平台识别
**选择基础原语而非应用层假设**——`GIT_DIR != GIT_COMMON` 跨所有宿主通用。

### 启示 5：基于来源的所有权
清理责任 = 创建责任。**最小权限原则的延伸**。

### 启示 6：操作顺序藏着智慧
**任何"操作顺序"问题都有微妙的正确解**——不能靠"删除再合并"等天真修复。

### 启示 7：spec 合并（subsume）模式
**明确标注新旧 spec 关系**——保留历史记录。

### 启示 8：延期是工程决策
**明确说明"为什么不修"**——比"忽略"更负责。

### 启示 9：跨平台测试矩阵
**当系统声称支持多平台时，必须有测试矩阵**——5/6 宿主有真实测试数据是稀有成就。

### 启示 10：detect-and-defer 可推广
**让工具"识别宿主"是普适的设计模式**——不限于 worktree。

---

## 十二、后续工作（推测）

1. **TDD 框架文档化**：`test-worktree-native-preference.sh` 抽象为通用 helper
2. **更多 RED-GREEN-REFACTOR 套件**：推广到其他技能
3. **PRESSURE 库**：构造更多 PRESSURE 场景
4. **submodule 嵌套场景**：扩展 guard 逻辑
5. **跨平台测试**：在 Windows、macOS、Linux 各跑一遍
6. **技能行为监控**：上线后监控 agent 是否实际遵循
7. **配套 GATE 工具**：自动跑所有 GATE 测试
8. **`writing-plans` 也用 TDD**：让"编写计划"本身可被测试
9. **PR 模板化**：自动生成 "fix: ..." commit message
10. **下一次翻新**：可能聚焦 dispatching-parallel-agents
