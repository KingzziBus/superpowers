# 视觉化脑暴重构：浏览器显示，终端命令

**日期：** 2026-02-19
**状态：** 已通过
**范围：** `lib/brainstorm-server/`、`skills/brainstorming/visual-companion.md`、`tests/brainstorm-server/`

## 问题

在视觉化脑暴期间，Claude 将 `wait-for-feedback.sh` 作为后台任务运行，并阻塞在 `TaskOutput(block=true, timeout=600s)` 上。这会完全占用 TUI——视觉化脑暴运行时用户无法向 Claude 输入。浏览器成为唯一的输入通道。

Claude Code 的执行模型是基于轮次的。在单个轮次内，Claude 无法同时监听两个通道。阻塞式 `TaskOutput` 模式是错误的原语——它模拟的是平台不支持的事件驱动行为。

## 设计

### 核心模型

**浏览器 = 交互式展示。** 显示 mockup，让用户点击选择选项。选择被服务端记录。

**终端 = 对话通道。** 始终未阻塞，始终可用。用户在这里与 Claude 交谈。

### 循环流程

1. Claude 向会话目录写入一个 HTML 文件
2. 服务器通过 chokidar 检测到文件存在，向浏览器推送 WebSocket 重新加载（不变）
3. Claude 结束本轮——告诉用户查看浏览器并在终端回复
4. 用户查看浏览器，可选地点击选择选项，然后在终端输入反馈
5. 在下一轮，Claude 读取 `$SCREEN_DIR/.events` 获得浏览器交互流（点击、选择），与终端文本合并
6. 迭代或前进

无后台任务。无 `TaskOutput` 阻塞。无轮询脚本。

### 关键删除：`wait-for-feedback.sh`

完全删除。其原本目的是桥接"服务器将事件记录到 stdout"和"Claude 需要接收这些事件"。`.events` 文件取代了这一点——服务器直接写入用户交互事件，Claude 用平台提供的任何文件读取机制读取它们。

### 关键新增：`.events` 文件（每屏事件流）

服务器将所有用户交互事件写入 `$SCREEN_DIR/.events`，每行一个 JSON 对象。这让 Claude 获得当前屏幕的完整交互流——不仅是最终选择，还有用户的探索路径（先点 A，再点 B，最后定 C）。

用户探索选项后文件示例内容：

```jsonl
{"type":"click","choice":"a","text":"Option A - Preset-First Wizard","timestamp":1706000101}
{"type":"click","choice":"c","text":"Option C - Manual Config","timestamp":1706000108}
{"type":"click","choice":"b","text":"Option B - Hybrid Approach","timestamp":1706000115}
```

- 在同一屏内仅追加。每个用户事件作为新行追加。
- 当 chokidar 检测到新的 HTML 文件（推送新屏幕）时，文件被清空（删除），防止陈旧事件传递。
- 若 Claude 读取时文件不存在，说明没有浏览器交互——Claude 仅使用终端文本。
- 文件仅包含用户事件（`click` 等）——不包含服务器生命周期事件（`server-started`、`screen-added`）。这让其保持小而聚焦。
- Claude 可读取完整流以理解用户的探索模式，或仅看最后一个 `choice` 事件获得最终选择。

## 按文件划分的变化

### `index.js`（服务器）

**A. 将用户事件写入 `.events` 文件。**

在 WebSocket `message` 处理器中，在将事件记录到 stdout 之后：通过 `fs.appendFileSync` 将事件作为 JSON 行追加到 `$SCREEN_DIR/.events`。仅写入用户交互事件（带 `source: 'user-event'` 的），不写服务器生命周期事件。

**B. 新屏幕到来时清空 `.events`。**

在 chokidar `add` 处理器中（检测到新的 `.html` 文件），若 `$SCREEN_DIR/.events` 存在则删除。这是"新屏幕"权威信号——比在每次重新加载触发的 `GET /` 上清空更可靠。

**C. 替换 `wrapInFrame` 内容注入。**

当前正则锚定在 `<div class="feedback-footer">`，而该元素即将被移除。改为注释占位符：移除 `#claude-content` 内现有的默认内容（`<h2>Visual Brainstorming</h2>` 和副标题段落），替换为单个 `<!-- CONTENT -->` 标记。内容注入变为 `frameTemplate.replace('<!-- CONTENT -->', content)`。更简单且不会因模板格式变化而中断。

### `frame-template.html`（UI 框架）

**移除：**
- `feedback-footer` div（textarea、Send 按钮、label、`.feedback-row`）
- 关联的 CSS（`.feedback-footer`、`.feedback-footer label`、`.feedback-row`、其中的 textarea 和 button 样式）

