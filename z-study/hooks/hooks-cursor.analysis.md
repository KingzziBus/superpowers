# hooks-cursor.json 深度分析报告

> 文件: `hooks/hooks-cursor.json`（11 行，~0.1 KB）
> 类型: Cursor 平台 Hook 配置（JSON Schema）
> 角色: superpowers 插件在 Cursor 上的"入口清单"（Cursor 专属版本）

---

## 1. 基本信息

| 字段 | 值 |
|------|-----|
| 文件路径 | `hooks/hooks-cursor.json` |
| 文件大小 | 约 0.1 KB（11 行） |
| 文件类型 | Cursor 平台 Hook 配置（JSON Schema） |
| 平台目标 | Cursor（AI 代码编辑器，基于 VSCode） |
| 顶层键 | `version` + `hooks` |
| Schema 版本 | `1` |
| 注册事件 | `sessionStart`（小驼峰） |
| 关联脚本 | `hooks/run-hook.cmd`（与 Claude Code 共用） |
| 关键环境变量 | `CURSOR_PLUGIN_ROOT`（在 `session-start` 中检测） |

---

## 2. 目的与背景

本文件是 superpowers 插件在 **Cursor 平台**的 Hook 配置文件。Cursor 是基于 VSCode 的 AI 代码编辑器，提供了与 Claude Code 类似但**完全不同 Schema**的 Hook 机制。

**本配置的核心作用**：

1. 注册 `sessionStart` 事件到 `./hooks/run-hook.cmd session-start`
2. 让 Cursor 在会话启动时调用 session-start 脚本
3. 与 Claude Code 配置**完全复用同一份业务逻辑**（`run-hook.cmd` + `session-start`），只在 Schema 层做平台适配
4. session-start 脚本内部通过 `CURSOR_PLUGIN_ROOT` 环境变量判断输出 Cursor 格式的 JSON

**与 Claude Code 配置的对比**：

| 维度 | Claude Code | Cursor |
|------|-------------|--------|
| 文件 | `hooks.json` | `hooks-cursor.json` |
| 事件名 | `SessionStart`（PascalCase） | `sessionStart`（camelCase） |
| 字段嵌套 | `hooks[].hooks[]` | `hooks[].command` |
| matcher 支持 | 是 | 否（不支持子事件过滤） |
| async 字段 | 是 | 否（不可配置） |
| 业务逻辑 | 同 | 同（通过 `run-hook.cmd` 复用） |

---

## 3. 核心技术决策

### 3.1 为什么需要独立的 JSON 文件？

即使最终调用的脚本是同一个 `run-hook.cmd`，Cursor 和 Claude Code 的 **Hook Schema 完全不同**：

- Claude Code 用 PascalCase 事件名（`SessionStart`）+ 嵌套 `hooks[].hooks[]` 数组
- Cursor 用 camelCase 事件名（`sessionStart`）+ 扁平 `hooks[].command` 字段

如果共用同一份 JSON，两边都不识别。**Schema 层的差异无法在脚本层抹平**，必须特化配置。

### 3.2 为什么命令用相对路径 `./hooks/run-hook.cmd`？

- Cursor 加载插件时，工作目录是插件根目录
- 因此 `./hooks/run-hook.cmd` 解析为 `<plugin_root>/hooks/run-hook.cmd`
- 不需要依赖任何环境变量（更简单）
- 但相对路径意味着 Cursor 必须从正确的工作目录执行——这一点是 Cursor 的隐式约定

### 3.3 为什么缺少 `matcher` 和 `async` 字段？

- **Cursor 不支持 matcher**：无法在 Cursor 层面区分 startup/clear/compact 子事件。这意味着 session-start 会在 Cursor 的**所有 sessionStart 触发点**执行，可能有性能损耗（但功能正确）
- **Cursor 不支持 async 字段**：不可配置同步/异步。这是 Cursor 平台的限制
- 缺这些字段不是疏忽，而是**平台的硬约束**

### 3.4 为什么 `version: 1`？

- Cursor 在 2024 年发布的 Hook Schema 中引入了 `version` 字段
- 当前值为 `1` 表示使用第一版 Schema
- 未来 Cursor 升级时，开发者会改为 `2` 并加新字段
- 这是一个**前向兼容**的版本控制机制

