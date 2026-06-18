# writing-skills/anthropic-best-practices.md 深度分析报告

> 本报告分析 Anthropic 官方发布的"技能编写最佳实践"文档。它是 `writing-skills/SKILL.md` 的官方补充参考资料。

---

## 1. 基本信息

| 项 | 值 |
|----|----|
| **文件名** | `anthropic-best-practices.md` |
| **总行数** | 1151 行 |
| **文档定位** | Anthropic 官方技能编写指南 |
| **与 SKILL.md 关系** | "complement the TDD-focused approach"（补充 TDD 聚焦的方法） |
| **来源** | Anthropic Agent Skills 官方文档（agentskills.io） |
| **目标读者** | 技能作者、AI 工程师、prompt 工程师 |
| **核心方法** | 评估驱动开发 + 与 Claude A/B 协作迭代 |

---

## 2. 目的与背景

### 2.1 文档的官方地位

这是 Anthropic 官方发布的技能编写指南，被 superpowers 项目作为权威参考纳入 `writing-skills/` 目录。它**不是** superpowers 自己写的，而是从 Anthropic Agent Skills 文档导入的。

### 2.2 三大核心命题

1. **"Concise is key"**（简洁是关键）—— 上下文窗口是公共资源
2. **"Match the level of specificity to the task's fragility and variability"**（匹配自由度与脆弱性）
3. **"Test with all models you plan to use"**（用所有模型测试）

### 2.3 解决的三个核心问题

| 问题 | 本文档提供的方案 |
|------|------------------|
| Token 浪费 | 简洁性原则 + Token 预算 500 行 |
| 指令模糊不清 | 三档自由度（高/中/低）+ 自由度的"机器人路径"类比 |
| 技能不可用 | 评估驱动开发 + 与 Claude A/B 协作迭代 |

---

## 3. 核心技术决策

### 3.1 决策 1：简洁性（Concise）作为首要原则

> "The context window is a public good."

**论证链：**
- 上下文窗口是有限的公共资源
- 技能与系统提示、对话历史、其他技能元数据竞争
- "Default assumption: Claude is already very smart"
- 简洁版本的 pdfplumber 示例：50 token vs 150 token（3x 差异）

**关键技术：**
- "Challenge each piece of information"（质疑每条信息）
- "Does Claude really need this explanation?"
- "Can I assume Claude knows this?"
- "Does this paragraph justify its token cost?"

### 3.2 决策 2：自由度的"机器人路径"类比

> "Think of Claude as a robot exploring a path"

**三档自由度：**

| 自由度 | 适用场景 | 示例 |
|--------|----------|------|
| 高（文本指令） | 多方法有效、决策依赖上下文 | 代码审查流程 |
| 中（伪代码+参数） | 首选模式存在、配置影响行为 | 报告生成 |
| 低（特定脚本） | 脆弱性高、一致性关键、必须按序 | 数据库迁移 |

**机器人路径类比：**
- **窄桥带悬崖** = 低自由度（具体护栏、精确指令）
- **开放田地无危险** = 高自由度（一般方向、信任 agent）

**价值：** 把抽象的"指令详细程度"问题转化为具体可操作的决策框架。

### 3.3 决策 3：多模型测试的"差异化"思维

> "What works perfectly for Opus might need more detail for Haiku."

**模型分级：**
- **Haiku**（快速、经济）：技能提供足够指导？
- **Sonnet**（平衡）：技能清晰高效？
- **Opus**（强大推理）：技能避免过度解释？

**风险：** 跨模型时指令的"最低公分母"问题——为 Haiku 写清楚的指令可能对 Opus 显得冗余。

### 3.4 决策 4：渐进式披露（Progressive Disclosure）作为核心架构

**4 个子模式：**

**模式 1：高级指南 + 参考**
```
SKILL.md（概述 + 入口）
├── FORMS.md（按需加载）
├── reference.md（按需加载）
└── examples.md（按需加载）
```

**模式 2：领域特定组织**
```
bigquery-skill/
├── SKILL.md（导航）
└── reference/
    ├── finance.md（按域）
    ├── sales.md
    ├── product.md
    └── marketing.md
```

