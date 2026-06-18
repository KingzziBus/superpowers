# tests/brainstorm-server/ 深度分析报告

> 目录: `tests/brainstorm-server/`（5 个文件）
> 角色: brainstorming 技能核心组件的"零依赖"实现测试套件
> 测试对象: `skills/brainstorming/scripts/server.cjs`（HTTP + WebSocket 视觉服务器）

---

## 1. 基本信息

| 字段 | 值 |
|------|-----|
| 目录路径 | `tests/brainstorm-server/` |
| 文件数量 | 5（4 测试 + 1 配置） |
| 总规模 | ~40 KB |
| 测试对象 | `skills/brainstorming/scripts/server.cjs`（zero-dep HTTP+WS） |
| 协议测试 | WebSocket RFC 6455 协议层 |
| 集成测试 | 启动真实 server，测试 HTTP/WS/文件监听 |
| 平台测试 | Windows MSYS2 lifecycle 行为 |
| 依赖 | Node.js + `ws` 包（仅测试用） |

### 5 个文件清单

| 文件 | 行数 | 角色 |
|------|------|------|
| `package.json` | 11 | npm 配置（仅声明 `ws` 依赖） |
| `server.test.js` | 428 | 集成测试：HTTP serving、WS 通信、文件监听 |
| `ws-protocol.test.js` | 393 | 单元测试：WebSocket RFC 6455 帧编解码 |
| `windows-lifecycle.test.sh` | 352 | 平台测试：Windows MSYS2 60 秒生命周期检查 |
| `package-lock.json` | - | npm 锁文件 |

---

## 2. 目的与背景

`brainstorming` 技能依赖一个**本地 HTTP + WebSocket 视觉服务器**（zero-dep），用于在浏览器中向用户展示设计选项、收集用户选择。这个服务器是 brainstorming 体验的核心：

- **HTTP**：serve HTML 屏幕（设计选项、布局对比、问题可视化）
- **WebSocket**：接收用户的点击/选择事件
- **文件监听**：检测 `content/` 目录的新 HTML 屏幕，触发浏览器 reload

**这个测试目录的存在是为了**：

1. **保障 zero-dep 承诺**：服务器只用 Node.js 内置模块（`http`、`fs`、`crypto`）实现 WebSocket
2. **RFC 6455 合规性**：WS 协议实现必须通过标准测试向量
3. **Windows 兼容性**：MSYS2 PID 命名空间问题导致 OWNER_PID 监控需特殊处理
4. **回归保护**：防止 WebSocket 帧大小、长度编码、mask 位等细节回归

---

## 3. 核心技术决策

### 3.1 为什么用 `ws` npm 包作为测试客户端？

来自 `server.test.js` 头部注释：

> Uses the `ws` npm package as a test client (test-only dependency, not shipped to end users).

**核心策略**：

- **服务器**：zero-dep（只用 Node.js 内置 `http` + `crypto`）
- **测试客户端**：用 `ws`（标准 npm 包，well-tested）
- **包大小**：测试需要 `ws` 依赖，但产品代码不依赖

**这是"测试可以重依赖，生产必须轻依赖"的工程原则**——测试需要可靠的参照系（成熟客户端），生产需要最小化依赖（兼容性 + 包大小）。

### 3.2 TDD 风格的"先测试后实现"

来自 `ws-protocol.test.js` 行 22-29：

```javascript
try {
  ws = require(SERVER_PATH);
} catch (e) {
  // Module doesn't exist yet (TDD — tests written before implementation)
  console.error(`Cannot load ${SERVER_PATH}: ${e.message}`);
  console.error('This is expected if running tests before implementation.');
  process.exit(1);
}
```

**TDD 实践**：

- 测试**先于**实现存在
- 文件头注释说"tests written before implementation"
- 如果模块不存在，**明确告知**这是 TDD 预期情况，而不是 crash

**这是一个"测试驱动设计"的范例**——测试是协议规格的"可执行文档"。

### 3.3 RFC 6455 协议测试向量的精确性

`ws-protocol.test.js` 行 50-56 引用了 RFC 6455 标准测试向量：

```javascript
test('computeAcceptKey produces correct RFC 6455 accept value', () => {
    // RFC 6455 Section 4.2.2 example
    // The magic GUID is "258EAFA5-E914-47DA-95CA-C5AB0DC85B11"
    const clientKey = 'dGhlIHNhbXBsZSBub25jZQ==';
    const expected = 's3pPLMBiTxaQ9kYGzzhZRbK+xOo=';
    assert.strictEqual(ws.computeAcceptKey(clientKey), expected);
});
```

