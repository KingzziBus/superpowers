# root-cause-tracing.md 分析报告

## 一、基本信息

- **文件路径**：`skills/systematic-debugging/root-cause-tracing.md`
- **文件大小**：170 行
- **文档类型**：技术参考（含 2 个 Graphviz 决策图）
- **配套主文件**：`SKILL.md`（系统化调试）
- **配套脚本**：`find-polluter.sh`（二分查找污染测试）

## 二、目的

这是系统化调试的**反向追踪技术参考**——专门解决"bug 在调用栈深处显现，但根因在源头" 的问题。

核心方法论：

- **从症状开始**（错误出现的地方）
- **反向追踪调用链**（谁调用了这个？）
- **在源头修复**（不只在症状处修复）
- **同时添加多层防御**（让 bug 不可能）

## 三、核心技术决策

### 3.1 "核心原则"的反向思维

> **核心原则：** 通过调用链反向追踪，直到找到原始触发器，然后在源头修复。

**这一句是反向思维的浓缩**：

- 正常调试：从入口开始向下追踪
- **反向追踪**：从症状反向到源头
- **在源头修复**（不只在症状处）

**"反向"是 LLM 调试中的关键方法**——LLM 倾向"在错误处修复"，但根因可能在 5 层之外。

### 3.2 两个决策图（Graphviz）

**决策图 1：when_to_use**

```dot
Bug appears deep in stack? (yes) → Can trace backwards? (yes) → Trace to original trigger
                                                   (no) → Fix at symptom point
```

**3 节点决策图**让"何时使用"成为可推理结构：

- bug 在调用栈深处? → 进入
- 能反向追踪? → 追踪到原始触发器
- 不能追踪（死胡同）? → 在症状点修复

**决策图 2：principle（关键原则）**

```dot
Found immediate cause → Can trace one level up? (yes) → Trace backwards → Is this the source?
                                  (no) → NEVER fix just the symptom                              (no) → Trace backwards
                                                                                                  (yes) → Fix at source → Add validation at each layer → Bug impossible
```

**这个图用红色八角形 "NEVER fix just the symptom"** —— 视觉上强提示"绝不要只修症状"。

**8 节点的详细决策图**让反向追踪流程成为**可推理的结构**。

### 3.3 "5 步追踪流程"

```
1. 观察症状
2. 找到直接原因
3. 问：谁调用了这个？
4. 继续向上追踪（传递了什么值？）
5. 找到原始触发器
```

**5 步流程**让 LLM 看到"具体如何反向追踪"：

- **步骤 1**：记录错误（症状）
- **步骤 2**：找直接执行错误的代码
- **步骤 3**：向上 1 层（调用方）
- **步骤 4**：追踪传递的值
- **步骤 5**：找到值的真正起源

**每一步都有具体的"问什么"** —— 不是模糊的"找原因"，而是"问这个具体问题"。

### 3.4 "真实示例：空的 projectDir" 5 层追踪

```
症状：.git 在 packages/core/（源代码）中创建

追踪链：
1. git init 在 process.cwd() 中运行 ← 空的 cwd 参数
2. WorktreeManager 被空的 projectDir 调用
3. Session.create() 传递空字符串
4. 测试在 beforeEach 之前访问了 context.tempDir
5. setupCoreTest() 最初返回 { tempDir: '' }
```

**5 层追踪**让 LLM 看到：

- **症状在第 1 层**（git init 在错误目录）
- **根因在第 5 层**（顶层变量初始化）
- **传递的值**（空字符串）一路向下传播

**这是 LLM 协作中"反向追踪"的具体范例**。

### 3.5 "添加堆栈跟踪" 的取证技术

```typescript
// 在有问题的操作之前
async function gitInit(directory: string) {
  const stack = new Error().stack;
  console.error('DEBUG git init:', {
    directory,
    cwd: process.cwd(),
    nodeEnv: process.env.NODE_ENV,
    stack,
  });

  await execFileAsync('git', ['init'], { cwd: directory });
}
```