**模式 3：条件性细节**
- 简单内容内联
- 高级内容链接到独立文件

**关键约束：**
- 引用保持 1 层深（避免"advanced.md → details.md"的 2 层嵌套）
- 超过 100 行的参考文件需要目录

### 3.5 决策 5：评估驱动开发

> "Create evaluations BEFORE writing extensive documentation"

**5 步流程：**
1. 识别差距：运行 Claude 无技能，记录具体失败
2. 创建评估：构建 3 个测试这些差距的场景
3. 建立基线：测量无技能的表现
4. 编写最小指令：刚好足够通过评估
5. 迭代：执行评估、与基线比较、改进

**评估结构示例：**
```json
{
  "skills": ["pdf-processing"],
  "query": "Extract all text from this PDF file...",
  "files": ["test-files/document.pdf"],
  "expected_behavior": [...]
}
```

**与 writing-skills/SKILL.md 的 TDD 方法的关系：**
- 两者都强调"先失败、后写"
- `writing-skills` 强调**压力场景 + 合理化表**（针对纪律类技能）
- `anthropic-best-practices` 强调**评估 + 期望行为列表**（针对技术/参考类技能）

### 3.6 决策 6：与 Claude A/B 协作的迭代模式

**双 Claude 协作模式：**

| 角色 | 任务 |
|------|------|
| **Claude A** | 帮助设计、改进技能 |
| **Claude B** | 实际使用技能执行任务 |

**理由：**
> "Claude models understand both how to write effective agent instructions and what information agents need."

**迭代循环：**
1. Claude A 创建技能
2. Claude B 在真实任务中使用
3. 观察 Claude B 的行为
4. 将观察带回 Claude A 进行改进
5. 重复

**观察的 4 个关键信号：**
- 意外的探索路径（结构不直观？）
- 错过的连接（链接不够明显？）
- 过度依赖某些部分（应移到主文件？）
- 忽略的内容（不必要或信号不足？）

### 3.7 决策 7：常见模式（模板、示例、条件工作流）

**3 个可复用模式：**

| 模式 | 用途 | 强度 |
|------|------|------|
| 模板 | 严格输出格式 | 强约束 |
| 示例 | 风格示范 | 弱约束 |
| 条件工作流 | 决策点引导 | 中等约束 |

**关键洞见：** 严格性应匹配需求——API 响应要严格，分析任务要灵活。

### 3.8 决策 8：可执行代码的"解决问题，不要推卸"

**Ousterhout 定律引用：**
> "Avoid 'voodoo constants' (Ousterhout's law)"

**好 vs 坏：**
- ✅ 处理错误（FileNotFoundError → 创建默认文件）
- ❌ 抛出异常让 Claude 处理

**魔法数字 vs 自文档化：**
- ✅ `# HTTP requests typically complete within 30 seconds`
- ❌ `TIMEOUT = 47  # Why 47?`

**预制脚本的好处：**
- 更可靠（vs 生成代码）
- 节省 token
- 节省时间
- 跨使用一致

**关键区别（执行 vs 阅读）：**
- ✅ "运行 `analyze_form.py` 提取字段"（执行）
- ✅ "有关字段提取算法，请参阅 `analyze_form.py`"（阅读）

### 3.9 决策 9：MCP 工具的全限定命名

> "Use ServerName:tool_name"

**示例：**
- ✅ `BigQuery:bigquery_schema`
- ✅ `GitHub:create_issue`

**理由：** 避免多 MCP 服务器环境下的"工具未找到"错误。

### 3.10 决策 10：避免 Windows 风格路径

> "Always use forward slashes in file paths, even on Windows"

**理由：** Unix 风格路径跨平台，Windows 反斜杠在 Unix 上失败。

---

## 4. 文件结构剖析

