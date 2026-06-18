# condition-based-waiting.md 分析报告

## 一、基本信息

- **文件路径**：`skills/systematic-debugging/condition-based-waiting.md`
- **文件大小**：116 行
- **文档类型**：技术参考
- **配套主文件**：`SKILL.md`（系统化调试）
- **配套示例**：`condition-based-waiting-example.ts`（实际代码示例）

## 二、目的

这是系统化调试的**专项技术参考**——专门解决"不稳定的测试"问题。

LLM 编写的测试常因"任意时间延迟"（`setTimeout(50)`、`sleep(100)`）而不稳定。本文档提供"基于条件的等待" 模式作为替代方案：

- **不要猜测时间** → 等待实际条件
- **不要硬编码延迟** → 轮询直到条件满足
- **总是有超时** → 避免永远循环

## 三、核心技术决策

### 3.1 "核心原则"的反 LLM 习惯

> **核心原则：** 等待你关心的实际条件，而非猜测需要多长时间。

**这一句精准打击 LLM 编写测试时的典型失败**：

- LLM 倾向"加个 50ms 延迟应该够" → 50ms 不一定够
- LLM 倾向"加个 100ms 保险" → 100ms 浪费时间
- LLM 倾向"延迟加倍应该稳" → 仍可能不稳定

**"等待条件"是治本**——不依赖时间，依赖实际状态。

### 3.2 决策图（when_to_use）

```dot
Test uses setTimeout/sleep? (yes) → Testing timing behavior? (yes) → Document WHY timeout needed
                                              (no) → Use condition-based waiting
```

**3 个节点的决策图**让"何时使用"成为可推理的结构：

- 用了 setTimeout/sleep? → 进入决策
- 在测时序行为? → 记录原因（这是合理使用）
- 否则 → 使用基于条件的等待

### 3.3 "之前 vs 之后" 对比

```typescript
// ❌ 之前：猜测时间
await new Promise(r => setTimeout(r, 50));
const result = getResult();
expect(result).toBeDefined();

// ✅ 之后：等待条件
await waitFor(() => getResult() !== undefined);
const result = getResult();
expect(result).toBeDefined();
```

**"之前 vs 之后" 对比**让 LLM 看到具体差异：

- 之前：硬编码 50ms 延迟
- 之后：等待 `getResult() !== undefined` 条件

**"之后" 的 waitFor 把"猜测时间"变成"等待条件"**。

### 3.4 "5 个快速模式" 表

| 场景 | 模式 |
|----------|---------|
| 等待事件 | `waitFor(() => events.find(e => e.type === 'DONE'))` |
| 等待状态 | `waitFor(() => machine.state === 'ready')` |
| 等待数量 | `waitFor(() => items.length >= 5)` |
| 等待文件 | `waitFor(() => fs.existsSync(path))` |
| 复杂条件 | `waitFor(() => obj.ready && obj.value > 10)` |

**5 个具体模式**覆盖 LLM 编写测试时的典型等待场景：

- 事件（异步消息）
- 状态（机/服务）
- 数量（集合）
- 文件（资源）
- 复杂（多条件组合）

每个模式都是"一行代码可复用"。

### 3.5 "通用 waitFor 实现"

```typescript
async function waitFor<T>(
  condition: () => T | undefined | null | false,
  description: string,
  timeoutMs = 5000
): Promise<T> {
  const startTime = Date.now();

  while (true) {
    const result = condition();
    if (result) return result;

    if (Date.now() - startTime > timeoutMs) {
      throw new Error(`Timeout waiting for ${description} after ${timeoutMs}ms`);
    }

    await new Promise(r => setTimeout(r, 10)); // 每 10ms 轮询一次
  }
}
```

**5 要素的完整实现**：

- **类型泛型 `<T>`** —— 任何类型
- **condition 闭包** —— 等待的条件
- **description 字符串** —— 错误信息
- **timeoutMs 5 秒** —— 默认超时
- **10ms 轮询间隔** —— 平衡 CPU 与响应

**这种"完整可复用实现" 让 LLM 直接复制使用**，避免 LLM 写错边界条件。

### 3.6 "3 个常见错误" + "修复"

```
❌ 轮询太快：setTimeout(check, 1) - 浪费 CPU
✅ 修复：每 10ms 轮询一次

❌ 没有超时：如果条件永远不满足则永远循环
✅ 修复：总是包含带清晰错误的超时

❌ 过期数据：在循环前缓存状态
✅ 修复：在循环内调用 getter 以获取新数据
```

**3 个常见错误**精准覆盖 LLM 实现 waitFor 时的 3 个典型 bug：

- 轮询 1ms → CPU 100%
- 没超时 → 永远循环
- 缓存状态 → 等的是旧数据

**每个错误配 1 个修复**——让 LLM **有据可循地**避免这些错误。

### 3.7 "何时任意超时是正确的" 的明确边界

