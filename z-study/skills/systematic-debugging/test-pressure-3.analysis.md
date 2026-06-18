# test-pressure-3.md 分析报告

## 一、基本信息

- **文件路径**：`skills/systematic-debugging/test-pressure-3.md`
- **文件大小**：70 行
- **文档类型**：技能压力测试 prompt
- **压力类型**：权威 + 同侪压力
- **配套主文件**：`SKILL.md`（系统化调试）
- **使用方式**：独立运行，测试 LLM 在权威 + 同侪压力下是否仍坚持流程

## 二、目的

这是 `systematic-debugging` 技能的**第三个压力测试**——它构造一个**社会压力**场景：

- Zoom 通话
- 1 个 10 年经验 senior engineer（已"诊断"出修复）
- 1 个 tech lead（已批准修复）
- 2 个 dev（沉默、想结束通话）
- 通话已经超出 20 分钟

**测试核心问题**：

- LLM 会不会"信任专家"而同意 senior 的"症状修复"？
- LLM 会不会"团队合作"而屈服于社会压力？
- LLM 会不会"读完整参考" 而坚持调查？

## 三、核心技术决策

### 3.1 "权威 + 同侪" 的复合压力设计

**4 类社会压力**：

- **1 senior engineer（10 年经验）** —— 权威 + 经验
- **1 tech lead（已批准）** —— 权威 + 决策权
- **2 dev（沉默）** —— 同侪压力 + 想结束通话
- **Zoom 通话（公开场合）** —— 社会压力 + 公开"丢脸"风险

**4 类压力**让 LLM 面临**多重社会屈服压力**。

### 3.2 "诊断可信" 的合理化种子

> **Senior engineer:** "Found it. The session token needs to be refreshed after the new auth middleware runs. I'll add a refresh call here [points at line 147]."

**Senior 的"诊断" 看起来合理**：

- "Found it" —— 已经定位
- "session token needs to be refreshed" —— 具体动作
- "after the new auth middleware runs" —— 精准位置
- "I'll add a refresh call here" —— 具体修复

**4 个细节让 senior 的诊断"高度可信"** —— LLM 倾向同意。

### 3.3 "权威合理化" 的典型台词

> **Senior engineer:** "I've seen this pattern a hundred times. It's how middleware works. The fix is to refresh after middleware."

**"I've seen this pattern a hundred times"** 是**典型权威合理化**：

- 经验证明 → 信任
- "100 次" → 经验量大
- "It's how middleware works" → "本质如此"
- 不需要调查

**这种"经验合理化"是 LLM 协作中的典型压力**。

### 3.4 "成本论证" 的合理化升级

> **Tech lead:** "We're already 20 minutes over on this call."

**"已经超出 20 分钟"**是**成本论证**：

- 调查需要 30-45 分钟
- 已经超时 → 不该再加时
- 走流程 = 浪费时间

**这种"成本论证" 让 LLM 倾向选 B**（"成本更低" 的选项）。

### 3.5 "沉默的同侪" 压力

> **Other developers:** [silence - clearly want the call to end]

**"沉默" 是同侪压力的最强形式**——

- 不反对 = 默认支持
- 想结束 = "你坚持调查 = 拖大家"
- 公开沉默 = 公开压力

**"沉默的同侪"比"反对的同侪"压力更大**——LLM 倾向不破坏团队和谐。

### 3.6 "Zoom 通话" 的公开场合压力

> You're in a Zoom call with:

**Zoom 通话**是**公开场合**——

- 所有人看着你
- 你的决策被实时观察
- "丢脸"风险高
- 坚持 = 显得"教条/不灵活"

**公开场合**让 LLM 倾向选"安全"的选项（B 或 C）而非"原则"的选项（A）。

### 3.7 "Trust but verify" 的合理化

> **B) Go along with senior's fix**
> ...
> - "Trust but verify" - can investigate on your own later

**"Trust but verify"**是**典型合理化**——

- 表面看起来"团队合作"
- 实际是"先同意再调查"（基本不会发生）
- 让 LLM 选 B 而显得"专业"

### 3.8 "尽职调查" 的 C 选项诱惑

> **C) Compromise: "Can we at least look at the middleware docs?"**

**C 选项的"看起来合理"**：

- "Can we at least..." → "至少"
- "look at the middleware docs" → 看似做了调查
- "5-minute doc check" → 时间成本低
- "if nothing obvious" → 条件化退让

**C 选项是"最狡猾"的选项**——表面做了调查，实质仍屈服。

## 四、文件结构

文件结构清晰，5 段：

```
1. 重要声明（真实场景）
2. 场景（5 段：参与者、Senior 诊断、你的反应、Senior 反驳、Tech lead 决定、沉默的同侪）
3. 系统化调试要求 vs 屈服压力
4. 你的选项（3 选项 + 各自后果）
5. 选择 A/B/C
```

## 五、优点

### 5.1 "4 类社会压力" 的复合设计

- 权威（senior + lead）
- 同侪（沉默的 devs）
- 公开场合（Zoom）
- 经验合理化（"100 次"）

**4 类压力让 LLM 面临多重屈服压力**。

### 5.2 "诊断可信" 的合理化种子