---

## 4. 文件结构剖析

```json
{
  "version": 1,                            // ← Schema 版本（Cursor 特有）
  "hooks": {                                // ← 顶层容器
    "sessionStart": [                       // ← 事件名（camelCase，Cursor 约定）
      {
        "command": "./hooks/run-hook.cmd session-start"
        // 注意：没有 type、matcher、async 字段
      }
    ]
  }
}
```

**字段语义对照表**：

| 字段 | 必需 | 含义 |
|------|------|------|
| `version` | 否 | Schema 版本号，未来兼容性 |
| `hooks` | 是 | 顶层容器，键为事件名 |
| `sessionStart` | - | 事件名，Cursor 用 camelCase |
| `command` | 是 | 要执行的命令字符串（扁平，无 type 包装） |

---

## 5. 优点亮点

- **极简**: 11 行（比 Claude Code 的 17 行还少 35%）
- **业务逻辑统一**: 复用 `run-hook.cmd` + `session-start`，没有重复实现
- **平台差异特化**: 显式承认 Cursor 和 Claude Code 的 Schema 不同
- **环境变量判断**: session-start 内部通过 `CURSOR_PLUGIN_ROOT` 检测平台，输出对应格式 JSON
- **相对路径简单**: 不依赖环境变量
- **符合 Cursor 约定**: 完整遵循 Cursor Hook Schema 文档

---

## 6. 缺点风险局限

- **不支持 matcher**: Cursor 在 compact 后**可能不会重新触发** sessionStart（行为待验证）。这意味着压缩后 superpowers 提示可能丢失
- **没有 `timeout` 字段**: 与 Claude Code 配置同样的问题
- **没有 `env` 字段**: 同上
- **相对路径耦合**: 命令必须从插件根目录执行，如果 Cursor 改变工作目录约定会失效
- **错误处理不可见**: session-start 失败时 Cursor 行为未明确
- **缺少 matcher 等价物**: Cursor Schema 当前不支持子事件过滤，session-start 会在所有 sessionStart 触发，包括可能是"恢复上次会话"等不需要重新注入的场景
- **`version: 1` 未说明升级路径**: 没有注释或文档说明未来如何升级

---

## 7. 关键洞察

1. **Schema 层差异是平台分裂的根源**: 同一业务逻辑需要 N 份平台特化配置。Cursor 和 Claude Code 的差异比看起来大得多（不只是大小写）

2. **环境变量是平台检测的最后手段**: `CURSOR_PLUGIN_ROOT` 和 `CLAUDE_PLUGIN_ROOT` 是 Cursor 和 Claude Code 注入的不同环境变量。`session-start` 内部用这两个变量来判断输出格式

3. **业务代码应"无知"于平台**: `session-start` 不知道调用方是哪个平台，它只检测环境变量并输出对应格式。这是一种"配置下沉，业务代码纯化"的模式

4. **matcher 缺失可能是个真实问题**: Claude Code 用 `startup|clear|compact` 三个 matcher 显式覆盖关键事件。Cursor 缺少这个能力可能是个潜在的功能缺口

5. **`version` 字段是"可演进性"投资**: Cursor 加了 version 字段是为了未来升级。但目前文档不全，何时升 version 不清楚

6. **极简不是错**: 11 行比 17 行还少 6 行——Cursor 的 Schema 设计**故意更窄**（支持的钩子类型更少）。这是一种"做少做对"的 API 哲学

7. **相对路径是 Cursor 的"工作目录隐式约定"**: 这是一个不太显式的契约。文档没写清楚，但实际工作。需要在 Cursor 升级时警惕

---

## 8. 关联文档

