# 视觉化脑暴重构实施计划 - 分析报告

> **对应原文**：`docs/superpowers/plans/2026-02-19-visual-brainstorming-refactor.md`
> **翻译文件**：`z-study/docs/superpowers/plans/2026-02-19-visual-brainstorming-refactor.zh-CN.md`
> **报告时间**：2026-06-03

---

## 一、基本信息

| 项目 | 内容 |
|---|---|
| 文档类型 | 实施计划（Implementation Plan） |
| Chunk 数量 | 1 个 Chunk（7 个 Task） |
| 总行数 | 524 行 |
| 关联 Spec | `docs/superpowers/specs/2026-02-19-visual-brainstorming-refactor-design.md` |
| 核心变更 | 阻塞 TUI 反馈 → 非阻塞"浏览器显示、终端命令"架构 |
| 删除脚本 | `wait-for-feedback.sh`（消除 TaskOutput 阻塞） |
| 新增机制 | 每屏一个 `.events` 文件（JSON Lines 格式） |
| 涉及文件 | 5 个修改 + 1 个删除 |

---

## 二、目的

将视觉化脑暴（visual brainstorming）的反馈循环从**阻塞式**重构为**非阻塞式**，解决 AI Agent 长时间挂起等待用户点击的问题。

### 痛点：阻塞式 TUI 反馈模型的缺陷

| 旧模式 | 表现 | 问题 |
|---|---|---|
| `wait-for-feedback.sh` | Bash 脚本同步等待用户点击 | 服务器在 5+ 分钟内持有 socket，agent 处于"假死"状态 |
| `TaskOutput(block=true)` | Claude 调用方阻塞等待 | token 消耗高，连接不稳定时易超时 |
| `sendToClaude` JS 函数 | 浏览器主动推回 agent | 把"对话"和"展示"混在一起，违反关注点分离 |

### 新模式："浏览器显示、终端命令"

```
用户：                        Agent（Claude）：
打开浏览器 ────► 看到屏幕    写 HTML 文件 ──► 结束本轮
              点击选项 ◄────────────────┐│
              │                        ││
              ▼                        ││
         写入 .events 文件             ││
                                       ││
用户在终端回复："A 不错"   ◄───────────┘│
                                  │     ││
读取 .events + 终端回复        ◄───────┘│
                                  │     │
判断是否需要下一轮         ─────────────┘
```

**核心创新**：
- 浏览器只显示，不推送
- 终端才是"对话"通道
- `.events` 文件作为"非实时信使"，解耦"看"和"说"

---

## 三、核心技术决策

### 1. 用 `.events` 文件替代 WebSocket 推送

**之前**：浏览器点击 → WebSocket → 服务器 → `wait-for-feedback.sh` 阻塞 → 唤醒 agent
**之后**：浏览器点击 → WebSocket → 服务器 → 追加写入 `.events` 文件 → agent 下轮读文件

**为什么用文件而不是 HTTP 回调**：
- ✅ 文件天然持久化（崩溃可恢复）
- ✅ 简单（`fs.appendFileSync` 即可）
- ✅ 调试方便（`cat .events` 直接看）
- ✅ 测试方便（断言文件内容）
- ⚠️ 不实时（但脑暴场景对实时无要求）

### 2. 屏幕到来时自动清空 `.events`

```javascript
if (filePath.endsWith('.html')) {
  const eventsFile = path.join(SCREEN_DIR, '.events');
  if (fs.existsSync(eventsFile)) fs.unlinkSync(eventsFile);
  // ...
}
```

**避免状态污染**：上一屏的点击记录不会影响下一屏的解读。

### 3. 删除 `wait-for-feedback.sh` 脚本

**为何坚决删除**：
- 阻塞式编程模型与"事件流"架构不兼容
- 一旦保留，就总有"等等看吧"的诱惑
- 心理上降低对"非阻塞"的承诺

### 4. 简化 helper.js：移除所有"主动推回"代码

