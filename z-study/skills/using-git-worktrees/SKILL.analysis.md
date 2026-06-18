# using-git-worktrees/SKILL.md 分析报告

## 基本信息

| 项目 | 内容 |
| --- | --- |
| 文档路径 | `skills/using-git-worktrees/SKILL.md` |
| 字数规模 | 约 216 行 / 7.0KB |
| 文档类型 | 技能定义（SKILL）/ 工作空间管理 |
| 主题 | 跨 AI 平台的统一 worktree 管理 |
| 核心定位 | "先检测 → 后用原生 → 最后 git 回退" |
| 涉及技术 | `git worktree`、`git rev-parse --git-dir/--git-common-dir`、submodule 防御 |
| 面向读者 | AI 代理（自我管理时） + 跨平台开发者 |

---

## 一、文档目的

解决一个跨平台 AI 编程工具的**根本性挑战**：**worktree 管理在每个平台上不一样**。

具体目标：
1. **统一抽象**：提供"先检测 → 后用原生 → 最后回退"的统一流程
2. **Submodule 防御**：避免误判"已经在 worktree"
3. **目录优先级**：明确项目级 vs 全局 vs 默认的选择规则
4. **安全验证**：避免 worktree 内容被 git 跟踪

定位上属于 **"跨平台 AI 工作流基础设施"**——和 `windows/polyglot-hooks.md` 类似，都是**让 Superpowers 跨平台工作的关键基础设施**。

---

## 二、核心技术决策

### 决策 1：先检测，后创建

```bash
GIT_DIR=$(cd "$(git rev-parse --git-dir)" 2>/dev/null && pwd -P)
GIT_COMMON=$(cd "$(git rev-parse --git-common-dir)" 2>/dev/null && pwd -P)
```

**关键洞察**：
- `--git-dir` 是当前工作空间的 `.git` 目录
- `--git-common-dir` 是**主仓库的 `.git` 目录**
- **两者相同 → 普通 repo**；**不同 → 已在 worktree**

**精妙之处**：用 git 自己的原语（不是文件名约定）来识别状态——**鲁棒**。

### 决策 2：Submodule 防御

```bash
# Submodule 也有 GIT_DIR != GIT_COMMON！
git rev-parse --show-superproject-working-tree 2>/dev/null
```

**为什么这是必须**：
- git submodule 也有独立的 `.git` 目录
- 如果不防御，会把"在 submodule"误判为"已在 worktree"
- 后果：永远跳过 worktree 创建，所有改动进入 submodule

**这是反 LLM 错误模式的硬性防御**。

### 决策 3：原生工具优先（detect-and-defer 哲学）

```text
1a. 原生 Worktree 工具（首选）
1b. Git Worktree 回退
```

**核心哲学**：**detect-and-defer**——如果平台已经有工具，**让它做**。

**为什么**：
- 原生工具知道"哪里放、怎么命名、怎么清理"——平台特有
- 用 `git worktree add` 创建的 worktree 是**"幽灵状态"**——harness 看不到
- 后果：清理困难、状态不一致

**这是 Superpowers 一贯的"不与 harness 作对"立场**。

### 决策 4：目录优先级明确

```text
1. 用户声明的偏好
2. 项目级 .worktrees/ 存在
3. 项目级 worktrees/ 存在
4. 全局 ~/.config/superpowers/worktrees/<project>
5. 默认项目级 .worktrees/
```

**设计意图**：
- 显式偏好 > 隐式状态——尊重用户
- 项目级 > 全局——避免污染用户家目录
- 隐藏目录 > 可见目录——减少视觉污染
- 全局路径只为向后兼容旧版

**这是一份"配置决策树"**——明确每个分支的判断标准。

### 决策 5：必须验证 .gitignore

```bash
git check-ignore -q .worktrees 2>/dev/null || git check-ignore -q worktrees 2>/dev/null
```

**如果 NOT ignored**：必须先 add + commit .gitignore。

**为什么至关重要**：
- worktree 是**独立的工作空间**——不应该被 commit
- 如果忘记 ignore，整个 worktree 内容会被 git 跟踪
- 后果：项目仓库里出现几千个无关文件

**这是反"灾难性后果"的硬性防御**。

### 决策 6：沙箱回退（防御性失败模式）

```text
如果 `git worktree add` 因权限错误失败（沙箱拒绝），告诉用户沙箱阻止了 worktree 创建，
而你改为在当前目录工作。
```

**设计意图**：
- 不要默默失败——要**明确告知**
- 沙箱拒绝 ≠ 不该工作——回退到原地
- 跑设置和基线测试——保持工作流完整性

**这是 "graceful degradation"（优雅降级）**的范例。

### 决策 7：基线测试必须在 worktree 创建后跑

