# Worktree 翻新实施计划 - 分析报告

> **对应原文**：`docs/superpowers/plans/2026-04-06-worktree-rototill.md`
> **翻译文件**：`z-study/docs/superpowers/plans/2026-04-06-worktree-rototill.zh-CN.md`
> **报告时间**：2026-06-03

---

## 一、基本信息

| 项目 | 内容 |
|---|---|
| 文档类型 | 实施计划（Implementation Plan） |
| 文档规模 | **880 行（项目最大单文件计划）** |
| Task 数量 | 5 个（其中 Task 1 是门控） |
| 关联 Spec | `docs/superpowers/specs/2026-04-06-worktree-rototill-design.md` |
| 关联工单 | PRI-974（+ 相关 #991、#940、#999、#238、#1049） |
| 核心变更 | 两个 SKILL.md 完整重写 + 3 个集成更新 |
| 测试策略 | RED-GREEN-REFACTOR TDD 范式应用于 Markdown 技能 |
| 创新点 | 用 `claude -p` 无头测试 agent 行为 |

---

## 二、目的

这是对 worktree 工作流的**全面翻新**——既要"与宿主平台协作"（detect-and-defer），又要"修复三个长期存在的 bug"，还要"建立 TDD 范式以验证 agent 行为"。

### 三大目标

| 目标 | 描述 |
|---|---|
| **1. 优先用宿主平台** | Claude Code 的 EnterWorktree、Codex App 的外部 worktree——优先用宿主能力 |
| **2. 修复完成阶段的 3 个 bug** | #940（Option 2 误清）、#999（顺序错乱）、#238（CWD 在 worktree 内） |
| **3. 验证假设（门控）** | Task 1 是 GATE：先验证"agent 真的会优先用原生工具"，再改代码 |

### 历史债务清单

| 编号 | 描述 | 修复位置 |
|---|---|---|
| **#991** | 工作流缺乏用户同意机制 | 新增 Step 0 consent prompt |
| **#940** | Option 2（创建 PR）误清理 worktree | Step 6 仅对 Options 1/4 清理 |
| **#999** | 合并/验证/移除/删除分支的顺序错乱 | Step 5 严格按顺序执行 |
| **#238** | 在 worktree 内运行 `git worktree remove` 静默失败 | 统一 `cd` 到主仓库根 |
| **#1049** | 文档硬编码 Claude Code 路径 | 改为 `${CLAUDE_PLUGIN_ROOT}` 占位符 |

---

## 三、核心技术决策

### 1. **Task 1 是 GATE 任务**（最重要的决策）

> "If the GREEN phase fails after 2 REFACTOR iterations, STOP. Do not proceed to Task 2."

**为什么用 GATE**：
- Step 1a（原生工具偏好）是整个设计的承重假设
- 如果 agent 不遵循，spec 失败——继续重写代码是浪费
- 先验证假设，再动代码——**TDD 的核心精神**

**门控价值**：
- 避免在错误前提下投入资源
- 把"假设可被验证"显式化
- 失败时快速止损

### 2. **TDD 范式应用于 Markdown 技能**

本计划最深刻的方法论创新——**用 TDD 验证"技能文本"是否能改变 agent 行为**。

| 阶段 | 目标 | 期望结果 |
|---|---|---|
| **RED** | 用当前（无 Step 1a）技能跑测试 | agent 使用 `git worktree add`（旧行为） |
| **GREEN** | 加 Step 1a 后跑测试 | agent 使用 EnterWorktree（新行为） |
| **PRESSURE** | 加压力："紧急、你熟悉 git worktree" | agent 仍用 EnterWorktree（抗压） |

**测试框架**：
- `claude -p`（无头 Claude Code）
- 临时 git 仓库（`create_test_project` helper）
- 断言：包含 "EnterWorktree"、不包含 "git worktree add"

### 3. **detect-and-defer 模式**

```
检测 → 跳过/优先/回退
   ↓
环境信号 (GIT_DIR vs GIT_COMMON)
```