```
anthropic-best-practices.md (1151 行)
├── 标题 + 概述 (8 行)
├── Core principles (143 行)
│   ├── Concise is key (50 行) - 50 vs 150 token 对比
│   ├── Set appropriate degrees of freedom (74 行) - 三档 + 机器人类比
│   └── Test with all models you plan to use (11 行)
├── Skill structure (266 行)
│   ├── YAML Frontmatter 提示 (8 行)
│   ├── Naming conventions (35 行) - 动名词形式
│   ├── Writing effective descriptions (50 行) - 第三人称 + 关键术语
│   ├── Progressive disclosure patterns (175 行) - 3 模式 + 视觉图
│   │   ├── 视觉概述（简单到复杂）(2 行)
│   │   ├── 模式 1：高级指南+参考 (24 行)
│   │   ├── 模式 2：领域特定组织 (19 行)
│   │   ├── 模式 3：条件性细节 (18 行)
│   │   ├── 避免深度嵌套 (28 行)
│   │   └── 用目录结构化长参考文件 (23 行)
├── Workflows and feedback loops (137 行)
│   ├── Use workflows for complex tasks (96 行) - 2 示例
│   └── Implement feedback loops (40 行) - 2 示例
├── Content guidelines (51 行)
│   ├── Avoid time-sensitive information (32 行)
│   └── Use consistent terminology (19 行)
├── Common patterns (123 行)
│   ├── Template pattern (50 行)
│   ├── Examples pattern (40 行)
│   └── Conditional workflow pattern (25 行)
├── Evaluation and iteration (102 行)
│   ├── Build evaluations first (33 行)
│   └── Develop Skills iteratively with Claude (60 行)
├── Anti-patterns to avoid (29 行)
│   ├── Avoid Windows-style paths (8 行)
│   └── Avoid offering too many options (16 行)
├── Advanced: Skills with executable code (240 行)
│   ├── Solve, don't punt (52 行)
│   ├── Provide utility scripts (62 行)
│   ├── Use visual analysis (20 行)
│   ├── Create verifiable intermediate outputs (33 行)
│   ├── Package dependencies (8 行)
│   ├── Runtime environment (37 行)
│   ├── MCP tool references (18 行)
│   └── Avoid assuming tools are installed (18 行)
├── Technical notes (10 行)
│   ├── YAML frontmatter requirements (5 行)
│   └── Token budgets (5 行)
├── Checklist for effective Skills (35 行)
│   ├── Core quality (10 项)
│   ├── Code and scripts (8 项)
│   └── Testing (4 项)
└── Next steps (15 行) - 3 个后续链接
```

**结构特点：**
1. **Skill structure 占 23%**（266 行）—— 反映"结构化"是核心
2. **Executable code 占 21%**（240 行）—— 大量示例针对带代码的技能
3. **Workflows + Feedback loops 占 12%**（137 行）—— 强调验证循环
4. **没有显式的"反合理化"章节**——与 writing-skills/SKILL.md 的关键区别

---

## 5. 优点 / 亮点

### 5.1 范式创新："为机器优化的文档学"

> "Default assumption: Claude is already very smart"

整篇文档围绕一个核心命题：**不要为初学者写文档**。Claude 已经聪明，你只需要补充它不知道的特定上下文。这与传统的"教学文档"哲学截然不同。

### 5.2 "机器人路径"类比的认知有效性

> "Think of Claude as a robot exploring a path"

- **窄桥带悬崖** = 低自由度（精确护栏）
- **开放田地无危险** = 高自由度（一般方向）

这个类比把抽象的"指令详细程度"问题转化为**视觉化、可决策**的框架。

### 5.3 渐进式披露的工程化

**4 个子模式 + 1 个关键约束**：
- 模式 1：高级指南 + 参考
- 模式 2：领域特定组织
- 模式 3：条件性细节
- 关键约束：1 层深引用

**Ousterhout 定律的引用**（"voodoo constants"）—— 借用成熟的软件工程原则。

### 5.4 评估驱动开发的明文化

> "Create evaluations BEFORE writing extensive documentation"

5 步流程 + 评估结构示例 + 与"先写文档"的反模式对比。这是"文档 TDD"的官方版本。

### 5.5 双 Claude 协作（A/B 模式）

