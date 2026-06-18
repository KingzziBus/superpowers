# testing-anti-patterns.md 分析报告

## 一、基本信息

- **文件路径**：`skills/test-driven-development/testing-anti-patterns.md`
- **文件大小**：300 行
- **文档类型**：反模式参考手册
- **配套主文件**：`SKILL.md`（TDD 主技能）
- **使用时机**：编写/修改测试、添加 mocks、给生产类添加仅测试方法时

## 二、目的

这是 TDD 主技能的**反模式参考**——它**列出 5 大典型反模式** + **3 条铁律** + **多个门控函数**：

让 LLM **遇到具体场景时**有据可循地识别反模式，并按"门控函数" 决定下一步。

它是 `SKILL.md` 的**具体反例**——SKILL.md 说"应该这样"，本文件说"不要那样"。

## 三、核心技术决策

### 3.1 "3 条铁律"

```
1. 永远不要测试 mock 行为
2. 永远不要给生产类添加仅测试方法
3. 永远不要在不理解依赖的情况下进行 mock
```

**3 条铁律** 简洁明了：

- **铁律 1**：测试对象错（mock 而非真实行为）
- **铁律 2**：污染生产代码（仅测试方法）
- **铁律 3**：盲目 mock（不理解依赖）

这种"3 条铁律" 设计比"长段描述" 更易记忆、引用、强制。

### 3.2 "5 大反模式" + "门控函数"模式

每个反模式都遵循统一结构：

```
1. 违规示例（代码）
2. 为什么这是错的（解释）
3. human partner 的纠正（自然语言提示）
4. 修复（代码）
5. 门控函数（决策树）
```

**这种统一结构让反模式识别成为可操作流程**：

- LLM 看到场景 → 找到对应反模式 → 看门控函数 → 决定下一步

**"门控函数"是本文件的核心创新**——它把"反模式识别" 变成"决策树"。

### 3.3 "human partner 的纠正" 自然语言提示

每个反模式都有一个"你的 human partner 的纠正"自然语言提示：

> **your human partner's correction:** "Are we testing the behavior of a mock?"

**这是 LLM 协作中的关键设计**——把"反模式识别" 包装成"自然语言对话"：

- LLM 收到提示 → 立即触发自我反思
- 比"违反原则 1" 更易理解
- 模拟 human partner 的真实反馈

**"human partner 的话" 是 LLM 协作中的稀缺设计**——它让 LLM **像与 human 协作一样**进行自我检查。

### 3.4 反模式 1：测试 Mock 行为

**核心问题**：测试断言的是 mock 元素（如 `*-mock` test ID），而非真实组件行为。

**为什么这是错的**：

- 测试通过 ≠ 代码正确
- 测试在 mock 存在/不存在时切换结果
- 告诉你的关于真实行为的**零信息**

**修复**：测试真实组件，或不 mock 它。

**门控函数**：

```
BEFORE asserting on any mock element:
  Ask: "Am I testing real component behavior or just mock existence?"

  IF testing mock existence:
    STOP - Delete the assertion or unmock the component

  Test real behavior instead
```

**设计亮点**：

- "BEFORE asserting" 强制在**断言前**问
- 二元问题（"测试真实行为吗？"）让 LLM 难以糊弄
- "STOP" 强制停止

### 3.5 反模式 2：生产类中的仅测试方法

**核心问题**：生产类中添加了**只在测试中调用**的方法（如 `destroy()` 仅在 `afterEach` 中调用）。

**为什么这是错的**：

- 生产类被仅测试代码污染
- 意外在生产中调用 → 危险
- 违反 YAGNI 和关注点分离
- 混淆对象生命周期与实体生命周期

**修复**：把测试逻辑移到 `test-utils/` 中。

**门控函数**：

```
BEFORE adding any method to production class:
  Ask: "Is this only used by tests?"

  IF yes:
    STOP - Don't add it
    Put it in test utilities instead

  Ask: "Does this class own this resource's lifecycle?"

  IF no:
    STOP - Wrong class for this method
```

**设计亮点**：

- **两个连续问题**：
  1. "只用于测试吗？" → 移到测试工具
  2. "这个类拥有资源的生命周期吗？" → 错误类
- **两个 IF 都用 STOP** —— 强制停止
- **第二个问题特别精妙**——"这个类拥有资源的生命周期吗" 是更深层的设计问题

### 3.6 反模式 3：未理解就 Mock