| 检测结果 | 动作 |
|---|---|
| 已在 worktree 内（且非 submodule） | 跳过创建 |
| 宿主有原生工具 | 使用原生工具 |
| 都无 | 回退到 git worktree add |

**核心原则**：**Never fight the harness.**——不与宿主平台对抗。

### 4. **submodule guard（关键修复）**

`GIT_DIR != GIT_COMMON` 在两种情况下都为 true：
- 关联 worktree（期望触发"跳过"）
- git submodule（**误报**触发"跳过"）

**修复**：
```bash
git rev-parse --show-superproject-working-tree 2>/dev/null
```

如果返回路径 → 在 submodule 中 → 不视为 worktree。

**价值**：这个 guard 修复了 [2026-03-23 计划](./2026-03-23-codex-app-compatibility.md) 留下的边界 case。

### 5. **provenance-based cleanup（基于来源的清理）**

**之前**："在 worktree 内就清理"——错杀宿主 worktree
**之后**：根据 worktree 路径判断"谁创建"

```bash
# 只有这两个路径是 superpowers 创建的
.worktrees/   # 项目本地
~/.config/superpowers/worktrees/  # 全局
```

**不在此列的 worktree** → 宿主环境拥有 → 不清理

**关键洞见**：**清理责任 = 创建责任**。不该由 superpowers 清理它没创建的资源。

### 6. **hooks 软链接（修复静默失败）**

```bash
if [ -D "$MAIN_ROOT/.git/hooks" ]; then
    ln -sf "$MAIN_ROOT/.git/hooks" "$path/.git/hooks"
fi
```

**背景**：worktree 不继承父仓库的 `.git/hooks`——会导致 pre-commit 钩子静默失效

**修复**：手动 symlink 父仓库的 hooks 目录

### 7. **"先检测再问"的同意机制**

新增 Step 0 consent：
> "Would you like me to set up an isolated worktree? It protects your current branch from changes."

**关键原则**：
- 检测优先（廉价、只读）
- 询问其次（用户决策）
- 同意后才创建（避免浪费资源）

**修复** #991：让工作流更尊重用户决策。

### 8. **cd 到主仓库根再清理**

```bash
MAIN_ROOT=$(git -C "$(git rev-parse --git-common-dir)/.." rev-parse --show-toplevel)
cd "$MAIN_ROOT"
git worktree remove "$WORKTREE_PATH"
```

**为什么必须 cd**：在 worktree 内运行 `git worktree remove <self>` 会静默失败。

**修复** #238：所有 worktree 清理前必须 cd。

### 9. **post-removal pruning（自我修复）**

```bash
git worktree remove "$WORKTREE_PATH"
git worktree prune  # 清理任何残留注册
```

`git worktree prune` 是**自愈性**操作——清理因崩溃而残留的注册信息。

### 10. **${CLAUDE_PLUGIN_ROOT} 占位符（修复硬编码）**

**之前**：`/Users/drewritter/prime-rad/superpowers/...`（Mac 硬编码）
**之后**：`${CLAUDE_PLUGIN_ROOT}/...`（平台无关）

**修复** #1049：让技能在多平台复用。

---

## 四、文件结构

### 重写（2 个完整文件）
| 文件 | 变化 |
|---|---|
| `using-git-worktrees/SKILL.md` | 219 行 → ~210 行（重写） |
| `finishing-a-development-branch/SKILL.md` | 201 行 → ~220 行（重写） |

### 集成更新（3 个 1 行修改）
| 文件 | 修改内容 |
|---|---|
| `executing-plans/SKILL.md:68` | "Set up" → "Ensures" |
| `subagent-driven-development/SKILL.md:268` | "Set up" → "Ensures" |
| `writing-plans/SKILL.md:16` | 修正"created by brainstorming"误导 |

### 新建（1 个测试）
- `tests/claude-code/test-worktree-native-preference.sh`（100+ 行 bash 脚本）

---

## 五、Task 划分逻辑