**新增：**
- `#claude-content` 内的 `<!-- CONTENT -->` 占位符，替换默认文本
- 原页脚位置的选择指示器栏，两种状态：
  - 默认："Click an option above, then return to the terminal"
  - 选中后："Option B selected — return to terminal to continue"
- 指示器栏的 CSS（微妙，视觉权重与现有 header 类似）

**保持不变：**
- 带有 "Brainstorm Companion" 标题和连接状态的 header 栏
- `.main` 包装器和 `#claude-content` 容器
- 所有组件 CSS（`.options`、`.cards`、`.mockup`、`.split`、`.pros-cons`、占位符、mock 元素）
- 暗/亮主题变量和媒体查询

### `helper.js`（客户端脚本）

**移除：**
- `sendToClaude()` 函数和 "Sent to Claude" 页面接管
- `window.send()` 函数（与已删除的 Send 按钮相关）
- 表单提交处理器——没有 feedback textarea 就没有用途，会增加日志噪声
- 输入变更处理器——同理
- `pageshow` 事件监听器（之前用于修复 textarea 持久化——现在没有 textarea）

**保留：**
- WebSocket 连接、重连逻辑、事件队列
- 重新加载处理器（服务器推送时 `window.location.reload()`）
- `window.toggleSelect()` 用于选择高亮
- `window.selectedChoice` 跟踪
- `window.brainstorm.send()` 和 `window.brainstorm.choice()`——这些与已删除的 `window.send()` 不同。它们调用通过 WebSocket 将事件记录到服务器的 `sendEvent`。对自定义完整文档页面有用。

**收窄：**
- Click 处理器：仅捕获 `[data-choice]` 点击，而非所有按钮/链接。当浏览器作为反馈通道时需要宽捕获；现在它仅用于选择跟踪。

**新增：**
- 在 `data-choice` 点击时，更新选择指示器栏文本以显示所选选项。

**从 `window.brainstorm` API 中移除：**
- `brainstorm.sendToClaude` —— 不再存在

### `visual-companion.md`（技能说明）

**重写 "The Loop" 小节**为上述非阻塞流程。移除所有对以下内容的引用：
- `wait-for-feedback.sh`
- `TaskOutput` 阻塞
- 超时/重试逻辑（600s 超时、30 分钟上限）
- 描述 `send-to-claude` JSON 的 "User Feedback Format" 小节

**替换为：**
- 新的循环（写 HTML → 结束轮次 → 用户在终端回复 → 读 `.events` → 迭代）
- `.events` 文件格式文档
- 指导说明终端消息是主要反馈；`.events` 提供完整浏览器交互流作为额外上下文

**保留：**
- 服务器启动/停止说明
- 内容片段 vs 完整文档指导
- CSS 类参考和可用组件
- 设计技巧（按问题缩放保真度、每屏 2-4 个选项等）

### `wait-for-feedback.sh`

**完全删除。**

### `tests/brainstorm-server/server.test.js`

需要更新的测试：
- 断言片段响应中存在 `feedback-footer` 的测试——更新为断言选择指示器栏或 `<!-- CONTENT -->` 替换
- 断言 `helper.js` 包含 `send` 的测试——更新以反映收窄后的 API
- 断言 `sendToClaude` CSS 变量用法的测试——移除（函数不再存在）

## 平台兼容性

服务器代码（`index.js`、`helper.js`、`frame-template.html`）完全平台无关——纯 Node.js 和浏览器 JavaScript。无 Claude Code 特有引用。已通过后台终端交互在 Codex 上验证可用。

技能说明（`visual-companion.md`）是平台适配层。每个平台的 Claude 使用自己的工具启动服务器、读取 `.events` 等。非阻塞模型天然跨平台工作，因为它不依赖任何平台特有的阻塞原语。

## 这能实现什么

- **TUI 在视觉化脑暴期间始终响应**
- **混合输入** —— 在浏览器中点击 + 在终端中输入，自然合并
- **优雅降级** —— 浏览器宕机或用户未打开？终端仍可用
- **更简单的架构** —— 无后台任务、无轮询脚本、无超时管理
- **跨平台** —— 同一服务器代码在 Claude Code、Codex 和任何未来平台工作

## 这会放弃什么

- **纯浏览器反馈工作流** —— 用户必须返回终端才能继续。选择指示器栏会引导他们，但这比旧的"点击 Send 并等待"流程多了一步。
- **来自浏览器的内联文本反馈** —— textarea 已消失。所有文本反馈都通过终端。这是刻意的——终端作为文本输入通道比 frame 中的小 textarea 更好。
- **浏览器 Send 后的即时响应** —— 旧系统在用户点击 Send 的瞬间让 Claude 响应。现在在用户切换到终端时存在间隙。实践中这只是几秒钟，用户可在终端消息中添加上下文。
