# 零依赖 Brainstorm 服务器实施计划 - 分析报告

> **对应原文**：`docs/superpowers/plans/2026-03-11-zero-dep-brainstorm-server.md`
> **翻译文件**：`z-study/docs/superpowers/plans/2026-03-11-zero-dep-brainstorm-server.zh-CN.md`
> **报告时间**：2026-06-03

---

## 一、基本信息

| 项目 | 内容 |
|---|---|
| 文档类型 | 实施计划（Implementation Plan） |
| Chunk 数量 | 3 个 Chunk，共 4 个 Task |
| 总行数 | 480 行 |
| 关联 Spec | `docs/superpowers/specs/2026-03-11-zero-dep-brainstorm-server-design.md` |
| 核心变更 | 替换 `index.js` + 714 个 `node_modules` 文件 → 单一零依赖 `server.js` |
| 技术栈 | Node.js 内置模块（`http`、`crypto`、`fs`、`path`） |
| 删除文件 | 4 个：`index.js`、`package.json`、`package-lock.json`、`node_modules/` |
| 净减重 | 约 700+ 个文件（依赖从 vendored 改为内置） |

---

## 二、目的

将脑暴服务器（brainstorm server）从一个"看似简单、实则臃肿"的多依赖应用，重构为一个**单文件、零依赖、可教学**的 WebSocket + HTTP 服务器。

### 当前痛点

1. **依赖膨胀**：`node_modules/` 共 714 个文件，但核心功能只用了 `express`、`ws`、`chokidar` 三个包
2. **安装门槛**：用户需要 `npm install` 才能跑服务器
3. **可读性差**：新手读 `index.js` 还要先理解 `express` 中间件、`ws` 事件循环
4. **可移植性差**：不同 Node 版本可能产生 `node_modules` lockfile 冲突
5. **测试摩擦**：单元测试需要完整的 `ws` 客户端库

### 目标收益

- ✅ **零安装**：克隆即可运行
- ✅ **教学价值**：单文件展示 WebSocket 协议、HTTP 路由、文件监听的完整实现
- ✅ **测试简化**：协议函数可独立导出和测试
- ✅ **体积骤降**：仓库体积减少 ~95%

---

## 三、核心技术决策

### 1. 手工实现 WebSocket 协议（不依赖 `ws` 库）

**实现要点**：
- `encodeFrame`：支持 3 种长度编码（< 126、126-65535、> 65535）
- `decodeFrame`：处理客户端掩码，累积不完整帧返回 `null`
- `computeAcceptKey`：SHA-1 + 魔术字符串
- 不实现 `Sec-WebSocket-Extensions` 等可选特性，**严格按 RFC 6455 最小可工作子集**

**决策权衡**：
- ✅ 学习价值高（让读者真正理解 WebSocket 协议）
- ✅ 完全控制字节布局
- ⚠️ 不支持二进制帧（`BINARY` opcode）—— 脑暴场景用不到
- ⚠️ 不支持扩展（permessage-deflate 等）—— 可接受的简化

### 2. 用 `http` 模块替代 `express`

**为什么不用 express**：
- 路由仅 2 个分支（`/` 渲染屏幕，`/files/` 提供静态文件）
- 中间件概念在脑暴场景冗余
- `http.createServer` + 手写路由代码量更少

### 3. 用 `fs.watch` 替代 `chokidar`

**简化要点**：
- 之前用 `chokidar` 是为跨平台稳定性（macOS/Linux/Windows 文件事件差异）
- 现在只在 Node 内置的 `fs.watch` 上加防抖（100ms）
- 添加 `watcher.on('error', ...)` 兜底

### 4. 协议函数独立导出供测试

```js
module.exports = { computeAcceptKey, encodeFrame, decodeFrame, OPCODES };
```

**设计哲学**：
- 协议层是"纯函数"（无副作用、可独立验证）
- HTTP/WS 处理层是"胶水代码"（靠集成测试覆盖）
- 分层清晰，单元测试不再需要起服务器

### 5. 防抖 100ms 处理快速连续写入

```js
debounceTimers.set(filename, setTimeout(() => { ... }, 100));
```

- 防止 `cp` 等命令产生的多个事件
- 100ms 足够短，用户无感知

### 6. 环境变量配置（向后兼容）

```js
PORT = process.env.BRAINSTORM_PORT || (49152 + Math.floor(Math.random() * 16383))
HOST = process.env.BRAINSTORM_HOST || '127.0.0.1'
SCREEN_DIR = process.env.BRAINSTORM_DIR || '/tmp/brainstorm'
```

- 端口范围 49152-65535（IANA 动态端口区间）
- 支持覆盖默认值用于测试和开发

---

## 四、文件结构