```typescript
// 工具每 100ms 滴答一次 —— 需要 2 个滴答来验证部分输出
await waitForEvent(manager, 'TOOL_STARTED'); // 首先：等待条件
await new Promise(r => setTimeout(r, 200));   // 然后：等待定时行为
// 200ms = 100ms 间隔的 2 个滴答 —— 已记录且合理
```

**3 条要求**给"任意超时"明确的边界：

1. **首先等待触发条件** → 仍然以条件为前提
2. **基于已知时序**（不是猜测）→ 200ms = 2 个 100ms 滴答
3. **注释解释原因** → 让 LLM 看到"为什么这个超时是合理的"

**这一段是反 LLM 教条的关键**——不是"永远不用任意超时"，而是"在合理场景下使用，并记录原因"。

### 3.8 "现实影响" 数据

> 来自调试会话（2025-10-03）：
> - 在 3 个文件中修复了 15 个不稳定测试
> - 通过率：60% → 100%
> - 执行时间：快 40%
> - 不再有竞态条件

**4 个对比数据**精准量化"条件等待 vs 任意延迟" 的差异：

- 修复 15 个不稳定测试
- 通过率：60% → 100%
- 执行时间：快 40%
- 0 竞态条件

**数据让 LLM 看到"条件等待的价值"**。

## 四、文件结构

文件结构清晰，9 段：

```
1. 标题
2. Overview（核心问题 + 核心原则）
3. When to Use（决策图 + 使用/不使用）
4. 核心模式（之前/之后对比）
5. 快速模式（5 场景表）
6. 实现（通用 waitFor 函数）
7. 常见错误（3 个 + 修复）
8. 何时任意超时是正确的（3 要求）
9. 现实影响（4 数据）
```

## 五、优点

### 5.1 极简而完整

116 行覆盖了"条件等待" 的完整内容：

- 何时用（决策图）
- 怎么用（之前/之后）
- 5 模式表
- 完整实现
- 3 错误+修复
- 任意超时的边界
- 现实数据

### 5.2 "之前 vs 之后" 对比让 LLM 立即理解

```typescript
// 之前：await new Promise(r => setTimeout(r, 50));
// 之后：await waitFor(() => getResult() !== undefined);
```

**"之前 vs 之后" 对比**让 LLM 看到**具体差异**——从"猜测时间"到"等待条件"。

### 5.3 完整可复用的 waitFor 实现

```typescript
async function waitFor<T>(condition, description, timeoutMs = 5000) { ... }
```

**5 要素的完整实现**（类型、condition、description、timeout、轮询间隔）让 LLM **直接复制使用**。

### 5.4 5 个快速模式覆盖典型场景

5 个 waitFor 模式覆盖 LLM 编写测试时的 5 大典型等待场景：

- 事件
- 状态
- 数量
- 文件
- 复杂条件

每个模式都是**一行代码可复用**。

### 5.5 3 错误 + 修复 配对

3 个常见错误精准覆盖 LLM 实现 waitFor 时的 3 个典型 bug：

- 轮询太快 → 浪费 CPU
- 没超时 → 永远循环
- 缓存数据 → 过期数据

**每个错误配 1 个修复**——让 LLM **有据可循地**避免这些错误。

### 5.6 "何时任意超时是正确的" 的反教条设计

不是"永远不用任意超时"——而是"在合理场景下使用，并记录原因"。

**3 条要求**给任意超时明确的边界：

1. 先等条件
2. 基于已知时序
3. 注释解释

**这种"反教条"设计让 LLM 不僵化**。

### 5.7 现实数据量化价值

4 个对比数据让 LLM 看到"条件等待" 的真实价值。

## 六、缺点 / 风险

### 6.1 默认 5 秒超时可能不合适

`timeoutMs = 5000` 是默认值，但不同场景可能需要不同：

- 单元测试 → 100ms 足够
- 集成测试 → 30s 才合理
- UI 测试 → 60s+

可以补充"超时时间的指导"。

### 6.2 10ms 轮询间隔可能不适用所有场景

10ms 是平衡值，但：

- 高频事件 → 1ms 更合理
- 慢操作 → 100ms 减少 CPU
- 嵌入式 → 50ms 平衡

可以补充"轮询间隔的指导"。

### 6.3 waitFor 自身可能不稳定

如果 condition 本身抛错：

- `waitFor(() => obj.property.deep.nested)` → 中间属性可能 undefined
- 应该 `waitFor(() => obj?.property?.deep?.nested)`

可以补充"安全访问" 的指导。

### 6.4 缺乏"轮询 vs 事件"的对比

条件等待是**轮询**模式，但很多情况下**事件订阅**更高效：

- 等待异步结果 → Promise/async-await
- 等待事件触发 → EventEmitter
- 轮询只在没有事件机制时使用

可以补充"轮询 vs 事件" 的对比。

### 6.5 "5 模式表" 没覆盖所有场景

5 模式覆盖典型场景，但 LLM 可能遇到：

- 等待网络请求完成
- 等待多个并行条件
- 等待某个特定时间点（cron-like）

可以补充更多模式。

## 七、关键洞察

### 7.1 这是 Superpowers 项目的"条件等待"专项技术

整个项目有多个专项技术参考：

