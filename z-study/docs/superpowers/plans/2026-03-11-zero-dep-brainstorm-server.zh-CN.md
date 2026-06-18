# 零依赖 Brainstorm 服务器实施计划

> **面向智能体（Agentic）工作者：** 必须：使用 `superpowers:subagent-driven-development`（如果支持子智能体）或 `superpowers:executing-plans` 技能来执行本计划。步骤使用复选框（`- [ ]`）语法进行跟踪。

**目标：** 用仅依赖 Node 内置模块的单一零依赖 `server.js` 替换脑暴服务器（brainstorm server）当前的 vendored node_modules 方案。

**架构：** 单个文件，包含 WebSocket 协议（RFC 6455 文本帧）、HTTP 服务器（`http` 模块）以及文件监听（`fs.watch`）。当作为模块导入时，导出协议函数用于单元测试。

**技术栈：** 仅使用 Node.js 内置模块：`http`、`crypto`、`fs`、`path`

**Spec（规格）：** `docs/superpowers/specs/2026-03-11-zero-dep-brainstorm-server-design.md`

**已有测试：** `tests/brainstorm-server/ws-protocol.test.js`（单元测试）、`tests/brainstorm-server/server.test.js`（集成测试）

---

## 文件清单

- **新建：** `skills/brainstorming/scripts/server.js` — 零依赖的替代实现
- **修改：** `skills/brainstorming/scripts/start-server.sh:94,100` — 将 `index.js` 改为 `server.js`
- **修改：** `.gitignore:6` — 删除 `!skills/brainstorming/scripts/node_modules/` 例外项
- **删除：** `skills/brainstorming/scripts/index.js`
- **删除：** `skills/brainstorming/scripts/package.json`
- **删除：** `skills/brainstorming/scripts/package-lock.json`
- **删除：** `skills/brainstorming/scripts/node_modules/`（共 714 个文件）
- **无需修改：** `skills/brainstorming/scripts/helper.js`、`skills/brainstorming/scripts/frame-template.html`、`skills/brainstorming/scripts/stop-server.sh`

---

## Chunk 1：WebSocket 协议层

### Task 1：实现 WebSocket 协议导出

**文件：**
- 新建：`skills/brainstorming/scripts/server.js`
- 测试：`tests/brainstorm-server/ws-protocol.test.js`（已存在）

- [ ] **第 1 步：** 在 `server.js` 中创建 OPCODES 常量和 `computeAcceptKey` 函数

```js
const crypto = require('crypto');

const OPCODES = { TEXT: 0x01, CLOSE: 0x08, PING: 0x09, PONG: 0x0A };
const WS_MAGIC = '258EAFA5-E914-47DA-95CA-C5AB0DC85B11';

function computeAcceptKey(clientKey) {
  return crypto.createHash('sha1').update(clientKey + WS_MAGIC).digest('base64');
}
```

- [ ] **第 2 步：** 实现 `encodeFrame`

服务器帧永远不进行掩码处理。三种长度编码：
- 负载 < 126：2 字节头（FIN+操作码，长度）
- 126-65535：4 字节头（FIN+操作码，126，16 位长度）
- &gt; 65535：10 字节头（FIN+操作码，127，64 位长度）

```js
function encodeFrame(opcode, payload) {
  const fin = 0x80;
  const len = payload.length;
  let header;

  if (len < 126) {
    header = Buffer.alloc(2);
    header[0] = fin | opcode;
    header[1] = len;
  } else if (len < 65536) {
    header = Buffer.alloc(4);
    header[0] = fin | opcode;
    header[1] = 126;
    header.writeUInt16BE(len, 2);
  } else {
    header = Buffer.alloc(10);
    header[0] = fin | opcode;
    header[1] = 127;
    header.writeBigUInt64BE(BigInt(len), 2);
  }

  return Buffer.concat([header, payload]);
}
```

- [ ] **第 3 步：** 实现 `decodeFrame`

客户端帧始终带掩码。返回 `{ opcode, payload, bytesConsumed }`，若不完整则返回 `null`，未带掩码的帧则抛出异常。