**5 个关键设计**：

- **在操作之前**记录（不是失败之后）
- **console.error** 在测试中（不是 logger）
- **多个上下文**（directory、cwd、env、stack）
- **`new Error().stack`** 捕获完整调用链
- **grep DEBUG** 快速过滤

**这种"主动取证" 设计是 LLM 调试中的关键模式**。

### 3.6 "4 个堆栈跟踪技巧"

```
- 在测试中：使用 console.error() 而非 logger - logger 可能被抑制
- 在操作之前：在危险操作之前记录，不是失败之后
- 包含上下文：目录、cwd、环境变量、时间戳
- 捕获堆栈：new Error().stack 显示完整调用链
```

**4 个技巧**精准覆盖 LLM 调试时的 4 大取证失败：

- 用 logger 看不到 → console.error
- 失败后记录太晚 → 操作前记录
- 缺少上下文 → 多个字段
- 缺堆栈 → `new Error().stack`

### 3.7 "find-polluter.sh" 二分查找脚本

```bash
./find-polluter.sh '.git' 'src/**/*.test.ts'
```

**bisection 算法的应用**——逐个运行测试，在第一个污染者处停止。

**这种"二分定位" 是 LLM 协作中的关键工具**——LLM 不用手动排查几千个测试。

### 3.8 "在源头修复 + 添加多层防御" 的双重收尾

```
根因：顶层变量初始化访问空值
修复：将 tempDir 改为 getter，如果在 beforeEach 之前访问则抛出

还添加了多层防御：
- 第 1 层：Project.create() 验证目录
- 第 2 层：WorkspaceManager 验证非空
- 第 3 层：NODE_ENV 守卫拒绝在临时目录外进行 git init
- 第 4 层：在 git init 之前记录堆栈跟踪
```

**双重收尾**让修复既治本又治标：

- **治本**：在源头修复（getter 验证）
- **治标**：添加多层防御（防止类似 bug）

**这种"治本 + 治标" 的双重收尾是 LLM 调试中的关键模式**。

### 3.9 "NEVER fix just the symptom" 视觉强化

```dot
"NEVER fix just the symptom" [shape=octagon, style=filled, fillcolor=red, fontcolor=white];
```

**红色八角形 + 白字**——视觉上**立即引起注意**。

**这种"视觉强化" 是 LLM 协作中的关键设计**——让 LLM 立即看到"绝不要"。

## 四、文件结构

文件结构清晰，11 段：

```
1. 标题
2. Overview（核心问题 + 核心原则）
3. 何时使用（决策图 + 4 使用场景）
4. 追踪流程（5 步）
5. 添加堆栈跟踪（5 关键设计）
6. 4 技巧
7. find-polluter.sh（bisection 工具）
8. 真实示例：空的 projectDir（5 层追踪 + 修复 + 多层防御）
9. 关键原则（Graphviz 决策图 + 红色八角形）
10. 堆栈跟踪技巧（4 条）
11. 现实影响（4 数据）
```

## 五、优点

### 5.1 极简而完整

170 行覆盖了"反向追踪" 的完整内容：

- 何时用
- 5 步流程
- 取证技术
- 真实示例
- 关键原则

### 5.2 2 个 Graphviz 决策图

**when_to_use**（何时用） + **principle**（关键原则）形成**完整的可视化决策流程**。

**principle 图用红色八角形"NEVER fix just the symptom"** —— 视觉上立即引起注意。

### 5.3 5 步追踪流程

```
1. 观察症状
2. 找到直接原因
3. 谁调用了这个？
4. 传递了什么值？
5. 找到原始触发器
```

**5 步**让 LLM 看到"具体如何反向追踪"——每步有具体的"问什么"。

### 5.4 真实示例 5 层追踪

**5 层追踪链**让 LLM 看到真实会话中的反向追踪：

