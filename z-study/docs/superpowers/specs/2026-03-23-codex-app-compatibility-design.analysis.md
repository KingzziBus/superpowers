# Codex App 兼容性设计 - 分析报告

> **对应原文**：`docs/superpowers/specs/2026-03-23-codex-app-compatibility-design.md`
> **翻译文件**：`z-study/docs/superpowers/specs/2026-03-23-codex-app-compatibility-design.zh-CN.md`
> **报告时间**：2026-06-03

---

## 一、基本信息

| 项目 | 内容 |
|---|---|
| 文档类型 | 设计 Spec |
| 总行数 | 245 行 |
| 关联计划 | `docs/superpowers/plans/2026-03-23-codex-app-compatibility.md` |
| 关联工单 | PRI-823 |
| 核心机制 | `git-dir` vs `git-common-dir` 对比检测 |
| 范围 | 5 个文件，~50 行变更 |
| 平台 | Codex App（沙盒化 worktree） |

---

## 二、目的

让 Superpowers 核心技能在 Codex App 沙盒环境中工作，**且不破坏** Claude Code / Codex CLI 既有行为。

### 关键洞察

> "Codex CLI 没有这个冲突——它没有内建的 worktree 管理。我们的手动 worktree 方法在那里填补了隔离空白。问题特定于 Codex App。"

**这是对"问题边界"的精确定位**——不是泛指"Codex 工具"问题，是 **Codex App 沙盒**特有的问题。

---

## 三、核心技术决策

### 1. 经验性发现驱动设计

spec 包含完整的"经验性发现"表（6 种操作 × 2 种沙盒），基于 2026-03-23 实际测试。**不是理论设计，是实验验证**。

### 2. 三条只读 git 命令

```bash
GIT_DIR=$(cd "$(git rev-parse --git-dir)" 2>/dev/null && pwd -P)
GIT_COMMON=$(cd "$(git rev-parse --git-common-dir)" 2>/dev/null && pwd -P)
BRANCH=$(git branch --show-current)
```

**为何选 `git-dir != git-common-dir` 而非 `show-toplevel`**：
- 普通仓库：两者相同
- 关联 worktree：不同（`.git/worktrees/<name>` vs `.git`）
- 子模块：相同（避免误报）

`show-toplevel` 在子模块中也会"不同"——**会产生误报**。

### 3. 通过 `cd && pwd -P` 解析

- 处理相对路径（普通仓库返回 `.git`，worktree 返回绝对）
- 处理符号链接（macOS 上 `/tmp` → `/private/tmp`）

**这是处理平台差异的细致工程**。

### 4. 决策矩阵（4 状态穷举）

| 关联 Worktree？ | 分离头指针？ | 动作 |
|---|---|---|
| 否 | 否 | 完整技能（不变） |
| 是 | 是 | 跳过创建 + 交接载荷 |
| 是 | 否 | 跳过创建 + 完整完成 |
| 否 | 是 | 正常创建 + 警告 |

**穷举所有状态 = 行为完备**。

### 5. 沙盒回退（late-detection fallback）

`GIT_DIR == GIT_COMMON` + 尝试 `git worktree add` 失败 → 回退到 "已在 worktree" 行为。

**价值**：应对未来沙盒策略变化——不被早期检测漏过困住。

### 6. 交接载荷（Path A 专属）

- 包含 commit SHA（用户引用方便）
- 包含数据丢失警告（防止误操作）
- 引导用 Codex App 原生控件
- 给出建议分支名/提交信息（可复制粘贴）

**这是把"用户体验最后一公里"做到极致的范例**。

### 7. Step 5 清理守卫（双保险）

完成时**重新**检测，不依赖之前输出。**因为 finishing 技能可能在不同会话运行**——状态可能已变。

### 8. "Ensures isolated workspace" 措辞

| 旧 | 新 |
|---|---|
| "Set up isolated workspace" | "Ensures isolated workspace" |

**"Ensures" 表达"幂等"**——可重复调用，可能创建也可能验证。

---

## 四、文件结构

| 文件 | 变更 |
|---|---|
| `using-git-worktrees/SKILL.md` | +12 行（Step 0） |
| `finishing-a-development-branch/SKILL.md` | +20 行（Step 1.5 + 清理守卫） |
| `subagent-driven-development/SKILL.md` | 1 行编辑 |
| `executing-plans/SKILL.md` | 1 行编辑 |
| `using-superpowers/references/codex-tools.md` | +15 行 |

**总计**：5 个文件，~50 行变更，零新增文件，零破坏性变更。

---

## 五、优点

| 维度 | 具体表现 |
|---|---|
| **经验性而非理论** | 基于实际测试数据设计 |
| **只读检测** | 零副作用 |
| **三路径穷举** | 4 种状态全覆盖 |
| **零破坏性** | 4 选项菜单在普通环境完全保留 |
| **可测性** | 5 个手动测试 + 5 个自动测试 |
| **可观测** | 详细的"经验性发现"表 |
| **明确边界** | "Codex App 特有问题" 而非泛指 |
| **沙盒回退** | 应对未来变化 |
| **可复用模式** | 未来其他沙盒可借鉴 |

---

## 六、缺点 / 风险