**删除清单**：
- `sendToClaude` 函数（点击后接管页面）
- `window.send` 函数（手动 Send 按钮）
- 表单提交处理（textarea 自由输入）
- `pageshow` 事件监听（清空 textarea）
- `inputTimeout` 变量

**保留清单**：
- `sendEvent` 基础发送
- `window.brainstorm` API（缩小到 `send` + `choice`）
- click 捕获（但收窄到 `[data-choice]` 元素）

**信条**："浏览器只看不发"——任何"主动推回"都违背新架构。

### 5. indicator-bar 替代 feedback-footer

**之前**（feedback-footer）：
- 大块文本输入框
- Send 按钮
- 实时显示对话

**之后**（indicator-bar）：
- 一行小字："Click an option above, then return to the terminal"
- 点击后变："<Option> selected — return to terminal to continue"
- 无输入框，无按钮

**设计哲学**：**视觉提示而非主动呼叫**——告诉用户"我看到了，但请到终端回复"。

### 6. 测试驱动：先写失败测试，再实现

每个 Task 都有"先写测试、运行确认失败、实现功能、运行确认通过"的标准 TDD 流程。这与项目推崇的 TDD 文化一致。

### 7. 保留 chokidar 文件监听

注意：本计划**没有**做零依赖改造（那是 [2026-03-11 计划](./2026-03-11-zero-dep-brainstorm-server.md)）。本计划保留 `express` + `ws` + `chokidar`，专注重构反馈模型。

---

## 四、文件结构

### 修改（5 个）
| 文件 | 主要变更 |
|---|---|
| `lib/brainstorm-server/index.js` | 增加 `.events` 写入/清空、简化 `wrapInFrame` |
| `lib/brainstorm-server/frame-template.html` | 删除 feedback footer + 文本框、新增 indicator-bar |
| `lib/brainstorm-server/helper.js` | 删除 send/feedback 函数、收窄到 click 捕获 |
| `skills/brainstorming/visual-companion.md` | 重写 "The Loop"、更新所有引用 |
| `tests/brainstorm-server/server.test.js` | 更新 4 个旧测试（Test 5/6/7/8） |

### 删除（1 个）
- `lib/brainstorm-server/wait-for-feedback.sh` — 阻塞式反馈不再需要

### 不变
- `lib/brainstorm-server/start-server.sh`、`stop-server.sh`
- `lib/brainstorm-server/frame-template.html` 的 CSS 主题（除 indicator-bar 样式外）

---

## 五、Task 划分逻辑

| Task | 范围 | 测试 |
|---|---|---|
| **Task 1** | frame-template.html 视觉重设计 | 模板仍可加载（测试 1-5） |
| **Task 2** | index.js 服务端逻辑（事件文件） | 新增 2 个失败测试 → 实现 → 通过 |
| **Task 3** | helper.js 客户端精简 | 不需新测试，沿用 Task 2 的事件测试 |
| **Task 4** | server.test.js 更新断言 | 修复 4 个旧测试以匹配新结构 |
| **Task 5** | 删除 wait-for-feedback.sh | 全测试通过（确认无引用） |
| **Task 6** | visual-companion.md 文档重写 | 文档级，无自动测试 |
| **Task 7** | 最终验证 + 残留引用扫描 | 全测试 + 手动冒烟 |

**价值**：
- Task 1-2 是基础设施层（视觉 + 服务端）
- Task 3-4 是客户端 + 测试同步
- Task 5-6 是清理 + 文档
- Task 7 是质量门控

---

## 六、优点

| 维度 | 具体表现 |
|---|---|
| **解决真实痛点** | 阻塞 TUI 是脑暴技能的最大使用摩擦——非阻塞化是质变 |
| **架构清晰** | "浏览器=展示、终端=对话、文件=非实时信使"——三方职责分明 |
| **可测试性强** | `.events` 是文件，可直接断言内容；click 流程端到端可测 |
| **故障恢复** | 文件持久化意味着 agent 崩溃重启后仍能读到 `.events` |
| **调试友好** | 开发者可 `cat .events` 查看用户点击流 |
| **文档同步** | `visual-companion.md` 与代码同步重写，避免文档与实现漂移 |
| **无破坏性** | 仍支持完整文档模式（`!DOCTYPE` / `<html`），不强制片段 |
| **心理学正确** | 删除 `wait-for-feedback.sh` 物理上避免开发者"恢复旧模式"的诱惑 |