**Claude A 改进 vs Claude B 使用**的分工：
> "Claude A helps you design and refine instructions, while Claude B tests them in real tasks."

**理论依据：**
> "Claude models understand both how to write effective agent instructions and what information agents need."

这是一种**元层面的能力认知**——LLM 既是被优化对象，也是优化器。

### 5.6 可执行代码的"解决问题"哲学

> "Solve, don't punt"

**Ousterhout 定律的应用**：
- 错误处理要明确（不要让 Claude 推断）
- 配置参数要自文档化（不要"voodoo constants"）
- 预制脚本要可靠（不要"让 Claude 生成"）

### 5.7 平台特定性的诚实承认

> "**claude.ai**: Can install packages from npm and PyPI and pull from GitHub repositories"
> "**Anthropic API**: Has no network access and no runtime package installation"

**价值：** 不假装"一招通用"——技能在不同平台有不同约束。

### 5.8 跨平台路径的一致性

> "Always use forward slashes in file paths, even on Windows"

简单但关键：跨平台兼容性是技能"可移植"的基础。

### 5.9 有效技能清单（Checklist）的实用性

3 类检查 × 22 项：
- 核心质量 10 项
- 代码和脚本 8 项
- 测试 4 项

**价值：** 把"好技能"从抽象品质转化为可验证的清单。

---

## 6. 缺点 / 风险 / 局限

### 6.1 与 writing-skills/SKILL.md 的潜在冲突

**冲突点 1：Token 预算 vs 抗合理化**
- `anthropic-best-practices`：SKILL.md < 500 行（强制约束）
- `writing-skills/SKILL.md`：抗合理化需要详细列举所有漏洞（支持扩展）

**冲突点 2：自由度 vs 铁律**
- `anthropic-best-practices`：高自由度适合代码审查等场景
- `writing-skills/SKILL.md`：纪律类技能需要"绝对命令"（低自由度）

**冲突点 3：评估 vs 压力场景**
- `anthropic-best-practices`：评估 = 期望行为列表（3 个场景）
- `writing-skills/SKILL.md`：压力场景 = 3+ 组合压力（纪律类）

**问题：** 文档没有给"何时遵循哪个"的优先级规则。

### 6.2 评估驱动开发的可操作性不足

> "Build three scenarios that test these gaps"

**问题：**
- 3 个场景的"质量标准"是什么？
- 场景如何生成？给方法了吗？
- 评估是手工还是自动化？文档说"Users can create their own evaluation system"——把责任推给用户。

### 6.3 缺少失败模式分析

文档关注"如何写好技能"但**没有分析"技能为什么会失败"**：
- 什么导致 agent 不发现技能？（CSO 失败）
- 什么导致 agent 不遵从技能？（抗合理化失败）
- 什么导致技能被错误触发？（误报）

### 6.4 双 Claude 协作的认知风险

> "Claude A helps you design and refine instructions, while Claude B tests them in real tasks."

**风险：**
- 假设 Claude A 永远比 Claude B 更有判断力
- 没有讨论"Claude A 的偏见"问题
- 当 Claude A 和 Claude B 实际是同一模型时，反馈循环可能强化偏见

### 6.5 模型分级的简化风险

> "Claude Haiku ... Does the Skill provide enough guidance?"

**问题：**
- 实际使用中 agent 可能是混合模型
- "足够的指导"对 Haiku 来说没有量化标准
- 文档没有给"为 Haiku 写技能 vs 为 Opus 写技能"的具体差异示例

### 6.6 缺少"何时不写技能"

文档没有讨论**技能的反面**：
- 什么时候**不要**写技能？
- 什么时候**应该**用普通文档？
- 什么时候**应该**用代码而非文档？

`writing-skills/SKILL.md` 的 "Don't create for" 部分做了这件事，本文档没做。

### 6.7 渐进式披露的"层数"边界不清

> "Keep references one level deep from SKILL.md"

**问题：**
- 1 层深的"反例"是什么？2 层一定不行？
- 跨领域组织（`reference/finance.md` → 内部子文件）算 1 层还是 2 层？
- "Layer" 的定义模糊。