**核心问题**：mock 阻止了测试依赖的副作用（如 mock 阻止了 config 写入），导致测试因错误原因通过或神秘失败。

**为什么这是错的**：

- mock 的方法有测试依赖的副作用
- 过度 mock "以防万一" 破坏实际行为
- 测试因错误原因通过或神秘失败

**修复**：在正确的层级 mock（mock 慢的部分，保留测试需要的行为）。

**门控函数**：

```
BEFORE mocking any method:
  STOP - Don't mock yet

  1. Ask: "What side effects does the real method have?"
  2. Ask: "Does this test depend on any of those side effects?"
  3. Ask: "Do I fully understand what this test needs?"

  IF depends on side effects:
    Mock at lower level (the actual slow/external operation)
    OR use test doubles that preserve necessary behavior
    NOT the high-level method the test depends on

  IF unsure what test depends on:
    Run test with real implementation FIRST
    Observe what actually needs to happen
    THEN add minimal mocking at the right level

  Red flags:
    - "I'll mock this to be safe"
    - "This might be slow, better mock it"
    - Mocking without understanding the dependency chain
```

**设计亮点**：

- **3 个连续问题**：副作用 / 依赖 / 理解
- **两种情况分别处理**：
  - 依赖副作用 → 在低层级 mock
  - 不确定依赖 → 先跑真实实现，再加最小 mock
- **3 条红旗** 显式禁止"以防万一" mock

**这个门控函数是 5 个反模式中最复杂的**——它把"mock 决策" 系统化为"先问 → 看依赖 → 最小 mock"。

### 3.7 反模式 4：不完整的 Mocks

**核心问题**：mock 只包含"你知道的字段"，但下游代码可能依赖其他字段。

**为什么这是错的**：

- 部分 mock 隐藏结构假设
- 下游代码可能依赖未包含的字段
- 测试通过但集成失败（mock 不完整，真实 API 完整）
- 虚假自信

**铁律**：Mock 真实存在的**完整**数据结构，而非你直接测试使用的字段。

**修复**：包含真实 API 的所有字段。

**门控函数**：

```
BEFORE creating mock responses:
  Check: "What fields does the real API response contain?"

  Actions:
    1. Examine actual API response from docs/examples
    2. Include ALL fields system might consume downstream
    3. Verify mock matches real response schema completely

  Critical:
    If you're creating a mock, you must understand the ENTIRE structure
    Partial mocks fail silently when code depends on omitted fields

  If uncertain: Include all documented fields
```

**设计亮点**：

- **"BEFORE creating"** 强制在创建 mock 前问
- **3 个 Actions** 给出具体步骤
- **"Critical" 段** 强调"必须理解完整结构"
- **"If uncertain" 给出保守策略**（"包含所有文档字段"）

### 3.8 反模式 5：集成测试作为事后补充

**核心问题**：实现完成 → 没有测试 → 声称"准备好测试了"。

**为什么这是错的**：

- 测试是实现的一部分，而非可选后续
- TDD 会捕获这个问题
- 没有测试不能声称完成

**修复**：TDD 循环（写失败测试 → 实现 → 重构 → 完成）。

**这是 5 个反模式中最"软"的**——它不涉及具体代码模式，而是**开发流程纪律**。

### 3.9 "When Mocks Become Too Complex" 警告信号

```
- Mock setup 比测试逻辑还长
- Mock 一切以让测试通过
- Mocks 缺少真实组件拥有的方法
- Mock 改变时测试就破坏
```

**4 个警告信号**精准覆盖"mock 过度"的典型情况：

- mock setup 太长
- mock 一切
- mock 不完整
- mock 脆弱

**human partner 的问题**："我们需要在这里使用 mock 吗？"

**这一问的精妙**：它让 LLM **质疑 mock 的必要性**——很多情况下，集成测试比复杂 mock 简单。

### 3.10 "TDD Prevents These Anti-Patterns" 闭环

```
1. 先写测试 → 迫使你思考你真正在测试什么
2. 看到它失败 → 确认测试测试的是真实行为，而非 mocks
3. 最少的实现 → 没有仅测试的方法潜入
4. 真实依赖 → 你在 mock 之前看到测试实际需要什么
```

**4 步 TDD 循环** + **4 种反模式预防** 形成闭环：

| TDD 步骤 | 防止的反模式 |
|---|---|
| 先写测试 | 反模式 5（事后补充） |
| 看到失败 | 反模式 1（mock 行为） |
| 最少实现 | 反模式 2（仅测试方法） |
| 真实依赖 | 反模式 3/4（盲目/不完整 mock） |