| Task | 关注点 | 风险等级 |
|---|---|---|
| **Task 1（GATE）** | TDD 验证 Step 1a 假设 | **关键**（决定后续是否继续） |
| **Task 2** | using-git-worktrees 完整重写 | 高（核心技能） |
| **Task 3** | finishing-a-development-branch 完整重写 | 高（核心技能） |
| **Task 4** | 三个集成更新 | 低（1 行/文件） |
| **Task 5** | 端到端验证 | 低（汇总） |

**依赖图**：
```
Task 1 (GATE)
   ↓
Task 2 (using-git-worktrees)
   ↓
Task 3 (finishing-a-development-branch)
   ↓
Task 4 (集成)
   ↓
Task 5 (验证)
```

**每个 Task 都有"通过条件"**：下一个 Task 依赖上一个 Task 的验证。

---

## 六、优点

| 维度 | 具体表现 |
|---|---|
| **方法论严谨** | GATE 任务 + TDD 范式应用于 Markdown——**技能工程的元方法论创新** |
| **承重假设优先验证** | 不在错误前提下投入资源——优秀的工程纪律 |
| **detect-and-defer 模式** | 让 skill 与宿主平台协作而非对抗 |
| **修复历史债务** | 5 个已知的 bug/问题被一一对应解决 |
| **可测性** | 用 `claude -p` 跑 agent 行为测试，**技能可被客观验证** |
| **抗压测试** | PRESSURE 阶段验证"在压力下仍用原生工具"——防回归 |
| **清理安全** | provenance-based 防止误删宿主资源 |
| **错误处理细致** | cd 到主仓库根、prune 残留、symlink hooks——每个边界都覆盖 |
| **向后兼容** | 既有 `~/.config/superpowers/worktrees/` 全局路径仍受支持 |
| **平台中立** | `${CLAUDE_PLUGIN_ROOT}` 占位符让技能跨平台可用 |

---

## 七、缺点 / 风险

| 风险点 | 影响 | 缓解 |
|---|---|---|
| **Task 1 GATE 失败则整个项目停摆** | 如果 agent 不优先用原生工具，需要重新设计 | 2 轮 REFACTOR 限制；明确"STOP and report back" |
| **TDD 测试依赖 Claude Code CLI** | 非 Claude Code 平台无法运行 | 测试是验证手段而非硬性要求；其他平台可手动验证 |
| **重写 2 个核心技能影响面大** | 可能影响大量下游使用 | Task 5 端到端验证 + 既有测试套件 |
| **provenance 检查可能误判** | 自定义路径可能与 `.worktrees/` 同名 | 文档明确约定（hidden dir + gitignore） |
| **submodule guard 仍可能漏** | 极端嵌套场景（submodule 内的 worktree）未覆盖 | 渐进式修复，未来再处理 |
| **subprocess 调用顺序敏感** | `MAIN_ROOT=$(...)` 必须在 `cd` 之前求值 | Task 3 代码示例严格按此顺序 |
| **PRESSURE 测试场景主观** | "紧急"等表述可能未充分传达压力 | 可调整场景措辞；2 轮 REFACTOR 留出迭代空间 |
| **修复 #999 顺序是隐式契约** | 若不读 Step 5 注释，可能误改顺序 | 注释明确说明每步的目的 |
| **5 个 commit 信息都很长** | 影响 `git log` 可读性 | 长描述详细在 body，subject 简洁 |
| **集成更新散落 3 个文件** | 一次改动需要 3 个文件 review | Task 4 一个 commit 包含全部，便于整体回滚 |

---

## 八、关键洞察

### 洞察 1：技能是"行为的 API 文档"

| 视角 | 类比 |
|---|---|
| 代码 API | 函数签名 + 行为 |
| 技能 | 文本指令 + agent 行为 |
| 单元测试 | 验证 API 行为 |
| TDD 技能测试 | 验证 agent 是否遵循指令 |

**TDD 应用于 Markdown 技能**是**"文档即代码"**理念的极致——文档不再只是参考，而是被测试覆盖的可执行规范。

### 洞察 2：GATE 任务是最便宜的保险