```text
Worktree ready at <full-path>
Tests passing (<N> tests, 0 failures)
Ready to implement <feature-name>
```

**为什么**：
- 不能假设 worktree 创建了 = 工作空间可用
- 必须验证**起点干净**——否则后面"我的修改导致 bug"vs"worktree 本来就有 bug"分不清

**这是"防止污染新工作空间"的硬性要求**。

### 决策 8：自动检测项目类型

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

**设计意图**：
- 4 种主流语言（Node/Rust/Python/Go）的**自动检测**
- 减少用户配置负担
- 跨语言项目支持

**缺失**：Java/Maven、C#/.NET、Ruby、Elixir 等。

---

## 三、文件结构

| 章节 | 内容 | 工程价值 |
| --- | --- | --- |
| 概述 | 核心原则 + 宣告 | 立场 |
| Step 0 | 现有隔离检测（含 submodule 防御） | 防御 |
| Step 1a | 原生工具路径 | 首选 |
| Step 1b | Git 回退（含目录优先级 + 安全验证） | 兜底 |
| Step 3 | 项目设置 | 自动检测 |
| Step 4 | 基线测试 | 质量门 |
| 快速参考 | 12 种情况 → 动作 | 速查 |
| 常见错误 | 5 类反模式 | 防误用 |
| 红旗 | 必做 / 必不做 | 强制规则 |

**结构特点**：**Step 0 → 1a → 1b → 3 → 4** 是**严格顺序**——不能跳过。

---

## 四、优点

### 1. detect-and-defer 哲学的极致体现

```text
1a. 原生工具（如果可用）
1b. Git 回退（如果 1a 不可用）
```

**优点**：
- 不重复造轮子
- 与平台生态一致
- 用户体验最好

**这是 Superpowers 的核心哲学**——`using-superpowers` 母技能也强调这个。

### 2. Submodule 防御是技术深度的体现

```bash
git rev-parse --show-superproject-working-tree 2>/dev/null
```

**优点**：
- 戳穿常见误判（submodule 看起来像 worktree）
- 用 git 原语解决，不是 hack
- 防止"永远跳过 worktree 创建"的灾难

**这是只有真正踩过坑的人才会写的防御**。

### 3. 目录优先级明确

5 级优先级 + "用户偏好 > 隐式状态" 的元规则——**完整的配置决策树**。

### 4. .gitignore 验证是"防止灾难"

```text
防止意外将 worktree 内容提交到仓库
```

**优点**：
- 单次失败 = 整个项目污染
- 显式验证 = 防御灾难
- "必须 add + commit 后再继续" = 不允许默默工作

### 5. 沙箱回退是 "graceful degradation" 范例

```text
告诉用户沙箱阻止了 worktree 创建，而你改为在当前目录工作
```

**优点**：
- 不默默失败
- 不拒绝继续工作
- 明确告知用户**真实状态**

### 6. 完整红旗清单

**永远不要 / 始终** 双重清单——明确边界。

**#1 错误**特意标出："**如果有原生工具就用它**"——这是最高优先级规则。

### 7. 与其他技能完美协作

- 上游：`writing-plans`（提供计划）
- 平行：`executing-plans`（开始前调用本技能）
- 下游：`finishing-a-development-branch`（结束时清理 worktree）

---

## 五、缺点与风险

### 1. `--git-dir` vs `--git-common-dir` 的区分在 LLM 认知中可能模糊

**问题**：
- LLM 容易把这两个搞混
- 实际差异微妙，普通开发者也容易错

**改进方向**：
- 提供一个 wrapper 脚本（shell 函数）封装判断
- 给出更详细的解释

### 2. 目录优先级对"两者都存在"的处理不够

```text
两者都存在 → 用 .worktrees/
```

**问题**：
- 为什么 `.worktrees` 优先？没说
- 如果用户已经用 `worktrees` 习惯了呢？

**缺失**：选择的**理由**说明。

### 3. 项目类型自动检测不完整

只支持 Node/Rust/Python/Go——**缺 Java、C#、Ruby、Elixir、Rust workspace**等。

**改进**：增加"其他项目类型"分支或明确的"未检测到项目类型"行为。

### 4. 沙箱回退时，AI "告诉用户"的方式不明确

```text
告诉用户沙箱阻止了 worktree 创建
```

**问题**：
- 怎么"告诉"？是 console 输出？chat 消息？
- 用户可能根本看不到

**改进**：明确"用什么机制通知"。

### 5. 缺少"worktree 命名冲突"的处理

**场景**：
- 已有同名 branch 的 worktree
- 已有同名目录

**缺失**：冲突检测和解决。

### 6. 缺少"worktree 状态报告"格式