**这是"协议实现必须通过标准向量"的金科玉律**——不依赖自己写的客户端，而是用 IETF 文档中的标准值。

### 3.4 边界值测试的"5 个关键尺寸"

来自 `ws-protocol.test.js`：

- 0 字节（empty frame）
- 125 字节（max small frame，< 126 阈值）
- 126 字节（boundary，刚好进 16-bit extended length）
- 65535 字节（max 16-bit length）
- 65536 字节（boundary，进 64-bit extended length）
- 70000 字节（典型 large frame）

**覆盖**：

- `encodeFrame` 在 6 个尺寸点都正确
- `decodeFrame` 在 4 个尺寸点都正确
- 不重不漏

**这是"边界值驱动测试"的工程实践**——协议 bug 常在边界处。

### 3.5 WS 测试客户端手写"伪装"成 RFC 客户端

来自 `ws-protocol.test.js` 行 149-179（`makeClientFrame` 函数）：

```javascript
function makeClientFrame(opcode, payload, fin = true) {
    const buf = Buffer.from(payload);
    const mask = crypto.randomBytes(4);
    const masked = Buffer.alloc(buf.length);
    for (let i = 0; i < buf.length; i++) {
      masked[i] = buf[i] ^ mask[i % 4];
    }
    // ... build header with mask bit set
}
```

**核心点**：

- **手写帧编码**：不依赖 `ws` 包，避免"测试用 `ws` 编帧、`ws` 解帧"循环验证
- **随机 mask key**：4 字节随机，与真实浏览器行为一致
- **XOR 解码**：mask = payload[i] XOR mask_key[i%4]（RFC 6455 标准）

**为什么手写而非用 `ws`**：

- "用 `ws` 测 `ws`" = 同一实现的两个版本互相验证，不能发现真实 bug
- 手写帧 = 独立验证，符合"独立参照系"原则

### 3.6 `bailout` 模式：测试运行遇错立即退出

`ws-protocol.test.js` 行 33-44：

```javascript
function runTests() {
    let passed = 0;
    let failed = 0;
    function test(name, fn) {
      try {
        fn();
        console.log(`  PASS: ${name}`);
        passed++;
      } catch (e) {
        console.log(`  FAIL: ${name}`);
        console.log(`    ${e.message}`);
        failed++;
      }
    }
    // ...
    if (failed > 0) process.exit(1);
}
```

**关键差异**：

- 与 `server.test.js` 不同，`ws-protocol.test.js` **不立即 bail**
- 每个 test 独立 try/catch
- 最终汇总 `failed > 0` 时退出
- **目的**：让协议测试**全跑完**，不因一个失败而漏掉其他失败的诊断

### 3.7 Windows 平台检测的"双保险"

`windows-lifecycle.test.sh` 行 100-106：

```bash
is_windows="false"
case "${OSTYPE:-}" in
    msys*|cygwin*|mingw*) is_windows="true" ;;
esac
if [[ -n "${MSYSTEM:-}" ]]; then
    is_windows="true"
fi
```

**为什么双检查**：

- `OSTYPE` 可能在某些 MSYS2 配置下未设置
- `MSYSTEM` 是 MSYS2 主动设置的环境变量
- 两者都检查 = 任一信号 = Windows

**这是"防御性平台检测"的最佳实践**。

### 3.8 Fake `node` 注入：捕获环境变量

来自 `windows-lifecycle.test.sh` 行 148-166：

```bash
FAKE_NODE_DIR="$TEST_DIR/fake-bin"
mkdir -p "$FAKE_NODE_DIR"
cat > "$FAKE_NODE_DIR/node" <<'FAKENODE'
#!/usr/bin/env bash
echo "CAPTURED_OWNER_PID=${BRAINSTORM_OWNER_PID:-__UNSET__}"
exit 0
FAKENODE
chmod +x "$FAKE_NODE_DIR/node"

captured=$(PATH="$FAKE_NODE_DIR:$PATH" bash "$START_SCRIPT" --project-dir "$TEST_DIR/session" --foreground 2>/dev/null || true)
```

**核心技巧**：

- 用 `PATH="$FAKE_NODE_DIR:$PATH"` 注入"假 node"到 PATH
- `start-server.sh` 会找到假 `node` 而非真 `node`
- 假 `node` 只打印环境变量然后退出
- 测试**捕获**输出，验证 `BRAINSTORM_OWNER_PID` 传递正确