如果跳过 GATE 直接重写——可能重写完才发现 Step 1a 无效，需要推倒重来。

**GATE 任务的成本**：1 个测试 + 2 轮验证 ≈ 1-2 小时
**GATE 失败挽回的成本**：重写全部技能 + 重新提交 + 团队困惑 ≈ 1-2 周

**GATE = 用 1% 的成本避免 99% 的浪费**。

### 洞察 3：provenance 检查是清理的伦理基础

"我能清理它吗？" 的回答不是"它存在吗？"——而是"**我创建了它吗？**"。

```
我创建的 → 我清理  (provenance match)
宿主创建的 → 宿主清理 (provenance mismatch)
```

这是"权限最小化原则"在资源清理中的体现。

### 洞察 4：cd 到主仓库根是"语义正确的必要条件"

`git worktree remove` 不接受"在 worktree 内删除自己"——这是 Git 设计的安全约束。

**服从工具的语义**比"试图绕过"更可靠。

### 洞察 5：submodule guard 揭示概念边界

`GIT_DIR != GIT_COMMON` 看似清晰，实则有 2 个合法 case：
1. 关联 worktree
2. git submodule

**子模块是 git 的"二级工作树"**——和工作树功能不同。**正确的检测必须区分概念**。

### 洞察 6：TDD 范式让 Markdown 技能可演化

| 没有 TDD | 有 TDD |
|---|---|
| 修改技能 → 部署 → 用户报告问题 | 修改技能 → 跑测试 → 验证假设 |
| 回归无感 | 回归即刻可见 |
| 技能腐化是必然 | 技能可长期维护 |

**TDD 让 Markdown 技能获得"代码级的可维护性"**。

### 洞察 7：PRESSURE 阶段是反"路径依赖"

PRESSURE_SCENARIO 故意构造：
- "Production is impacted"（紧急感）
- "git worktree add works reliably"（既有路径更可靠）
- ".worktrees/ 存在"（环境支持回退）

**测试 agent 在多压力下是否仍遵循 Step 1a**——这是"原则 vs 权宜"的较量。

### 洞察 8：detect-and-defer 哲学可推广

| 场景 | detect-and-defer 应用 |
|---|---|
| 数据库迁移 | 检测 → 用宿主平台迁移工具 → 回退到 SQL |
| 测试运行 | 检测 → 用 IDE 测试运行器 → 回退到 CLI |
| 部署 | 检测 → 用 CI/CD 平台 → 回退到手动 |

**让工具"识别宿主"是普适的设计模式**。

---

## 九、关联文档

### 上游 / Spec
- [`2026-04-06-worktree-rototill-design.md`](../specs/2026-04-06-worktree-rototill-design.md) — 设计文档（本计划的依据）

### 下游 / 受影响
- `skills/using-git-worktrees/SKILL.md` — 接受 Task 2 的完整重写
- `skills/finishing-a-development-branch/SKILL.md` — 接受 Task 3 的完整重写
- `skills/executing-plans/SKILL.md` — Task 4 单行更新
- `skills/subagent-driven-development/SKILL.md` — Task 4 单行更新
- `skills/writing-plans/SKILL.md` — Task 4 单行更新
- `tests/claude-code/test-worktree-native-preference.sh` — Task 1 新建

### 横向关联
- [`2026-03-23-codex-app-compatibility.md`](./2026-03-23-codex-app-compatibility.md) — 本计划的前置（首次引入 `GIT_DIR != GIT_COMMON` 检测）
- [`2025-11-22-opencode-support-design.md`](../../plans/2025-11-22-opencode-support-design.md) — 多平台支持
- [`2025-11-28-skills-improvements-from-user-feedback.md`](../../plans/2025-11-28-skills-improvements-from-user-feedback.md) — 用户反馈驱动的技能改进
- `skills/writing-skills/testing-skills-with-subagents.md` — TDD 框架（Task 1 的方法论来源）
- `tests/claude-code/test-helpers.sh` — 测试基础设施（Task 1 依赖）