- `root-cause-tracing.md` —— 根因追踪
- `defense-in-depth.md` —— 多层防御
- **`condition-based-waiting.md`（本文件）** —— 条件等待

**本文件是"反不稳定测试" 的专项技术**。

### 7.2 "等待条件 vs 猜测时间" 是测试稳定性核心

> **核心原则：** 等待你关心的实际条件，而非猜测需要多长时间。

**这一句是测试稳定性的核心** —— LLM 倾向"加延迟"，但延迟不能保证稳定性。

**条件等待** 是"基于状态"——状态确定，测试就稳定。

### 7.3 "5 模式" 形成 waitFor 的"模式库"

5 个 waitFor 模式覆盖 LLM 编写测试时的 5 大典型等待场景：

- 事件
- 状态
- 数量
- 文件
- 复杂条件

**这种"模式库"是 LLM 协作中的关键设计**——LLM 找到对应模式即可复用。

### 7.4 "3 错误 + 修复" 的反 LLM 实现 bug

3 个常见错误精准覆盖 LLM 实现 waitFor 时的 3 个典型 bug：

- 轮询 1ms → CPU 100%
- 没超时 → 永远循环
- 缓存数据 → 过期数据

**这种"反 LLM 实现 bug" 的设计是 LLM 协作中的关键模式**。

### 7.5 "何时任意超时是正确的" 的反教条

不是"永远不用任意超时"——而是"在合理场景下使用，并记录原因"。

**3 条要求**给任意超时明确的边界：

1. 先等条件
2. 基于已知时序
3. 注释解释

**这种"反教条"设计让 LLM 不僵化**。

### 7.6 现实数据量化"条件等待" 的价值

4 个对比数据让 LLM 看到"条件等待" 的真实价值：

- 修复 15 个不稳定测试
- 通过率：60% → 100%
- 执行时间：快 40%
- 0 竞态条件

## 八、关联文档

### 8.1 同目录

- `SKILL.md`（[systematic-debugging/SKILL.md](file:///e:/WorkStation/MyProject/superpowers/skills/systematic-debugging/SKILL.md)）—— 系统化调试主技能
- `root-cause-tracing.md` —— 根因追踪
- `defense-in-depth.md` —— 多层防御
- `condition-based-waiting-example.ts` —— 完整代码示例

### 8.2 配套技能

- `test-driven-development/SKILL.md` —— TDD 中的"测试稳定性" 集成
- `verification-before-completion/SKILL.md` —— 完成前验证

## 九、状态评价

**成熟度**：✅ **生产可用 + 专项技术**

- 文档结构清晰、技术成熟
- 反 LLM 习惯设计完整
- 现实数据量化价值

**完备度**：⭐⭐⭐⭐ (4/5)

- 核心机制完整
- 默认超时/轮询间隔的指导不足
- 缺乏"轮询 vs 事件" 对比
- 5 模式可能不够

**风险等级**：🟢 低

- 主要风险是 LLM 不知道这个技能
- 文档本身已经成熟
- 风险在于使用而非设计

## 十、给读者的启示

### 10.1 启示 1："等待条件 vs 猜测时间" 是测试稳定性核心

> **核心原则：** 等待你关心的实际条件，而非猜测需要多长时间。

**"等待条件"是治本**——不依赖时间，依赖实际状态。

### 10.2 启示 2：完整可复用的实现 + 模式库是关键

```typescript
async function waitFor<T>(condition, description, timeoutMs = 5000) { ... }
```

**完整可复用的实现** + **5 模式库** 让 LLM 编写测试时**直接复制使用**。

### 10.3 启示 3：3 错误 + 修复 配对是反 LLM bug 的关键

3 个常见错误精准覆盖 LLM 实现 waitFor 时的 3 个典型 bug——**有据可循地**避免这些错误。

### 10.4 启示 4：反教条设计让 LLM 不僵化

不是"永远不用任意超时"——而是"在合理场景下使用，并记录原因"。

**这种"反教条"设计让 LLM 不僵化**。

### 10.5 启示 5：现实数据量化价值

4 个对比数据让 LLM 看到"条件等待" 的真实价值。

## 十一、后续工作

### 11.1 可能的改进

1. **添加"超时时间"的指导**
   - 单元测试 vs 集成测试 vs UI 测试
   - 默认值的调整

2. **添加"轮询间隔"的指导**
   - 高频 vs 低频
   - CPU 与响应的平衡

3. **添加"安全访问"的指导**
   - condition 内部的可选链
   - 避免抛错

4. **添加"轮询 vs 事件"对比**
   - 何时用轮询
   - 何时用事件订阅
   - 性能差异

5. **扩展"5 模式表"**
   - 网络请求完成
   - 多个并行条件
   - 特定时间点

### 11.2 监控点

- waitFor 在 LLM 实际代码中的使用率
- 不稳定测试的减少率
- 任意超时的合理使用率

### 11.3 长期演进

- 引入"自动不稳定测试检测"
- 集成 LLM 测试工具
- 形成"条件等待" 标准库