---

## 七、缺点 / 风险

| 风险点 | 影响 | 缓解 |
|---|---|---|
| **失去实时性** | agent 不知道用户已经点击——必须等用户主动回复 | 这是设计意图（解耦"看"和"说"），不是缺陷 |
| **多设备复杂度** | 用户切换设备时 `.events` 路径不同 | 脑暴是单设备单用户场景，可接受 |
| **需要用户双通道** | 用户必须看浏览器 + 写终端——增加认知负担 | 通过清晰的指示器栏 + "return to terminal to continue" 提示缓解 |
| **删除老代码风险** | `sendToClaude` 等被硬删除——若外部代码引用则破坏 | Task 7 步骤 3 用 grep 全仓库扫描 |
| **路径硬编码** | `lib/brainstorm-server/` 路径出现在文档示例中 | 文档使用 `${CLAUDE_PLUGIN_ROOT}` 占位符 |
| **测试迁移摩擦** | 4 个旧测试需要更新——易遗漏 | Task 4 明确列出每个测试的更新点 |
| **`fs.appendFileSync` 阻塞** | 高频 click 可能成为瓶颈 | 脑暴场景 click 频率 < 1 Hz，完全可接受 |
| **顺序语义** | 多 click 时，最后一个才是最终选择？ | 文档明确说"通常"是，模式也很重要 |

---

## 八、关键洞察

### 洞察 1：阻塞 vs 非阻塞的范式跃迁

**阻塞式**的隐含假设是："agent 必须实时响应"。
**非阻塞式**的隐含假设是："agent 可以异步消费信息"。

后者更符合**批处理**心智模型——agent 主动拉取信息（`.events`），而不是被信息推送打断。

### 洞察 2：文件作为通用集成点

`.events` 文件这个设计极其简洁，却**极具扩展性**：
- 可被 `cat` 读取（人）
- 可被 `node fs.appendFileSync` 写入（程序）
- 可被 `grep` 过滤（运维）
- 可被 `tail -f` 监控（实时调试）

**用文件做 IPC** 是 Unix 哲学的现代回响。

### 洞察 3：删除是更强的承诺

`git rm wait-for-feedback.sh` 比"标记为 deprecated"更有效——它从物理上断绝旧模式的回归路径。**真正的架构改进需要删除而不仅仅是"标记"**。

### 洞察 4：测试驱动让"反馈模型"可被客观验证

| 旧 | 新 |
|---|---|
| 阻塞 → agent 是否被唤醒？只有人类能感知 | 写入文件 → 可断言文件内容 |
| 推送 → 用户是否收到？需打开浏览器 | click 记录 → 可单元测试 |

**非阻塞架构+文件IPC 让"用户行为"可被机器验证**——这是测试覆盖率的质变。

### 洞察 5：浏览器和终端的角色再分工

| 通道 | 优势 | 新职责 |
|---|---|---|
| 浏览器 | 视觉化、可交互 | **展示**（单向） |
| 终端 | 文本流、可编程 | **对话**（双向） |

这种分工对应 **"GUI vs CLI"** 哲学之争的现代解答——让 GUI 回到"展示"的本质，让 CLI 主导"对话"。

### 洞察 6：文档必须与代码同步重构

Task 6 重写 `visual-companion.md` 与代码 Task 1-5 同步进行。**没有这一步，文档会撒谎**——开发者读文档会按老 API 实现，导致不一致。

---

## 九、关联文档

### 上游 / Spec
- [`2026-02-19-visual-brainstorming-refactor-design.md`](../specs/2026-02-19-visual-brainstorming-refactor-design.md) — 设计文档（本计划的依据）

