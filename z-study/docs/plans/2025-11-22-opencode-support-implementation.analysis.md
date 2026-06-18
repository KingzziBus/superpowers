# 文档分析报告：OpenCode 支持实施计划

> **原文档**：[2025-11-22-opencode-support-implementation.md](file://e:/WorkStation/MyProject/superpowers/docs/plans/2025-11-22-opencode-support-implementation.md)
> **翻译版本**：[2025-11-22-opencode-support-implementation.zh-CN.md](file://e:/WorkStation/MyProject/superpowers/z-study/docs/plans/2025-11-22-opencode-support-implementation.zh-CN.md)
> **文档类型**：实施计划（Implementation Plan）
> **分析日期**：2026-06-03

---

## 📋 一、文档基本信息

| 字段 | 内容 |
|------|------|
| 标题 | OpenCode Support Implementation Plan |
| 日期 | 2025-11-22 |
| 作者 | Jesse（推断） |
| 状态 | 与配套设计文档一起发布 |
| 字数 | ~1096 行 / ~27KB |
| 类别 | plans（项目开发计划） |
| 任务数 | 5 阶段 18 任务 |

## 🎯 二、文档目的

将 [设计文档](file://e:/WorkStation/MyProject/superpowers/docs/plans/2025-11-22-opencode-support-design.md) 中的架构方案，**任务接任务地**拆解为可执行的 TDD（测试驱动开发）实施步骤。本文档是**面向 AI Agent** 的执行手册。

## 🏗️ 三、实施结构（5 阶段 18 任务）

### 阶段 1：创建共享核心模块（任务 1-4）

| 任务 | 目标 | 关键文件 |
|------|------|---------|
| 1 | 提取 `extractFrontmatter` 函数 | `lib/skills-core.js` |
| 2 | 提取 `findSkillsInDir` 函数 | `lib/skills-core.js` |
| 3 | 提取 `resolveSkillPath` 函数（支持覆盖） | `lib/skills-core.js` |
| 4 | 提取 `checkForUpdates` 函数 | `lib/skills-core.js` |

### 阶段 2：重构 Codex 使用共享核心（任务 5-8）

每个任务对应替换一个函数（`extractFrontmatter`、`findSkillsInDir`、`checkForUpdates`）并验证 Codex 仍工作。

### 阶段 3：构建 OpenCode 插件（任务 9-12）

| 任务 | 目标 |
|------|------|
| 9 | 创建插件目录结构与骨架 |
| 10 | 实现 `use_skill` 工具 |
| 11 | 实现 `find_skills` 工具 |
| 12 | 实现 `session.started` 钩子 |

### 阶段 4：文档（任务 13-15）

- 任务 13：创建 `.opencode/INSTALL.md`
- 任务 14：更新主 README
- 任务 15：更新 RELEASE-NOTES

### 阶段 5：最终验证（任务 16-18）

- 任务 16：测试 Codex 仍工作
- 任务 17：验证文件结构
- 任务 18：最终提交与总结

## 📊 四、文档结构特点

### 4.1 任务粒度

每个 Task 包含：
- **Files**：明确指出创建/修改/参考的文件（含行号）
- **步骤 1+**：具体操作（含完整代码/命令）
- **验证步骤**：`node -c`、`ls`、实际命令执行
- **提交步骤**：明确的 commit message

**这是"机器可执行"的实施手册**——AI Agent 拿到文档就能照着做。

### 4.2 TDD 原则的体现

虽然文档没有显式说"TDD"，但每个任务都有：
1. **写代码**（步骤 1）
2. **验证语法**（步骤 2，如 `node -c`）
3. **验证功能**（步骤 3，真实命令）
4. **原子提交**（步骤 4，commit message）

这是 **小步提交、频繁验证** 的工程实践。

### 4.3 渐进式重构

阶段 2 重构 Codex 时：
- 任务 5：先加 import 语句（不删除原代码）
- 任务 6：删除并替换 `extractFrontmatter`（每次只换一个）
- 任务 7-8：同理

**保证了重构的可逆性**——如果新版本失败，可以回滚到上一个 commit。

## 🔍 五、文档优缺点分析

### ✅ 优点

1. **任务极其具体**：行号定位、完整代码、可复制粘贴的命令
2. **TDD 思维贯穿**：每步都有验证，CI/CD 友好
3. **提交信息规范化**：每个 commit 都是原子操作，便于 code review
4. **文档即代码**：INSTALL.md 直接以 markdown 形式包含在任务里
5. **配套手动测试指南**：第 1074-1084 行提供了 OpenCode 实测步骤
6. **成功标准明确**：第 1086-1095 行 8 条可勾选的成功标准

### ⚠️ 缺点 / 风险点

1. **缺少自动化测试**：所有验证都是手动命令，无单元测试设计
2. **没有"如何回滚"**：如果发现设计错误怎么办？缺少回滚预案
3. **没有"如何调试"**：插件加载失败时如何排查？缺乏调试指南
4. **依赖项未明确**：OpenCode 插件 API 的版本、zod 库的版本约束未说明
5. **性能基准缺失**：没有"启动时延应 < X 秒"等可量化指标
6. **缺少 CI 集成**：如何确保 PR 不会破坏 Codex + OpenCode 双平台？

## 💡 六、关键洞察

### 洞察 1：这是"执行机器可读计划"的范式

文档开头明确写：
> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

这是 **AI Agent 作为执行者**的标志性设计：文档本身就是程序。

**对比传统软件工程**：传统实施计划是"给人看的"；这里的实施计划是"给 AI 看的"，因此粒度更细、指令更明确。

### 洞察 2：共享核心 = 重构的真正价值

任务 1-4 看似在做"提取代码"，但本质是**为未来所有平台适配铺路**。

```
未来添加 Cursor 支持时：
✓ 直接使用 lib/skills-core.js 的 4 个函数
✓ 只需写 Cursor 平台的薄封装
✓ 无需重新实现技能发现/解析逻辑
```

**这是架构演进的复利效应**——每个新平台都让项目更值钱。

### 洞察 3：Tool Mapping 是跨平台 AI 的"巴别塔"

任务 12 中的 `toolMapping` 字符串是核心创新点：
```javascript
const toolMapping = `
**Tool Mapping for OpenCode:**
- TodoWrite → update_plan
- Task with subagents → @mention
- Skill tool → use_skill
- Read, Write, Edit, Bash → Your native equivalents
```

这本质是**为不同 AI Agent 提供"语言翻译表"**。当 Superpowers 技能（用 Claude Code 工具名）被 OpenCode 加载时，这个映射表告诉 OpenCode "用你的本地工具代替"。

**这种"工具翻译层"后来成为 [using-superpowers/SKILL.md](file://e:/WorkStation/MyProject/superpowers/skills/using-superpowers/SKILL.md) 中"Platform Adaptation"段落的标准模式。**

### 洞察 4：手动测试 vs 自动化测试的权衡

文档第 1074 行写"This step requires OpenCode to be installed"。

**这暴露了一个工程现实**：跨 AI 平台测试很难完全自动化，因为：
- OpenCode 的真实环境复杂
- 启动会话涉及 GUI、Token 计费、用户交互
- CI 环境难以复现

**所以选择了"自动化 + 手动"混合测试策略**：CI 验证语法和函数；开发者手动验证端到端。

## 🔗 七、与其他文档的关联

| 关联文档 | 关联方式 |
|---------|---------|
| [opencode-support-design.md](file://e:/WorkStation/MyProject/superpowers/docs/plans/2025-11-22-opencode-support-design.md) | **配套设计文档**，本文档的实施依据 |
| [.opencode/plugins/superpowers.js](file://e:/WorkStation/MyProject/superpowers/.opencode/plugins/superpowers.js) | **实际产物**，任务 9-12 的最终代码 |
| [lib/skills-core.js](file://e:/WorkStation/MyProject/superpowers/lib/skills-core.js) | **实际产物**，任务 1-4 的最终代码（但后来在 2026-02 删除） |
| [.opencode/INSTALL.md](file://e:/WorkStation/MyProject/superpowers/.opencode/INSTALL.md) | **实际产物**，任务 13 的最终文档 |
| [using-superpowers/SKILL.md](file://e:/WorkStation/MyProject/superpowers/skills/using-superpowers/SKILL.md) | 任务 12 引入的"工具映射"在此成为正式模式 |

**有趣的历史细节**：从 RELEASE-NOTES.md 看，`lib/skills-core.js` 在 2026-02 已被删除（"Deleted `lib/skills-core.js` and its test - unused since February 2026"）。这说明**计划文档中的设计并不一定与最终实现完全一致**——演进过程中可能发现不需要共享核心了。

## 📝 八、文档状态评价

| 评价维度 | 评分 | 说明 |
|---------|------|------|
| **任务明确度** | ⭐⭐⭐⭐⭐ | 每个步骤都可直接执行 |
| **可验证性** | ⭐⭐⭐⭐ | 每步都有验证命令，但缺自动化测试 |
| **TDD 体现** | ⭐⭐⭐⭐ | 体现了小步提交、频繁验证 |
| **完整性** | ⭐⭐⭐ | 缺回滚方案、调试指南、CI 集成 |
| **机器可读性** | ⭐⭐⭐⭐⭐ | 是 AI Agent 实施手册的范式 |
| **总分** | **4.3/5** | 工程实践到位，缺测试和回滚设计 |

## 🎓 九、给读者的启示

### 如果你是新加入项目的开发者

1. **本文档配合设计文档一起读**——先看"为什么"（设计），再看"怎么做"（实施）
2. **关注任务 1-4 的代码模式**——这是后续所有平台适配的基础设施
3. **学习任务 12 的 `toolMapping` 模式**——这是跨平台 AI 的核心难点

### 如果你是 AI Agent 实施者

1. **逐任务执行，不要跳步**——每个 commit 都是验证点
2. **遇到错误先回滚**——不要"调试到对为止"，而是"回滚到上一个 working 状态"
3. **手动测试要实际跑 OpenCode**——不要只信任单元测试

### 如果你是项目维护者

1. **警惕"计划文档与最终实现的偏离"**——`lib/skills-core.js` 最终被删除说明设计需要演进
2. **为新平台适配预留"薄封装"接口**——避免每个平台都重新实现
3. **重视手动测试指南**——跨平台 AI 的测试无法完全自动化

## 🚀 十、可能的后续工作

1. **重构为更通用的 npm 包**：`@superpowers/core` 替代 `lib/skills-core.js`
2. **补充自动化测试套件**：用 mock 替代 OpenCode API 调用
3. **添加 CI 集成**：GitHub Actions 同时验证 Codex + OpenCode
4. **完善监控/日志**：在 plugin 中加埋点，便于问题排查
5. **支持更多平台**：Cursor、Windsurf、Trae、Qoder（参考 [Qoder 适配分析](file://e:/WorkStation/MyProject/superpowers/z-study/docs/plans/2025-11-22-opencode-support-design.analysis.md)）
