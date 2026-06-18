# 视觉化脑暴重构设计 - 分析报告

> **对应原文**：`docs/superpowers/specs/2026-02-19-visual-brainstorming-refactor-design.md`
> **翻译文件**：`z-study/docs/superpowers/specs/2026-02-19-visual-brainstorming-refactor-design.zh-CN.md`
> **报告时间**：2026-06-03

---

## 一、基本信息

| 项目 | 内容 |
|---|---|
| 文档类型 | 设计 Spec |
| 总行数 | 163 行 |
| 关联计划 | `docs/superpowers/plans/2026-02-19-visual-brainstorming-refactor.md` |
| 状态 | 已通过 |
| 范围 | 3 个目录：`lib/brainstorm-server/`、`skills/brainstorming/visual-companion.md`、`tests/brainstorm-server/` |
| 核心创新 | `.events` 文件 + 浏览器展示 / 终端对话分离 |
| 跨平台 | Claude Code、Codex、未来平台 |

---

## 二、目的

解决视觉化脑暴中"阻塞式 TUI"的根本问题：Claude 用 `wait-for-feedback.sh` + `TaskOutput(block=true)` 后台等待浏览器反馈，导致 TUI 完全卡死。

### 根本洞察

> "Claude Code 的执行模型是基于轮次的。在单个轮次内，Claude 无法同时监听两个通道。阻塞式 `TaskOutput` 模式是错误的原语——它模拟的是平台不支持的事件驱动行为。"

**这是对"用错误原语模拟想要行为"的批判**。

---

## 三、核心技术决策

### 1. 浏览器 / 终端职责再分工

| 通道 | 旧职责 | 新职责 |
|---|---|---|
| 浏览器 | 反馈 + 展示 | **仅展示** |
| 终端 | （被阻塞） | **对话通道** |
| 文件 | （未用） | **非实时信使**（.events） |

**核心原则**："用对工具做对的事"——浏览器是"展示"的工具，终端是"对话"的工具。

### 2. `.events` 文件作为解耦的桥梁

| 旧 | 新 |
|---|---|
| 浏览器 → WebSocket → 服务器 → 后台任务 → 阻塞 → 唤醒 agent | 浏览器 → WebSocket → 服务器 → 写文件 → 下轮 agent 读文件 |

**为什么用文件**：
- 持久化（崩溃可恢复）
- 简单（`fs.appendFileSync` 即可）
- 调试友好（`cat .events`）
- 测试可断言

### 3. 新屏幕清空 `.events`

chokidar 检测到新 HTML 文件时删除 `.events`——**比 GET / 清空更可靠**（GET / 在每次重载触发）。

### 4. 删除 `wait-for-feedback.sh`

**彻底删除**而非"标记 deprecated"——**避免旧模式回归**。

### 5. indicator-bar 替代 feedback-footer

- 旧：textarea + Send 按钮
- 新：一行小字提示

**设计哲学**："视觉提示而非主动呼叫"。

### 6. helper.js API 收窄

| 旧 API | 新 API |
|---|---|
| `sendToClaude()` | （删除） |
| `window.send()` | （删除） |
| 表单/输入处理 | （删除） |
| `[data-choice]` click | 保留（收窄） |
| `window.brainstorm.send/choice` | 保留（自定义完整文档用） |

**原则**：**任何"主动推回"都违背新架构**。

### 7. 内容注入改用注释占位符

```js
frameTemplate.replace('<!-- CONTENT -->', content)
```

**为何比 `<div class="feedback-footer">` 锚定好**：
- 不依赖具体 HTML 结构
- 模板格式变更不会破坏
- 简单直白

---

## 四、文件结构

### 修改（5 个）
| 文件 | 变化 |
|---|---|
| `index.js` | 增加 `.events` 写入/清空、简化 `wrapInFrame` |
| `frame-template.html` | 移除 footer、新增 indicator-bar |
| `helper.js` | 删除 send/feedback、收窄到 click 捕获 |
| `visual-companion.md` | 重写 "The Loop" |
| `server.test.js` | 更新 4 个旧测试 |

### 删除（1 个）
- `wait-for-feedback.sh`

---

## 五、优点

| 维度 | 具体表现 |
|---|---|
| **解决真实痛点** | 阻塞 TUI 是脑暴最大使用摩擦 |
| **架构清晰** | 浏览器=展示、终端=对话、文件=信使——职责分明 |
| **可测试性强** | `.events` 是文件，可直接断言 |
| **故障恢复** | 文件持久化，崩溃后仍能读取 |
| **调试友好** | `cat .events` 直接查看用户点击流 |
| **跨平台** | 同一代码在 Claude Code 和 Codex 都可用 |
| **删除作为承诺** | 物理删除 `wait-for-feedback.sh` 而非"标记" |
| **降级优雅** | 浏览器宕机时终端仍可用 |

---

## 六、缺点 / 风险

