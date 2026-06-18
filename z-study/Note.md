
# 疑问

## 个人笔记部分
- skill 还能有插件？(原文说特定场景的请做成插件)
- skill 如何使用第三方依赖？
- OpenCode 怎样注入skill?
  - 注入引导上下文：通过 experimental.chat.system.transform 钩子，在每次对话中加入 superpowers 相关的提示信息。
  - 注册技能目录：通过 config 钩子，让 OpenCode 无需 symlink 或手动配置即可发现所有 superpowers 技能。
- 工具映射表是支持多种 Agent 的核心资产
---

# 📚 Superpowers 学习笔记（基于本轮对话整理）

## 1. CLAUDE.md 的核心作用

### 文件身份
- `CLAUDE.md` 是 Claude Code 约定的"代理指令文件"，会话开始时由 AI 编码助手自动加载
- 本项目存在 3 个并列的指令文件（互为别名/引用）：

| 文件 | 内容 | 服务对象 |
|------|------|---------|
| `CLAUDE.md` | 完整贡献者守则（8.3KB） | Claude Code |
| `AGENTS.md` | 一行：`CLAUDE.md`（**符号链接**） | 其他通用代理 |
| `GEMINI.md` | `@./skills/using-superpowers/SKILL.md` 引用 | Gemini CLI |

### 优先级（最高）
在 `skills/using-superpowers/SKILL.md` 中明确定义：
1. **用户的显式指令**（CLAUDE.md / GEMINI.md / AGENTS.md / 直接请求）← 最高
2. Superpowers 技能
3. 工具默认系统提示 ← 最低

### 加载机制
- 不靠 AI 主动读
- 通过 `SessionStart` 钩子（hooks/hooks.json）自动注入
- OpenCode 插件在每条用户消息中注入 using-superpowers 内容

### 这份 CLAUDE.md 的特殊之处
跟普通项目不同，**Superpowers 的 CLAUDE.md 不是"项目介绍"，而是"AI 贡献者行为守则"**：
- 因为 Superpowers 本身就是给 AI 代理用的工具集
- 维护者统计：94% 的 PR 被拒
- 内容分四块：6 项强制 PR 检查、9 类不接受的 PR、新 Harness 验收测试、技能修改评估要求

---

## 2. 与"项目级 CLAUDE.md"完全不冲突

### 物理位置隔离

| 文件 | 位置 | 加载者 | 是否会冲突 |
|------|------|-------|----------|
| `ProjectA/CLAUDE.md` | 项目根目录 | Claude Code 自动加载 | ❌ 不会 |
| Superpowers 的 CLAUDE.md | `~/.claude/plugins/superpowers/` | **不会**被加载到你的项目会话 | ❌ 不会 |

### 注入的是 using-superpowers，不是 CLAUDE.md
看 `hooks/session-start` 源码：注入的是 `using-superpowers` 技能的内容（作为 `additionalContext`），不是 CLAUDE.md 本身。

### 用户项目 CLAUDE.md 优先级最高
即使你写"不要用 TDD"，superpowers 技能说"必须 TDD"，AI 也听你的（用户永远优先）。

---

## 3. 多 Agent 的安装路径

### 6 套插件清单适配 6 个 Agent
- `.claude-plugin/plugin.json` → Claude Code
- `.codex-plugin/plugin.json` → Codex CLI / Codex App
- `.cursor-plugin/plugin.json` → Cursor
- `package.json`（`"main": ".opencode/plugins/superpowers.js"`）→ OpenCode
- `gemini-extension.json` → Gemini CLI
- 无元数据 → Factory Droid / Copilot CLI（走 marketplace）

### 各 Agent 的具体安装路径

| Agent | 安装命令 | 安装路径 | 关键被安装文件 |
|-------|---------|---------|--------------|
| **Claude Code** | `/plugin install superpowers@claude-plugins-official` | `~/.claude/plugins/superpowers/` | `skills/` + `hooks/` + `assets/` |
| **Codex CLI** | `/plugins` → 搜索安装 | `~/.codex/extensions/superpowers/` | `skills/` + `assets/` + `.codex-plugin/`（**不带 hooks**） |
| **Cursor** | `/add-plugin superpowers` | `~/.cursor/extensions/superpowers/` | `skills/` + `agents/` + `commands/` + `hooks-cursor.json` |
| **OpenCode** | npm 安装 + `opencode.json` 声明 | `~/.config/opencode/node_modules/superpowers/` | **整包**（含 `.opencode/plugins/superpowers.js`） |
| **Gemini CLI** | `gemini extensions install <url>` | `~/.gemini/extensions/superpowers/` | `skills/` + `gemini-extension.json` + `GEMINI.md` |

### 核心被安装文件
**`skills/` 13 个技能 + `assets/` 图标 = 100% 必装**

### 关键洞察
- `hooks/` 只给 Claude 和 Cursor 用
- OpenCode 是唯一需要 `.js` 入口的
- 根目录的 `CLAUDE.md` / `AGENTS.md` / `GEMINI.md` 永远不会被装到用户项目里

