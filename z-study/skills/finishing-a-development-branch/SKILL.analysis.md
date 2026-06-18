# finishing-a-development-branch/SKILL.md 分析报告

## 基本信息

| 项目 | 内容 |
| --- | --- |
| 文档路径 | `skills/finishing-a-development-branch/SKILL.md` |
| 字数规模 | 约 252 行 / 8.0KB |
| 文档类型 | 技能定义（SKILL）/ 工作流收尾 |
| 主题 | 完成开发分支时的决策与执行 |
| 核心定位 | "3 状态 × 4 选项" 的结构化收尾流程 |
| 核心哲学 | 检测 → 决策 → 执行 → 清理（带出处感知） |
| 面向读者 | AI 代理（执行完成时） + 协作伙伴（做决策时） |

---

## 一、文档目的

解决一个看似简单实则微妙的问题：**开发完成时，怎么让 AI 给出清晰的"下一步"选项并安全执行**。

具体目标：
1. **防止"什么该做"的模糊性**：用结构化菜单替代开放式问题
2. **防止破坏性操作**：丢弃/merge/force-push 都需要确认
3. **worktree 出处感知**：只清理自己创建的 worktree
4. **环境自适应**：3 种环境状态对应不同菜单和清理逻辑

定位上属于 **"工作流收尾"**——是 `executing-plans` 的下游消费者，提供"开发完成 → 集成/清理"的完整路径。

---

## 二、核心技术决策

### 决策 1：3 步前置检查（验证 → 检测 → 确定基础）

**Step 1: 验证测试**
- 没通过测试 → 停下
- 通过 → 继续

**Step 2: 检测环境（基于 git 原语）**
- `GIT_DIR == GIT_COMMON` → 普通 repo
- `GIT_DIR != GIT_COMMON` + 命名分支 → linked worktree
- `GIT_DIR != GIT_COMMON` + 分离 HEAD → 外部管理的 worktree

**Step 3: 确定基础分支**
- 用 `git merge-base` 找最近的 main/master
- 或询问用户

**设计意图**：
- **Step 1 防止**："merge 损坏代码"
- **Step 2 防止**："在错误的工作空间执行命令"
- **Step 3 防止**："merge 到错误的目标分支"

**这是"决策前置"原则**——所有决策都基于**已验证的事实**。

### 决策 2：3 种环境状态对应不同菜单

| 状态 | 菜单 | 清理 |
|------|------|------|
| 普通 repo | 4 选项 | 无 |
| linked worktree | 4 选项 | 出处感知 |
| 分离 HEAD | 3 选项（无 merge） | 无（外部管理） |

**关键洞察**：
- 分离 HEAD **没有"本地 merge"** 选项——因为 HEAD 不在任何分支上
- 分离 HEAD **没有清理**——因为 harness 拥有
- 4 选项 vs 3 选项 = **环境自适应**

### 决策 3：严格呈现 4（或 3）个选项

```text
"严格呈现这 4 个选项"（exactly these 4 options）
"严格呈现这 3 个选项"（exactly these 3 options）
"不要添加解释"（Don't add explanation）
```

**关键洞察**：
- **不要"自由发挥"**——AI 可能添加看似合理的第 5 个选项
- **不要"补充说明"**——简洁 = 减少用户认知负担
- **不要"开放式提问"**——"你想做什么？" 是反模式

**这是"反 LLM 自主发挥"**——AI 应该**严格按规则**而不是"看起来合理"。

### 决策 4：操作前 CWD 安全检查

```bash
MAIN_ROOT=$(git -C "$(git rev-parse --git-common-dir)/.." rev-parse --show-toplevel)
cd "$MAIN_ROOT"
```

**为什么**：
- 不能在 worktree 内部跑 `git worktree remove`（静默失败）
- 不能在 worktree 内部跑 `git checkout <base-branch>`（会失败）
- **必须先 cd 到主 repo 根**

**这是"防御性 CWD 管理"**——AI 必须明确切换工作目录。

### 决策 5：merge 后必须验证测试

```bash
git merge <feature-branch>

# 在 merge 结果上验证测试
<test command>
```

**关键洞察**：
- 单独的 feature 测试通过 ≠ merge 后测试通过
- merge 可能产生**冲突解决 bug**
- 必须在**合并后的状态**上跑测试

**这是"测试真实状态"原则**——不是测试"理论上应该"。