**这种"TDD 循环 = 反模式预防" 的映射是 LLM 协作中的关键洞察**——它让"严格 TDD" 的价值**超越测试本身**，延伸到"防止开发反模式"。

### 3.11 "Quick Reference" 表

| 反模式 | 修复 |
|--------------|-----|
| 断言 mock 元素 | 测试真实组件或不 mock |
| 生产中的仅测试方法 | 移到测试工具中 |
| 未理解就 mock | 先理解依赖，最小化 mock |
| 不完整的 mocks | 完整镜像真实 API |
| 测试作为事后补充 | TDD —— 测试优先 |
| 过度复杂的 mocks | 考虑集成测试 |

**6 行对照表**让 LLM **快速找到对应的修复**。

### 3.12 "Red Flags" 6 条

```
- 断言检查 `*-mock` 测试 ID
- 方法仅在测试文件中被调用
- Mock setup 占测试的 50% 以上
- 当你移除 mock 时测试失败
- 无法解释为何需要 mock
- "为了安全" 而 mock
```

**6 条红旗**是"识别反模式"的硬触发器：

- 每条对应一个具体可观察的现象
- 出现任一条 → 反模式存在
- **"为了安全" 而 mock** 是最经典的借口

## 四、文件结构

文件结构清晰，12 段：

```
1. 标题 + 加载时机
2. Overview（核心原则 + TDD 防止反模式）
3. The Iron Laws（3 条铁律）
4. 反模式 1：测试 Mock 行为（违规 + 错因 + 修复 + 门控）
5. 反模式 2：生产类中的仅测试方法
6. 反模式 3：未理解就 Mock
7. 反模式 4：不完整的 Mocks
8. 反模式 5：集成测试作为事后补充
9. When Mocks Become Too Complex（4 警告 + 1 关键问题）
10. TDD Prevents These Anti-Patterns（4 步循环 → 4 预防）
11. Quick Reference（6 行对照表）
12. Red Flags（6 条）+ Bottom Line（最终结论）
```

## 五、优点

### 5.1 统一的反模式结构

每个反模式都遵循 5 段统一结构：

- 违规示例（代码）
- 为什么这是错的（解释）
- human partner 的纠正（自然语言）
- 修复（代码）
- 门控函数（决策树）

**这种统一让反模式识别成为可操作流程**。

### 5.2 "门控函数"是 LLM 协作的关键创新

每个反模式都有一个**门控函数** —— 决策树形式：

- 强制在某个动作前问某个问题
- 给出 IF-ELSE 决策
- 关键节点用 STOP 强制停止

**门控函数把"反模式识别"变成"决策树"** —— 这是 LLM 协作中的关键创新。

### 5.3 "human partner 的纠正" 自然语言提示

每个反模式都有一个"human partner 的纠正"自然语言：

> "Are we testing the behavior of a mock?"
> "Do we need to be using a mock here?"

**这种"自然语言提示"让 LLM 像与 human 协作一样**进行自我检查。

### 5.4 TDD 循环 = 反模式预防 的闭环

4 步 TDD 循环 → 4 种反模式预防：

- 先写测试 → 防止事后补充
- 看到失败 → 防止 mock 行为
- 最少实现 → 防止仅测试方法
- 真实依赖 → 防止盲目/不完整 mock

**这种"循环 = 预防"的映射**让"严格 TDD" 的价值**超越测试本身**。

### 5.5 "Quick Reference" + "Red Flags" 双重工具

- **Quick Reference**：6 行对照表 → 快速找修复
- **Red Flags**：6 条红旗 → 快速识别反模式

**两个工具让 LLM 在不同场景下都能快速反应**。

### 5.6 "Bottom Line" 终极判断

> **Mocks are tools to isolate, not things to test.**
> If TDD reveals you're testing mock behavior, you've gone wrong.
> Fix: Test real behavior or question why you're mocking at all.

**3 句话总结全文**：

- Mocks 是工具（用于隔离）
- 不是被测试对象
- 如果 TDD 揭示你在测 mock → 你走错了

**这种"终极判断"让 LLM 在所有反模式中都有一致的方向**。

## 六、缺点 / 风险

### 6.1 "human partner" 假设可能不通用

文档多处提到"你的 human partner"——但 LLM 协作场景中**未必有 human partner**：