**这是"测试依赖行为而非副作用"的工程模式**——fake `node` 模拟 node，但只暴露我们关心的行为。

### 3.9 60 秒生命周期检查的核心防御

来自 `windows-lifecycle.test.sh` 行 209-255：

- 启动 server，**等待 75 秒**（> 60 秒阈值）
- 验证 server **还活着**
- 验证 HTTP **还响应**
- 验证日志中**没有**"owner process exited" 误杀消息

**反例控制**（行 262-302）：

- 故意用**坏的** `OWNER_PID`（如 99999）
- 启动 server
- 等 75 秒
- 验证 server **已死**（生命周期检查正确触发）
- 验证日志有 "owner process exited"

**这是"正面 + 反面"对照实验设计**——证明行为只在应该时触发。

### 3.10 macOS 路径解析的特殊处理

`server.test.js` 行 173：

```javascript
TEST_PROJECT_REAL=$(cd "$TEST_PROJECT" && pwd -P)
SESSION_DIR="$HOME/.claude/projects/$(echo "$TEST_PROJECT_REAL" | sed 's|[^a-zA-Z0-9]|-|g')"
```

**核心问题**：

- macOS `mktemp` 返回 `/var/folders/...`
- 但 Claude 把 cwd 规范化为 `/private/var/folders/...`
- 不解析 → 找不到 session transcript → 测试失败

**解决**：

- `pwd -P`：解析所有符号链接，得到真实路径
- `sed 's|[^a-zA-Z0-9]|-|`：Claude 的目录命名规范（非字母数字 → `-`）

**这是"跨平台真实路径"的精确处理**。

---

## 4. 文件结构剖析

```
┌─ package.json (11 行) ──────────────────────────────────┐
│ 唯一依赖: ws ^8.19.0 (测试客户端)                       │
│ script: node server.test.js                             │
└──────────────────────────────────────────────────────────┘

┌─ ws-protocol.test.js (393 行) ─ 协议层单元测试 ─────────┐
│ Handshake:                                              │
│   - computeAcceptKey (RFC 6455 §4.2.2)                │
│   - Random keys → valid base64                         │
│                                                          │
│ Frame Encoding (server → client):                       │
│   - 5 sizes: 0, 5, 125, 126, 200, 70000                │
│   - CLOSE, PONG, never-masked                           │
│                                                          │
│ Frame Decoding (client → server):                       │
│   - 4 sizes masked                                      │
│   - Incomplete frames → null                            │
│   - Reject unmasked client frames                       │
│   - Multiple frames in one buffer                       │
│   - Unmask with all mask byte values                    │
│                                                          │
│ Boundary at 65535/65536:                                │
│   - 16-bit vs 64-bit length encoding                   │
│                                                          │
│ Close frame: status code + reason                       │
│ JSON roundtrip                                          │
└──────────────────────────────────────────────────────────┘

┌─ server.test.js (428 行) ─ 集成测试 ─────────────────────┐
│ 启动 server (child_process.spawn node)                  │
│ 等待 "server-started" stdout 消息                       │
│                                                          │
│ Server Startup (3 tests):                               │
│   - server-started JSON 格式正确                       │
│   - state/server-info 写入正确                          │
│                                                          │
│ HTTP Serving (7 tests):                                 │
│   - 等待页 (no screens)                                │
│   - helper.js 注入                                     │
│   - Content-Type 正确                                  │
│   - 完整 HTML 文档不包装                               │
│   - 片段 wrap 到 frame template                        │
│   - 最新文件按 mtime 排序                              │
│   - 忽略非 HTML                                       │
│   - 404 路径                                           │
│                                                          │
│ WebSocket (5 tests):                                    │
│   - Upgrade on /                                       │
│   - 事件传到 stdout + source field                     │
│   - choice 写入 state/events                          │
│   - 非 choice 不写 events                             │
│   - 多客户端并发                                      │
│   - closed 客户端清理                                 │
│   - malformed JSON 优雅处理                           │
│                                                          │
│ File Watching (5 tests):                               │
│   - 新文件 → reload                                   │
│   - 修改文件 → reload                                 │
│   - 非 .html → no reload                             │
│   - state/events 清空                                 │
│   - 正确的日志消息                                    │
│                                                          │
│ Static Content (2 tests):                              │
│   - helper.js 必需 API                                │
│   - frame-template.html 必需结构                       │
│                                                          │
│ finally: server.kill(), cleanup()                      │
└──────────────────────────────────────────────────────────┘

┌─ windows-lifecycle.test.sh (352 行) ─ 平台特定测试 ──────┐
│ 配置:                                                   │
│   - REPO_ROOT 推导                                      │
│   - TEST_DIR 唯一                                       │
│                                                          │
│ Helpers:                                                 │
│   - cleanup() (杀进程 + 删目录)                       │
│   - pass/fail/skip                                     │
│   - wait_for_server_info (50×0.1s 循环)               │
│   - get_port_from_info (grep/sed)                      │
│   - http_check (node 内联)                             │
│   - platform detection (OSTYPE + MSYSTEM)             │
│                                                          │
│ Test 1: OWNER_PID is empty on Windows (skip non-Windows)│
│   模拟 start-server.sh 行 104-112                      │
│   在 Windows 上清空 OWNER_PID                          │
│                                                          │
│ Test 2: start-server.sh 传空 BRAINSTORM_OWNER_PID     │
│   Fake node 捕获 BRAINSTORM_OWNER_PID                  │
│   验证 start-server.sh 传空字符串                      │
│                                                          │
│ Test 3: Windows 自动检测 foreground mode               │
│   Fake node 捕获 FOREGROUND_MODE=true                 │
│   无 --foreground 时, Windows 仍应进 foreground 路径   │
│                                                          │
│ Test 4: Server 在 75s 后仍存活                          │
│   BRAINSTORM_OWNER_PID="" 启动                         │
│   sleep 75                                              │
│   kill -0 验证仍活                                     │
│   http_check 验证仍响应                                │
│   日志验证无 "owner process exited"                   │
│                                                          │
│ Test 5: 坏 OWNER_PID → 自杀 (control)                 │
│   BRAINSTORM_OWNER_PID=99999 (不存在)                 │
│   sleep 75                                              │
│   验证已死 + 日志有 "owner process exited"            │
│                                                          │
│ Test 6: stop-server.sh 干净停止                       │
│   bash stop-server.sh                                  │
│   验证 PID 已死                                        │
└──────────────────────────────────────────────────────────┘
```