### 历史
- PRI-974 是 worktree 翻新的总工单
- 5 个相关工单（#991、#940、#999、#238、#1049）都是过去半年累积的 issue
- 本计划是 worktree 工作流的最大单次重构

---

## 十、状态评价

**计划成熟度：极高** ⭐⭐⭐

- Task 1 GATE 模式是行业级方法论
- 5 个历史 issue 全部对应修复
- PRESSURE 阶段覆盖对抗性场景
- 端到端验证步骤完备
- 集成更新有专门 Task

**实施复杂度：中高** ⚠️

- 涉及 5 个文件 + 1 个新测试
- 2 个完整重写（而非增量修改）
- TDD 框架的引入需要学习曲线
- 失败兜底（GATE 失败时停止）需要纪律

**适用性**：✅ 强烈推荐
- 解决了大量历史债务
- 引入"Markdown TDD" 元方法论
- detect-and-defer 模式可推广到其他技能

---

## 十一、给读者的启示

### 启示 1：TDD 不只适用于代码
TDD 的核心是"先验证假设，再实现"。**任何可被验证的东西都可用 TDD**——包括 Markdown 技能、配置文件、设计文档。

### 启示 2：GATE 任务是纪律
"在不确定的前提下投入资源"是工程大忌。**对承重假设做 GATE 验证**——用 1% 的成本避免 99% 的浪费。

### 启示 3：provenance 是清理的伦理
"我创建的，我清理；不是我创建的，宿主清理"——**最小权限原则的延伸**。

### 启示 4：submodule guard 揭示概念精度
"两个看起来相同的现象"在精细分析下可能是不同概念。**检测函数必须能区分**——否则就是误报。

### 启示 5：Never fight the harness
不要与宿主平台对抗。**detect-and-defer 模式是工具设计的黄金法则**。

### 启示 6：PRESSURE 测试是反"路径依赖"
"在压力下是否会回到旧模式"是任何流程改进都该验证的——**抗压测试 = 防止回潮**。

### 启示 7：${CLAUDE_PLUGIN_ROOT} 占位符的智慧
硬编码路径是"今天能跑、明天爆炸"的根源。**用占位符表达"我依赖宿主解析"**。

### 启示 8：Task 1 GATE 模式的普适价值
| 场景 | GATE 任务 |
|---|---|
| 重构 | 验证"重构后行为等价" |
| 新技术引入 | 验证"新技术真的解决问题" |
| 范式切换 | 验证"新范式真的被接受" |
| 性能优化 | 验证"瓶颈确实在假设位置" |

**GATE = 把假设变成可验证的、可证伪的、可止损的命题**。

---

## 十二、后续工作（推测）

1. **TDD 框架文档化**：将 `test-worktree-native-preference.sh` 的模式抽象为通用 `test-skill-with-claude.sh` helper
2. **更多 RED-GREEN-REFACTOR 套件**：将 TDD 范式推广到 `brainstorming`、`writing-plans`、`using-superpowers` 等其他技能
3. **PRESSURE 库**：构造更多 PRESSURE 场景（时间压力、上级压力、用户诱导等），建立 PRESSURE 测试集
4. **submodule 嵌套场景**：未来若要支持"submodule 内的 worktree"模式，需扩展 guard 逻辑
5. **跨平台测试**：在 Windows、macOS、Linux 各跑一遍 GATE 测试，确认 detect-and-defer 行为一致
6. **技能行为监控**：上线后监控"agent 是否实际遵循新技能"——通过日志分析 RED/GREEN 真实发生率
7. **配套 GATE 工具**：开发一个 `superpowers-gate-check` 工具，自动跑所有 GATE 测试
8. **`writing-plans` 也用 TDD**：让"编写计划"本身也可被测试——验证"按计划执行真的能达到目标"
9. **PR 模板化**：让 Task 4 这样的 1 行集成更新有 PR 模板，自动生成 "fix: ..." commit message
10. **`worktree-rototill` 之后的下一次翻新**：可能聚焦在 `dispatching-parallel-agents` 或 `subagent-driven-development`