### 6.8 实用脚本的"执行 vs 阅读"区分不够具体

> "**Execute the script** (most common)"
> "**Read it as reference** (for complex logic)"

**问题：**
- "复杂逻辑"的定义是什么？
- 当脚本既有"数据获取"又有"算法"两部分时怎么办？

### 6.9 缺少"技能的版本管理"

- 技能如何升级？
- 旧版本的 agent 会用新版本的技能吗？
- 如何处理"breaking changes"？
- 文档完全没有讨论这一点。

### 6.10 视觉图（screenshots）无法翻译

文档中包含 3 张 Anthropic 截图（simple-file、bundling-content、executable-scripts），这些图片**无法在中文版本中重新生成**。中文翻译保留了 `<img src="...">` 标签但截图本身仍是英文的。

**影响：** 视觉示例的价值在中文版本中部分丧失。

### 6.11 评估结构的局限性

```json
{
  "expected_behavior": [
    "Successfully reads the PDF file using an appropriate PDF processing library or command-line tool",
    "Extracts text content from all pages in the document without missing any pages",
    "Saves the extracted text to a file named output.txt in a clear, readable format"
  ]
}
```

**问题：**
- 期望行为是"叙述性"而非"可断言"——没有"成功/失败"的二元判断
- 难以做"通过率"统计
- 没有给出"评分量表"

### 6.12 缺乏"反模式检测"工具

文档有"避免的反模式"小节（Windows 路径、太多选择），但**没有方法论**：
- 如何自动检测这些反模式？
- 是否有 lint 工具？
- 反馈机制是什么？

### 6.13 调色板（formatting）不一致

文档使用 `<Note>`、`<Warning>`、`<Tip>`、`<CardGroup>`、`<Card>` 等 Mintlify 组件。这些组件：
- 不是标准 Markdown
- 在其他渲染器中可能不显示
- 限制了可移植性

### 6.14 缺少"跨文化"考量

- 文档全英文，未讨论多语言技能的设计
- 翻译策略、文化适配、本地化未涉及

---

## 7. 关键洞察

### 7.1 洞察 1："为机器优化的文档学"是新范式

传统文档学的优化目标：可读性、教育性、完整性。
本范式的优化目标：**可发现性（CSO）+ 可触发性 + 可遵从性 + Token 经济性**。

这是文档学从"给人读"到"给 agent 加载"的根本转变。

### 7.2 洞察 2：自由度是"具体性的连续谱"而非"二元"

三档（高/中/低）实操上需要：
- **任务脆弱性**：操作可逆吗？失败成本高吗？
- **上下文依赖**：决策需要什么上下文？
- **配置复杂度**：参数越多自由度应越低

"机器人路径"类比把这个判断视觉化。

### 7.3 洞察 3：渐进式披露的工程价值

> "No context penalty for large files"

这是**文件系统作为认知架构**的设计。Claude 不需要"理解整个技能库"，它只需要"知道技能存在 + 按需加载"。

**对软件工程的类比：** 类似于"包管理 + 按需 import"取代"全量 include"。

### 7.4 洞察 4：评估驱动开发是"文档的 TDD"

5 步流程：识别差距 → 创建评估 → 建立基线 → 编写最小指令 → 迭代。

**与 writing-skills/SKILL.md 的"压力场景 TDD"的对比：**

| 维度 | Anthropic 评估驱动 | Superpowers 压力 TDD |
|------|---------------------|----------------------|
| 目标 | 测功能性 | 测抗压性 |
| 失败信号 | 期望行为未达成 | 钻空子/合理化 |
| 强度 | 中等 | 高（3+ 组合压力） |
| 适用 | 技术/参考类技能 | 纪律类技能 |

**两者互补不互斥。**

### 7.5 洞察 5：双 Claude 协作是"元层面"的能力反思

> "Claude models understand both how to write effective agent instructions and what information agents need."

这是承认 LLM 既是"作者"也是"读者"的**自我指涉特性**。文档利用这一特性构建了迭代改进的飞轮。

### 7.6 洞察 6：MCP 工具的全限定命名反映"命名空间焦虑"

