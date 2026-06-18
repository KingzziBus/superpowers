# hooks.json 深度分析报告

> 文件: `hooks/hooks.json`（17 行，~0.3 KB）
> 类型: Claude Code 平台 Hook 配置（JSON Schema）
> 角色: superpowers 插件在 Claude Code 上的"入口清单"

---

## 1. 基本信息

| 字段 | 值 |
|------|-----|
| 文件路径 | `hooks/hooks.json` |
| 文件大小 | 约 0.3 KB（17 行） |
| 文件类型 | Claude Code 平台 Hook 配置（JSON Schema） |
| 平台目标 | Claude Code（Anthropic 官方 CLI） |
| 顶层键 | `hooks` |
| 注册事件 | `SessionStart`（大驼峰） |
| 事件匹配器 | `startup\|clear\|compact`（`\|` 分隔的"或"匹配） |
| 关联脚本 | `hooks/run-hook.cmd`（被本配置调用） |
| 间接调用 | `hooks/session-start`（被 `run-hook.cmd` 调用） |
| 关键环境变量 | `${CLAUDE_PLUGIN_ROOT}`（Claude Code 注入） |
| 同步/异步 | `async: false`（同步阻塞） |

---

## 2. 目的与背景

本文件是 superpowers 插件在 **Claude Code 平台**的 Hook 配置文件。Hook 是 Claude Code 在特定生命周期事件（`SessionStart`、`PostToolUse`、`UserPromptSubmit`、`PreCompact` 等）触发时执行自定义命令的能力。

**本配置的核心作用**：

1. 注册 `SessionStart` 事件监听器
2. 当用户开启新会话、调用 `/clear`、或触发 `/compact` 时，调用 `run-hook.cmd session-start`
3. 让 `session-start` 脚本读取 `using-superpowers` 技能内容并注入到模型上下文
4. 实现 **"模型始终知道自己拥有 superpowers 技能"** 的硬约束

**为什么这件事重要**：

superpowers 的核心价值主张是"教会 AI 如何系统地使用技能"。如果模型在每个会话开始时不知道自己有技能，那么整个 superpowers 库就形同虚设。这个 17 行的 JSON 文件就是 **把 using-superpowers 内容强喂给模型的管道**——是整个 superpowers 系统能被"自动使用"的关键开关。

---

## 3. 核心技术决策

### 3.1 为什么用 `run-hook.cmd` 作为入口而非直接 `bash session-start`？

这是最关键的设计决策，源于 Claude Code 在 Windows 上的一个已知 bug：

- Claude Code 在 Windows 上会自动检测命令是否包含 `.sh` 扩展名
- 如果检测到，会自动在命令前加 `bash` 前缀
- 这意味着 `"command": "bash session-start.sh"` 会被改写为 `"bash bash session-start.sh"`，导致双重 bash 调用而失败

**解决方案**：

- hook 脚本使用**无扩展名**文件名（`session-start` 而非 `session-start.sh`）
- 统一通过 `run-hook.cmd` 入口调度（它内部跨平台分发）
- 这样既绕过 Windows 自动检测的 bug，又获得跨平台一致性

**参考资料**：`docs/windows/polyglot-hooks.md` 详述了此设计。

### 3.2 为什么 `matcher` 是 `startup|clear|compact`？

这三个事件**都意味着"上下文刚发生重大变化"**：

| 事件 | 触发时机 | 重新注入原因 |
|------|---------|-------------|
| `startup` | 新会话开始 | 全新上下文，需要 superpowers 提示 |
| `clear` | 用户执行 `/clear` | 上下文被清空，需要重新加载 |
| `compact` | 自动压缩上下文 | 压缩后可能丢失之前的能力说明，必须重灌 |

**漏掉任何一个事件都会导致严重问题**：
- 漏 `startup`：用户首次对话没有 superpowers
- 漏 `clear`：用户清空后失去 superpowers
- 漏 `compact`：长会话压缩后 Claude 忘记它有技能，可能从 TDD 切换到"想怎么写就怎么写"

### 3.3 为什么 `async: false`？

- `async: false`：同步阻塞，Claude Code 等待命令完成才开始处理用户消息
- `async: true`：异步执行，命令和用户消息并行处理

**SessionStart 的语义要求"先注入，再对话"**：
- 如果异步，Claude 可能在上下文注入完成前就开始回复用户
- 用户首条回复可能完全没有 superpowers 提示，违反"硬约束"

代价：用户启动会话时会有几百毫秒的额外延迟（session-start 读取+转义+JSON 输出），这是可接受的。

### 3.4 为什么用 `${CLAUDE_PLUGIN_ROOT}` 而非绝对路径？