| 风险点 | 影响 | 缓解 |
|---|---|---|
| **submodule 边界 case** | `git-dir != git-common-dir` 在 submodule 内也是 true | 未来 spec 需 submodule guard（已被 [2026-04-06 plan](../plans/2026-04-06-worktree-rototill.md) 修复） |
| **检测状态可能滞后** | 创建 worktree 后 `.git/worktrees/<name>` 立即可用 | 使用 `cd && pwd -P` 强制解析 |
| **未来沙盒策略变化** | Codex App 可能限制 `git rev-parse` | 检测只读，理论上不受限 |
| **Path A 用户认知负担** | 用户需理解 "external management" | 交接载荷用自然语言 |
| **建议分支名不一定合适** | 模板"前 5 词 slugify"可能尴尬 | 文档明确 "omittable" |
| **测试脚本 Windows 兼容** | `mktemp`、`trap` 是 Unix 习惯 | 可能需 WSL |
| **完成时双检测开销** | Step 5 重新检测增加 100ms | 可接受 |

---

## 七、关键洞察

### 洞察 1：经验性设计 > 理论设计

spec 包含 2026-03-23 实际测试的"经验性发现"表。**不是推测可能的行为，是测过的**。这是工程严谨性的体现。

### 洞察 2：精确的问题边界

不是"Codex 工具"问题，是 **Codex App 沙盒**特有。**精确识别问题边界避免过度泛化**。

### 洞察 3：submodule 边界揭示概念精度

`show-toplevel` 在 submodule 内也"不同"——会产生误报。**检测函数必须能区分相似但不同的概念**。

### 洞察 4：cd && pwd -P 是平台差异的处理

- 处理相对路径 vs 绝对
- 处理符号链接（macOS）

**这是工程上的细致**——很多"看似对的"代码在 macOS 上会出错。

### 洞察 5：决策矩阵是状态完备性的体现

4 种状态穷举所有可能——**不留死角**。这是"状态机问题"的标准解法。

### 洞察 6：沙盒回退应对未来变化

> "If `GIT_DIR == GIT_COMMON` and the skill proceeds to Creation Steps, but `git worktree add -b` fails..."

**"未来沙盒可能加新的限制"是常识**——回退机制是"假设会变化"的工程哲学。

### 洞察 7：交接载荷是"用户工程的最后一公里"

包含 commit SHA、分支建议、数据警告——**让用户复制粘贴即可继续**。

### 洞察 8：Step 5 重新检测的智慧

> "Re-run the detection at cleanup time (do not rely on earlier skill output — the finishing skill may run in a different session)."

**不依赖之前状态 = 鲁棒**。这是分布式系统设计的常识在单人脚本中的体现。

---

## 八、关联文档

### 下游 / 实施
- [`2026-03-23-codex-app-compatibility.md`](../plans/2026-03-23-codex-app-compatibility.md) — 实施计划

### 横向关联
- [`2026-04-06-worktree-rototill-design.md`](./2026-04-06-worktree-rototill-design.md) — 紧随其后的"worktree 翻新"，**subsumes 本 spec**
- [`2026-01-22-document-review-system-design.md`](./2026-01-22-document-review-system-design.md) — 同时期 spec
- [`2026-02-19-visual-brainstorming-refactor-design.md`](./2026-02-19-visual-brainstorming-refactor-design.md) — 同时期 spec
- [`2025-11-22-opencode-support-design.md`](../../plans/2025-11-22-opencode-support-design.md) — 多平台支持

### 受影响
- 5 个技能文件（修改范围明确）
- Codex App 平台用户

### 历史
- Codex App 推出后，用户报告"worktree 技能在沙盒中失败"
- 本 spec 是该问题的一次性修复

---

## 九、状态评价

**设计成熟度：极高** ⭐⭐⭐

- 经验性数据驱动
- 状态矩阵完备
- 测试计划详尽
- 风险缓解明确

**实施复杂度：低-中** ⬇️

- 范围明确（5 文件，~50 行）
- 不引入新工具
- 每个 Task 步骤清晰

**适用性**：✅ 强烈推荐
- 解决真实问题
- 改动局部，不破坏现有用户
- 模式可推广到其他平台

---

## 十、给读者的启示

### 启示 1：经验性 > 理论
**先测再设计**——spec 包含 2026-03-23 实际测试数据。理论推测易出错，实验验证更可靠。

### 启示 2：精确问题边界
"Codex 工具" vs "Codex App 沙盒"——**精确识别问题边界避免过度泛化**。

### 启示 3：cd && pwd -P 是细致工程
处理相对路径、符号链接——**很多代码"看似对"在 macOS 上会出错**。

### 启示 4：决策矩阵是状态完备性
4 种状态穷举 = 不留死角。**状态机问题用矩阵覆盖**。

### 启示 5：沙盒回退应对未来
**假设平台策略会变化**——回退机制是工程智慧。

### 启示 6：交接载荷是"用户工程最后一公里"
包含 commit SHA、分支建议——**让用户成功路径最短**。

### 启示 7：不依赖之前状态
Step 5 重新检测——**分布式系统设计的常识应用到单人脚本**。

### 启示 8：Ensures 表达幂等
"Ensures isolated workspace" 表达"幂等"——**API 命名要传达行为契约**。

---

## 十一、后续工作（推测）

1. **更多沙盒平台支持**：CircleCI、GitHub Codespaces、Gitpod
2. **检测脚本提取为库**：供更多技能复用
3. **交接载荷的 i18n**：英文外的语言
4. **Codex App API 集成**：未来若提供"创建分支"API
5. **submodule guard 增强**：[2026-04-06 plan](../plans/2026-04-06-worktree-rototill.md) 已加，但边缘 case 仍需测试
6. **Codex App 版本兼容矩阵**：不同版本的沙盒策略可能不同
7. **路径统一化**：`${CLAUDE_PLUGIN_ROOT}` 占位符