| 文档/文件 | 关系 |
|----------|------|
| [`hooks/hooks.json`](file://e:/WorkStation/MyProject/superpowers/hooks/hooks.json) | Claude Code 平台的兄弟配置（17 行） |
| [`hooks/run-hook.cmd`](file://e:/WorkStation/MyProject/superpowers/hooks/run-hook.cmd) | 被两个配置共同调用的跨平台入口 |
| [`hooks/session-start`](file://e:/WorkStation/MyProject/superpowers/hooks/session-start) | 内部通过 `CURSOR_PLUGIN_ROOT` 判断输出格式 |
| [`docs/windows/polyglot-hooks.md`](file://e:/WorkStation/MyProject/superpowers/docs/windows/polyglot-hooks.md) | 解释跨平台 Hook 机制的设计文档 |
| Cursor Hook 官方文档 | 外部参考（未在本仓库） |

---

## 9. 状态评价

| 维度 | 评分 | 说明 |
|------|------|------|
| 完整性 | 7/10 | 缺 matcher、timeout、env 字段（平台限制） |
| 正确性 | 9/10 | 语法正确，语义符合 Cursor 约定 |
| 可维护性 | 10/10 | 11 行即一目了然 |
| 可移植性 | 9/10 | 相对路径在 Cursor 约定下可工作 |
| 跨平台 | 8/10 | 业务逻辑跨平台，但配置层不跨平台 |
| 健壮性 | 7/10 | 缺 timeout 防御、缺错误处理 |
| 文档化 | 6/10 | 没有任何 inline 注释说明差异 |
| 教育价值 | 9/10 | 展示了"平台特化配置 + 业务代码统一"模式 |

**总体评价**: 8/10 - 简洁而符合 Cursor 约定，但功能完整度受平台 Schema 限制。

**改进优先级**:
1. **中**: 监控 Cursor 是否在 compact 后重新触发 sessionStart，必要时在 session-start 内部加 workaround
2. **低**: 添加 inline 注释说明 `version: 1` 含义和升级路径
3. **低**: 关联 `docs/hook-mechanism.md` 说明整体设计

---

## 10. 给读者的启示

- **同一份业务，N 份平台配置**: 这是跨平台 SDK 的常态。理解这个模式有助于设计多平台抽象
- **平台 Schema 差异不是"小问题"**: PascalCase vs camelCase、嵌套 vs 扁平、matcher 支持 vs 不支持——每一项都是真实差异
- **环境变量是平台分发的"暗线"**: `CURSOR_PLUGIN_ROOT` vs `CLAUDE_PLUGIN_ROOT` 让同一份 `session-start` 脚本能识别平台
- **业务代码应"无知"**: 业务代码不应"知道"自己被谁调用——这才是好的关注点分离
- **API 越窄不一定越差**: Cursor 的 Schema 比 Claude Code 窄（不支持 type、matcher、async），但**更简单**也是一种价值
- **`version` 字段是"可演进性"投资**: 设计 API 时加个 version 字段不会增加复杂度，但能为未来省很多麻烦
- **极简配置可读性最强**: 11 行比 17 行更易读——删除不必要的字段是种美德

---

## 11. 后续工作

### 11.1 短期改进（1 小时）

- [ ] 监控 Cursor 在 compact/clear 后是否重新触发 sessionStart
- [ ] 如果不触发，编写 plugin-internal 补偿机制（如定期重新注入）
- [ ] 添加 `docs/hook-mechanism.md` 说明跨平台配置差异

### 11.2 中期改进（半天）

- [ ] 如果 Cursor 升级 Schema 到 v2，评估是否需要升级 `version: 1` → `version: 2`
- [ ] 探索 Cursor 是否后续支持 `matcher` 或 `timeout` 字段
- [ ] 在 `session-start` 中加 platform 标识日志，便于调试

### 11.3 长期演进

- [ ] 抽象平台配置到 `hooks/_lib/platform-configs/`，让每个平台配置作为模板渲染
- [ ] 编写 Cursor 配置的单元测试，验证 Schema 兼容性
- [ ] 探索"配置生成器"模式：用一个高层声明，自动生成 N 份平台特化配置

### 11.4 风险预警

- ⚠️ Cursor 升级可能改变 `sessionStart` 事件名（虽然概率低）
- ⚠️ Cursor 改变工作目录约定会让相对路径失效（待观察）
- ⚠️ 缺少 matcher 可能导致 superpowers 提示在某些 Cursor 事件中意外丢失

### 11.5 跨平台对比扩展

- [ ] 编写 `docs/hook-mechanism.md` 时，建议在文档中以表格形式对比 Claude Code / Cursor / Copilot CLI 的 Schema 差异
- [ ] 在每个配置文件中加 `// 对应 Cursor 平台` 注释，方便维护者一眼识别

---