---

## 5. 优点亮点

- **zero-dep 测试对象**：`server.cjs` 不依赖任何 npm 包（只 `http` + `crypto` + `fs`）
- **RFC 标准测试向量**：`computeAcceptKey` 用 IETF 官方示例
- **手写 WS 客户端帧**：不依赖 `ws` 包的编帧，独立验证解码
- **5 个边界尺寸**：125/126/65535/65536/70000 全覆盖
- **平台差异处理**：macOS `/var/folders/` ↔ `/private/var/folders/`
- **fake node 注入**：`PATH` 覆盖技术，零侵入验证环境变量传递
- **正面 + 反面控制实验**：60s 生命周期测试有对照组
- **TDD 友好**：测试在实现之前写好
- **`finally` 清理 server**：集成测试确保资源不泄漏
- **8 个测试 phase**：Startup → HTTP → WS → FileWatch → Static → Cleanup
- **状态隔离**：每个 test 间无状态依赖（除显式 file cleanup）
- **JSONL 事件验证**：检查文件实际写入而非 mock
- **`pwd -P` 真实路径**：避免 macOS 符号链接陷阱
- **超时控制 + 命令输出截断**：避免错误信息淹没终端

---

## 6. 缺点风险局限

- **`server.test.js` 用 `test()` 同步函数 + async fn**：内部 `await test(...)`，错误处理复杂
- **没有 describe/it 嵌套结构**：所有测试平铺，难以看出分组（注释弥补）
- **`ws-protocol.test.js` 用 `if (failed > 0) process.exit(1)`**：失败时仍跑完，但 `server.test.js` 风格不同
- **没有并发运行**：每个 phase 串行执行，集成测试总时长较长
- **没有 cleanup 失败时重试**：tmp 目录残留可能跨测试污染
- **macOS `mktemp` 路径处理硬编码**：Linux 上 `/tmp` 不需要 `pwd -P`
- **`windows-lifecycle.test.sh` 75s 等待**：单测时间长
- **fake node 不模拟真实 node 行为**：只能验证环境变量传递
- **没有 CI timeout 处理**：可能挂死整个 pipeline
- **`os.tmpdir()` 与 `mktemp` 路径不一致**：测试跨平台时可能找不到 tmp
- **`sendEvent` / `selectedChoice` 验证不深**：仅检查字符串存在，未验证实际行为
- **测试间无状态清理断言**：依赖 cleanup()，未验证清理彻底性
- **没有 frame size 上限测试**：恶意大 frame 可能 OOM
- **没有认证测试**：WS 协议接受任何来源