### 决策 6：worktree 出处感知

```text
如果 worktree 路径在 .worktrees/、worktrees/ 或 ~/.config/superpowers/worktrees/ 下：
  Superpowers 创建了这个 worktree —— 我们负责清理

否则：
  宿主环境（harness）拥有这个工作空间。不要删除它。
```

**关键洞察**：
- 路径前缀 = **创建者身份** = **清理责任**
- Superpowers 创建的 → Superpowers 清理
- Harness 创建的 → Harness 清理（或保留）
- **不要清理别人的 worktree**——会导致"幽灵状态"

**这是"所有权感知"原则**——资源管理要看**谁创建**、**谁拥有**。

### 决策 7：必须 cd 到主 repo 根再 worktree remove

```bash
MAIN_ROOT=$(...)
cd "$MAIN_ROOT"
git worktree remove "$WORKTREE_PATH"
```

**为什么必须**：
- `git worktree remove` 在 worktree 内部跑会**静默失败**
- 这是 git 的"反人类"行为
- 显式 cd = 防御灾难

**这是"git 内部行为的硬性防御"**。

### 决策 8：prune 作为自愈

```bash
git worktree remove "$WORKTREE_PATH"
git worktree prune  # 自愈：清理任何过时的注册
```

**为什么 prune**：
- `git worktree remove` 可能不完整
- 过时的 `.git/worktrees/<name>` 目录会保留
- `prune` 清理这些"幽灵注册"

**这是"防御性收尾"**——主动清理可能的不完整状态。

### 决策 9：丢弃需要 typed confirmation

```text
Type 'discard' to confirm.
```

**为什么这么严**：
- 丢弃 = **不可逆**
- 任何"软确认"（"yes"/"确认"）都太容易误触
- `discard` 是**显式、有意识**的输入

**这是"反意外删除"的硬性防御**。

### 决策 10：选项 2（PR）保持 worktree 活着

```text
不要清理 worktree —— 用户需要它活着以便对 PR 反馈进行迭代
```

**关键洞察**：
- PR 流程中用户可能要求"再改一点"
- worktree 是用户迭代的"工作空间"
- 提前清理 = 阻止后续迭代

**这是"工作流完整性"**——收尾不能**破坏**正在进行的流程。

### 决策 11：强制 `git worktree prune`

**为什么必做**：
- 防御性清理
- 防止 `.git/worktrees/<name>` 残留
- 下次 git 操作时不会被"幽灵 worktree"干扰

**这是"git 内部状态的卫生"**。

---

## 三、文件结构

| 章节 | 内容 | 工程价值 |
| --- | --- | --- |
| 概述 | 核心原则 + 宣告 | 立场 |
| Step 1 | 验证测试 | 质量门 |
| Step 2 | 检测环境 | 上下文 |
| Step 3 | 确定基础分支 | 决策准备 |
| Step 4 | 呈现选项 | 严格 4/3 选项 |
| Step 5 | 执行选择（4 个子选项） | 核心操作 |
| Step 6 | 清理工作空间 | 资源管理 |
| 快速参考 | 4 选项 × 4 维度 | 速查 |
| 常见错误 | 7 类 | 防误用 |
| 红旗 | 必做 / 必不做 | 强制规则 |

**结构特点**：**6 步流程 + 4 选项 + 7 错误 + 8 红旗**——完整的"收尾防御体系"。

---

## 四、优点

### 1. 严格 4/3 选项 = "反 LLM 自由发挥"

```text
"严格呈现这 4 个选项"
"不要添加解释"
```

**关键洞察**：
- LLM 默认会**添加"看起来合理"**的选项（如"先做 lint"、"再跑一次 build"）
- 严格 4 选项 = **结构化决策**
- 简化用户决策 = **降低认知负担**

**这是"反 LLM 决策扩散"**——AI 应该**收敛**到固定选项。

### 2. 出处感知的清理

```text
.worktrees/、worktrees/、~/.config/superpowers/worktrees/ → Superpowers 清理
其他 → Harness 清理
```

**优点**：
- 防止"误删别人的 worktree"
- 路径前缀 = 身份标识
- **所有权清晰**

**这是"资源管理伦理"**——只清理自己的。

### 3. 防御性 CWD 管理

```bash
cd "$MAIN_ROOT"  # 必须！
git worktree remove ...
```