```text
Worktree ready at <full-path>
Tests passing (<N> tests, 0 failures)
Ready to implement <feature-name>
```

**只列了"成功"格式**——**失败/降级**的报告格式缺失。

### 7. Step 3 跳过 "Step 2"

**奇怪**：文档说 "Step 1a" 然后 "Step 1b"，但中间没有 Step 2。后面直接是 "Step 3"。

**问题**：
- 是 typo？
- 还是有 Step 2 被故意省略？
- 读者会困惑

**改进**：明确解释（或者真的补上 Step 2，或者重命名）。

### 8. 全局目录路径硬编码

```text
~/.config/superpowers/worktrees/$project
```

**问题**：
- Windows 下路径是 `~/.config/...`？
- 用户在多台机器上用，路径可能不一致

**改进**：用 cross-platform 路径处理。

### 9. 没有"worktree 清理"的指导

**缺失**：
- 创建了 worktree 后，怎么知道要清理？
- `finishing-a-development-branch` 涵盖，但本技能没提示

### 10. "必须 cd 到 worktree" 的提示不够

```bash
git worktree add "$path" -b "$BRANCH_NAME"
cd "$path"
```

**问题**：
- `cd` 是 shell 局部——不会影响调用者
- AI 代理可能"以为"自己 cd 了，实际没有

**改进**：明确说"AI 代理必须用绝对路径继续操作"。

---

## 六、关键洞察

### 洞察 1：detect-and-defer 是 Superpowers 的核心哲学

```text
先检测 → 后用原生 → 最后回退
```

**哲学根源**：
- 与 `using-superpowers` 母技能一致
- 与 `windows/polyglot-hooks.md` 一致
- 与 `README.opencode.md` 的"双钩子注入"一致

**核心立场**：**永远不与 harness 作对**——如果平台有工具，让它做。

**这是 Superpowers 的"非侵入性"价值观**。

### 洞察 2：用 git 原语而非文件名约定

```bash
git rev-parse --git-dir
git rev-parse --git-common-dir
git rev-parse --show-superproject-working-tree
```

**优点**：
- 不依赖 `.worktrees` 目录是否存在
- 不依赖任何文件名约定
- 跨平台、跨 git 版本都鲁棒

**这是"工程级鲁棒性"的范例**——用工具的**原语**而不是**约定**。

### 洞察 3：Submodule 防御是"真实踩坑"的证据

```bash
git rev-parse --show-superproject-working-tree 2>/dev/null
```

**洞察**：
- 这条防御**不是凭空设计**——是真实遇到过的 bug
- 没踩过的人不会想到 submodule 也有 `GIT_DIR != GIT_COMMON`

**这是"防御性编程"的成熟标志**。

### 洞察 4：.gitignore 验证是"防灾难"

```text
如果不验证：worktree 内容被 commit → 项目污染 → 难以清理
```

**设计意图**：
- 一次失败 = 大量清理工作
- 显式验证 = 一次性成本
- **"花 1 秒验证，省 1 小时清理"**

**这是"低成本防御"原则**。

### 洞察 5：原生工具 vs git 回退 = "工具链哲学"

```text
原生工具：平台做得好，让它做
Git 回退：平台没工具，自己做
```

**哲学立场**：
- 不试图"统一所有平台的 worktree 体验"
- 而是"用平台提供的，不用平台没提供的"
- **"借用" > "重写"**

**这是 Unix 哲学 "use existing tools" 的体现**。

### 洞察 6：基线测试是"防污染"

```text
worktree ready at <path>
Tests passing (34/34 pass)
```

**为什么**：
- 不能假设 worktree 创建 = 工作空间可用
- 必须验证**起点干净**
- 防止"我的修改导致 bug"vs"worktree 本来就有 bug"的混淆

**这是"信任但验证"原则**。

### 洞察 7：Step 编号不连续（"Step 0" + "Step 1a/1b" + "Step 3"）暗示演进史

**观察**：
- 现在的"Step 0"曾经可能是后加的
- "Step 1a"和"Step 1b"暗示曾经只有 "Step 1"
- "Step 3"暗示曾经有 "Step 2" 被删除

**启示**：技术文档的"编号跳跃"是**演进痕迹**——可以接受，但应该注释。

---

## 七、关联文档

| 文档 | 关系 |
| --- | --- |
| `docs/superpowers/plans/2026-04-06-worktree-rototill.md` | 上游：worktree 轮作的设计起源 |
| `docs/superpowers/specs/2026-04-06-worktree-rototill-design.md` | 上游：设计文档 |
| `skills/finishing-a-development-branch/SKILL.md` | 下游：worktree 清理逻辑 |
| `skills/executing-plans/SKILL.md` | 平行：执行计划前调用本技能 |
| `skills/subagent-driven-development/SKILL.md` | 平行：subagent 工作流中也会用到 worktree |
| `tests/claude-code/test-worktree-native-preference.sh` | 测试：原生工具优先 |
| `docs/windows/polyglot-hooks.md` | 平行：跨平台哲学一致 |
| `docs/README.opencode.md` | 平行：detect-and-defer 哲学一致 |