- 症状在第 1 层
- 根因在第 5 层
- 值的传播路径

### 5.5 "console.error + 多个上下文" 的取证设计

```typescript
console.error('DEBUG git init:', {
  directory,
  cwd: process.cwd(),
  nodeEnv: process.env.NODE_ENV,
  stack,
});
```

**5 要素的取证设计**让调试信息丰富而可操作。

### 5.6 "治本 + 治标" 的双重收尾

- **治本**：在源头修复（getter 验证）
- **治标**：添加多层防御

**双重收尾**是 LLM 调试中的关键模式。

### 5.7 "NEVER fix just the symptom" 视觉强化

**红色八角形 + 白字**让 LLM 立即看到"绝不要"——视觉强化设计。

### 5.8 find-polluter.sh 二分定位

**bisection 算法**让 LLM 不用手动排查几千个测试——这是 LLM 协作中的关键工具。

## 六、缺点 / 风险

### 6.1 5 步流程在某些场景下不够

5 步流程针对"调用链清晰" 的场景，但**异步调用**可能模糊调用链：

- Promise 链 → 跨 event loop
- 事件订阅 → 不在直接调用栈
- setTimeout 回调 → 异步执行

可以补充"异步调用追踪" 的指导。

### 6.2 缺乏"何时停止追踪"的指导

5 步流程没说什么时候停止：

- 找到原始触发器 → 停止
- 追溯到 main 入口 → 停止
- 追溯到测试 → 停止
- 陷入循环 → ?

可以补充"停止条件"。

### 6.3 "添加堆栈跟踪" 可能改变行为

```typescript
async function gitInit(directory: string) {
  const stack = new Error().stack;
  console.error(...);
  await execFileAsync('git', ['init'], { cwd: directory });
}
```

**`new Error().stack` 有性能开销**——在 hot path 中可能影响性能。

可以补充"何时不加堆栈跟踪" 的指导。

### 6.4 find-polluter.sh 的前提假设

`find-polluter.sh` 假设测试可以**逐个运行且不互相干扰**：

- 测试有共享状态 → 逐个运行结果不同
- 测试有 beforeAll → 多次执行可能不一致
- 集成测试需要外部服务 → 逐个运行开销大

可以补充"find-polluter.sh 的局限性"。

### 6.5 缺乏"多线程/多进程追踪"的指导

5 步流程针对单线程调用链，但**多线程/多进程**调试：

- worker threads
- child processes
- distributed tracing

不在本文档范围内。可以补充"分布式追踪" 的指引。

### 6.6 "在源头修复" 可能在源头处改动过大

如果源头是**核心 API**（如 `process.cwd()`），改动可能影响全局：

- 加 getter → 全局所有调用都要重写
- 加环境守卫 → 影响其他代码
- 改 API 签名 → 大范围破坏

可以补充"源头不可改时" 的回退策略。

## 七、关键洞察

### 7.1 这是 Superpowers 项目的"反向追踪" 专项技术

整个项目有多个专项技术参考：

- **`root-cause-tracing.md`（本文件）** —— 反向追踪
- `defense-in-depth.md` —— 多层防御
- `condition-based-waiting.md` —— 条件等待

**本文件是"反向追踪" 的专项技术**——专门解决"症状在深处，根因在源头"的问题。

### 7.2 "反向追踪"是治本调试的核心

> **核心原则：** 通过调用链反向追踪，直到找到原始触发器，然后在源头修复。

**"反向"是 LLM 调试中的关键方法**——LLM 倾向"在错误处修复"，但根因可能在 5 层之外。

### 7.3 "在源头修复 + 多层防御" 的双重收尾

- **治本**：在源头修复（getter 验证）
- **治标**：添加多层防御

**双重收尾**让修复既治本又治标——是 LLM 调试中的关键模式。

### 7.4 "console.error + 多个上下文" 的取证设计

