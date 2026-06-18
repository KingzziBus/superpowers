# 零依赖 Brainstorm 服务器

将脑暴伴侣服务器当前 vendored（内嵌）的 `node_modules`（express、ws、chokidar——共 714 个入仓文件）替换为仅使用 Node.js 内置模块的单一零依赖 `server.js`。

## 动机

将 `node_modules` 内嵌入 git 仓库会带来供应链风险：冻结的依赖得不到安全补丁；714 个第三方代码文件未经审计就被提交；对内嵌代码的修改看起来像普通提交。虽然实际风险较低（仅本地主机的开发服务器），但消除它很简单。

## 架构

单个 `server.js` 文件（约 250-300 行），使用 `http`、`crypto`、`fs` 和 `path`。该文件承担两个角色：

- **直接运行时**（`node server.js`）：启动 HTTP/WebSocket 服务器
- **作为模块加载时**（`require('./server.js')`）：导出 WebSocket 协议函数供单元测试使用

### WebSocket 协议

仅实现 RFC 6455 的文本帧：

**握手（Handshake）：** 使用 SHA-1 + RFC 6455 的魔术 GUID，根据客户端的 `Sec-WebSocket-Key` 计算 `Sec-WebSocket-Accept`。返回 101 Switching Protocols。

**帧解码（客户端到服务器）：** 处理三种带掩码的长度编码：
- 小型：负载 < 126 字节
- 中型：126-65535 字节（16 位扩展）
- 大型：> 65535 字节（64 位扩展）

使用 4 字节掩码键对负载做 XOR 解掩。返回 `{ opcode, payload, bytesConsumed }`，缓冲区不完整时返回 `null`。拒绝未带掩码的帧。

**帧编码（服务器到客户端）：** 不带掩码的帧，同样支持三种长度编码。

**支持的 Opcodes：** TEXT (0x01)、CLOSE (0x08)、PING (0x09)、PONG (0x0A)。未识别的 opcode 返回状态码 1003（Unsupported Data）的关闭帧。

**刻意跳过：** 二进制帧、分片消息、扩展（permessage-deflate）、子协议。这些对 localhost 客户端间的小型 JSON 文本消息来说并非必需。扩展和子协议在握手中协商——不声明它们，则永远不会激活。

**缓冲区累积：** 每个连接维护一个缓冲区。在 `data` 事件中追加并循环调用 `decodeFrame`，直到返回 null 或缓冲区为空。

### HTTP 服务器

三条路由：

1. **`GET /`** — 按 mtime 从屏幕目录提供最新的 `.html` 文件。检测完整文档与片段，将片段包裹在 frame 模板中，注入 helper.js。返回 `text/html`。当不存在任何 `.html` 文件时，提供硬编码的等待页面（"Waiting for Claude to push a screen..."），并注入 helper.js。
2. **`GET /files/*`** — 从屏幕目录提供静态文件，通过硬编码的扩展名映射表查询 MIME 类型（html、css、js、png、jpg、gif、svg、json）。未找到时返回 404。
3. **其他所有** — 404。

WebSocket 升级通过 HTTP 服务器的 `'upgrade'` 事件处理，与请求处理器分离。

### 配置

环境变量（全部可选）：

- `BRAINSTORM_PORT` — 绑定的端口（默认：随机高端口 49152-65535）
- `BRAINSTORM_HOST` — 绑定的网络接口（默认：`127.0.0.1`）
- `BRAINSTORM_URL_HOST` — 启动 JSON 中 URL 使用的主机名（默认：当 host 为 `127.0.0.1` 时为 `localhost`，否则与 host 相同）
- `BRAINSTORM_DIR` — 屏幕目录路径（默认：`/tmp/brainstorm`）

### 启动序列

1. 若 `SCREEN_DIR` 不存在则创建（`mkdirSync` 递归）
2. 从 `__dirname` 加载 frame 模板和 helper.js
3. 在配置的 host:port 上启动 HTTP 服务器
4. 在 `SCREEN_DIR` 上启动 `fs.watch`
5. 成功 listen 后，向 stdout 输出 `server-started` JSON：`{ type, port, host, url_host, url, screen_dir }`
6. 将同样的 JSON 写入 `SCREEN_DIR/.server-info`，以便代理在 stdout 被隐藏（后台执行）时仍能获取连接详情

### 应用层 WebSocket 消息

当来自客户端的 TEXT 帧到达时：

1. 解析为 JSON。解析失败则记录到 stderr 并继续。
2. 作为 `{ source: 'user-event', ...event }` 记录到 stdout。
3. 如果事件包含 `choice` 属性，将 JSON 追加到 `SCREEN_DIR/.events`（每行一个事件）。

### 文件监听

`fs.watch(SCREEN_DIR)` 取代 chokidar。HTML 文件事件：

- 新文件（`rename` 事件且文件存在）：若存在则删除 `.events` 文件（`unlinkSync`），将 `screen-added` 作为 JSON 记录到 stdout
- 文件变更（`change` 事件）：将 `screen-updated` 作为 JSON 记录到 stdout（**不**清空 `.events`）
- 两种事件：向所有连接的 WebSocket 客户端发送 `{ type: 'reload' }`

按文件名做约 100ms 的防抖，防止重复事件（macOS 和 Linux 上常见）。

### 错误处理

- WebSocket 客户端发来的格式错误 JSON：记录到 stderr，继续
- 未处理的 opcode：以 1003 状态码关闭
- 客户端断开：从广播集合中移除
- `fs.watch` 错误：记录到 stderr，继续
- 无优雅停机逻辑——shell 脚本通过 SIGTERM 处理进程生命周期

## 变更内容

| 之前 | 之后 |
|---|---|
| `index.js` + `package.json` + `package-lock.json` + 714 个 `node_modules` 文件 | `server.js`（单一文件） |
| express、ws、chokidar 依赖 | 无 |
| 无静态文件服务 | `/files/*` 从屏幕目录提供文件 |

## 保持不变

- `helper.js`——无变更
- `frame-template.html`——无变更
- `start-server.sh`——一行更新：`index.js` 改为 `server.js`
- `stop-server.sh`——无变更
- `visual-companion.md`——无变更
- 所有现有服务器行为和外部契约

## 平台兼容性

- `server.js` 仅使用跨平台的 Node 内置模块
- `fs.watch` 在 macOS、Linux 和 Windows 上对单个扁平目录可靠
- Shell 脚本需要 bash（Windows 上的 Git Bash，Claude Code 所需）

## 测试

**单元测试**（`ws-protocol.test.js`）：通过 `require('server.js')` 导出直接测试 WebSocket 帧编解码、握手计算和协议边界场景。

**集成测试**（`server.test.js`）：测试完整服务器行为——HTTP 服务、WebSocket 通信、文件监听、脑暴工作流。使用 `ws` npm 包作为仅供测试的客户端依赖（不交付给最终用户）。