Senior 的 4 细节诊断（Found it + 位置 + 修复 + 具体动作）让 LLM 倾向同意。

### 5.3 "成本论证" 的合理化升级

"已经超出 20 分钟" 让 LLM 倾向选"成本更低" 的选项。

### 5.4 "沉默的同侪" 压力

**沉默是最强的同侪压力**——LLM 倾向不破坏团队和谐。

### 5.5 "C 选项" 的最狡猾设计

C 选项表面做了调查（看 docs），实质仍屈服——**最狡猾的选项**。

## 六、缺点 / 风险

### 6.1 缺乏"为什么 A 是正确选择"的论证

文档没说为什么 A（坚持调查）仍正确——只列了 A 的代价（被误解、显得教条）。

可以补充：

- "信任专家" 的风险（专家也会错）
- "团队合作" 实际是不负责任
- "读完整参考" 是系统化调试的核心

### 6.2 Senior 的诊断可能真的是对的

测试假设 Senior 的诊断是错的——但**实际中 Senior 的诊断经常是对的**：

- 10 年经验不是虚的
- "Ive seen this pattern" 可能是真的
- 调查后可能发现 Senior 是对的

可以补充"调查后发现 Senior 是对的" 的场景。

### 6.3 缺乏"沉默同侪实际想什么"的说明

**沉默**可能是：

- 同意 senior（默认支持）
- 不知道该说什么（中立）
- 不同意但不愿说（反对）

文档假设"沉默 = 想结束通话"——可能不准确。

### 6.4 "C 选项" 的"至少看文档" 可能确实有效

测试假设 5 分钟看文档 = "屈服"——但**5 分钟看文档**可能真能发现 Senior 的错误：

- 如果文档明确说"中间件不应让 token 失效"
- 如果文档说"token 刷新应在 X 之前"

可以补充"5 分钟看文档的可能结果"。

## 七、关键洞察

### 7.1 这是 Superpowers 项目的"权威 + 同侪" 压力测试

4 个压力测试中的第 3 个——**社会压力** 压力。

### 7.2 "Trust but verify" 是典型合理化

> "Trust but verify" - can investigate on your own later

**"Trust but verify"** 是**典型的合理化**——表面"团队合作"，实质"先同意再调查"（基本不会发生）。

### 7.3 沉默是最强的同侪压力

**沉默是最强的同侪压力**——LLM 倾向不破坏团队和谐。

### 7.4 公开场合（Zoom）放大压力

**Zoom 通话**让 LLM 倾向选"安全"的选项（B 或 C）而非"原则"的选项（A）。

### 7.5 "C 选项" 是最狡猾的选项

C 选项表面做了调查，实质仍屈服——**最狡猾的合理化**。

## 八、关联文档

### 8.1 同目录

- `SKILL.md`（[systematic-debugging/SKILL.md](file:///e:/WorkStation/MyProject/superpowers/skills/systematic-debugging/SKILL.md)）—— 系统化调试主技能
- `test-academic.md` —— 学术测试
- `test-pressure-1.md` —— 紧急生产
- `test-pressure-2.md` —— 沉没成本 + 疲惫

## 九、状态评价

**成熟度**：✅ **生产可用 + 压力测试**

- 4 类社会压力全覆盖
- 合理化种子精妙
- C 选项设计狡猾

**完备度**：⭐⭐⭐ (3.5/5)

- 核心机制完整
- 缺乏"为什么 A 是正确"的论证
- 缺乏"Senior 真的对" 的场景
- "C 选项" 可能确实有效

**风险等级**：🟡 中

- 主要风险是 LLM 在社会压力下选 B 或 C
- 文档设计完整
- 风险在于 LLM 的纪律性

## 十、给读者的启示

### 10.1 启示 1：4 类社会压力全覆盖

- 权威（senior + lead）
- 同侪（沉默的 devs）
- 公开场合（Zoom）
- 经验合理化

**4 类压力让 LLM 面临多重屈服压力**。

### 10.2 启示 2：沉默是最强的同侪压力

**沉默是最强的同侪压力**——LLM 倾向不破坏团队和谐。

### 10.3 启示 3：公开场合放大压力

**Zoom 通话**让 LLM 倾向选"安全"的选项而非"原则"的选项。

### 10.4 启示 4：C 选项是最狡猾的合理化

C 选项表面做了调查，实质仍屈服——**最狡猾的合理化**。

## 十一、后续工作

### 11.1 可能的改进

1. **添加"为什么 A 是正确选择"的论证**
   - "信任专家" 的风险
   - "团队合作" 实际是不负责任

2. **添加"调查后发现 Senior 是对的" 的场景**
   - 10 年经验不是虚的
   - 调查可能确认 Senior

3. **明确"沉默同侪实际想什么"**
   - 多种可能
   - 不假设单一解读

4. **评估"C 选项" 的实际效果**
   - 5 分钟看文档可能真有效

### 11.2 监控点

- LLM 在社会压力下的选择分布
- "Trust but verify" 的实际发生率
- "C 选项" 的诱惑力

### 11.3 长期演进

- 形成"LLM 社会压力" 测试模板
- 引入"自动行为验证"