**优点**：
- 防御 `git worktree remove` 在 worktree 内部静默失败
- 防御 `git checkout` 在 worktree 内部失败
- 显式切换 = 不依赖隐式行为

**这是"git 反人类行为的硬性防御"**。

### 4. 强 typed confirmation

```text
Type 'discard' to confirm.
```

**优点**：
- 不可逆操作的硬性保护
- `discard` 是显式输入
- 防止误触

### 5. 选项 2 保持 worktree 活着

```text
不要清理 worktree —— 用户需要它活着以便对 PR 反馈进行迭代
```

**关键洞察**：
- 收尾 ≠ 销毁所有资源
- **保持工作流完整性**比"清理干净"重要
- PR 迭代需要 worktree

**这是"工作流优先于清理"**。

### 6. 完整红旗清单

**永远不要 / 始终** 双重清单——7 个"必不做" + 6 个"必做"。

### 7. 8 KB 信息密度极高

- 6 步流程
- 4 个选项的完整操作
- 3 种环境状态
- 7 类错误
- 8 条红旗

**单位字符承载的规则数**——极高。

---

## 五、缺点与风险

### 1. "测试通过"的定义未明

```text
<test command>
```

**问题**：
- 不同项目有不同测试命令
- 测试通过 = 0 失败？还是 0 失败 + 0 warning？
- Flaky test 怎么办？

**缺失**：**"测试通过"的形式化标准**。

### 2. "基础分支"判断不鲁棒

```bash
git merge-base HEAD main 2>/dev/null || git merge-base HEAD master 2>/dev/null
```

**问题**：
- 只尝试 main 和 master
- 现实中有 develop、release 等
- 如果都不存在会失败

**改进**：增加更多常见基础分支或**询问用户**。

### 3. "merge 后必须验证" 在大项目中成本高

**问题**：
- 大项目完整测试 30+ 分钟
- AI 决策流程中"先合并后验证"拖慢节奏

**改进**：分层验证（小合并跑子集，大合并跑全套）。

### 4. 强制 typed confirmation 缺乏"取消"机制

```text
Type 'discard' to confirm.
```

**问题**：
- 如果用户**暂时不想决定**怎么办？
- 必须输入 `discard` 才能取消确认
- 强确认可能让用户**放弃确认流程**

**改进**：明确"取消"选项（输入其他内容 = 取消）。

### 5. PR 模板过于简单

```text
## Summary
<2-3 bullets of what changed>

## Test Plan
- [ ] <verification steps>
```

**问题**：
- 缺少截图、影响、迁移指南
- 与社区标准 PR 模板差异大

**改进**：可配置 PR 模板。

### 6. worktree 路径匹配"太硬"

```text
.worktrees/、worktrees/、~/.config/superpowers/worktrees/
```

**问题**：
- 如果用户用了 `/tmp/my-worktree/` 怎么办？
- 强制路径 = **降低灵活性**

**改进**：询问用户"是否清理"或检测"git worktree list" 的来源。

### 7. 没有"部分完成"的状态

**问题**：
- 如果用户选了"merge"，但 merge 冲突了怎么办？
- 没有"merge 失败 → 回到选项"的流程
- 没有"取消 merge"的指导

**缺失**：**失败恢复路径**。

### 8. 缺少"事后审计"机制

**问题**：
- 操作完成后，没有"操作清单"给用户确认
- 如果选错了选项，需要 git reflog 救场

**改进**：操作前显示"将执行 X / Y / Z" 摘要。

### 9. "不要添加解释"可能与"提供上下文"冲突

**问题**：
- 用户可能不理解"为什么只有 4 个选项"
- 严格不解释 = **不够用户友好**

**缺失**：**首次使用时的解释机制**。

### 10. 没有"工作流分支"的支持

**问题**：
- GitFlow、GitHub Flow、GitLab Flow 是不同的工作流
- 本技能假设"feature branch + PR"模式
- 不支持"release 分支"、"hotfix 分支"等

**改进**：增加工作流配置或识别。

---

## 六、关键洞察

### 洞察 1：这是"工作流收尾"的形式化

**传统开发**：
- 完成开发 = 提交代码
- 集成 = push + PR
- 收尾 = 没有"结构化"概念

**本技能**：
- 完成开发 = 6 步流程
- 集成 = 4 个结构化选项
- 收尾 = 出处感知的资源清理