### 新增
- `skills/brainstorming/scripts/server.js` — 单一文件，~400 行

### 修改
- `skills/brainstorming/scripts/start-server.sh`（2 行） — 改用 `server.js`
- `.gitignore`（1 行） — 删除 node_modules 例外

### 删除
- `skills/brainstorming/scripts/index.js`
- `skills/brainstorming/scripts/package.json`
- `skills/brainstorming/scripts/package-lock.json`
- `skills/brainstorming/scripts/node_modules/`（714 个文件）

### 不变
- `skills/brainstorming/scripts/helper.js`
- `skills/brainstorming/scripts/frame-template.html`
- `skills/brainstorming/scripts/stop-server.sh`

---

## 五、Chunk 划分逻辑

| Chunk | 关注点 | 测试 |
|---|---|---|
| **Chunk 1：WebSocket 协议层** | 协议正确性（握手、帧编解码、边界） | 单元测试 `ws-protocol.test.js` |
| **Chunk 2：HTTP + WS 应用层** | 端到端功能（路由、广播、文件监听） | 集成测试 `server.test.js` |
| **Chunk 3：切换与清理** | 切换文件路径、删除旧代码 | 两套测试同时通过 + 冒烟测试 |

**分层价值**：
- Chunk 1 完成时，协议层已可独立测试（不必启服务器）
- Chunk 2 在 Chunk 1 基础上叠加 HTTP 路由
- Chunk 3 是纯清理工作，不引入新功能

---

## 六、优点

| 维度 | 具体表现 |
|---|---|
| **依赖洁癖** | 0 个第三方依赖，最大化"克隆即用"体验 |
| **教学价值** | 单文件展示 WebSocket 协议完整实现——可作为协议教学材料 |
| **可测试性** | 协议层函数可独立单元测试；HTTP 层集成测试 |
| **分层清晰** | 协议层（纯函数）/ 应用层（胶水代码）/ 配置层（环境变量）三段分明 |
| **性能** | 协议层直接 `Buffer` 操作，比 `ws` 库少一层抽象 |
| **体积** | 仓库净减重 700+ 文件，git clone 更快 |
| **错误处理** | `decodeFrame` 显式检查 `masked` 位、错误时主动关闭连接 |
| **运维友好** | 支持环境变量覆盖端口/主机/目录，便于容器化和测试 |

---

## 七、缺点 / 风险

| 风险点 | 影响 | 缓解 |
|---|---|---|
| **维护负担转移** | 第三方库的 bug 修复需要自己移植 | 协议层代码量小，< 100 行，bug 易于排查 |
| **不支持二进制帧** | 若未来要传图片/音频，需扩展 | 当前脑暴场景纯 JSON 文本，可接受 |
| **不支持扩展** | 不实现 `permessage-deflate` 等压缩扩展 | 当前负载小，无需压缩 |
| **`fs.watch` 跨平台差异** | Windows / macOS / Linux 事件行为不一致 | 防抖 + error handler 兜底；脑暴场景仅在开发机使用 |
| **单元测试客户端仍需 `ws`** | 集成测试要用 `ws` 客户端连接 | 测试目录的 `package.json` 保留 `ws` 依赖（不提交 node_modules） |
| **缓冲区分片逻辑复杂** | `decodeFrame` 的累积 buffer 处理需小心 | 已有 5 步单元测试覆盖各种边界 |
| **`start-server.sh` 中的 `nohup` 模式** | 进程管理逻辑需保留 | 该脚本不属于本次重构范围，保持不动 |

---

## 八、关键洞察

### 洞察 1：依赖 vs 教学价值的反向关系

| 阶段 | 价值取向 | 例子 |
|---|---|---|
| 早期原型 | 速度优先 | 直接 `npm install express` |
| 中期生产 | 稳定性优先 | 用成熟的 `ws` 库 |
| 成熟期/教学期 | 透明性优先 | 亲手实现，零依赖 |

**本计划处于第三阶段**——Superpowers 项目的目标用户群（AI Agent 开发者）需要理解协议本身。零依赖方案把"黑盒"变"白盒"。

### 洞察 2：协议层和应用层的解耦

```js
// 协议层：纯函数
module.exports = { computeAcceptKey, encodeFrame, decodeFrame, OPCODES };

// 应用层：胶水代码（用上面的函数）
function handleUpgrade(req, socket) { ... }
```

这种分层让"协议正确性"和"业务正确性"可被分别测试——这是软件工程的经典分层思想在小型项目中的体现。

### 洞察 3：环境变量是配置的最佳默认值

```js
const PORT = process.env.BRAINSTORM_PORT || (49152 + Math.floor(Math.random() * 16383));
```