---

## 4. 各 Agent 加载技能机制差异

### 共同点（"事实标准"）
所有 Agent 都接受 `skills/<name>/SKILL.md` 这个**目录结构约定** + YAML frontmatter + Markdown 格式。

### 差异点（每家都各搞一套）

| 维度 | Claude Code | Codex | Cursor | OpenCode | Gemini |
|------|------------|-------|--------|----------|--------|
| **目录结构** | 同 | 同 | 同 | 任意 | 同 |
| **发现机制** | Claude 原生扫描 | 读 plugin.json | 读 plugin.json | **JS 编程注册** | 读 extension.json |
| **引导注入** | SessionStart 钩子 | defaultPrompt 字段 | hooks-cursor | **JS 钩子函数** | `contextFileName` + `@` 引用 |
| **激活工具** | 原生 Skill 工具 | Codex 自家 | Cursor 自家 | OpenCode `skill` 工具 | `activate_skill` |
| **是否官方规范** | ✅ 是 | ⚠️ 换皮 | ⚠️ 换皮 | ❌ 完全自写 | ⚠️ 换皮 |

### 工作流对比
- **Claude Code**：钩子 → 读 `using-superpowers/SKILL.md` → 注入 additionalContext → Skill 工具按需加载
- **Codex CLI**：读 plugin.json → 扫 skills/ → defaultPrompt 注入引导 → 自家机制激活
- **Cursor**：读 plugin.json → sessionStart 调 `session-start` 脚本 → Cursor Skill 系统加载
- **OpenCode**：npm 装包 → 加载 `superpowers.js` → 编程读取 SKILL.md → 注册 `chat.messages.transform` 钩子
- **Gemini CLI**：读 extension.json → 加载 GEMINI.md → `@` 引用展开 → `activate_skill` 激活

---

## 5. 关键概念类比

### CLAUDE.md 的"员工守则"类比
- `CLAUDE.md` = 公司的"AI 代理员工守则"
- `skills/*/SKILL.md` = 各部门 SOP
- `using-superpowers` bootstrap = 入职培训（强制先看）
- `hooks/hooks.json` = 门卫（强制执行入职培训）

### 技能加载机制的"图书馆"类比
- 书架摆法（`skills/<name>/SKILL.md`）= 大家都用 ISBN 编号
- 检索系统（发现机制）= 每家都有自己的搜索引擎
- 借书流程（激活工具）= 每家都有自己的借书证
- 入馆须知（引导/钩子）= 每家都有自己的迎宾员

### 项目设计哲学
为什么 worktree 技能里要写 "your project's agent instruction file (CLAUDE.md, GEMINI.md, AGENTS.md, .cursorrules, or equivalent)"？
因为"指令文件"在 5 个 Agent 里没有统一命名，需要兼容。

---

## 6. 一句话总结

> Superpowers 是一个**跨多 AI Agent** 的可复用技能库设计：
> - 共同点：统一的目录约定（`skills/<name>/SKILL.md`）
> - 差异点：每个 Agent 自己的发现/注册/激活机制
> - Claude Code 100% 官方规范
> - OpenCode 完全自写 JS 插件
> - 其他介于两者之间
> - 用户始终掌控（CLAUDE.md 优先级最高）

---

## 7. Qoder / QoderCN 的 Skill 机制（基于外部资料整理）

> ⚠️ 本节内容来自 Qoder 官方文档（docs.qoder.com / help.aliyun.com），**不是从 Superpowers 源码推出**。Superpowers 官方未适配 Qoder。

### 7.1 Qoder vs QoderCN 的关系

| 维度 | Qoder | QoderCN |
|------|-------|---------|
| 发布方 | 阿里达摩院 | 阿里云灵码（国内品牌） |
| 用户级配置 | `~/.qoder/` | `~/.qoder-cn/` |
| 项目级配置 | `.qoder/` | `.qodercn/` |
| 官方文档 | docs.qoder.com | help.aliyun.com/zh/lingma |
| Skill 体系 | **完全相同** | **完全相同**（仅路径前缀不同） |

> 顺带一提：你的系统路径是 `C:\Users\Q\AppData\Roaming\QoderCN\...`，所以你用的是 QoderCN。

### 7.2 Skill 安装路径

| 位置 | 路径 | 作用域 |
|------|------|--------|
| Qoder 用户级 | `~/.qoder/skills/{skill-name}/SKILL.md` | 当前用户所有项目 |
| Qoder 项目级 | `.qoder/skills/{skill-name}/SKILL.md` | 仅当前项目 |
| QoderCN 用户级 | `~/.qoder-cn/skills/{skill-name}/SKILL.md` | 当前用户所有项目 |
| QoderCN 项目级 | `.qodercn/skills/{skill-name}/SKILL.md` | 仅当前项目 |

**优先级规则**：同名时，**项目级覆盖用户级**。

### 7.3 Skill 文件格式（跟 Anthropic 官方规范 100% 兼容）