- 插件可能安装到不同位置（用户 home、system-wide、developer mode）
- 硬编码绝对路径会让插件**不可移植**
- `${CLAUDE_PLUGIN_ROOT}` 由 Claude Code 在调用 hook 时自动注入为插件根目录
- 配合双引号（`"${CLAUDE_PLUGIN_ROOT}/hooks/run-hook.cmd"`），正确处理路径中可能存在的空格

---

## 4. 文件结构剖析

```json
{
  "hooks": {                              // ← 顶层容器（Claude Code 约定）
    "SessionStart": [                     // ← 事件名（大驼峰命名）
      {
        "matcher": "startup|clear|compact",  // ← 子匹配器（多值用 | 分隔）
        "hooks": [                        // ← 钩子命令数组（可多个）
          {
            "type": "command",            // ← 命令类型（目前唯一支持类型）
            "command": "\"${CLAUDE_PLUGIN_ROOT}/hooks/run-hook.cmd\" session-start",
            "async": false                // ← 同步阻塞模式
          }
        ]
      }
    ]
  }
}
```

**字段语义对照表**：

| 字段 | 必需 | 含义 |
|------|------|------|
| `hooks` | 是 | 顶层容器，键为事件名 |
| `SessionStart` | - | 事件名，PascalCase 命名 |
| `matcher` | 否 | 子事件匹配，`|` 分隔多值，缺省则匹配所有 |
| `hooks[]` | 是 | 该匹配条件下的命令数组 |
| `type: "command"` | 是 | 钩子类型，目前 Claude Code 只支持 shell 命令 |
| `command` | 是 | 要执行的命令字符串 |
| `async` | 否 | true=异步, false=同步, 缺省=true |

---

## 5. 优点亮点

- **极简**: 17 行实现完整 Hook 链，没有冗余
- **声明式**: 不写代码，只描述"何时触发 + 调什么命令"
- **平台约定一致**: 完全遵循 Claude Code Hooks Schema 文档规范
- **可移植性**: `${CLAUDE_PLUGIN_ROOT}` 让插件可在不同安装位置工作
- **正确处理 Windows 路径**: 双引号包裹 + 环境变量展开
- **安全考量**: `async: false` 确保数据完整性优先
- **教育价值高**: 是一个"配置即架构"的优秀范例

---

## 6. 缺点风险局限

- **未指定 `timeout` 字段**: 如果 `run-hook.cmd` 卡住（如 bash 启动失败但没错误），Claude Code 会等多久未知。建议显式 `"timeout": 10000`（10 秒）
- **未设置 `env` 字段**: 无法向 `run-hook.cmd` 传递额外环境变量。如果未来需要在不同部署模式下调整行为（如开发模式标志），需要修改 `session-start` 内部
- **`.cmd` 后缀在 Unix 上是冗余的**: 在 Mac/Linux 上执行 `"${CLAUDE_PLUGIN_ROOT}/hooks/run-hook.cmd"` 会调用 run-hook.cmd 中的 bash 段（heredoc 被忽略），虽然能工作但路径语义略奇怪
- **`compact` 事件名可能是简写**: 完整事件名可能是 `PreCompact`，需要查阅 Claude Code 最新文档确认 `compact` 是否为官方简写
- **未指定 `if` 条件钩子**: Claude Code 新版支持条件钩子（只在满足条件时触发），本配置未利用
- **错误处理不可见**: 如果 `session-start` 失败退出，Claude Code 的行为未在本配置中体现

---

## 7. 关键洞察

1. **Hook 是元技能加载机制**: superpowers 之所以能"被 Claude 自动使用"，核心就是这个 17 行的 JSON 文件。它把"技能可用性"从"用户记得调用"提升到"系统自动注入"

2. **平台差异强制特化**: 三个平台（Claude Code、Cursor、Copilot CLI）需要**三个独立的 hooks JSON**，因为事件名格式、字段名、嵌套结构都不同。这是 Hook 机制的"低公分母"代价

3. **`.cmd` 后缀是反 `.sh`-bug 的 hack**: 一个极有教育意义的工程应对——把 Claude Code 的"自动加 bash 前缀"行为变成 feature 而不是 bug，方法是**故意不写 `.sh` 扩展名**

4. **startup|clear|compact 三选一**: 漏掉 `compact` 是个常见但隐蔽的 bug。Claude Code 官方应该把它默认为匹配所有 SessionStart 子事件，但显式列出更安全

5. **同步注入是 SessionStart 的核心 SLA**: SessionStart 的语义决定它必须同步。PostToolUse 等其他事件则可以异步

6. **配置文件也是"产品"**: 不要小看"几行 JSON"。理解这个文件是理解 superpowers 整体工作原理的第一步

7. **Superpowers 的"魔法时刻"**: 用户打开 Claude Code → SessionStart 触发 → session-start 读取 using-superpowers → 注入到上下文 → Claude 首条回复就自动使用 Skill 工具。这一切都由这 17 行 JSON 启动

---

## 8. 关联文档