> "Use ServerName:tool_name to avoid 'tool not found' errors"

随着 MCP 生态扩展，工具冲突会增多。这是一种"早期规范"——现在定义规则比事后重构便宜。

### 7.7 洞察 7：可执行代码的"解决问题"哲学

> "Solve, don't punt"

这是一种**对 LLM 能力的信任边界**的明确表态：
- LLM 不应该"承担错误处理"——这是脚本的责任
- LLM 不应该"推断魔法数字"——这是设计者的责任
- LLM 不应该"决定执行还是阅读"——这是文档的责任

**哲学：** LLM 是协作者而非兜底者。

### 7.8 洞察 8：Token 经济性是新的文档质量维度

> "保持 SKILL.md 主体在 500 行以内"

文档质量的传统维度：准确性、完整性、可读性。
**新增维度：Token 成本**。

每个加载的技能都是成本。这改变了文档学的优先级——**信息密度**比"完整性"更重要。

### 7.9 洞察 9：平台特定性的诚实承认

文档明确区分 claude.ai（可安装包）和 Anthropic API（无网络），**不假装通用**。

这是"文档工程"的成熟态度：承认约束、给具体方案、不抽象到无意义。

### 7.10 洞察 10：有效技能清单的"原子化"质量标准

3 类 × 22 项的 checklist 把"好技能"分解为可独立验证的子项。这种"原子化"是质量工程的成熟方法。

---

## 8. 关联文档

### 8.1 内部关联（同目录）

| 文件 | 关系 |
|------|------|
| `SKILL.md` | **主技能**——明确说"complement the TDD-focused approach" |
| `persuasion-principles.md` | 心理学基础——本指南的"权威、承诺"语言使用 |
| `testing-skills-with-subagents.md` | 测试方法论——更详细的压力测试 |
| `graphviz-conventions.dot` | Graphviz 风格——本指南没怎么用流程图 |
| `examples/CLAUDE_MD_TESTING.md` | 测试用例——可作为本指南的实战示例 |

### 8.2 外部关联（其他技能）

| 技能 | 关系 |
|------|------|
| `test-driven-development` | TDD 是文档评估的灵感来源 |
| `using-superpowers` | 元技能——本指南是其补充 |
| `systematic-debugging` | 评估场景设计的参考 |
| `subagent-driven-development` | Claude B 角色的实施方式 |

### 8.3 上游文档