```yaml
---
name: api-doc-generator
description: Generate comprehensive API documentation from code. Use when...
---

# API Documentation Generator

When generating API documentation:
1. Identify all API endpoints
2. Document request/response formats
...
```

✅ 跟 Claude Code / Superpowers 的 `SKILL.md` 格式**完全相同**。

### 7.4 三种创建方式

1. **`/create-skill` 引导**：`/create-skill <描述>`（不熟悉语法时推荐）
2. **`npx skills` CLI**：`npx skills add xxx -a qoder`（从 skills.sh 或 GitHub 安装第三方）
3. **手动创建**：写 `.md` 到指定目录

### 7.5 触发与加载机制

- **自动触发**：模型根据用户请求和 Skill 描述自主判断何时使用
- **手动触发**：在对话框输入 `/skill-name`
- **加载时机**：创建或修改后，**新会话自动加载**；输入 `/` 查看已加载列表

### 7.6 Qoder 独特扩展（其他 Agent 都没有的）

| 特性 | 说明 | 跟其他 Agent 对比 |
|------|------|------------------|
| **可视化界面**（v0.5.0） | 技能/智能体/指令的 GUI 管理 | Claude Code / Cursor 都没有 |
| **Hook 机制** | 5 个节点：UserPromptSubmit、PreToolUse、PostToolUse、PostToolUseFailure、Stop | Claude Code 是 SessionStart 等 |
| **`npx skills` CLI** | 一键从 skills.sh / GitHub 安装第三方 | Qoder **独家** |
| **Quest 模式** | 智能体团队自主完成多步骤任务 | Qoder 1.0 核心卖点 |
| **内置 Skills** | `/create-skill`、`/create-subagent`、`/vercel-deploy`、`/canvas` | 风格类似 Claude Code |

### 7.7 6 个 Agent 综合对比

| 维度 | Qoder | Claude Code | Codex | Cursor | OpenCode | Gemini |
|------|-------|------------|-------|--------|----------|--------|
| 目录约定 | `skills/<n>/SKILL.md` | 同 | 同 | 同 | JS 加载 | 同 + GEMINI.md |
| 是否官方规范 | ✅ **100% 兼容** | ✅ 是 | ⚠️ 换皮 | ⚠️ 换皮 | ❌ 自写 | ⚠️ 换皮 |
| Agent 目录 | `~/.qoder/agents/*.md` | 无独立概念 | 混在 skills | `agents/` 目录 | 编程注册 | 无 |
| 指令(Commands) | ❌ **已并入 Skills** | `commands/` | 无 | `commands/` | 编程注册 | 无 |
| Hook 机制 | ✅ 5 个节点 | ✅ SessionStart 等 | ❌ | ✅ 1 个节点 | ✅ JS 钩子 | ❌ |
| CLI 安装 | ✅ `npx skills` | ❌ | ❌ | 有 `openskills` | ❌ | ❌ |
| 可视化管理 | ✅ GUI | ❌ | ❌ | ❌ | ❌ | ❌ |

**关键发现**：
- **Qoder 在 6 个 Agent 里适配最干净**（目录约定 100% 兼容 Anthropic 官方规范）
- **Qoder 是唯一把 commands 合并到 skills 的**（Cursor 保留独立目录）
- **Qoder Hook 节点最多**（5 个，覆盖 Agent 全生命周期）

### 7.8 Superpowers 装到 Qoder 的可行性分析

**理论分析**（未实测），分三步：

✅ **可以做**：
- 目录约定兼容（直接复制 `superpowers/skills/*` 到 `~/.qoder-cn/skills/` 即可识别）

⚠️ **需要适配**：
- **工具名映射**（Superpowers 技能里用的是 Claude Code 工具名：TodoWrite、Task、Skill → Qoder 的 todowrite、skill 等）
- **重新实现 using-superpowers bootstrap**（Claude Code 靠 SessionStart 钩子注入，Qoder 需要用 PreToolUse 或 UserPromptSubmit 节点）

🔧 **做不到自动**：
- Qoder **没有** `.claude-plugin/` 这种 manifest 机制
- 所以 skills 只能靠目录扫描被发现
- 无法像 Claude Code 那样走 marketplace 自动更新

### 7.9 Qoder 的设计哲学

> **模型提供智能，Harness 决定这份智能能否转化为可用交付。**
> —— 2026-05-15 Qoder 1.0 发布稿

这跟 Superpowers 的核心哲学**完全一致**：
- 都不优化模型本身
- 而是优化"模型运行的环境"（Harness Engineering / 驾驭工程）
- 通过 Skill + Hook + Agent 协作来塑造行为

**这可能就是两者的 Skill 规范如此相似的原因**——都站在 Anthropic 官方规范这个"事实标准"上。

### 7.10 一句话总结

> Qoder 是 6 个 Agent 里**适配最干净**的（目录 100% 兼容 Anthropic 官方规范），但 Superpowers 未官方适配；想装 Superpowers 到 Qoder，目录复制即可，**关键是重写 `using-superpowers` 引导注入机制**（用 Qoder 的 Hook 节点实现）。