- 完全自动化的 LLM 协作
- human 在环但不在场
- human partner 的角色不明确

可以补充"如果没有 human partner" 的回退策略。

### 6.2 5 个反模式可能不够

5 个反模式覆盖了典型情况，但 LLM 可能创造**新反模式**：

- 测试时间依赖性（time-of-day）
- 测试并发（race conditions）
- 测试全局状态（singletons）
- 测试副作用（file system, network）

可以补充更多反模式或提供"扩展机制"。

### 6.3 "门控函数"的"决策"质量依赖 LLM

门控函数给出 IF-ELSE 决策，但 LLM 仍然要**自己判断**：

- "Am I testing real component behavior or just mock existence?" → LLM 怎么算"real"？
- "What side effects does the real method have?" → LLM 怎么识别"side effects"？

可以补充更多具体的判断标准。

### 6.4 反模式 5 缺乏代码示例

反模式 1-4 都有具体代码示例（TypeScript），但反模式 5（集成测试作为事后补充）**没有**：

- 反模式 5 是流程问题，不是代码模式
- 没有具体代码示例 → LLM 可能难以识别

可以补充"流程反模式" 的代码示例（如 anti-pattern git commit message）。

### 6.5 "When Mocks Become Too Complex" 缺乏反模式编号

4 个警告信号**没归入"反模式 6"**——结构上不统一：

- 反模式 1-5 有明确编号
- 警告信号是另一个章节

可以归入"反模式 6"。

### 6.6 "Quick Reference" 表的"修复"过于简洁

6 行对照表的"修复"列只有几个字：

- "测试真实组件或不 mock" —— 不够具体
- "移到测试工具中" —— 不够具体

可以提供具体步骤或指向反模式正文的链接。

### 6.7 缺乏"反模式严重度"的标注

5 个反模式的严重度可能不同：

- 反模式 1（测试 mock 行为）→ 严重（测试无意义）
- 反模式 5（事后补充）→ 中等（可以补救）
- 反模式 2（仅测试方法）→ 严重（生产代码污染）

可以标注严重度（Critical / Important / Minor）。

## 七、关键洞察

### 7.1 这是 Superpowers 项目的"反模式参考手册"

整个项目有多个参考文档：

- `using-superpowers/SKILL.md` —— 12 条红旗
- `verification-before-completion/SKILL.md` —— 5 步门控
- **`test-driven-development/testing-anti-patterns.md`（本文件）** —— 5 大反模式 + 6 门控

**本文件是 TDD 主技能的"反例"**——SKILL.md 说"应该这样"，本文件说"不要那样"。

### 7.2 "门控函数"是 LLM 协作的关键创新

每个反模式都有一个**门控函数** —— 决策树形式：

- 强制在某个动作前问某个问题
- 给出 IF-ELSE 决策
- 关键节点用 STOP 强制停止

**门控函数把"反模式识别"变成"决策树"** —— 这是 LLM 协作中的关键创新，可以推广到：

- 代码审查决策树
- 测试策略决策树
- 重构决策树

### 7.3 "human partner 的纠正" 自然语言提示

每个反模式都有一个"human partner 的纠正"自然语言：

> "Are we testing the behavior of a mock?"
> "Do we need to be using a mock here?"

**这种"自然语言提示"让 LLM 像与 human 协作一样**进行自我检查——这是 LLM 协作中**反"工具冷漠" 的设计**。

### 7.4 "TDD 循环 = 反模式预防" 的闭环

4 步 TDD 循环 → 4 种反模式预防：

- 先写测试 → 防止事后补充
- 看到失败 → 防止 mock 行为
- 最少实现 → 防止仅测试方法
- 真实依赖 → 防止盲目/不完整 mock

**这种"循环 = 预防" 的映射让"严格 TDD" 的价值超越测试本身**。

### 7.5 "Mocks are tools to isolate, not things to test" 的核心原则

> **Mocks are tools to isolate, not things to test.**

**这一句是测试哲学的浓缩**——它澄清了 mock 的**真正角色**：

- mock = 隔离工具
- mock ≠ 测试对象

很多 LLM 协作中，**测试的不是代码而是 mock**——这是测试的根本失败。

### 7.6 "If you're testing mock behavior, you've gone wrong" 的诊断

> **If TDD reveals you're testing mock behavior, you've gone wrong.**

**TDD 本身就有"反模式检测"功能**：