```js
function decodeFrame(buffer) {
  if (buffer.length < 2) return null;

  const firstByte = buffer[0];
  const secondByte = buffer[1];
  const opcode = firstByte & 0x0F;
  const masked = (secondByte & 0x80) !== 0;
  let payloadLen = secondByte & 0x7F;
  let offset = 2;

  if (!masked) throw new Error('Client frames must be masked');

  if (payloadLen === 126) {
    if (buffer.length < 4) return null;
    payloadLen = buffer.readUInt16BE(2);
    offset = 4;
  } else if (payloadLen === 127) {
    if (buffer.length < 10) return null;
    payloadLen = Number(buffer.readBigUInt64BE(2));
    offset = 10;
  }

  const maskOffset = offset;
  const dataOffset = offset + 4;
  const totalLen = dataOffset + payloadLen;
  if (buffer.length < totalLen) return null;

  const mask = buffer.slice(maskOffset, dataOffset);
  const data = Buffer.alloc(payloadLen);
  for (let i = 0; i < payloadLen; i++) {
    data[i] = buffer[dataOffset + i] ^ mask[i % 4];
  }

  return { opcode, payload: data, bytesConsumed: totalLen };
}
```

- [ ] **第 4 步：** 在文件底部添加模块导出

```js
module.exports = { computeAcceptKey, encodeFrame, decodeFrame, OPCODES };
```

- [ ] **第 5 步：** 运行单元测试

执行：`cd tests/brainstorm-server && node ws-protocol.test.js`
预期：所有测试通过（握手、编码、解码、边界、边缘情况）

- [ ] **第 6 步：** 提交

```bash
git add skills/brainstorming/scripts/server.js
git commit -m "Add WebSocket protocol layer for zero-dep brainstorm server"
```

---

## Chunk 2：HTTP 服务器与应用逻辑

### Task 2：添加 HTTP 服务器、文件监听和 WebSocket 连接处理

**文件：**
- 修改：`skills/brainstorming/scripts/server.js`
- 测试：`tests/brainstorm-server/server.test.js`（已存在）

- [ ] **第 1 步：** 在 `server.js` 顶部（require 之后）添加配置和常量

```js
const http = require('http');
const fs = require('fs');
const path = require('path');

const PORT = process.env.BRAINSTORM_PORT || (49152 + Math.floor(Math.random() * 16383));
const HOST = process.env.BRAINSTORM_HOST || '127.0.0.1';
const URL_HOST = process.env.BRAINSTORM_URL_HOST || (HOST === '127.0.0.1' ? 'localhost' : HOST);
const SCREEN_DIR = process.env.BRAINSTORM_DIR || '/tmp/brainstorm';

const MIME_TYPES = {
  '.html': 'text/html', '.css': 'text/css', '.js': 'application/javascript',
  '.json': 'application/json', '.png': 'image/png', '.jpg': 'image/jpeg',
  '.jpeg': 'image/jpeg', '.gif': 'image/gif', '.svg': 'image/svg+xml'
};
```

- [ ] **第 2 步：** 在模块作用域添加 `WAITING_PAGE`、模板加载和辅助函数

在模块作用域加载 `frameTemplate` 和 `helperInjection`，以便 `wrapInFrame` 和 `handleRequest` 访问。它们仅从 `__dirname`（scripts 目录）读取文件，无论作为模块导入还是直接运行都有效。

