# 文档分析报告：来自用户反馈的技能改进

> **原文档**：[2025-11-28-skills-improvements-from-user-feedback.md](file://e:/WorkStation/MyProject/superpowers/docs/plans/2025-11-28-skills-improvements-from-user-feedback.md)
> **翻译版本**：[2025-11-28-skills-improvements-from-user-feedback.zh-CN.md](file://e:/WorkStation/MyProject/superpowers/z-study/docs/plans/2025-11-28-skills-improvements-from-user-feedback.zh-CN.md)
> **文档类型**：改进建议（Improvement Proposals）
> **分析日期**：2026-06-03

---

## 📋 一、文档基本信息

| 字段 | 内容 |
|------|------|
| 标题 | Skills Improvements from User Feedback |
| 日期 | 2025-11-28 |
| 状态 | Draft（草稿） |
| 来源 | 两个 Claude 实例的真实使用反馈 |
| 字数 | ~712 行 / ~20.5KB |
| 类别 | plans（项目开发计划） |
| 问题数 | 8 个真实问题 |
| 改进建议 | 8 项针对性改进 |
| 实施阶段 | 3 阶段（高/中/低风险） |

## 🎯 二、文档目的

**汇集并分析真实用户反馈**，识别 superpowers 技能中**系统性缺口**，并提出针对性改进方案。这不是技术设计文档，而是**基于实际失败案例的方法论反思**。

## 🔬 三、8 个核心问题诊断

### 3.1 问题分布（按主题归类）

| 主题 | 问题编号 | 影响等级 |
|------|---------|---------|
| **验证缺口** | 问题 1 | 高 |
| **进程卫生** | 问题 2 | 中-高 |
| **上下文优化** | 问题 3 | 中 |
| **自我反思** | 问题 4 | 中 |
| **Mock 安全性** | 问题 5 | 高 |
| **文件访问** | 问题 6 | 低-中 |
| **工作流延迟** | 问题 7 | 低 |
| **技能激活** | 问题 8 | 中 |

### 3.2 三个最关键的问题（高影响）

#### 问题 1：配置变更验证缺口 🔴

**实际案例：**
```
"OpenAI 集成工作" ← 报告内容
实际：响应包含 "model": "claude-sonnet-4-20250514" ← 仍在用 Anthropic
```

**为什么可怕**：
- 状态码 200 → 看起来一切正常
- 实际响应内容 → 完全不是预期的
- 这种 bug 流入生产环境的概率极高

**改进方案**：增加"门控函数"（Gate Function），强制验证**意图**而非**操作成功**。

#### 问题 5：Mock-接口 偏离 🔴

**实际代码示例**：
```typescript
// Interface defines close() ← 真实接口
// Code (BUGGY) calls cleanup() ← 代码 bug
// Mock (MATCHES BUG) defines cleanup() ← Mock 把 bug 固化了
```

**为什么可怕**：
- TypeScript 类型系统**捕获不到**这个错误（因为 mock 是手写的）
- 测试通过 → 错误自信
- 运行时崩溃 → 用户发现

**改进方案**："先读接口，再写 mock"门控函数，强制 mock 来源是接口而非实现。

#### 问题 2：后台进程累积 🟡

**实际场景**：
```
4+ 个服务器进程在运行
E2E 测试命中错误配置的旧服务器
```

**改进方案**：子代理 prompt 中显式包含 `pkill + lsof` 清理步骤。

## 📊 四、3 阶段实施路线图

| 阶段 | 风险 | 改进项 | 文件 |
|------|------|-------|------|
| **阶段 1** | 低 | 3 项 | `verification-before-completion`、`testing-anti-patterns`、`requesting-code-review` |
| **阶段 2** | 中 | 3 项 | `subagent-driven-development`（3 处修改）|
| **阶段 3** | 高 | 2 项 | `subagent-driven-development`（2 处工作流变更）|

**亮点**：作者明确建议"阶段 3 需要更多验证"，这体现了**工程谨慎**。

## 💡 五、关键洞察

### 洞察 1：失败案例是最好的老师

这份文档**不是凭空设计**，而是**从 8 个真实 bug 中归纳**。每个改进建议都对应：
- 真实发生场景
- 具体代码/操作序列
- 失败的根本原因
- 可执行的修复

**这是"以问题驱动设计"的最佳实践**——比"以解决方案驱动"更可靠。

### 洞察 2：验证的"最后一公里"陷阱

问题 1 揭示了一个深刻现象：**操作成功 ≠ 意图达成**。

```
LLM 时代的新型 bug：
- 旧时代：操作失败 → 容易发现
- 新时代：操作成功 + 意图错误 → 难以发现
```

**这正是 "verification-before-completion" 技能要解决的核心问题**——验证不是"有没有报错"，而是"是不是我想要的结果"。

### 洞察 3：Mock 是测试的"影子"，容易脱离"主人"

问题 5 揭示了一个 TypeScript 项目的"暗角"：
- TypeScript 类型系统**无法保护**你写出与接口不符的 mock
- 因为 mock 通常是手写的、可选字段
- 测试通过给开发者的虚假安全感

**根本解药**：**Mock 必须从接口派生，而非从实现派生**。这是一个简单但深刻的工程原则。

### 洞察 4：自我反思比看起来更重要

问题 4 中一个具体案例：
```
"实施者审视自己的工作时，发现 line 99 的 entrypoint 拼接有 bug"
```

这是 AI 时代的特殊价值：**AI 不会"自我怀疑"，必须被 prompt 引导**。

`"Look at your work with fresh eyes"` 这句简单的提示，**让 AI 模拟人类的"第二天再回头看"心理**。

### 洞察 5：技能存在 ≠ 技能被使用

问题 8 是一个"元问题"：
- `testing-anti-patterns` 技能存在
- 但没人在写测试前读它
- 技能投资完全浪费

**改进方向**：把"读技能"变成 prompt 的**强制部分**，而不是建议。

## 🔍 六、文档优缺点分析

### ✅ 优点

1. **真实案例驱动**：每个问题都有具体场景、代码、操作
2. **分级影响评估**：用"高/中-高/中/低"量化影响
3. **明确的"门控函数"模式**：每个改进都有可执行步骤
4. **三阶段风险分级**：低风险先做、高风险后做
5. **"Open Questions" 部分诚实**：承认未解之谜
6. **"Success Metrics" 可衡量**：不是"会更好"，而是"零次这类 bug"
7. **风险与缓解措施完整**：预见 4 类风险 + 对策

### ⚠️ 缺点 / 风险点

1. **缺少用户验证**：两个 Claude 实例的反馈是否具有普遍性？
2. **没有"反对意见"**：所有改进都是单方面支持，没有讨论 trade-off
3. **缺少 A/B 测试设计**：怎么知道新 prompt 真的有效？
4. **没考虑对人类开发者的影响**：这些规则对人类也适用吗？
5. **"自我反思疲劳"未量化**：30 秒/任务是估计还是实测？
6. **Mock 门控的可行性存疑**：要求开发者"先看接口"在快速开发中可能反人性

## 🔗 七、与其他文档的关联

| 关联文档 | 关联方式 |
|---------|---------|
| [verification-before-completion/SKILL.md](file://e:/WorkStation/MyProject/superpowers/skills/verification-before-completion/SKILL.md) | 本文档直接驱动此技能的更新 |
| [testing-anti-patterns/SKILL.md](file://e:/WorkStation/MyProject/superpowers/skills/test-driven-development/testing-anti-patterns.md) | 添加"反模式 6：Mock-接口 偏离" |
| [subagent-driven-development/SKILL.md](file://e:/WorkStation/MyProject/superpowers/skills/subagent-driven-development/SKILL.md) | 4 处更新（流程卫生、精简上下文、自我反思、技能读取） |
| [requesting-code-review/SKILL.md](file://e:/WorkStation/MyProject/superpowers/skills/requesting-code-review/SKILL.md) | 添加"显式文件读取" |
| [2025-11-22-opencode-support-design.md](file://e:/WorkStation/MyProject/superpowers/docs/plans/2025-11-22-opencode-support-design.md) | OpenCode 设计中已经预见了"工具映射"问题，呼应了本文档的"问题 8" |

## 📝 八、文档状态评价

| 评价维度 | 评分 | 说明 |
|---------|------|------|
| **问题诊断深度** | ⭐⭐⭐⭐⭐ | 真实案例、具体代码、根因分析 |
| **改进可执行性** | ⭐⭐⭐⭐ | 门控函数清晰，但缺验证 |
| **风险识别** | ⭐⭐⭐⭐⭐ | 4 类风险 + 缓解措施 |
| **可衡量性** | ⭐⭐⭐⭐ | 成功指标明确 |
| **诚实性** | ⭐⭐⭐⭐ | 承认未解之谜和权衡 |
| **总分** | **4.4/5** | 一份高质量的"问题-解决方案"文档 |

## 🎓 九、给读者的启示

### 如果你是 AI Agent 的开发者

1. **真实失败案例比抽象原则更说服人**——本文档的成功在于"具体场景"
2. **"验证意图而非操作"是 AI 时代的新主题**——这是 Claude Code 用户最常踩的坑
3. **Mock 安全性是 LLM 编程的暗角**——TypeScript 不会保护你

### 如果你是项目维护者

1. **重视用户反馈的系统化收集**——本文档是"用户驱动演进"的范式
2. **分阶段实施降低风险**——不要一次性应用所有改进
3. **为"自我反思"等元能力投资**——这是 AI 编程相比传统编程的新维度

### 如果你是 AI Agent 实施者

1. **遇到 mock 相关问题，先看接口**——本文档问题 5 的方法
2. **配置变更后，验证"差异"而非"成功"**——本文档问题 1 的方法
3. **报告完成前，退后一步自我审视**——本文档问题 4 的方法

## 🚀 十、可能的后续工作

1. **A/B 测试这些改进**：在真实项目中对比改进前后的 bug 率
2. **建立反馈收集机制**：让用户问题能被系统化地反映到技能更新
3. **添加自动化检测**：例如 lint 规则检测 "TypeScript mock 与接口不符"
4. **跨 AI 平台验证**：这些规则在 OpenCode、Codex 上是否同样有效？
5. **编写"反模式 v2"**：把 8 个问题扩展为可独立引用的反模式库