- 先写测试 → 强迫你思考在测什么
- 看到失败 → 强迫你看到真实行为
- 最少实现 → 强迫你不增加无关方法
- 真实依赖 → 强迫你理解依赖

**如果你违反 TDD 后出现了反模式 → 那就是 TDD 在告诉你走错了**。

## 八、关联文档

### 8.1 同目录

- `SKILL.md`（[test-driven-development/SKILL.md](file:///e:/WorkStation/MyProject/superpowers/skills/test-driven-development/SKILL.md)）—— TDD 主技能

### 8.2 集成技能

- `subagent-driven-development/implementer-prompt.md` —— 实施者子代理遵循 TDD + 反模式
- `verification-before-completion/SKILL.md` —— 完成前验证
- `using-superpowers/SKILL.md` —— 母技能

### 8.3 测试

- `tests/skill-triggering/prompts/test-driven-development.txt` —— 触发测试
- `tests/explicit-skill-requests/prompts/use-systematic-debugging.txt` —— 调试与 TDD 集成

## 九、状态评价

**成熟度**：✅ **生产可用 + 反模式参考**

- 文档结构清晰、纪律强
- 5 大反模式 + 6 门控函数 + 3 铁律 + 6 红旗
- 已经在项目中被实际使用

**完备度**：⭐⭐⭐⭐ (4/5)

- 核心机制完整
- 反模式 5 缺乏代码示例
- "When Mocks Become Too Complex" 缺乏反模式编号
- 缺乏反模式严重度标注

**风险等级**：🟡 中

- 主要风险是 LLM 创造新反模式
- 门控函数提供决策树但仍依赖 LLM 判断
- 需要主代理具备"识别新反模式" 的能力

## 十、给读者的启示

### 10.1 启示 1：反模式 = 违规 + 错因 + 修复 + 门控

LLM 协作中的反模式文档应当遵循 5 段统一结构：

- 违规示例（代码）
- 为什么这是错的（解释）
- human partner 的纠正（自然语言）
- 修复（代码）
- 门控函数（决策树）

**这种统一让反模式识别成为可操作流程**。

### 10.2 启示 2：门控函数是 LLM 协作的关键创新

每个反模式都应当有一个**门控函数** —— 决策树形式：

- 强制在某个动作前问某个问题
- 给出 IF-ELSE 决策
- 关键节点用 STOP 强制停止

**门控函数把"反模式识别"变成"决策树"** —— 可以推广到所有 LLM 协作场景。

### 10.3 启示 3：自然语言提示让 LLM 像与 human 协作

> "Are we testing the behavior of a mock?"

**这种"自然语言提示"让 LLM 像与 human 协作一样**进行自我检查——这是 LLM 协作中**反"工具冷漠" 的设计**。

### 10.4 启示 4：TDD 循环 = 反模式预防

4 步 TDD 循环 → 4 种反模式预防：

- 先写测试 → 防止事后补充
- 看到失败 → 防止 mock 行为
- 最少实现 → 防止仅测试方法
- 真实依赖 → 防止盲目/不完整 mock

**这种"循环 = 预防" 的映射让"严格 TDD" 的价值超越测试本身**。

### 10.5 启示 5："Mocks are tools to isolate, not things to test" 是测试哲学的浓缩

> **Mocks are tools to isolate, not things to test.**

**这一句澄清了 mock 的真正角色**——很多 LLM 协作中，测试的不是代码而是 mock，这是测试的根本失败。

## 十一、后续工作

### 11.1 可能的改进

1. **添加"没有 human partner" 的回退策略**
   - 在完全自动化场景下的决策路径
   - human-in-loop 但不在场的处理

2. **添加更多反模式**
   - 时间依赖性测试
   - 并发测试
   - 全局状态测试
   - 副作用测试

3. **提供"门控函数"的判断标准**
   - 具体的"real behavior" 定义
   - 具体的"side effects" 识别方法

4. **反模式 5 添加代码示例**
   - git commit message 反例
   - PR 描述反例

5. **"When Mocks Become Too Complex" 归入反模式 6**

6. **标注反模式严重度**
   - Critical / Important / Minor

### 11.2 监控点

- 反模式实际触发频率
- 门控函数被 LLM 严格执行的频率
- "human partner" 提示的实际效果
- Quick Reference 6 行的使用频率

### 11.3 长期演进

- 引入"自动反模式检测"（基于代码静态分析）
- 引入"反模式预防训练"（基于历史反模式案例）
- 与 LLM 测试工具集成