```js
const WAITING_PAGE = `<!DOCTYPE html>
<html>
<head><title>Brainstorm Companion</title>
<style>body { font-family: system-ui, sans-serif; padding: 2rem; max-width: 800px; margin: 0 auto; }
h1 { color: #333; } p { color: #666; }</style>
</head>
<body><h1>Brainstorm Companion</h1>
<p>Waiting for Claude to push a screen...</p></body></html>`;

const frameTemplate = fs.readFileSync(path.join(__dirname, 'frame-template.html'), 'utf-8');
const helperScript = fs.readFileSync(path.join(__dirname, 'helper.js'), 'utf-8');
const helperInjection = '<script>\n' + helperScript + '\n</script>';

function isFullDocument(html) {
  const trimmed = html.trimStart().toLowerCase();
  return trimmed.startsWith('<!doctype') || trimmed.startsWith('<html');
}

function wrapInFrame(content) {
  return frameTemplate.replace('<!-- CONTENT -->', content);
}

function getNewestScreen() {
  const files = fs.readdirSync(SCREEN_DIR)
    .filter(f => f.endsWith('.html'))
    .map(f => {
      const fp = path.join(SCREEN_DIR, f);
      return { path: fp, mtime: fs.statSync(fp).mtime.getTime() };
    })
    .sort((a, b) => b.mtime - a.mtime);
  return files.length > 0 ? files[0].path : null;
}
```

- [ ] **第 3 步：** 添加 HTTP 请求处理器

```js
function handleRequest(req, res) {
  if (req.method === 'GET' && req.url === '/') {
    const screenFile = getNewestScreen();
    let html = screenFile
      ? (raw => isFullDocument(raw) ? raw : wrapInFrame(raw))(fs.readFileSync(screenFile, 'utf-8'))
      : WAITING_PAGE;

    if (html.includes('</body>')) {
      html = html.replace('</body>', helperInjection + '\n</body>');
    } else {
      html += helperInjection;
    }

    res.writeHead(200, { 'Content-Type': 'text/html' });
    res.end(html);
  } else if (req.method === 'GET' && req.url.startsWith('/files/')) {
    const fileName = req.url.slice(7); // strip '/files/'
    const filePath = path.join(SCREEN_DIR, path.basename(fileName));
    if (!fs.existsSync(filePath)) {
      res.writeHead(404);
      res.end('Not found');
      return;
    }
    const ext = path.extname(filePath).toLowerCase();
    const contentType = MIME_TYPES[ext] || 'application/octet-stream';
    res.writeHead(200, { 'Content-Type': contentType });
    res.end(fs.readFileSync(filePath));
  } else {
    res.writeHead(404);
    res.end('Not found');
  }
}
```

- [ ] **第 4 步：** 添加 WebSocket 连接处理

```js
const clients = new Set();

function handleUpgrade(req, socket) {
  const key = req.headers['sec-websocket-key'];
  if (!key) { socket.destroy(); return; }

  const accept = computeAcceptKey(key);
  socket.write(
    'HTTP/1.1 101 Switching Protocols\r\n' +
    'Upgrade: websocket\r\n' +
    'Connection: Upgrade\r\n' +
    'Sec-WebSocket-Accept: ' + accept + '\r\n\r\n'
  );

  let buffer = Buffer.alloc(0);
  clients.add(socket);

  socket.on('data', (chunk) => {
    buffer = Buffer.concat([buffer, chunk]);
    while (buffer.length > 0) {
      let result;
      try {
        result = decodeFrame(buffer);
      } catch (e) {
        socket.end(encodeFrame(OPCODES.CLOSE, Buffer.alloc(0)));
        clients.delete(socket);
        return;
      }
      if (!result) break;
      buffer = buffer.slice(result.bytesConsumed);

      switch (result.opcode) {
        case OPCODES.TEXT:
          handleMessage(result.payload.toString());
          break;
        case OPCODES.CLOSE:
          socket.end(encodeFrame(OPCODES.CLOSE, Buffer.alloc(0)));
          clients.delete(socket);
          return;
        case OPCODES.PING:
          socket.write(encodeFrame(OPCODES.PONG, result.payload));
          break;
        case OPCODES.PONG:
          break;
        default:
          // Unsupported opcode — close with 1003
          const closeBuf = Buffer.alloc(2);
          closeBuf.writeUInt16BE(1003);
          socket.end(encodeFrame(OPCODES.CLOSE, closeBuf));
          clients.delete(socket);
          return;
      }
    }
  });

  socket.on('close', () => clients.delete(socket));
  socket.on('error', () => clients.delete(socket));
}

function handleMessage(text) {
  let event;
  try {
    event = JSON.parse(text);
  } catch (e) {
    console.error('Failed to parse WebSocket message:', e.message);
    return;
  }
  console.log(JSON.stringify({ source: 'user-event', ...event }));
  if (event.choice) {
    const eventsFile = path.join(SCREEN_DIR, '.events');
    fs.appendFileSync(eventsFile, JSON.stringify(event) + '\n');
  }
}

function broadcast(msg) {
  const frame = encodeFrame(OPCODES.TEXT, Buffer.from(JSON.stringify(msg)));
  for (const socket of clients) {
    try { socket.write(frame); } catch (e) { clients.delete(socket); }
  }
}
```

- [ ] **第 5 步：** 添加防抖定时器 Map

```js
const debounceTimers = new Map();
```

文件监听逻辑被内联到 `startServer`（第 6 步）中，使监听器生命周期与服务器生命周期保持一致，并按规范包含 `error` 处理器。

- [ ] **第 6 步：** 添加 `startServer` 函数和条件式主入口

`frameTemplate` 和 `helperInjection` 已经在模块作用域中（第 2 步）。`startServer` 仅负责创建屏幕目录、启动 HTTP 服务器、文件监听器并输出启动信息。

```js
function startServer() {
  if (!fs.existsSync(SCREEN_DIR)) fs.mkdirSync(SCREEN_DIR, { recursive: true });

  const server = http.createServer(handleRequest);
  server.on('upgrade', handleUpgrade);

  const watcher = fs.watch(SCREEN_DIR, (eventType, filename) => {
    if (!filename || !filename.endsWith('.html')) return;
    if (debounceTimers.has(filename)) clearTimeout(debounceTimers.get(filename));
    debounceTimers.set(filename, setTimeout(() => {
      debounceTimers.delete(filename);
      const filePath = path.join(SCREEN_DIR, filename);
      if (eventType === 'rename' && fs.existsSync(filePath)) {
        const eventsFile = path.join(SCREEN_DIR, '.events');
        if (fs.existsSync(eventsFile)) fs.unlinkSync(eventsFile);
        console.log(JSON.stringify({ type: 'screen-added', file: filePath }));
      } else if (eventType === 'change') {
        console.log(JSON.stringify({ type: 'screen-updated', file: filePath }));
      }
      broadcast({ type: 'reload' });
    }, 100));
  });
  watcher.on('error', (err) => console.error('fs.watch error:', err.message));

  server.listen(PORT, HOST, () => {
    const info = JSON.stringify({
      type: 'server-started', port: Number(PORT), host: HOST,
      url_host: URL_HOST, url: 'http://' + URL_HOST + ':' + PORT,
      screen_dir: SCREEN_DIR
    });
    console.log(info);
    fs.writeFileSync(path.join(SCREEN_DIR, '.server-info'), info + '\n');
  });
}

if (require.main === module) {
  startServer();
}
```

- [ ] **第 7 步：** 运行集成测试

测试目录中已有 `package.json` 并将 `ws` 作为依赖。如有需要先安装，再运行测试。

执行：`cd tests/brainstorm-server && npm install && node server.test.js`
预期：所有测试通过

- [ ] **第 8 步：** 提交

```bash
git add skills/brainstorming/scripts/server.js
git commit -m "Add HTTP server, WebSocket handling, and file watching to server.js"
```

---

## Chunk 3：切换与清理

### Task 3：更新 `start-server.sh` 并删除旧文件

**文件：**
- 修改：`skills/brainstorming/scripts/start-server.sh:94,100`
- 修改：`.gitignore:6`
- 删除：`skills/brainstorming/scripts/index.js`
- 删除：`skills/brainstorming/scripts/package.json`
- 删除：`skills/brainstorming/scripts/package-lock.json`
- 删除：`skills/brainstorming/scripts/node_modules/`（整个目录）

- [ ] **第 1 步：** 更新 `start-server.sh` —— 将 `index.js` 改为 `server.js`

需要修改两行：

第 94 行：`env BRAINSTORM_DIR="$SCREEN_DIR" BRAINSTORM_HOST="$BIND_HOST" BRAINSTORM_URL_HOST="$URL_HOST" node server.js`

第 100 行：`nohup env BRAINSTORM_DIR="$SCREEN_DIR" BRAINSTORM_HOST="$BIND_HOST" BRAINSTORM_URL_HOST="$URL_HOST" node server.js > "$LOG_FILE" 2>&1 &`

- [ ] **第 2 步：** 移除 `.gitignore` 中关于 `node_modules` 的例外项

在 `.gitignore` 中，删除第 6 行：`!skills/brainstorming/scripts/node_modules/`

- [ ] **第 3 步：** 删除旧文件

```bash
git rm skills/brainstorming/scripts/index.js
git rm skills/brainstorming/scripts/package.json
git rm skills/brainstorming/scripts/package-lock.json
git rm -r skills/brainstorming/scripts/node_modules/
```

- [ ] **第 4 步：** 运行两套测试

执行：`cd tests/brainstorm-server && node ws-protocol.test.js && node server.test.js`
预期：所有测试通过

- [ ] **第 5 步：** 提交

```bash
git add skills/brainstorming/scripts/ .gitignore
git commit -m "Remove vendored node_modules, swap to zero-dep server.js"
```

### Task 4：手动冒烟测试

- [ ] **第 1 步：** 手动启动服务器

```bash
cd skills/brainstorming/scripts
BRAINSTORM_DIR=/tmp/brainstorm-smoke BRAINSTORM_PORT=9876 node server.js
```

预期：输出 `server-started` JSON，其中端口为 9876

- [ ] **第 2 步：** 在浏览器中打开 `http://localhost:9876`

预期：显示等待页面，文案为 "Waiting for Claude to push a screen..."

- [ ] **第 3 步：** 在屏幕目录中写入一个 HTML 文件

```bash
echo '<h2>Hello from smoke test</h2>' > /tmp/brainstorm-smoke/test.html
```

预期：浏览器自动刷新，显示 "Hello from smoke test"，包裹在 frame 模板中

- [ ] **第 4 步：** 验证 WebSocket 工作正常 —— 检查浏览器控制台

打开浏览器开发者工具。WebSocket 连接应显示为已连接（控制台无错误）。frame 模板的状态指示器应显示 "Connected"。

- [ ] **第 5 步：** 通过 `Ctrl-C` 停止服务器，并清理

```bash
rm -rf /tmp/brainstorm-smoke
```