**关键洞察**：把"开发完成"从**感觉**变成**结构化协议**。

### 洞察 2：4 选项 = "决策收敛"原则

```text
"严格呈现这 4 个选项"（exactly these 4 options）
```

**哲学立场**：
- 不给用户"开放性问题"（"你想做什么？"）
- 不给用户"看起来合理"的额外选项
- **决策结构化 = 认知负担最小化**

**这是"反 LLM 自由发挥"**——AI 不应该"补充"用户没问的事。

### 洞察 3：3 种环境状态 = "上下文感知"

**观察**：开发者工作空间有 3 种本质不同的状态——普通 repo、linked worktree、分离 HEAD。

**本技能的形式化**：
- 每种状态 = **不同的菜单 + 不同的清理逻辑**
- 不用"if-else 嵌套"实现——用"决策表"实现

**这是"环境自适应"原则**——行为应该适配上下文。

### 洞察 4：出处感知 = "资源管理伦理"

```text
.worktrees/、worktrees/、~/.config/superpowers/worktrees/ → Superpowers 清理
其他 → Harness 清理（或保留）
```

**关键洞察**：
- 路径前缀 = 创建者 = 责任方
- **只清理自己的**——不要越界
- 防止"幽灵状态"

**这是"所有权清晰"原则**。

### 洞察 5：防御性 CWD = "git 反人类行为的硬性防御"

```bash
cd "$MAIN_ROOT"  # 必须先做
git worktree remove ...
```

**观察**：`git worktree remove` 在 worktree 内部跑会**静默失败**。

**设计意图**：
- 显式 cd = **不依赖 git 的默认行为**
- 这是"git 内部行为的形式化防御"

**这是"反工具怪异行为"原则**。

### 洞察 6：merge 后必须验证 = "测试真实状态"

```bash
git merge <feature-branch>
<test command>  # 必须在 merge 后跑
```

**关键洞察**：
- feature 测试通过 ≠ merge 后通过
- merge 冲突解决可能引入 bug
- **真实状态测试** = 唯一可靠标准

**这是"测试是最终验证"原则**。

### 洞察 7：丢弃需要 typed confirmation = "防御性确认"

```text
Type 'discard' to confirm.
```

**设计意图**：
- 不可逆操作 = 强确认
- "discard" 是**显式有意识**的输入
- 防止"软确认"误触

**这是"反意外删除"原则**。

### 洞察 8：选项 2 保持 worktree = "工作流优先"

**关键洞察**：
- 收尾 ≠ 销毁所有资源
- **保持工作流完整性**比"清理干净"重要
- PR 迭代需要 worktree 活着

**这是"工作流优先于清理"原则**。

### 洞察 9：与 using-git-worktrees 的对称

| using-git-worktrees | finishing-a-development-branch |
|---------------------|--------------------------------|
| 创建 worktree | 清理 worktree |
| 路径优先级（创建时） | 路径匹配（清理时） |
| 出处感知（"是 Superpowers 创建的"） | 出处感知（"是 Superpowers 清理的"） |

**关键洞察**：两个技能**首尾呼应**——共同构成 worktree 生命周期的完整管理。

### 洞察 10：强制 prune = "自愈式清理"

```bash
git worktree remove ...
git worktree prune  # 自愈
```

**关键洞察**：
- `remove` 可能不完整
- `prune` 主动清理"幽灵注册"
- 这是**自愈式设计**——主动处理潜在不完整

**这是"git 内部状态卫生"原则**。

---

## 七、关联文档

| 文档 | 关系 |
| --- | --- |
| `skills/executing-plans/SKILL.md` | 上游：本技能是其下游 |
| `skills/using-git-worktrees/SKILL.md` | 平行：worktree 创建 ↔ 清理 |
| `skills/writing-plans/SKILL.md` | 上游：计划来源 |
| `docs/superpowers/plans/2026-04-06-worktree-rototill.md` | 上游：worktree 轮作设计 |
| `docs/superpowers/specs/2026-04-06-worktree-rototill-design.md` | 上游：设计文档 |
| `tests/claude-code/test-worktree-native-preference.sh` | 测试：原生工具优先 |

---

## 八、状态评价