| 文档/文件 | 关系 |
|----------|------|
| [`hooks/hooks-cursor.json`](file://e:/WorkStation/MyProject/superpowers/hooks/hooks-cursor.json) | 兄弟配置文件（Cursor 平台，事件名为 camelCase） |
| [`hooks/run-hook.cmd`](file://e:/WorkStation/MyProject/superpowers/hooks/run-hook.cmd) | 被调用的跨平台 polyglot 包装器 |
| [`hooks/session-start`](file://e:/WorkStation/MyProject/superpowers/hooks/session-start) | 实际执行 SessionStart 逻辑的 bash 脚本 |
| [`skills/using-superpowers/SKILL.md`](file://e:/WorkStation/MyProject/superpowers/skills/using-superpowers/SKILL.md) | 被注入到上下文的技能内容 |
| [`docs/windows/polyglot-hooks.md`](file://e:/WorkStation/MyProject/superpowers/docs/windows/polyglot-hooks.md) | 解释跨平台 Hook 机制的设计文档 |

---

## 9. 状态评价

| 维度 | 评分 | 说明 |
|------|------|------|
| 完整性 | 9/10 | 覆盖了主要场景，缺 `timeout` 字段 |
| 正确性 | 9/10 | 语法正确，语义符合 Claude Code 约定 |
| 可维护性 | 10/10 | 17 行即一目了然 |
| 可移植性 | 10/10 | `${CLAUDE_PLUGIN_ROOT}` + run-hook.cmd 抽象 |
| 跨平台 | 8/10 | 通过 run-hook.cmd 抽象，但 .cmd 后缀在 Unix 上冗余 |
| 健壮性 | 7/10 | 缺 `timeout` 防御，缺错误处理规范 |
| 文档化 | 6/10 | 没有任何 inline 注释说明 matcher 选择原因 |
| 教育价值 | 10/10 | 简洁而完整的 Hook 配置范例 |

**总体评价**: 9/10 - 一个非常优雅的、符合"最小惊讶原则"的配置。

**改进优先级**:
1. **高**: 添加 `timeout: 10000` 防止 session-start 卡死
2. **中**: 补充 inline 注释或关联文档，说明 `startup|clear|compact` 选择原因
3. **低**: 考虑添加 `env` 字段，为开发模式留扩展点

---

## 10. 给读者的启示

- **配置文件的 17 行也是架构**: 不要小看"几行 JSON"，每一个字段背后都是工程权衡
- **Hook 是技能自动发现的关键**: 理解本文件是理解 superpowers 工作原理的入口
- **平台差异需要特化配置**: 不能假设一份配置适用所有平台（参见 hooks-cursor.json）
- **环境变量是配置的"超能力"**: `${CLAUDE_PLUGIN_ROOT}` 让插件摆脱绝对路径束缚
- **同步 vs 异步反映 SLA 要求**: SessionStart 必须同步，PostToolUse 可异步
- **Bug 也能变 Feature**: Claude Code 的 ".sh 自动加 bash" bug 被故意利用成跨平台分发机制
- **Matcher 模式是 Hook 的"事件订阅"模型**: 理解了 startup\|clear\|compact 就理解了 Claude Code 的会话生命周期
- **配置即文档**: 一个好的配置本身就是一份"何时该发生什么"的清晰说明

---

## 11. 后续工作

### 11.1 短期改进（1-2 小时）

- [ ] 添加 `"timeout": 10000` 字段
- [ ] 在 `docs/` 下补充一份 `docs/hook-mechanism.md` 说明整体设计
- [ ] 编写集成测试，模拟 SessionStart 事件并验证输出

### 11.2 中期改进（半天）

- [ ] 考虑迁移到 Claude Code Hooks v2 Schema（如果平台升级支持）
- [ ] 将 startup 和 clear 拆分为两个不同事件，注入略有差异的提示
- [ ] 添加 `env` 字段支持开发模式开关

### 11.3 长期演进

- [ ] 抽象 hooks 公共部分到 `hooks/_lib/`，让每个平台 hook 共享
- [ ] 编写 schema 验证脚本，启动时检测 hooks.json 格式正确性
- [ ] 探索 Claude Code 的"条件钩子"（if 字段）减少不必要注入

### 11.4 监控/观测

- [ ] 在 session-start 中输出结构化日志（事件类型、注入耗时、警告状态）
- [ ] 设计可观测性指标：每次 SessionStart 注入成功率、平均延迟

### 11.5 风险预警

- ⚠️ Claude Code 升级可能改变 Hooks Schema 约定（如事件名 `compact` → `PreCompact`），需要保持 schema 验证
- ⚠️ 如果 Claude Code 改变 `.sh` 自动检测行为，本配置需要重新评估是否仍需 `run-hook.cmd` 抽象
- ⚠️ `${CLAUDE_PLUGIN_ROOT}` 在某些异常安装场景下可能未设置，需要 session-start 有降级方案

---