---

## 八、状态评价

| 维度 | 评分 | 说明 |
| --- | --- | --- |
| **跨平台兼容性** | ⭐⭐⭐⭐⭐ (5/5) | detect-and-defer 完美处理多平台 |
| **防御性** | ⭐⭐⭐⭐⭐ (5/5) | submodule 防御、.gitignore 验证、沙箱回退 |
| **决策形式化** | ⭐⭐⭐⭐⭐ (5/5) | 目录优先级、12 种情况速查 |
| **可操作性** | ⭐⭐⭐⭐ (4/5) | 流程清晰，但"cd 后继续操作"提示不够 |
| **项目类型覆盖** | ⭐⭐⭐ (3/5) | 只支持 4 种主流语言 |
| **错误恢复** | ⭐⭐⭐⭐ (4/5) | 沙箱回退好，但 worktree 命名冲突缺处理 |
| **文档一致性** | ⭐⭐⭐ (3/5) | Step 编号不连续（Step 0/1a/1b/3/4） |
| **生态一致性** | ⭐⭐⭐⭐⭐ (5/5) | 与其他技能完美协作 |

**总体评价**：**跨平台 worktree 管理的范本**。detect-and-defer 哲学贯穿始终，submodule 防御、.gitignore 验证、沙箱回退构成**完整的防御体系**。**项目类型覆盖、文档一致性、错误恢复** 3 个方面可以提升。

---

## 九、给读者的启示

### 如果你是 AI 代理
- **执行计划前**：跑 Step 0 检测现有隔离
- **创建 worktree 前**：跑 .gitignore 验证
- **优先用原生工具**（`EnterWorktree`、`/worktree` 等）
- **基线测试必须跑**——不能跳过
- **沙箱拒绝时**：明确告知用户，回退到原地

### 如果你维护多平台 AI 工具
- **借鉴**：
  - "先检测 → 后用原生 → 最后回退"的统一流程
  - "git 原语优先于文件名约定"
  - "防御性检测"（submodule）
  - ".gitignore 验证"（防污染）
  - "graceful degradation"（沙箱回退）

### 如果你做 worktree 相关工具
- **本技能展示了"worktree 管理的 5 大核心问题"**：
  1. 如何检测当前状态
  2. 如何选择创建机制
  3. 如何选择目录
  4. 如何避免污染
  5. 如何处理失败

### 如果你维护 Superpowers
- **改进建议 1**：补充 Step 2（或者解释为什么没 Step 2）
- **改进建议 2**：扩展项目类型支持（Java、C#、Ruby、Elixir）
- **改进建议 3**：明确"沙箱回退时用什么机制通知用户"
- **改进建议 4**：增加"worktree 命名冲突"处理
- **改进建议 5**：在 worktree 创建后明确"AI 必须用绝对路径"

### 如果你在 AI 团队做工作流规范
- **本技能可作为"AI 工作空间管理"的核心条款**
- 强调"原生工具优先"——避免 AI 重复造轮子
- 强调"基线测试"——防止"修改污染"
- 强调".gitignore 验证"——防止灾难

---

## 十、后续工作

按重要性排序：

1. **高优先级**：补充 Step 2（或者解释为什么没 Step 2）
2. **高优先级**：扩展项目类型支持（Java/CSharp/Ruby/Elixir）
3. **中优先级**：明确"沙箱回退时用什么机制通知用户"
4. **中优先级**：增加"worktree 命名冲突"处理
5. **中优先级**：增加 Windows 路径处理的指导（`~/.config/` 在 Windows 的位置）
6. **低优先级**：把"原生工具优先"做成一个 API check（如果平台没原生工具就给出提示）
7. **可选**：增加"worktree 状态查询"功能——AI 主动询问"你希望我创建 worktree 吗？"

---

**总结**：这个 216 行的技能是 **"跨平台 worktree 管理的工程范本"**。detect-and-defer 哲学贯穿始终，submodule 防御、.gitignore 验证、沙箱回退构成**完整的防御体系**。**与 Superpowers 其他技能（尤其是 polyglot-hooks、opencode README）形成统一的"跨平台、非侵入性"哲学**。

对任何需要在 AI 编程工具中管理 worktree 的人来说，**这是必读必用的规范**。其工程深度（用 git 原语）和防御思维（submodule 防御、.gitignore 验证）都值得深入学习。