| 维度 | 评分 | 说明 |
| --- | --- | --- |
| **结构化** | ⭐⭐⭐⭐⭐ (5/5) | 4/3 选项 + 6 步流程 = 严格决策 |
| **环境自适应** | ⭐⭐⭐⭐⭐ (5/5) | 3 种状态对应不同菜单/清理 |
| **防御性** | ⭐⭐⭐⭐⭐ (5/5) | CWD 检查、prune、typed confirmation |
| **出处感知** | ⭐⭐⭐⭐⭐ (5/5) | 路径前缀 = 所有权 |
| **可操作性** | ⭐⭐⭐⭐ (4/5) | 步骤清晰，但部分决策细节不鲁棒 |
| **错误恢复** | ⭐⭐ (2/5) | 缺失败恢复路径 |
| **工作流支持** | ⭐⭐⭐ (3/5) | 只支持"feature branch + PR"模式 |
| **生态一致性** | ⭐⭐⭐⭐⭐ (5/5) | 与 using-git-worktrees 完美对称 |

**总体评价**：**"工作流收尾"的范本**。3 状态 × 4 选项 × 6 步 × 7 错误 × 8 红旗构成**完整的"开发收尾"防御体系**。**反 LLM 自由发挥、防御性 CWD、出处感知清理、merge 后必须验证**等都是亮点。**错误恢复路径、工作流支持、用户友好性** 3 个方面可以提升。

---

## 九、给读者的启示

### 如果你是 AI 代理
- **完成开发时**：先验证测试，再检测环境，再呈现选项
- **选项必须严格 4/3 个**——不要"自由发挥"
- **merge 后必须再跑测试**——feature 通过 ≠ merge 通过
- **cd 到主 repo 根**再 worktree remove
- **跑 prune** 清理幽灵注册
- **discard 操作必须 typed confirmation**

### 如果你是协作伙伴
- **看到 4/3 选项时**：放心选——AI 不会加额外选项
- **discard 时**：必须输入"discard"——这是安全机制
- **PR 流程中**：AI 会保留 worktree——你可以继续迭代
- **merge 失败时**：AI 会停下让你处理冲突

### 如果你做 AI 工作流编排
- **借鉴**：
  - "严格 4 选项"作为**结构化决策协议**
  - "出处感知"作为**资源管理原则**
  - "merge 后验证"作为**质量门**
  - "prune 自愈"作为**卫生机制**

### 如果你维护 Superpowers
- **改进建议 1**：增加"错误恢复路径"——merge 冲突、命令失败怎么办
- **改进建议 2**：增加"工作流分支"支持（GitFlow、GitHub Flow、GitLab Flow）
- **改进建议 3**：增加"测试通过"的形式化标准（区分 0 失败、0 失败 + 0 warning）
- **改进建议 4**：增加"事后审计"机制——操作前显示摘要
- **改进建议 5**：增加"取消"机制——typed confirmation 中允许"取消"

### 如果你在 AI 团队做工作流规范
- **本技能可作为"AI 收尾规范"的核心条款**
- 强调"结构化选项"——避免 AI 自由发挥
- 强调"merge 后验证"——避免"feature 通过 = 集成通过"
- 强调"出处感知清理"——避免越界

---

## 十、后续工作

按重要性排序：

1. **高优先级**：增加"错误恢复路径"——merge 冲突、命令失败、用户中断怎么办
2. **高优先级**：增加"工作流分支"支持（GitFlow/GitHub Flow/GitLab Flow）
3. **中优先级**：增加"测试通过"的形式化标准
4. **中优先级**：增加"事后审计"机制——操作前显示摘要
5. **中优先级**：增加"取消"机制——typed confirmation 中允许"取消"
6. **低优先级**：增加"部分完成"状态——merge 失败时回到选项
7. **可选**：增加 PR 模板配置（适应团队约定）

---

**总结**：这个 252 行的技能是 **"工作流收尾的形式化范本"**。3 状态 × 4 选项 × 6 步 × 7 错误 × 8 红旗构成**完整的"开发收尾"防御体系**。

最显著的优点是 **"反 LLM 自由发挥"（严格 4 选项）、"环境自适应"（3 状态对应不同行为）、"出处感知清理"（路径 = 所有权）**；最显著的不足是 **"错误恢复路径、工作流支持、用户友好性"** 3 个方面可以提升。

对任何需要 AI 代理完成开发收尾的团队来说，**这是必读必用的核心规范**。其结构化决策、防御性 CWD、出处感知清理的工程美学值得深入学习。
