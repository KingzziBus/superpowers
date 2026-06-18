# README.opencode.md 分析报告

## 基本信息

| 项目 | 内容 |
| --- | --- |
| 文档路径 | `docs/README.opencode.md` |
| 字数规模 | 约 158 行 / 1.7KB（轻量级 README） |
| 文档类型 | 平台适配指南 / 用户安装使用文档 |
| 目标平台 | [OpenCode.ai](https://opencode.ai) |
| 面向读者 | 想在 OpenCode 中使用 Superpowers 的开发者 |
| 文档定位 | OpenCode 平台的"专属安装使用手册" |

---

## 一、文档目的

本文档是 Superpowers 跨平台战略的产物之一——为 **OpenCode** 这个新兴的开源 AI 编程助手（类似 Claude Code）提供一份"零到一"的安装与使用指南。

具体要回答 4 个问题：

1. **怎么装**（在 OpenCode 中如何让 Superpowers 跑起来）
2. **怎么用**（技能如何查找、加载、扩展）
3. **怎么升级**（如何处理 OpenCode 缓存导致的更新问题）
4. **出了事怎么办**（故障排查 + Windows 变通方案）

定位上与 `README.md`（项目主文档）、`.opencode/INSTALL.md`（本地安装说明）形成互补——本文档是 **面向终端用户的简化版安装指南**，重点在于"几行命令搞定"。

---

## 二、核心技术决策

### 决策 1：用 OpenCode 原生 plugin 数组，而不是 hooks 或 symlink

```json
{
  "plugin": ["superpowers@git+https://github.com/obra/superpowers.git"]
}
```

**设计意图**：
- **拥抱平台原生能力**：与 `using-superpowers` 技能提倡的"detect-and-defer"理念一致，优先使用 OpenCode 自己的插件系统
- **避免维护多套机制**：不需要在 OpenCode 上单独开发一套 Superpowers 加载器
- **未来兼容**：OpenCode 升级 plugin API 时，只需要更新这个调用点，技能本身不用改

### 决策 2：双钩子注入（bootstrap + skills 注册）

文档揭示了 OpenCode 插件的核心工作机制（"How It Works" 章节）：

| 钩子 | 作用 |
| --- | --- |
| `experimental.chat.system.transform` | 注入引导上下文，让 AI 每次对话都"意识到"自己加载了 superpowers |
| `config` | 注册技能目录，让 OpenCode 自动发现所有 superpowers 技能 |

**精妙之处**：这两个钩子都符合 OpenCode 官方 API，**完全没有侵入式 hack**。这也回答了之前对话中用户问的"对 skill 的处理机制"——OpenCode 走的是"系统级钩子注入 + 自动发现"路线。

### 决策 3：工具自动映射（Tool Mapping）

```text
TodoWrite       → todowrite
Task (subagent) → @mention
Skill           → skill
文件操作         → OpenCode 原生
```

**这是文档中"含金量最高"的部分**。它揭示了一个跨 AI 编程工具的关键技术：**抽象工具名 + 平台特定实现**。

含义：
- 技能作者写一次，可以"自动"在多个平台运行
- 但前提是平台要提供"等价物"——OpenCode 的 `@mention` 系统必须能完成 Claude Code 的 `Task` 子代理功能
- 这也暗示了"工具映射表"是 Superpowers 跨平台策略的核心资产，未来扩展 Cursor/Gemini/Copilot 时都需要这个表

### 决策 4：明确的优先级规则

```text
项目技能 > 个人技能 > Superpowers 技能
```

**为什么重要**：这条规则是整个技能系统的"宪法"——它防止了：
- Superpowers 的"通才技能"覆盖了用户在项目中定义的"专才技能"
- 用户的个人偏好被通用插件压制

这与 `using-superpowers` 技能中的"工具能力边界"哲学一脉相承。

### 决策 5：git 备份的 npm spec + 版本 pin

```json
"superpowers@git+https://github.com/obra/superpowers.git#v5.0.3"
```

**设计意图**：
- 默认走 `main` 分支（持续获得最新功能）
- 想要稳定可用 `#tag` 锁定版本
- 这与 npm 生态的 `git+https#semver` 模式一致，零学习成本

---

## 三、文件结构

文档结构极简，**全部 158 行没有冗余**，每个章节都对应一个明确问题：

| 章节 | 行数 | 解决的问题 |
| --- | --- | --- |
| 安装 | 23 | 怎么装 |
| → Migrating from symlink | 14 | 旧用户如何迁移 |
| 使用 | 42 | 怎么用 |
| → 查找/加载/个人/项目技能 | — | 4 个子场景 |
| 更新 | 12 | 怎么升级 |
| 工作原理 | 14 | 内部机制（可选读） |
| → Tool Mapping | 9 | 跨平台兼容细节 |
| 故障排查 | 36 | 出问题怎么办 |
| → 4 类常见问题 | — | 加载/Windows/技能/引导 |
| 获取帮助 | 5 | 找谁求助 |

**信息密度极高**：没有前言、没有"为什么需要 Superpowers"这种营销文案，直接进入"如何做"。

---

## 四、优点

### 1. 零环境依赖
- 唯一前置：OpenCode 自身 + 网络
- 安装就是改一个 JSON 数组项，对非开发者也友好

### 2. 平滑迁移路径
- 专门有"Migrating from symlink"小节照顾早期用户
- 列出具体的 `rm` 命令而不是"请删除旧文件"

### 3. 主动暴露 Windows 痛点
- 不是"理论支持 Windows"，而是直接承认"某些 Windows 版 OpenCode 有上游 bug"
- 提供了"用 npm 安装本地包"的兜底方案
- 这一点非常工程化——很多文档只会写"理论上能跑"

### 4. 工具映射表
- 这是文档的"技术资产"——一旦维护好这个表，新增平台就只是"加一行映射"的工作

### 5. 优先级规则
- 简单到只有 3 行（项目>个人>Superpowers），但解决了"扩展性与可控性"的核心矛盾

---

## 五、缺点与风险

### 1. "How It Works" 信息不足
- 仅说"通过 `experimental.chat.system.transform` 钩子注入"，但没解释：
  - 这个钩子在什么时机触发？
  - 注入的"superpowers 意识"具体长什么样？
  - 是否会被 OpenCode 自己的 system prompt 覆盖？
- 想要深挖的开发者需要去看 `.opencode/plugins/` 下的源码

### 2. Tool Mapping 表不完整
- 只列了 4 个映射，但 Superpowers 实际用到的工具远不止这些
- 没有说明"如果某个工具在 OpenCode 没有等价物怎么办"
  - 是降级？
  - 是报错？
  - 还是悄悄跳过？

### 3. Windows 变通方案是"绕过"而非"修复"
- 推荐的 `npm install + 本地路径` 方案，本质上是"绕开 OpenCode 自己的 git 解析器"
- 这会让用户失去"通过 OpenCode 一键升级"的能力
- 文档没说明"用本地路径后怎么升级 Superpowers"

### 4. 缓存问题描述模糊
- "某些 OpenCode 和 Bun 版本会在 lockfile 或 cache 中固定"——但没说：
  - 是哪些版本？
  - 怎么手动清缓存？
  - 清缓存会不会影响其他插件？

### 5. 缺少对 OpenCode 自身工作流的介绍
- 对从没接触过 OpenCode 的用户，文档假设他们已经会用 OpenCode 的 `skill` 工具
- 实际上 OpenCode 的很多 API（如 `@mention`）都有自己独特的约定

### 6. 没有"卸载"章节
- 只有"从 symlink 迁移"的步骤，但没有"完全卸载 Superpowers for OpenCode"的方法
- 这在"试用了想退出"的场景下会成为障碍

---

## 六、关键洞察

### 洞察 1：Superpowers 的"多平台适配"是渐进的、平台无关技能 + 平台相关插件

```
┌──────────────────────────────────┐
│  skills/                          │  (Claude Code 写的)
│  - brainstorming/SKILL.md        │  → 引用 TodoWrite, Task, Skill
│  - ...                            │  → 假设 Claude Code 工具集
└──────────────────────────────────┘
                ↓ 加载
┌──────────────────────────────────┐
│  .opencode/plugins/               │  (OpenCode 适配层)
│  - 将 TodoWrite 翻译为 todowrite  │
│  - 将 Task 翻译为 @mention        │
│  - ...                            │
└──────────────────────────────────┘
                ↓ 注册
┌──────────────────────────────────┐
│  OpenCode 平台                    │
│  - skill tool                     │
│  - experimental.chat.system.transform │
└──────────────────────────────────┘
```

**启示**：要新增一个平台（假设未来出"Superpowers for Zed"），工作量是：
- 写一个 `<platform>/plugins/` 适配器
- 维护一个工具映射表
- **不需要改任何 skill 文件**

### 洞察 2：`experimental.chat.system.transform` 是"开挂点"

这个钩子的命名带 `experimental`——意味着 OpenCode 自己也认为这个 API 不稳定。但 Superpowers 选择了重度依赖它（bootstrap 上下文）。

**风险**：OpenCode 一旦把这个 API 改掉，Superpowers 就会断。
**应对**：文档末尾的"故障排查 → Bootstrap not appearing"已经预警了这一点。

### 洞察 3：Windows 兜底方案揭示了 Bun 生态的早期阶段

文档中提到"Bun not finding git.exe even when it works in a normal terminal"——这是 Bun 作为新兴 runtime 的典型早期问题。Superpowers 不得不提供"绕开 Bun"的方法。

这说明：**Superpowers 并不完全信任 OpenCode 的安装链路**，它更愿意让用户"自己 npm 装好，然后让 OpenCode 引用"。

### 洞察 4：版本 pin 策略保守

```json
"superpowers@git+https://github.com/obra/superpowers.git#v5.0.3"
```

默认走 `main` 是激进的（持续拿到最新功能），但提供 `#tag` 是保守的（生产可锁版本）。

这种"激进默认 + 保守兜底"是优秀工具的标配。

---

## 七、关联文档

| 文档 | 关系 |
| --- | --- |
| `.opencode/INSTALL.md` | 更详细的本地开发安装说明（面向贡献者） |
| `docs/plans/2025-11-22-opencode-support-design.md` | OpenCode 支持的设计文档（解释为什么这么设计） |
| `docs/plans/2025-11-22-opencode-support-implementation.md` | 实施记录（记录实际怎么落地的） |
| `skills/using-superpowers/SKILL.md` | "detect-and-defer"理念的源头 |
| `README.md` | 项目主文档，提供总体介绍 |

---

## 八、状态评价

| 维度 | 评分 | 说明 |
| --- | --- | --- |
| **完整性** | ⭐⭐⭐⭐ (4/5) | 覆盖安装/使用/升级/排查，但缺卸载和深度的"工作原理" |
| **准确性** | ⭐⭐⭐⭐⭐ (5/5) | 描述与 OpenCode 实际 API 一致 |
| **可操作性** | ⭐⭐⭐⭐⭐ (5/5) | 复制粘贴即可跑通 |
| **可读性** | ⭐⭐⭐⭐⭐ (5/5) | 极简、信息密度高、章节分明 |
| **可维护性** | ⭐⭐⭐⭐ (4/5) | Tool Mapping 表需要随 OpenCode 演化而更新 |
| **多平台一致性** | ⭐⭐⭐ (3/5) | 与 Claude Code/Codex 版本的"安装说明"风格不统一（这个是 OpenCode 特定的） |

**总体评价**：作为平台专属 README，**堪称范本**——短小精悍、信息密度高、对 Windows 痛点诚实。但作为跨平台战略的一部分，**缺少对总体架构的指引**（如"如何为新平台写适配器"）。

---

## 九、给读者的启示

### 如果你是 OpenCode 用户
- **必读**：安装、查找/加载技能、故障排查
- **选读**：迁移步骤（仅当你用过 symlink 安装时）
- **了解即可**：Tool Mapping 表（你不需要懂，但知道它存在会让你更信任这个工具）

### 如果你是其他平台的开发者（想给 Cursor/Gemini/Copilot 写适配器）
- **重点研究**：Tool Mapping 表——这是 Superpowers 跨平台的核心资产
- **模仿模式**：双钩子注入（bootstrap + skills 注册）
- **避免的坑**：重度依赖某个平台带 `experimental` 前缀的 API

### 如果你是 Superpowers 维护者
- **改进建议 1**：补充"完全卸载"章节
- **改进建议 2**：详细说明 `experimental.chat.system.transform` 的内部行为（可以加一节"内部机制"指向源码）
- **改进建议 3**：把 Windows 变通方案独立成"Windows 安装专项"文档

### 如果你在做类似的跨平台工具
- **借鉴**：
  - 平台无关核心 + 平台特定适配器
  - 默认激进（拉 main）+ 兜底保守（pin tag）
  - 对平台 bug 诚实（"这是上游问题，我们提供变通方案"）
- **警惕**：依赖 `experimental` API 的长期风险

---

## 十、后续工作

按重要性排序：

1. **高优先级**：补充"完全卸载 Superpowers"章节，覆盖两种安装方式（git spec 和本地路径）
2. **中优先级**：维护 Tool Mapping 表的版本控制——随着 OpenCode 演化，可能需要新增映射项
3. **中优先级**：增加"性能与缓存最佳实践"小节——目前只说"清缓存"，没说什么时候清
4. **低优先级**：考虑用一段简短的"OpenCode 是什么"小节帮助 OpenCode 新用户上手
5. **可选**：把"工作原理"小节展开成单独的架构文档（`docs/architecture/opencode-adapter.md`）

---

**总结**：这份文档用 158 行完成了"让一个新平台用户从 0 到能用 Superpowers"的任务，是 Superpowers 跨平台战略的"前线门面"。它的设计哲学（轻量级 README、零假设前提、诚实的故障排查）值得所有"多平台工具"的作者学习。