**5 要素的取证设计**让调试信息丰富而可操作。

### 7.5 "NEVER fix just the symptom" 视觉强化

**红色八角形 + 白字**让 LLM 立即看到"绝不要"——**视觉强化设计**是 LLM 协作中的关键。

### 7.6 find-polluter.sh 二分定位

**bisection 算法**让 LLM 不用手动排查几千个测试——**自动化定位**是 LLM 协作中的关键工具。

## 八、关联文档

### 8.1 同目录

- `SKILL.md`（[systematic-debugging/SKILL.md](file:///e:/WorkStation/MyProject/superpowers/skills/systematic-debugging/SKILL.md)）—— 系统化调试主技能
- `defense-in-depth.md` —— 多层防御（在源头修复后添加）
- `condition-based-waiting.md` —— 条件等待
- `find-polluter.sh` —— 二分定位脚本

### 8.2 配套技能

- `test-driven-development/SKILL.md` —— TDD 中的"反向追踪 bug" 集成

## 九、状态评价

**成熟度**：✅ **生产可用 + 反向追踪方法论**

- 文档结构清晰、方法论完整
- 2 个决策图 + 5 步流程 + 真实示例
- 视觉强化 + 二分定位工具

**完备度**：⭐⭐⭐⭐ (4/5)

- 核心机制完整
- 缺乏"异步调用追踪" 指导
- 缺乏"何时停止追踪" 指导
- 缺乏"分布式追踪" 指导

**风险等级**：🟢 低

- 主要风险是 LLM 跳过反向直接修症状
- 红色八角形视觉强化大部分缓解
- 风险在于使用而非设计

## 十、给读者的启示

### 10.1 启示 1：反向追踪是治本调试核心

> **核心原则：** 通过调用链反向追踪，直到找到原始触发器，然后在源头修复。

**"反向"是 LLM 调试中的关键方法**——LLM 倾向"在错误处修复"，但根因可能在 5 层之外。

### 10.2 启示 2：在源头修复 + 多层防御 是双重收尾

- **治本**：在源头修复（getter 验证）
- **治标**：添加多层防御

**双重收尾**让修复既治本又治标。

### 10.3 启示 3：取证设计要 5 要素齐全

```typescript
console.error('DEBUG:', {
  directory, cwd, nodeEnv, stack,
});
```

**5 要素齐全**让调试信息丰富而可操作。

### 10.4 启示 4：视觉强化让 LLM 立即看到关键约束

**红色八角形 + 白字**让 LLM 立即看到"绝不要"——**视觉强化**是 LLM 协作中的关键设计。

### 10.5 启示 5：bisection 算法是定位工具

**二分定位**让 LLM 不用手动排查几千个测试——**自动化定位**是 LLM 协作中的关键工具。

## 十一、后续工作

### 11.1 可能的改进

1. **添加"异步调用追踪" 指导**
   - Promise 链
   - 事件订阅
   - setTimeout 回调

2. **添加"何时停止追踪" 指导**
   - 找到原始触发器
   - 追溯到 main 入口
   - 陷入循环时

3. **添加"何时不加堆栈跟踪" 指导**
   - hot path 性能开销
   - 生产环境

4. **添加"find-polluter.sh 局限性" 说明**
   - 共享状态测试
   - beforeAll
   - 集成测试

5. **添加"分布式追踪" 指引**
   - worker threads
   - child processes
   - 跨服务追踪

6. **添加"源头不可改时" 的回退**
   - 核心 API 改动过大
   - 加 getter 影响全局
   - 改 API 签名破坏性

### 11.2 监控点

- 5 步流程的实际使用率
- 真实会话中"追踪层数" 的分布
- find-polluter.sh 的有效性
- 多层防御的触发频率

### 11.3 长期演进

- 集成 LLM 调试工具（OpenTelemetry）
- 引入"自动反向追踪" 工具
- 形成"反向追踪" 标准库