---

## 7. 关键洞察

1. **"测试可以重依赖，生产必须轻依赖"是工程金科玉律**：`ws` 包是测试专属依赖，生产服务器 zero-dep

2. **RFC 标准测试向量是协议正确性的唯一真理**：`s3pPLMBiTxaQ9kYGzzhZRbK+xOo=` 是 IETF 文档的官方值

3. **"手写客户端帧"比"用库"更可靠**：避免"用 `ws` 编帧，`ws` 解帧"的循环验证

4. **fake 二进制注入是 Unix 哲学的体现**：`PATH` 覆盖是 Unix 测试的"标准武器"

5. **60s 生命周期测试需要对照组**：Test 4 (应该活) + Test 5 (应该死) = 行为完整验证

6. **TDD 模式"测试先于实现"在协议层最有效**：测试是协议规格的"可执行文档"

7. **边界值驱动测试覆盖协议 bug 高发区**：125/126/65535/65536 是 WS 长度编码的关键拐点

8. **macOS 路径解析 `/var/folders/` ↔ `/private/var/folders/` 是真实陷阱**：`pwd -P` 是必备工具

9. **`finally { server.kill() }` 是集成测试的卫生标准**：资源清理不可省略

10. **平台检测"双保险" `OSTYPE` + `MSYSTEM`**：单信号可能失效

11. **`mktemp -d` + `mktemp` 文件**：测试用 tmp 资源是标准 Unix 模式

12. **不立即 bail 跑完所有 test**：让诊断信息更全（特别在协议层）

13. **JSON 嵌入字符串的转义**：`session-start` 脚本用 `${s//old/new}` 一次性替换，这是 server.cjs 测试存在的原因之一

14. **server 输出 `server-started` JSON 到 stdout**：`parent process` 用 `if (stdout.includes('server-started')) resolve()` 等待 server 就绪

15. **CWD-based session isolation**：`~/.claude/projects/$(normalize_cwd)` 把每个测试隔离到独立项目目录

16. **`test()` 内部 `try/catch` 永不 throw**：失败信息主动打印而非 propagate

17. **fake `node` 行为 vs 真实 `node` 行为差异**：测试只关心被测代码的契约

18. **`stop-server.sh` 测的是"调用是否能 kill server PID"**：不验证信号类型或 graceful shutdown

19. **WS mask 强制（RFC 6455 §5.1）**：测试明确拒绝 unmasked 客户端帧

20. **HTML 片段 vs 完整 HTML 文档的"包装"判断**：通过 `indicator-bar` 等特征字符串识别

---

## 8. 关联文档