- 默认值够用（随机端口避免冲突）
- 环境变量可覆盖（测试 / 容器化场景）
- 不需要配置文件（避免配置管理的复杂性）

这是 **"convention over configuration"** 原则的实践。

### 洞察 4：删除代码比增加代码更难

注意 Task 3 中删除 4 个旧文件（`index.js`、`package.json`、`package-lock.json`、`node_modules/`）需要 4 个独立的 `git rm` 命令——其中 `node_modules` 是 `-r` 递归删除，且要从 git 索引中移除。**重构不仅是写新代码，更是审慎地退场**。

### 洞察 5：删除 `node_modules` 例外项反映价值观

```gitignore
# Before
!skills/brainstorming/scripts/node_modules/

# After
# (line removed)
```

之前把 `node_modules/` 加入 git 是"以备不时之需"的兜底；删除这行则**明确表态：依赖应该由包管理器管理，不应入仓**。这是项目治理层面的文化声明。

---

## 九、关联文档

### 上游 / Spec
- [`2026-03-11-zero-dep-brainstorm-server-design.md`](../specs/2026-03-11-zero-dep-brainstorm-server-design.md) — 设计文档（本计划的依据）

### 下游 / 受影响
- `skills/brainstorming/scripts/` 目录 — 减重 700+ 文件
- `tests/brainstorm-server/ws-protocol.test.js` — 协议层单元测试
- `tests/brainstorm-server/server.test.js` — 集成测试

### 横向关联
- [`2026-02-19-visual-brainstorming-refactor.md`](./2026-02-19-visual-brainstorming-refactor.md) — 同时期的视觉化脑暴重构（动 `index.js`）
- [`2026-01-17-visual-brainstorming.md`](../../plans/2026-01-17-visual-brainstorming.md) — 引入脑暴服务器的初始计划
- [`2025-11-22-opencode-support-implementation.md`](../../plans/2025-11-22-opencode-support-implementation.md) — 多平台支持（也涉及 server 端）

### 历史
- 原始 `index.js` 创建于 `2026-01-17-visual-brainstorming.md` 计划
- `node_modules` 入仓是当时"为方便克隆"的妥协

---

## 十、状态评价

**计划成熟度：高** ✅

- 每个 Task 都有具体代码片段
- 单元测试和集成测试都有明确预期
- commit message 预写好
- 删除和修改的边界条件清晰

**实施复杂度：中** ⚖️

- 协议层代码需细心（边界条件、字节序、掩码）
- HTTP 路由较简单
- 文件监听有跨平台细节
- 删除步骤需谨慎（git rm vs rm）

**适用性**：✅ 强烈推荐
- 减少了项目的"魔法"
- 让任何读者都能从单文件理解系统
- 与项目"教学优先"的价值观一致

---

## 十一、给读者的启示

### 启示 1：协议学习的最短路径是亲手实现
不要再"读 RFC + 读第三方库代码"——**自己写一遍 WebSocket 协议实现**，比任何文档都更深刻。300 行代码胜过 30 页规范。

### 启示 2：分层让小项目也能保持清晰
即使是 ~400 行的单文件，也分出了"协议层/应用层/配置层"。**分层的价值不取决于项目规模，而取决于未来调试/测试的便利**。

### 启示 3：依赖是"方便"也是"锁链"
每个 `npm install` 都是一次对上游的信任投票。**在项目成熟期，审视哪些依赖真的"必要"**——很多时候"必要"其实是"习惯"。

### 启示 4：环境变量是配置的最小公约数
不要为了"配置灵活性"引入配置文件。**环境变量 + 合理默认值** 已能覆盖 95% 的场景。

### 启示 5：删除也是一种工程活动
注意本计划中"删除 node_modules"需要 4 个独立命令、涉及 git 索引操作、且有先后顺序。**删除代码的工程量经常被低估**——它需要测试、commit message、可能的回滚预案。

---

## 十二、后续工作（推测）

1. **协议层文档化**：将 `encodeFrame` / `decodeFrame` 的 RFC 引用注释化，作为活文档
2. **性能基准**：对比 `ws` 库 vs 手工实现的吞吐量差异
3. **扩展性预留**：设计 `extensions` 字段，未来可加 `permessage-deflate` 等
4. **二进制帧支持**：若脑暴需要传图/音频，需扩展 `BINARY` opcode 处理
5. **`fs.watch` 替换方案**：评估迁移到 `fs.watchFile`（轮询）或 `node:fs/promises` 的可能性
6. **零依赖进一步推广**：审视项目其他 `node_modules/` 目录，看是否可同步精简
7. **CI 集成**：在 CI 中强制 `npm ls` 检查，确保无新增依赖未在 `package.json` 中声明