| 来源 | 关系 |
|------|------|
| [Anthropic Agent Skills 官方文档](https://docs.claude.com/en/docs/agents-and-tools/agent-skills/overview) | **官方源文档**——本文件是导入版本 |
| Ousterhout 的软件哲学 | "voodoo constants" 概念来源 |
| 评估驱动开发（EDP）文献 | 评估结构的灵感 |

---

## 9. 状态评价

### 9.1 完成度：★★★★★（5/5）

- 12 个主要章节覆盖完整（原则、结构、工作流、内容、模式、评估、反模式、高级、清单）
- 22 项 checklist 提供可操作验证
- 3 个渐进式披露模式 + 1 个关键约束
- 多模型测试建议

### 9.2 一致性：★★★★（4/5）

- 内部自洽（简洁、自由度匹配、评估驱动）
- **扣分点**：与 writing-skills/SKILL.md 在某些方面存在未解决的冲突

### 9.3 可操作性：★★★★（4/5）

- 23 个代码示例 + 2 个工作流示例 + 1 个评估结构示例
- 22 项 checklist
- **扣分点**：评估生成、压力场景生成、模型分级具体差异等"方法论"层面不够具体

### 9.4 创新度：★★★★（4/5）

- "为机器优化的文档学"是范式创新
- 机器人路径类比是认知创新
- 双 Claude 协作是元层面创新
- **扣分点**：渐进式披露、评估驱动开发等概念在其他领域已存在

### 9.5 整体评价：**8.5 / 10**

这是一份**官方权威的、范式正确的、可操作的**技能编写指南。它与 superpowers 的 `writing-skills/SKILL.md` 形成互补：
- **Anthropic**：官方权威、覆盖全面、面向通用技能
- **superpowers**：方法论深入、纪律性更强、针对压力场景

**两者协同而非冗余**，但用户需要知道"何时用哪个"。

---

## 10. 给读者的启示

### 10.1 对技能作者

1. **简洁是首要原则**：挑战每条信息——"Does Claude really need this?"
2. **匹配自由度与脆弱性**：数据库迁移要低自由度，代码审查可高自由度
3. **渐进式披露**：SKILL.md 是目录不是全集——大型参考放单独文件
4. **测试用所有模型**：Haiku、Sonnet、Opus 行为不同
5. **执行意图明确**：写"运行 X"还是"阅读 X"——必须二选一

### 10.2 对 AI 工程师

1. **文档学范式转变**：从"教学文档"到"接口文档"
2. **Token 是成本**：每个加载都消耗，写得越短越好
3. **评估驱动开发**：先建评估、后写文档
4. **Ousterhout 定律适用**：不要让 LLM 推断魔法数字
5. **平台特定性**：承认约束、不假装通用

### 10.3 对 superpowers 用户

1. **本指南是 writing-skills 的补充**：两者结合使用
2. **Anthropic 评估 vs Superpowers 压力测试**：测不同维度，可同时使用
3. **MCP 工具的全限定命名**：用 ServerName:tool_name
4. **跨平台路径**：用 `/` 不用 `\`
5. **每条指令都"挑刺"**：默认 Claude 已经聪明

### 10.4 对更广泛的提示工程

1. **CSO 是新 SEO**：description、关键词、命名 = LLM 发现机制
2. **Token 经济学的兴起**：信息密度 > 信息完整性
3. **评估驱动 vs 压力测试**：互补的方法论
4. **Ousterhout 定律的 LLM 适配**：让脚本"自文档化"
5. **预制 vs 生成的权衡**：预制脚本更可靠、更省 token

---

## 11. 后续工作 / 改进方向

### 11.1 应补充的内容

1. **与 writing-skills/SKILL.md 的冲突解决**：何时用哪个？
2. **评估的具体生成方法**：3 个场景如何产生？
3. **模型分级的具体差异示例**：同一技能在 Haiku/Sonnet/Opus 下的具体不同
4. **何时不写技能的反面论证**：技能 vs 普通文档的边界
5. **技能版本管理**：升级、breaking changes、向后兼容

### 11.2 应展开的讨论

1. **跨文化技能设计**：多语言、本地化
2. **自动 lint 工具**：检测反模式
3. **量化指标**：评估的通过率、token 节省率
4. **平台特定性更详细**：每个平台的约束完整列表
5. **团队协作工作流**：多开发者同时维护技能库

### 11.3 元层面

本指南是 Anthropic 官方文档，**不应被 superpowers 改写**，而应被引用和补充。建议：

- 在 superpowers `using-superpowers` 技能中明确"何时参考 Anthropic 官方文档 vs 何处使用 superpowers 自己的方法"
- 提供"两种方法"的对照表
- 在测试场景中混合使用两种方法（评估 + 压力）

### 11.4 哲学层面

最大的开放问题是：**"LLM 时代的文档学"是否会形成独立学科**？

当前的迹象：
- CSO（LLM 搜索优化）类似 SEO
- Token 经济学类似性能工程
- 评估驱动开发类似测试驱动开发
- 双 Claude 协作类似结对编程

**未来可能的方向：**
- "Skill Engineering" 作为正式学科
- "SkillOps" 作为 DevOps 的对应
- 技能市场 / 技能版本控制的标准协议

---

## 附录：本分析的方法论

本报告按 11 节统一结构组织：
1. 基本信息
2. 目的与背景
3. 核心技术决策
4. 文件结构剖析
5. 优点/亮点
6. 缺点/风险/局限
7. 关键洞察
8. 关联文档
9. 状态评价
10. 给读者的启示
11. 后续工作

所有论断都基于 anthropic-best-practices.md 文本和其在 superpowers 生态中的角色。**不做超出文档声明的推论**。