| 风险点 | 影响 | 缓解 |
|---|---|---|
| **失去实时性** | agent 不知道用户已点击 | 这是设计意图（解耦"看"和"说"） |
| **多设备复杂度** | 切换设备时 `.events` 路径不同 | 脑暴是单设备场景 |
| **用户双通道认知负担** | 必须看浏览器+写终端 | 指示器栏 + "return to terminal" 提示 |
| **删除老代码风险** | 外部代码若引用会破坏 | 计划 Task 7 用 grep 扫描 |
| **fs.appendFileSync 阻塞** | 高频 click 可能瓶颈 | 脑暴场景 < 1 Hz，可接受 |
| **顺序语义** | 多 click 时最后是最终选择 | 文档明确"通常"是，模式也重要 |

---

## 七、关键洞察

### 洞察 1：用错误原语模拟想要行为是反模式

> "阻塞式 TaskOutput 模式模拟的是平台不支持的事件驱动行为。"

**这是对"硬上"思维的批判**——如果平台不支持，并发模型就是不对的，应该重构流程而非模拟。

### 洞察 2：浏览器 vs 终端的"通道哲学"

不同通道有不同的**擅长**：
- GUI 擅长**视觉**（一次展示大量信息）
- CLI 擅长**对话**（快速输入、文本流）

**让通道回到它们擅长的角色**。

### 洞察 3：文件作为通用 IPC

`.events` 文件是"用文件做进程间通信"——`cat`、`grep`、`tail -f`、测试断言都能直接用。

**用文件做集成的系统天然可调试、可测试、可观测**。

### 洞察 4：删除是更强的承诺

`git rm wait-for-feedback.sh` 比"标记 deprecated"更有效——**真正的架构改进需要删除而不仅仅是"标记"**。

### 洞察 5：测试驱动让"用户行为"可机器验证

非阻塞架构 + 文件 IPC 让"用户是否点击 A"从"问用户"变为"`assert event.choice === 'a'`"。

**可测性是非阻塞架构的副产品**。

### 洞察 6：跨平台兼容性的自然实现

由于不依赖任何平台特有阻塞原语，**同一代码天然跨平台**。这是"不假设平台能力"的回报。

### 洞察 7：设计哲学的"放弃"也是设计

"这会放弃什么"一节明确说明：
- 纯浏览器反馈（需要切到终端）
- 内联文本反馈（textarea 没了）
- 即时响应（用户切换间隙）

**好的设计文档应该诚实说明"放弃了什么"**——避免承诺无法兑现的体验。

---

## 八、关联文档

### 下游 / 实施
- [`2026-02-19-visual-brainstorming-refactor.md`](../plans/2026-02-19-visual-brainstorming-refactor.md) — 实施计划

### 横向关联
- [`2026-01-17-visual-brainstorming.md`](../../plans/2026-01-17-visual-brainstorming.md) — 视觉化脑暴的初始实现
- [`2026-03-11-zero-dep-brainstorm-server-design.md`](./2026-03-11-zero-dep-brainstorm-server-design.md) — 紧随其后的零依赖重构
- [`2025-11-28-skills-improvements-from-user-feedback.md`](../../plans/2025-11-28-skills-improvements-from-user-feedback.md)

### 历史
- 阻塞 TUI 模式是早期脑暴原型的妥协
- 用户反馈（`skills-improvements` 计划）标记其为反模式
- 本 spec 是该妥协的撤销方案

---

## 九、状态评价

**设计成熟度：极高** ⭐⭐⭐

- 问题诊断准确（"用错误原语"）
- 解决方案清晰（"通道再分工"）
- 跨平台兼容性自然获得
- 诚实说明"放弃什么"

**实施复杂度：中** ⚖️

- 涉及前端 + 后端 + 文档 + 测试
- 关键是测试与代码同步演进
- 心理上"删除工作流"比"增加功能"更难

**适用性**：✅ 强烈推荐
- 解决最常见的用户痛点
- 架构改进而非功能堆叠
- 测试可量化

---

## 十、给读者的启示

### 启示 1：用错误原语模拟想要行为是反模式
**先问"平台支持什么"再设计**——而非"模拟想要的行为"。

### 启示 2：通道应回到它们擅长的角色
GUI 展示、CLI 对话、文件信使。**让工具做它擅长的事**。

### 启示 3：文件是万能 IPC
不要小看文件——`cat`、`grep`、`tail -f`、测试断言都能直接用。

### 启示 4：删除比注释更有效
"// TODO: remove later" 几乎永远不会被删除。**`git rm` 才是真的告别**。

### 启示 5：可测性是架构选择的副产品
非阻塞 + 文件 IPC 让"用户行为"可被机器验证。**选择架构时考虑可测性**。

### 启示 6：诚实说明"放弃了什么"
好的设计文档明确说明"trade-off"——**避免承诺无法兑现的体验**。

### 启示 7：跨平台是"不假设"的回报
不依赖平台特有原语 → 天然跨平台。**不假设 = 普适**。

---

## 十一、后续工作（推测）

1. **`.events` schema 形式化**：定义 JSON Schema（type、choice、text、metadata）
2. **时间戳归一化**：跨时区用户的脑暴
3. **多用户协作支持**：每用户独立 `.events.{userId}` 文件
4. **事件回放 UI**：开发简单 HTML 页面回放点击流
5. **替代方案对比**：若用 SQLite 替代文件，可支持更复杂查询
6. **浏览器扩展集成**：Chrome Extension 识别"超级脑暴页面"提供快捷键
