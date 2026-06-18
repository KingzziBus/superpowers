# 文档分析报告：OpenCode 支持设计

> **原文档**：[2025-11-22-opencode-support-design.md](file://e:/WorkStation/MyProject/superpowers/docs/plans/2025-11-22-opencode-support-design.md)
> **翻译版本**：[2025-11-22-opencode-support-design.zh-CN.md](file://e:/WorkStation/MyProject/superpowers/z-study/docs/plans/2025-11-22-opencode-support-design.zh-CN.md)
> **文档类型**：设计文档（Design）
> **分析日期**：2026-06-03

---

## 📋 一、文档基本信息

| 字段 | 内容 |
|------|------|
| 标题 | OpenCode Support Design |
| 日期 | 2025-11-22 |
| 作者 | Bot & Jesse |
| 状态 | Design Complete, Awaiting Implementation |
| 字数 | ~295 行 / ~12KB |
| 类别 | plans（项目开发计划） |

## 🎯 二、文档目的

为 **OpenCode.ai** 添加完整的 superpowers 支持。核心思路是构建一个**原生 OpenCode 插件**（使用其 JavaScript/TypeScript 插件系统），并**与现有的 Codex 实现共享核心代码**（通过 `lib/skills-core.js`）。

## 🏗️ 三、核心技术决策

### 3.1 三大平台架构对比

| 平台 | 插件机制 | superpowers 实现方式 |
|------|---------|---------------------|
| Claude Code | 原生 Anthropic 插件 + 基于文件的技能 | 最直接 |
| Codex | 无插件系统 | bootstrap markdown + CLI 脚本 |
| **OpenCode** | **JS/TS 插件 + 事件钩子 + 自定义工具** | **本次设计目标** |

### 3.2 共享核心模块设计

设计提取了 4 个核心函数到 `lib/skills-core.js`：

| 函数 | 职责 |
|------|------|
| `extractFrontmatter(filePath)` | 解析 YAML frontmatter（name + description） |
| `findSkillsInDir(dir, maxDepth)` | 递归发现目录中的 SKILL.md |
| `findAllSkills(dirs)` | 扫描多个目录 |
| `resolveSkillPath(skillName, dirs)` | 处理覆盖关系（个人 > 核心） |
| `checkForUpdates(repoDir)` | 检查 git 远程更新 |

### 3.3 OpenCode 插件架构

**两个自定义工具**：

1. **`use_skill`**：相当于 Claude 的 Skill 工具，加载特定技能内容到对话
2. **`find_skills`**：列出所有可用技能及其元数据

**一个关键钩子**：
- `session.started` 钩子：会话启动时自动注入 3 样东西
  1. `using-superpowers` 技能的完整内容
  2. 技能列表
  3. 工具映射说明（TodoWrite → update_plan 等）

**技能目录覆盖机制**（关键设计点）：
- `~/.config/opencode/superpowers/skills/`（核心）
- `~/.config/opencode/skills/`（个人，可覆盖核心）
- 用 `superpowers:` 前缀显式访问核心技能

## 📐 四、文件结构

```
superpowers/
├── lib/
│   └── skills-core.js           # 新建：共享逻辑
├── .codex/
│   ├── superpowers-codex        # 更新：使用 skills-core
│   ├── superpowers-bootstrap.md
│   └── INSTALL.md
├── .opencode/
│   ├── plugin/
│   │   └── superpowers.js       # 新建：OpenCode 插件
│   └── INSTALL.md               # 新建：安装指南
└── skills/                       # 不变
```

## 📊 五、实施计划（3 阶段 16 任务）

| 阶段 | 任务 | 目标 |
|------|------|------|
| **阶段 1** | 重构共享核心 | 提取 4 个核心函数，更新 Codex 用共享代码 |
| **阶段 2** | 构建 OpenCode 插件 | 实现 use_skill、find_skills、session.started 钩子 |
| **阶段 3** | 文档与打磨 | 更新 README、RELEASE-NOTES、INSTALL |

## 🔍 六、文档优缺点分析

### ✅ 优点

1. **架构合理**：采用"共享核心 + 平台封装"模式，DRY 原则贯彻到位
2. **渐进式重构**：分阶段实施，每阶段可独立验证
3. **明确工具映射**：列出 Claude Code 工具到 OpenCode 工具的映射表
4. **关注可扩展性**：方案文档明确提到 "Easy to add future platforms (Cursor, Windsurf, etc.)"
5. **覆盖机制清晰**：个人技能可覆盖核心技能的设计很优雅

### ⚠️ 缺点 / 风险点

1. **缺少错误处理设计**：技能文件格式错误、权限问题等异常情况未说明
2. **未讨论性能影响**：每次会话启动注入所有技能元数据是否会消耗 token？是否需要分页/懒加载？
3. **版本兼容性未提**：OpenCode 插件 API 是否有版本要求？未来 API 变化如何应对？
4. **测试策略偏弱**：只有手动测试步骤，没有自动化测试设计
5. **缺少迁移路径**：现有 Codex 用户的配置是否会受影响？
6. **Frontmatter 解析手写**：手写 YAML 解析器，如果格式复杂可能 bug

## 💡 七、关键洞察

### 洞察 1：这是"复用驱动设计"的典型案例

文档第 44-57 行是核心创新点：把"为新平台做适配"转化为"为已有平台抽公共部分"。**这就是一个好的架构设计的本质——让新功能的成本越来越低。**

### 洞察 2：工具映射是跨平台适配的核心难题

跨 AI 平台移植最大的挑战不是目录结构（各家都学会了 `skills/<n>/SKILL.md`），而是**工具名差异**：
- `TodoWrite` → `update_plan`
- `Task`（subagent）→ `@mention`
- `Skill` → `use_skill`

**这是 Superpowers 后续设计 `using-superpowers/SKILL.md` 中"Platform Adaptation"段落的雏形。**

### 洞察 3：覆盖机制（Shadowing）值得借鉴

```javascript
// 命名空间语法
superpowers:brainstorming  // 强制使用核心技能
brainstorming              // 默认：个人 > 核心
```

这种**显式命名空间**设计借鉴了 Node.js、Python 等包管理器的成熟经验。

## 🔗 八、与其他文档的关联

| 关联文档 | 关联方式 |
|---------|---------|
| [2025-11-22-opencode-support-implementation.md](file://e:/WorkStation/MyProject/superpowers/docs/plans/2025-11-22-opencode-support-implementation.md) | 配套的实施计划（1096 行） |
| [using-superpowers/SKILL.md](file://e:/WorkStation/MyProject/superpowers/skills/using-superpowers/SKILL.md) | 设计中的"工具映射"在此文档中标准化为"Platform Adaptation" |
| [README.opencode.md](file://e:/WorkStation/MyProject/superpowers/docs/README.opencode.md) | 最终面向用户的 OpenCode 安装文档 |
| [.opencode/INSTALL.md](file://e:/WorkStation/MyProject/superpowers/.opencode/INSTALL.md) | 设计文档中提到的安装指南的最终版本 |

## 📝 九、文档状态评价

| 评价维度 | 评分 | 说明 |
|---------|------|------|
| **目标清晰度** | ⭐⭐⭐⭐⭐ | 目标明确，动机充分 |
| **架构合理性** | ⭐⭐⭐⭐⭐ | 共享核心 + 平台封装是正确选择 |
| **可实施性** | ⭐⭐⭐⭐ | 步骤明确但缺少自动化测试 |
| **完整性** | ⭐⭐⭐ | 缺少错误处理、性能、回滚方案 |
| **文档质量** | ⭐⭐⭐⭐ | 简洁清晰，代码示例丰富 |
| **总分** | **4.2/5** | 一份质量不错的轻量级设计文档 |

## 🎓 十、给读者的启示

如果你是新加入 Superpowers 项目的开发者，这篇文档是**必读**的，因为：

1. **它揭示了项目的核心架构思想**——共享核心 + 平台封装
2. **它展示了"为新平台适配"的标准工作流**——设计 → 实施 → 文档
3. **它定义了跨平台工具映射的范式**——这是后续所有平台适配的基础

**但更推荐配套阅读** [2025-11-22-opencode-support-implementation.md](file://e:/WorkStation/MyProject/superpowers/docs/plans/2025-11-22-opencode-support-implementation.md)（1096 行的详细实施计划），看看抽象的设计如何落地为具体的 TDD 任务。

## 🚀 十一、可能的后续工作

基于本设计，**值得进一步探索**的方向：

1. **将 `lib/skills-core.js` 抽象为更通用的包**（`@superpowers/core`）发布到 npm
2. **添加更多平台支持**（Cursor、Windsurf、Trae 等）
3. **改进 frontmatter 解析**（使用 `yaml` 库替代手写解析）
4. **添加自动化测试套件**（当前依赖手动测试）
5. **实现增量更新检查**（避免每次都全量 fetch）
