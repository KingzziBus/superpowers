# 文档分析报告：可视化头脑风暴伴侣实施计划

> **原文档**：[2026-01-17-visual-brainstorming.md](file://e:/WorkStation/MyProject/superpowers/docs/plans/2026-01-17-visual-brainstorming.md)
> **翻译版本**：[2026-01-17-visual-brainstorming.zh-CN.md](file://e:/WorkStation/MyProject/superpowers/z-study/docs/plans/2026-01-17-visual-brainstorming.zh-CN.md)
> **文档类型**：实施计划（Implementation Plan）
> **分析日期**：2026-06-03

---

## 📋 一、文档基本信息

| 字段 | 内容 |
|------|------|
| 标题 | Visual Brainstorming Companion Implementation Plan |
| 日期 | 2026-01-17 |
| 状态 | 实施计划 |
| 字数 | ~572 行 / ~14.8KB |
| 类别 | plans（项目开发计划） |
| 任务数 | 5 任务 |
| 技术栈 | Node.js + Express + WebSocket + chokidar |

## 🎯 二、文档目的

为 **Claude Code 添加浏览器端的可视化头脑风暴伴侣**。这是一个**进程外 UI 增强方案**——通过本地 Node.js 服务器 + 文件监视 + WebSocket，**让 Claude 能够在终端对话中"展示"HTML 界面给用户**。

## 🏗️ 三、核心架构

### 3.1 数据流（3 个进程协同）

```
┌──────────────────┐     写文件        ┌──────────────────┐
│  Claude (Agent)  │ ───────────────→ │ /tmp/brainstorm  │
│                  │                   │   /screen.html   │
└──────────────────┘                   └────────┬─────────┘
                                                │ 文件变化
                                                ▼
┌──────────────────┐     HTTP         ┌──────────────────┐
│  User Browser    │ ←─────────────── │  Node.js Server  │
│  (可视化 UI)     │ ──── WebSocket──→│  (localhost:3333)│
└──────────────────┘   用户点击/输入   └────────┬─────────┘
                                                │ stdout (JSON)
                                                ▼
                                       ┌──────────────────┐
                                       │  Claude (stdout) │
                                       │  读取用户响应     │
                                       └──────────────────┘
```

**关键创新**：
- **用文件作为 Claude 与浏览器的桥梁**（不依赖复杂 API）
- **用 stdout 让服务器反馈到 Claude**（标准 Unix 模式）
- **WebSocket 处理实时双向通信**

### 3.2 三个关键组件

| 组件 | 文件 | 职责 |
|------|------|------|
| **Node.js 服务器** | `lib/brainstorm-server/index.js` | HTTP + WebSocket + 文件监视 |
| **浏览器辅助库** | `lib/brainstorm-server/helper.js` | 自动捕获点击/表单/输入事件 |
| **集成测试** | `tests/brainstorm-server/server.test.js` | 4 个集成测试 |

## 📋 四、5 任务实施分解

| 任务 | 目标 | 关键代码量 |
|------|------|----------|
| 1 | 创建服务器基础（含文件监视、WebSocket） | ~120 行 |
| 2 | 创建浏览器辅助库（事件自动捕获） | ~95 行 |
| 3 | 编写集成测试（4 个测试用例） | ~110 行 |
| 4 | 修改 brainstorming 技能 + 创建 visual-companion.md | ~80 行 |
| 5 | 添加 node_modules 到 .gitignore | ~3 行 |

## 💡 五、关键技术亮点

### 5.1 "写文件 + 文件监视" 桥接模式

```javascript
// Claude 端：写 HTML
fs.writeFileSync('/tmp/brainstorm/screen.html', html);

// 服务器端：监视文件变化
chokidar.watch(SCREEN_FILE).on('change', () => {
  // 通知所有浏览器重新加载
});
```

**巧妙之处**：
- Claude **不需要特殊 API** 来"推送 UI"
- 任何能写文件的 Claude 都能使用这个功能
- 文件系统是**最简单的 IPC**（进程间通信）

### 5.2 "stdout + JSON" 事件回流

```javascript
// 服务器端：把 WebSocket 事件写入 stdout
ws.on('message', (data) => {
  const event = JSON.parse(data.toString());
  console.log(JSON.stringify({ type: 'user-event', ...event }));
});
```

**优势**：
- Claude 只需"读后台任务输出"就能获取用户响应
- 不需要新的 API
- JSON 格式让 Claude 容易解析

### 5.3 helper.js 的"零配置事件捕获"

```javascript
// 自动捕获点击
document.addEventListener('click', (e) => {
  const target = e.target.closest('button, a, [data-choice], [role="button"], input[type="submit"]');
  if (!target) return;
  e.preventDefault();
  send({ type: 'click', text: target.textContent.trim(), choice: target.dataset.choice || null });
});
```

**优雅之处**：
- Claude 写的 HTML **不需要写 JS** 就能与用户交互
- 用 `data-choice` 属性标识可选元素
- 自动捕获 `click`、`submit`、`input` 三类事件
- 500ms 防抖避免 input 事件风暴

## 🔍 六、文档优缺点分析

### ✅ 优点

1. **架构极简**：3 个文件、5 个任务就能跑通
2. **零依赖 API**：复用现有 file IO、stdout、WebSocket
3. **完整测试覆盖**：4 个集成测试覆盖关键路径
4. **良好的关注点分离**：服务器、helper、文档分离
5. **HTML 模式库**：visual-companion.md 提供了丰富的 HTML 模式
6. **Git ignore 提示**：避免 node_modules 污染仓库

### ⚠️ 缺点 / 风险点

1. **依赖 /tmp 目录**：在 Windows 上可能需要调整
2. **端口硬编码 3333**：多个会话可能冲突
3. **无身份验证**：任何能访问 localhost:3333 的人都能看到内容
4. **无跨平台测试**：未提供 Windows/macOS 兼容性测试
5. **错误恢复弱**：服务器崩溃后浏览器辅助库只是重连，无状态恢复
6. **未处理大量 HTML**：超大 HTML 可能影响 chokidar 性能

## 💡 七、关键洞察

### 洞察 1：进程外 UI 是"AI Coding 的视觉扩展"

传统 AI Coding 工具**只有终端界面**。本文档的创新在于：
- **不修改 Claude 的核心架构**（不用新 API）
- **通过文件系统 + 本地服务器扩展 UI 能力**
- **用标准 Unix 模式（stdout）回传数据**

这种"外挂式增强"思路在后续 OpenCode、Qoder 等 AI IDE 中被广泛采用。

### 洞察 2：辅助库的"零配置"哲学是核心 UX

helper.js 的设计哲学：**Claude 写纯 HTML，用户就能交互**。
- 无需 `<script>` 标签
- 无需 `addEventListener`
- 无需 WebSocket 代码

**这是"框架应当消失"的最佳实践**——好的框架让用户感受不到它的存在。

### 洞察 3：4 个测试用例覆盖了完整数据流

```
Test 1: 服务器启动并输出 JSON（验证 stdout 流）
Test 2: GET / 返回带 helper 的 HTML（验证 HTTP 流）
Test 3: WebSocket 转发事件到 stdout（验证反向数据流）
Test 4: 文件变化通知浏览器（验证 chokidar 流）
```

**测试设计的"完整性"**：每个测试对应一条数据流路径。**这是端到端测试的范式**。

### 洞察 4：visual-companion.md 是"技能即文档"的最佳示例

`skills/brainstorming/visual-companion.md` 不只是说明文档，而是**HTML 模式库**：
- Choice Cards（选择卡片）
- Interactive Mockup（交互 mockup）
- Form with Notes（带备注的表单）
- Explicit JavaScript（显式 JavaScript）

**这种"模式库"让技能具备真正的可复用性**。

## 🔗 八、与其他文档的关联

| 关联文档 | 关联方式 |
|---------|---------|
| [brainstorming/SKILL.md](file://e:/WorkStation/MyProject/superpowers/skills/brainstorming/SKILL.md) | 任务 4 直接修改此技能 |
| [brainstorming/visual-companion.md](file://e:/WorkStation/MyProject/superpowers/skills/brainstorming/visual-companion.md) | 任务 4 创建此参考文档 |
| [brainstorming/scripts/server.cjs](file://e:/WorkStation/MyProject/superpowers/skills/brainstorming/scripts/server.cjs) | **最终代码**——任务 1 的产物（但经过重构） |
| [brainstorming/scripts/helper.js](file://e:/WorkStation/MyProject/superpowers/skills/brainstorming/scripts/helper.js) | **最终代码**——任务 2 的产物 |
| [brainstorming/scripts/frame-template.html](file://e:/WorkStation/MyProject/superpowers/skills/brainstorming/scripts/frame-template.html) | 任务 4 设计的 HTML 模式最终落到了模板 |
| [2026-02-19-visual-brainstorming-refactor.md](file://e:/WorkStation/MyProject/superpowers/docs/superpowers/plans/2026-02-19-visual-brainstorming-refactor.md) | **后续重构计划**——把 Express 换成纯 Node.js |
| [2026-03-11-zero-dep-brainstorm-server.md](file://e:/WorkStation/MyProject/superpowers/docs/superpowers/plans/2026-03-11-zero-dep-brainstorm-server.md) | **再后续**——实现零依赖服务器 |

**有趣的时间线**：
1. **2026-01-17**：本文档（最初设计）
2. **2026-02-19**：重构（去掉 Express）
3. **2026-03-11**：再重构（实现零依赖服务器）

**这是一个"演进式架构"**——从有依赖到少依赖到零依赖，每一步都是对前一步的反思。

## 📝 九、文档状态评价

| 评价维度 | 评分 | 说明 |
|---------|------|------|
| **架构创新度** | ⭐⭐⭐⭐⭐ | 文件 + stdout 桥接模式极具巧思 |
| **任务清晰度** | ⭐⭐⭐⭐⭐ | 每个任务都明确可执行 |
| **测试完整性** | ⭐⭐⭐⭐⭐ | 4 个测试覆盖完整数据流 |
| **可扩展性** | ⭐⭐⭐⭐ | HTML 模式库设计良好 |
| **跨平台考虑** | ⭐⭐ | 假设 Unix /tmp，未考虑 Windows |
| **总分** | **4.3/5** | 一份创新性极强、工程性扎实的实施计划 |

## 🎓 十、给读者的启示

### 如果你是 AI Coding 工具开发者

1. **进程外 UI 是扩展 AI 能力的有效路径**——无需修改 AI 核心
2. **文件系统 + stdout 是被低估的 IPC 机制**——简单、跨平台、跨语言
3. **"零配置"辅助库是优秀 UX 的核心**——让用户写 HTML 就能交互

### 如果你是项目维护者

1. **每个"完美"的设计都有后续重构**——本文档的演进路径值得研究
2. **依赖项是技术债的来源**——后续两次重构都在"减依赖"
3. **好的技能包含完整的"模式库"**——visual-companion.md 是典范

### 如果你是 AI Agent 实施者

1. **写 UI 时优先使用 `data-choice` 等语义化属性**——helper.js 会自动捕获
2. **检查后台任务输出获取用户响应**——不需要新的 API
3. **在头脑风暴中考虑使用视觉伴侣**——尤其涉及 UI/UX 设计时

## 🚀 十一、可能的后续工作

1. **跨平台支持**：Windows 上用 `%TEMP%` 替代 `/tmp`
2. **端口冲突检测**：自动选择空闲端口
3. **多用户协作**：支持多个浏览器同时连接
4. **可访问性增强**：ARIA 标签、键盘导航
5. **状态恢复**：服务器崩溃后浏览器自动恢复现场