### 下游 / 受影响
- 所有使用脑暴技能的 skill（如 `brainstorming`）— 现在遵循新循环
- 所有依赖 `wait-for-feedback.sh` 的旧文档/工具 — Task 7 步骤 3 扫描

### 横向关联
- [`2026-01-17-visual-brainstorming.md`](../../plans/2026-01-17-visual-brainstorming.md) — 视觉化脑暴的初始实现
- [`2026-03-11-zero-dep-brainstorm-server.md`](./2026-03-11-zero-dep-brainstorm-server.md) — 紧随其后的零依赖重构（同一文件的下一步优化）
- [`2025-11-28-skills-improvements-from-user-feedback.md`](../../plans/2025-11-28-skills-improvements-from-user-feedback.md) — 用户反馈驱动的技能改进

### 历史
- `wait-for-feedback.sh` 创建于早期脑暴原型
- 阻塞 TUI 模式在 2025 年底的 [skills-improvements 计划](../../plans/2025-11-28-skills-improvements-from-user-feedback.md) 中被标记为反模式

---

## 十、状态评价

**计划成熟度：高** ✅

- 每个 Task 有失败测试 → 实现的 TDD 流程
- 7 个 Task 全部对应 commit
- 最终验证步骤 3 包含全仓库 grep 扫描

**实施复杂度：中** ⚖️

- 涉及前端 + 后端 + 文档 + 测试，多领域同步
- 关键是测试与代码同步演进（4 个旧测试需要更新）
- 心理上"删除工作流"比"增加功能"更困难

**适用性**：✅ 强烈推荐
- 解决最常见的用户痛点
- 架构改进而非功能堆叠
- 测试可量化（文件断言）

---

## 十一、给读者的启示

### 启示 1：非阻塞是现代 AI Agent 的关键能力
AI agent 在"等用户"时浪费大量 token 和 wall-clock 时间。**把"等待"变成"读文件"**是巨大效率提升。

### 启示 2：文件是万能 IPC
不要小看文件——`cat`、`grep`、`tail -f`、测试断言都能直接用。**用文件做集成的系统天然可调试、可测试、可观测**。

### 启示 3：删除比注释更有效
`// TODO: remove later` 几乎永远不会被删除。**`git rm` 才是真的告别**。本计划直接删除 `wait-for-feedback.sh` 是务实的工程实践。

### 启示 4：UI 应回到"展示"的本质
现代 web 应用的"主动推送"经常让用户/agent 抓狂。**让浏览器只展示，让 CLI 对话**——这是更健康的职责分离。

### 启示 5：测试驱动让"用户行为"可机器验证
"用户是否点击了 A？"以前只能问用户，现在可直接 `assert event.choice === 'a'`。**可测性是非阻塞架构的副产品**。

### 启示 6：文档必须与代码同步重构
Task 6 看似是"收尾工作"，实则是"质量门控"。**没有同步的文档，架构改进会随着开发者入职而退化**。

---

## 十二、后续工作（推测）

1. **`.events` schema 形式化**：定义事件的 JSON Schema（type、choice、text、metadata），用 TypeScript 或 JSON Schema 校验
2. **时间戳归一化**：当前用 `Date.now()`，跨时区用户的脑暴可能产生时间错位
3. **多用户协作支持**：未来若要支持多人脑暴（每用户独立 `.events.{userId}` 文件）
4. **事件回放 UI**：开发一个简单 HTML 页面，按时间顺序回放用户点击流（教学价值）
5. **替代方案对比**：若用 SQLite 替代 `.events` 文件，可支持更复杂查询
6. **浏览器扩展集成**：未来可在 Chrome Extension 中识别"超级脑暴页面"，提供快捷键
7. **零依赖协同**：与 [`2026-03-11` 计划](./2026-03-11-zero-dep-brainstorm-server.md) 合并后，`index.js` 中本计划添加的代码需要迁移到 `server.js`