| 文档/文件 | 关系 |
|----------|------|
| [`skills/brainstorming/scripts/server.cjs`](file://e:/WorkStation/MyProject/superpowers/skills/brainstorming/scripts/server.cjs) | 本目录的核心测试对象 |
| [`skills/brainstorming/scripts/helper.js`](file://e:/WorkStation/MyProject/superpowers/skills/brainstorming/scripts/helper.js) | 测试验证必需 API |
| [`skills/brainstorming/scripts/frame-template.html`](file://e:/WorkStation/MyProject/superpowers/skills/brainstorming/scripts/frame-template.html) | 测试验证必需结构 |
| [`skills/brainstorming/scripts/start-server.sh`](file://e:/WorkStation/MyProject/superpowers/skills/brainstorming/scripts/start-server.sh) | Windows 测试验证其 OWNER_PID 行为 |
| [`skills/brainstorming/scripts/stop-server.sh`](file://e:/WorkStation/MyProject/superpowers/skills/brainstorming/scripts/stop-server.sh) | Windows 测试验证其能 kill server |
| `docs/superpowers/specs/2026-03-11-zero-dep-brainstorm-server-design.md` | 零依赖服务器设计 spec |
| `docs/superpowers/plans/2026-03-11-zero-dep-brainstorm-server.md` | 零依赖服务器实现 plan |
| RFC 6455 (WebSocket Protocol) | WS 协议标准（被 `ws-protocol.test.js` 严格遵循） |
| Node.js 内置模块 `http` / `fs` / `crypto` | server.cjs 的实现基础 |

---

## 9. 状态评价

| 维度 | 评分 | 说明 |
|------|------|------|
| 完整性 | 9/10 | 覆盖协议 + 集成 + 平台三个层次 |
| 正确性 | 10/10 | 用 RFC 标准测试向量 |
| 可维护性 | 8/10 | `server.test.js` 平铺结构，注释分 phase |
| 可读性 | 9/10 | 测试名清晰，断言语义明确 |
| 性能 | 7/10 | 集成测试串行，Windows 75s 等待 |
| 健壮性 | 9/10 | 多层 cleanup、fake node 注入、平台检测 |
| 可移植性 | 9/10 | macOS / Linux / Windows 全覆盖 |
| 文档化 | 8/10 | 文件头注释充分，缺少总体 README |
| 教育价值 | 10/10 | zero-dep 协议实现、fake 注入、TDD 范例 |

**总体评价**: 9/10 - 一个精心设计的、多层次的测试套件。

**改进优先级**:

1. **中**: 把 `server.test.js` 改为 describe/it 嵌套结构
2. **中**: 并行运行独立 phase（HTTP、WS、FileWatch）
3. **中**: 加 README.md 说明测试架构
4. **低**: 统一 `if (failed > 0) process.exit(1)` 模式
5. **低**: 验证 tmp 资源彻底清理

---

## 10. 给读者的启示

- **"测试可以重依赖"是工程金科玉律**：`ws` 包是测试依赖，生产 zero-dep
- **RFC 标准测试向量是协议正确性的真理**：`s3pPLMBiTxaQ9kYGzzhZRbK+xOo=` 是 IETF 官方
- **"手写客户端帧"比"用库"更可靠**：避免循环验证
- **fake 二进制注入是 Unix 哲学的体现**：`PATH` 覆盖
- **生命周期测试需要对照组**：Test 4 (活) + Test 5 (死)
- **TDD 在协议层最有效**：测试是"可执行规格"
- **边界值驱动测试覆盖协议 bug 高发区**：125/126/65535/65536
- **macOS `/var/folders/` ↔ `/private/var/folders/` 是真实陷阱**
- **`finally { server.kill() }` 是集成测试卫生标准**
- **平台检测"双保险" `OSTYPE` + `MSYSTEM`**
- **不立即 bail 跑完所有 test**：诊断信息更全
- **HTML 片段 vs 完整文档的"包装"判断**：特征字符串识别
- **WS mask 强制（RFC 6455 §5.1）**：拒绝 unmasked 客户端帧
- **CWD-based session 隔离**：`normalize_cwd()` → 独立项目目录

---

## 11. 后续工作

### 11.1 短期改进（2-3 小时）

- [ ] `server.test.js` 改用 describe/it 结构
- [ ] 并行化独立 phase 测试
- [ ] 加 `README.md` 描述测试架构
- [ ] 统一 `process.exit(1)` 模式
- [ ] 验证 tmp 资源彻底清理

### 11.2 中期改进（半天）

- [ ] 添加 frame size 上限测试（防 OOM）
- [ ] 添加 WS 认证/源验证测试
- [ ] 添加 reconnect 行为测试
- [ ] 添加 server 启动失败路径测试
- [ ] 添加并发 client 写竞争测试

### 11.3 长期演进

- [ ] 探索 property-based testing（自动生成边界 case）
- [ ] 探索 fuzzing（随机输入测试）
- [ ] 探索 performance benchmark（吞吐 + 延迟）
- [ ] 添加 visual regression（截图对比）
- [ ] 跨平台 CI 矩阵（macOS / Linux / Windows）

### 11.4 风险预警

- ⚠️ Node.js `http` 模块 API 变化可能影响 server
- ⚠️ macOS mktemp 路径语义变化（已用 `pwd -P` 缓解）
- ⚠️ RFC 6455 后续修订（如 RFC 8441）需要更新
- ⚠️ WS 协议中"无认证"导致可能被 CSRF
- ⚠️ 75s lifecycle 测试可能超过 CI 默认 timeout

### 11.5 教育价值挖掘

- [ ] 在 `docs/` 下补充 `docs/testing-zero-dep-servers.md`
- [ ] 录制视频讲解 RFC 标准测试向量
- [ ] 写博客："fake 二进制注入是 Unix 哲学的体现"
- [ ] 演示"手写客户端帧" vs "用库"的差异

### 11.6 跨平台 CI 矩阵

- [ ] 在 macOS / Linux / Windows 上分别跑
- [ ] 收集跨平台结果差异
- [ ] 探索 GitHub Actions matrix

---
