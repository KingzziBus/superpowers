# 技能编写最佳实践

> 学习如何编写 Claude 能够发现并成功使用的有效技能。

好的技能应简洁、良好结构化，并通过实际使用进行测试。本指南提供实用的编写决策，帮助你编写 Claude 能够发现并有效使用的技能。

关于技能如何工作的概念背景，请参阅 [技能概述](/en/docs/agents-and-tools/agent-skills/overview)。

## 核心原则

### 简洁是关键

[上下文窗口](https://platform.claude.com/docs/en/build-with-claude/context-windows)是一种公共资源。你的技能与其他 Claude 需要知道的内容共享上下文窗口，包括：

* 系统提示
* 对话历史
* 其他技能的元数据
* 你的实际请求

并不是技能中的每个 token 都有即时成本。在启动时，只有所有技能的元数据（名称和描述）被预加载。Claude 仅在技能变得相关时才读取 SKILL.md，并且仅在需要时读取其他文件。然而，在 SKILL.md 中保持简洁仍然很重要：一旦 Claude 加载它，每个 token 都会与对话历史和其他上下文竞争。

**默认假设**：Claude 已经非常聪明

只添加 Claude 还没有的上下文。对每条信息进行质疑：

* "Claude 真的需要这个解释吗？"
* "我可以假设 Claude 知道这个吗？"
* "这段话是否值得它的 token 成本？"

**好示例：简洁**（约 50 个 token）：

````markdown  theme={null}
## 提取 PDF 文本

使用 pdfplumber 提取文本：

```python
import pdfplumber

with pdfplumber.open("file.pdf") as pdf:
    text = pdf.pages[0].extract_text()
```
````

**坏示例：太冗长**（约 150 个 token）：

```markdown  theme={null}
## 提取 PDF 文本

PDF（便携文档格式）文件是一种常见的文件格式，包含
文本、图像和其他内容。要从 PDF 中提取文本，你需要
使用一个库。有许多可用于 PDF 处理的库，但我们
推荐 pdfplumber，因为它易于使用且能处理大多数情况。
首先，你需要使用 pip 安装它。然后你可以使用下面的代码...
```

简洁版本假设 Claude 知道 PDF 是什么以及库是如何工作的。

### 设置适当的自由度

将具体性级别与任务的脆弱性和可变性相匹配。

**高自由度**（基于文本的指令）：

在以下情况下使用：

* 多种方法都有效
* 决策取决于上下文
* 启发式方法指导方法

示例：

```markdown  theme={null}
## 代码审查流程

1. 分析代码结构和组织
2. 检查潜在的 bug 或边缘情况
3. 建议可读性和可维护性的改进
4. 验证是否遵守项目约定
```

**中等自由度**（带参数的伪代码或脚本）：

在以下情况下使用：

* 存在首选模式
* 一些变化是可接受的
* 配置影响行为

示例：

````markdown  theme={null}
## 生成报告

使用此模板并根据需要进行自定义：

```python
def generate_report(data, format="markdown", include_charts=True):
    # 处理数据
    # 以指定格式生成输出
    # 可选择包含可视化
```
````

**低自由度**（特定脚本，参数少或无）：

在以下情况下使用：

* 操作脆弱且容易出错
* 一致性至关重要
* 必须遵循特定顺序

示例：

````markdown  theme={null}
## 数据库迁移

完全按照此脚本运行：

```bash
python scripts/migrate.py --verify --backup
```

不要修改命令或添加其他标志。
````

**类比**：将 Claude 想象成探索路径的机器人：

* **两侧有悬崖的窄桥**：只有一种安全的前进方式。提供具体的护栏和精确的指令（低自由度）。示例：必须按精确顺序运行的数据库迁移。
* **没有危险的开放田地**：许多路径都通向成功。给出一般方向并信任 Claude 找到最佳路线（高自由度）。示例：上下文决定最佳方法的代码审查。

### 用你计划使用的所有模型测试

技能作为模型的补充，其有效性取决于底层模型。使用你计划使用的所有模型测试你的技能。

**按模型的测试考虑因素**：

* **Claude Haiku**（快速、经济）：技能是否提供足够的指导？
* **Claude Sonnet**（平衡）：技能是否清晰高效？
* **Claude Opus**（强大推理）：技能是否避免过度解释？

对 Opus 完全适用的内容可能对 Haiku 需要更多细节。如果你计划在多个模型上使用你的技能，请让指令在所有模型上都能良好工作。

## 技能结构

<Note>
  **YAML 前言**：SKILL.md 前言需要两个字段：

  * `name` - 技能的人类可读名称（最多 64 个字符）
  * `description` - 一行描述技能做什么以及何时使用它（最多 1024 个字符）

  有关完整的技能结构详细信息，请参阅 [技能概述](/en/docs/agents-and-tools/agent-skills/overview#skill-structure)。
</Note>

### 命名约定

使用一致的命名模式使技能更易于引用和讨论。我们建议对技能名称使用**动名词形式**（动词 + -ing），因为这清楚地描述了技能提供的活动或能力。

**好的命名示例（动名词形式）**：

* "处理 PDF"
* "分析电子表格"
* "管理数据库"
* "测试代码"
* "编写文档"

**可接受的替代方案**：

* 名词短语："PDF 处理"、"电子表格分析"
* 面向动作："处理 PDF"、"分析电子表格"

**避免**：

* 模糊的名称："Helper"、"Utils"、"Tools"
* 过于通用："Documents"、"Data"、"Files"
* 技能集合中的不一致模式

一致的命名使以下更容易：

* 在文档和对话中引用技能
* 一眼理解技能做什么
* 组织和搜索多个技能
* 维护专业且有凝聚力的技能库

### 编写有效的描述

`description` 字段启用技能发现，应包括技能做什么和何时使用它。

<Warning>
  **始终使用第三人称**。描述被注入到系统提示中，不一致的视角会导致发现问题。

  * **好：** "处理 Excel 文件并生成报告"
  * **避免：** "我可以帮助你处理 Excel 文件"
  * **避免：** "你可以使用它来处理 Excel 文件"
</Warning>

**具体并包含关键术语**。包括技能做什么和何时使用它的具体触发器/上下文。

每个技能恰好有一个 description 字段。描述对技能选择至关重要：Claude 使用它从可能 100+ 个可用技能中选择正确的技能。你的描述必须为 Claude 提供足够的细节以知道何时选择此技能，而 SKILL.md 的其余部分提供实现细节。

有效示例：

**PDF 处理技能**：

```yaml  theme={null}
description: Extract text and tables from PDF files, fill forms, merge documents. Use when working with PDF files or when the user mentions PDFs, forms, or document extraction.
```

**Excel 分析技能**：

```yaml  theme={null}
description: Analyze Excel spreadsheets, create pivot tables, generate charts. Use when analyzing Excel files, spreadsheets, tabular data, or .xlsx files.
```

**Git 提交助手技能**：

```yaml  theme={null}
description: Generate descriptive commit messages by analyzing git diffs. Use when the user asks for help writing commit messages or reviewing staged changes.
```

避免像这样的模糊描述：

```yaml  theme={null}
description: Helps with documents
```

```yaml  theme={null}
description: Processes data
```

```yaml  theme={null}
description: Does stuff with files
```

### 渐进式披露模式

SKILL.md 作为概述，在需要时将 Claude 指向详细材料，就像入门指南中的目录。有关渐进式披露如何工作的解释，请参阅概述中的[技能如何工作](/en/docs/agents-and-tools/agent-skills/overview#how-skills-work)。

**实用指南**：

* 保持 SKILL.md 主体在 500 行以内以获得最佳性能
* 接近此限制时将内容拆分为单独的文件
* 使用下面的模式有效地组织指令、代码和资源

#### 视觉概述：从简单到复杂

一个基本的技能从仅包含元数据和指令的 SKILL.md 文件开始：

<img src="https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-simple-file.png?fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=87782ff239b297d9a9e8e1b72ed72db9" alt="简单的 SKILL.md 文件显示 YAML 前言和 markdown 主体" data-og-width="2048" width="2048" data-og-height="1153" height="1153" data-path="images/agent-skills-simple-file.png" data-optimize="true" data-opv="3" srcset="https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-simple-file.png?w=280&fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=c61cc33b6f5855809907f7fda94cd80e 280w, https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-simple-file.png?w=560&fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=90d2c0c1c76b36e8d485f49e0810dbfd 560w, https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-simple-file.png?w=840&fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=ad17d231ac7b0bea7e5b4d58fb4aeabb 840w, https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-simple-file.png?w=1100&fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=f5d0a7a3c668435bb0aee9a3a8f8c329 1100w, https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-simple-file.png?w=1650&fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=0e927c1af9de5799cfe557d12249f6e6 1650w, https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-simple-file.png?w=2500&fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=46bbb1a51dd4c8202a470ac8c80a893d 2500w" />

随着你的技能增长，你可以捆绑 Claude 仅在需要时加载的额外内容：

<img src="https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-bundling-content.png?fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=a5e0aa41e3d53985a7e3e43668a33ea3" alt="捆绑其他参考文件如 reference.md 和 forms.md" data-og-width="2048" width="2048" data-og-height="1327" height="1327" data-path="images/agent-skills-bundling-content.png" data-optimize="true" data-opv="3" srcset="https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-bundling-content.png?w=280&fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=f8a0e73783e99b4a643d79eac86b70a2 280w, https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-bundling-content.png?w=560&fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=dc510a2a9d3f14359416b706f067904a 560w, https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-bundling-content.png?w=840&fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=82cd6286c966303f7dd914c28170e385 840w, https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-bundling-content.png?w=1100&fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=56f3be36c77e4fe4b523df209a6824c6 1100w, https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-bundling-content.png?w=1650&fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=d22b5161b2075656417d56f41a74f3dd 1650w, https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-bundling-content.png?w=2500w" />

完整的技能目录结构可能如下所示：

```
pdf/
├── SKILL.md              # 主指令（触发时加载）
├── FORMS.md              # 表单填写指南（按需加载）
├── reference.md          # API 参考（按需加载）
├── examples.md           # 使用示例（按需加载）
└── scripts/
    ├── analyze_form.py   # 实用脚本（执行，不加载）
    ├── fill_form.py      # 表单填写脚本
    └── validate.py       # 验证脚本
```

#### 模式 1：带参考的高级指南

````markdown  theme={null}
---
name: PDF Processing
description: Extracts text and tables from PDF files, fills forms, and merges documents. Use when working with PDF files or when the user mentions PDFs, forms, or document extraction.
---

# PDF Processing

## 快速开始

使用 pdfplumber 提取文本：
```python
import pdfplumber
with pdfplumber.open("file.pdf") as pdf:
    text = pdf.pages[0].extract_text()
```

## 高级功能

**表单填写**：有关完整指南，请参阅 [FORMS.md](FORMS.md)
**API 参考**：有关所有方法，请参阅 [REFERENCE.md](REFERENCE.md)
**示例**：有关常见模式，请参阅 [EXAMPLES.md](EXAMPLES.md)
````

Claude 仅在需要时加载 FORMS.md、REFERENCE.md 或 EXAMPLES.md。

#### 模式 2：领域特定组织

对于具有多个领域的技能，按领域组织内容以避免加载不相关的上下文。当用户询问销售指标时，Claude 只需要读取与销售相关的模式，而不需要财务或营销数据。这保持了低 token 使用率和上下文聚焦。

```
bigquery-skill/
├── SKILL.md（概述和导航）
└── reference/
    ├── finance.md（收入、计费指标）
    ├── sales.md（机会、管道、账户）
    ├── product.md（API 使用、功能、采用）
    └── marketing.md（活动、归因、邮件）
```

````markdown SKILL.md theme={null}
# BigQuery 数据分析

## 可用数据集

**财务**：收入、ARR、计费 → 请参阅 [reference/finance.md](reference/finance.md)
**销售**：机会、管道、账户 → 请参阅 [reference/sales.md](reference/sales.md)
**产品**：API 使用、功能、采用 → 请参阅 [reference/product.md](reference/product.md)
**营销**：活动、归因、邮件 → 请参阅 [reference/marketing.md](reference/marketing.md)

## 快速搜索

使用 grep 查找特定指标：

```bash
grep -i "revenue" reference/finance.md
grep -i "pipeline" reference/sales.md
grep -i "api usage" reference/product.md
```
````

#### 模式 3：条件性细节

显示基本内容，链接到高级内容：

```markdown  theme={null}
# DOCX 处理

## 创建文档

使用 docx-js 创建新文档。请参阅 [DOCX-JS.md](DOCX-JS.md)。

## 编辑文档

对于简单的编辑，直接修改 XML。

**对于跟踪更改**：请参阅 [REDLINING.md](REDLINING.md)
**对于 OOXML 细节**：请参阅 [OOXML.md](OOXML.md)
```

Claude 仅在用户需要这些功能时读取 REDLINING.md 或 OOXML.md。

### 避免深度嵌套的引用

当从其他被引用的文件引用时，Claude 可能部分读取文件。遇到嵌套引用时，Claude 可能使用 `head -100` 之类的命令预览内容而不是读取整个文件，导致信息不完整。

**保持引用从 SKILL.md 起一层深**。所有参考文件应直接从 SKILL.md 链接，以确保 Claude 在需要时读取完整的文件。

**坏示例：太深**：

```markdown  theme={null}
# SKILL.md
See [advanced.md](advanced.md)...

# advanced.md
See [details.md](details.md)...

# details.md
Here's the actual information...
```

**好示例：一层深**：

```markdown  theme={null}
# SKILL.md

**基本用法**：[SKILL.md 中的指令]
**高级功能**：请参阅 [advanced.md](advanced.md)
**API 参考**：请参阅 [reference.md](reference.md)
**示例**：请参阅 [examples.md](examples.md)
```

### 用目录结构化较长的参考文件

对于超过 100 行的参考文件，在顶部包含一个目录。这确保 Claude 即使在部分读取预览时也能看到可用信息的完整范围。

**示例**：

```markdown  theme={null}
# API 参考

## 目录
- 身份验证和设置
- 核心方法（创建、读取、更新、删除）
- 高级功能（批处理操作、webhook）
- 错误处理模式
- 代码示例

## 身份验证和设置
...

## 核心方法
...
```

然后 Claude 可以读取完整的文件或跳到特定的部分。

有关此基于文件系统的架构如何启用渐进式披露的详细信息，请参阅下面高级部分中的[运行时环境](#runtime-environment)部分。

## 工作流和反馈循环

### 使用工作流处理复杂任务

将复杂的操作分解为清晰的顺序步骤。对于特别复杂的工作流，提供一个 Claude 可以复制到其响应中并在进行过程中勾选的清单。

**示例 1：研究综合工作流**（针对无代码的技能）：

````markdown  theme={null}
## 研究综合工作流

复制此清单并跟踪你的进度：

```
Research Progress:
- [ ] 步骤 1：阅读所有源文档
- [ ] 步骤 2：识别关键主题
- [ ] 步骤 3：交叉引用声明
- [ ] 步骤 4：创建结构化摘要
- [ ] 步骤 5：验证引用
```

**步骤 1：阅读所有源文档**

审查 `sources/` 目录中的每个文档。注意主要论点和支持证据。

**步骤 2：识别关键主题**

跨来源查找模式。哪些主题重复出现？来源在哪里一致或不一致？

**步骤 3：交叉引用声明**

对于每个主要声明，验证它是否出现在源材料中。注意哪个来源支持每个观点。

**步骤 4：创建结构化摘要**

按主题组织发现。包括：
- 主要声明
- 源材料中的支持证据
- 冲突的观点（如果有）

**步骤 5：验证引用**

检查每个声明是否引用了正确的源文档。如果引用不完整，返回步骤 3。
````

此示例显示了工作流如何应用于不需要代码的分析任务。清单模式适用于任何复杂的多步过程。

**示例 2：PDF 表单填写工作流**（针对有代码的技能）：

````markdown  theme={null}
## PDF 表单填写工作流

复制此清单并在完成时勾选项目：

```
Task Progress:
- [ ] 步骤 1：分析表单（运行 analyze_form.py）
- [ ] 步骤 2：创建字段映射（编辑 fields.json）
- [ ] 步骤 3：验证映射（运行 validate_fields.py）
- [ ] 步骤 4：填写表单（运行 fill_form.py）
- [ ] 步骤 5：验证输出（运行 verify_output.py）
```

**步骤 1：分析表单**

运行：`python scripts/analyze_form.py input.pdf`

这会提取表单字段及其位置，保存到 `fields.json`。

**步骤 2：创建字段映射**

编辑 `fields.json` 以添加每个字段的值。

**步骤 3：验证映射**

运行：`python scripts/validate_fields.py fields.json`

在继续之前修复任何验证错误。

**步骤 4：填写表单**

运行：`python scripts/fill_form.py input.pdf fields.json output.pdf`

**步骤 5：验证输出**

运行：`python scripts/verify_output.py output.pdf`

如果验证失败，返回步骤 2。
````

清晰的步骤可防止 Claude 跳过关键的验证。清单帮助 Claude 和你跟踪多步工作流的进度。

### 实现反馈循环

**常见模式**：运行验证器 → 修复错误 → 重复

此模式大大提高了输出质量。

**示例 1：风格指南遵从性**（针对无代码的技能）：

```markdown  theme={null}
## 内容审查流程

1. 按照 STYLE_GUIDE.md 中的指南起草你的内容
2. 根据清单进行审查：
   - 检查术语一致性
   - 验证示例遵循标准格式
   - 确认所有必需的部分都存在
3. 如果发现问题：
   - 用具体的部分引用记录每个问题
   - 修改内容
   - 再次审查清单
4. 仅当满足所有要求时才继续
5. 完成并保存文档
```

这显示了使用参考文档而不是脚本的验证循环模式。"验证器"是 STYLE\_GUIDE.md，Claude 通过阅读和比较来执行检查。

**示例 2：文档编辑流程**（针对有代码的技能）：

```markdown  theme={null}
## 文档编辑流程

1. 对 `word/document.xml` 进行编辑
2. **立即验证**：`python ooxml/scripts/validate.py unpacked_dir/`
3. 如果验证失败：
   - 仔细查看错误消息
   - 在 XML 中修复问题
   - 再次运行验证
4. **仅当验证通过时才继续**
5. 重新打包：`python ooxml/scripts/pack.py unpacked_dir/ output.docx`
6. 测试输出文档
```

验证循环可以及早捕获错误。

## 内容指南

### 避免时间敏感信息

不要包含会过时的信息：

**坏示例：时间敏感**（将变得错误）：

```markdown  theme={null}
If you're doing this before August 2025, use the old API.
After August 2025, use the new API.
```

**好示例**（使用"旧模式"部分）：

```markdown  theme={null}
## 当前方法

使用 v2 API 端点：`api.example.com/v2/messages`

## 旧模式

<details>
<summary>遗留 v1 API（已弃用 2025-08）</summary>

v1 API 使用：`api.example.com/v1/messages`

此端点不再受支持。
</details>
```

旧模式部分提供历史上下文而不使主要内容杂乱。

### 使用一致的术语

选择一个术语并在整个技能中使用它：

**好 - 一致**：

* 始终使用 "API endpoint"
* 始终使用 "field"
* 始终使用 "extract"

**差 - 不一致**：

* 混合使用 "API endpoint"、"URL"、"API route"、"path"
* 混合使用 "field"、"box"、"element"、"control"
* 混合使用 "extract"、"pull"、"get"、"retrieve"

一致性有助于 Claude 理解和遵循指令。

## 常见模式

### 模板模式

为输出格式提供模板。将严格性级别与你的需求相匹配。

**对于严格的要求**（如 API 响应或数据格式）：

````markdown  theme={null}
## 报告结构

始终使用此确切模板结构：

```markdown
# [分析标题]

## 执行摘要
[关键发现的一段概述]

## 主要发现
- 发现 1 与支持数据
- 发现 2 与支持数据
- 发现 3 与支持数据

## 建议
1. 具体可行的建议
2. 具体可行的建议
```
````

**对于灵活的指导**（当适应有用时）：

````markdown  theme={null}
## 报告结构

这是一个合理的默认格式，但根据分析使用你的最佳判断：

```markdown
# [分析标题]

## 执行摘要
[概述]

## 主要发现
[根据你发现的内容调整部分]

## 建议
[针对具体上下文量身定制]
```

根据具体分析类型按需调整部分。
````

### 示例模式

对于输出质量取决于查看示例的技能，提供输入/输出对，就像在常规提示中一样：

````markdown  theme={null}
## 提交消息格式

按照这些示例生成提交消息：

**示例 1：**
输入：使用 JWT 令牌添加用户身份验证
输出：
```
feat(auth): implement JWT-based authentication

Add login endpoint and token validation middleware
```

**示例 2：**
输入：修复报告中日期显示错误的 bug
输出：
```
fix(reports): correct date formatting in timezone conversion

Use UTC timestamps consistently across report generation
```

**示例 3：**
输入：更新依赖项并重构错误处理
输出：
```
chore: update dependencies and refactor error handling

- Upgrade lodash to 4.17.21
- Standardize error response format across endpoints
```

遵循此样式：type(scope): 简要描述，然后详细解释。
````

示例帮助 Claude 比单独的描述更清楚地理解所需的样式和详细程度。

### 条件工作流模式

指导 Claude 做出决策：

```markdown  theme={null}
## 文档修改工作流

1. 确定修改类型：

   **创建新内容？** → 按照下面的"创建工作流"
   **编辑现有内容？** → 按照下面的"编辑工作流"

2. 创建工作流：
   - 使用 docx-js 库
   - 从头开始构建文档
   - 导出为 .docx 格式

3. 编辑工作流：
   - 解包现有文档
   - 直接修改 XML
   - 每次更改后验证
   - 完成后重新打包
```

<Tip>
  如果工作流变得庞大或复杂且有许多步骤，请考虑将它们推送到单独的文件中，并告诉 Claude 根据手头的任务读取适当的文件。
</Tip>

## 评估和迭代

### 首先建立评估

**在编写大量文档之前创建评估。** 这确保了你的技能解决实际问题而不是记录想象的问题。

**评估驱动的开发**：

1. **识别差距**：在没有技能的情况下对代表性任务运行 Claude。记录具体的失败或缺失的上下文
2. **创建评估**：构建三个测试这些差距的场景
3. **建立基线**：衡量 Claude 在没有技能的情况下的表现
4. **编写最小指令**：创建刚好足够的内容来解决差距并通过评估
5. **迭代**：执行评估，与基线比较，并改进

这种方法确保你正在解决实际问题而不是预测可能永远不会出现的需求。

**评估结构**：

```json  theme={null}
{
  "skills": ["pdf-processing"],
  "query": "Extract all text from this PDF file and save it to output.txt",
  "files": ["test-files/document.pdf"],
  "expected_behavior": [
    "Successfully reads the PDF file using an appropriate PDF processing library or command-line tool",
    "Extracts text content from all pages in the document without missing any pages",
    "Saves the extracted text to a file named output.txt in a clear, readable format"
  ]
}
```

<Note>
  此示例演示了一个带简单测试评分标准的数据驱动评估。我们目前不提供内置的方式来运行这些评估。用户可以创建自己的评估系统。评估是你衡量技能有效性的真相来源。
</Note>

### 与 Claude 一起迭代开发技能

最有效的技能开发过程涉及 Claude 本身。与一个 Claude 实例（"Claude A"）合作创建一个将供其他实例（"Claude B"）使用的技能。Claude A 帮助你设计和改进指令，而 Claude B 在实际任务中测试它们。这是有效的，因为 Claude 模型既理解如何编写有效的 agent 指令，也理解 agent 需要什么信息。

**创建新技能**：

1. **在没有技能的情况下完成任务**：使用普通提示与 Claude A 一起处理问题。在工作时，你会自然地提供上下文、解释偏好并分享程序性知识。注意你重复提供的信息。

2. **识别可重用模式**：完成任务后，识别你提供的对类似未来任务有用的上下文。

   **示例**：如果你完成了 BigQuery 分析，你可能提供了表名、字段定义、过滤规则（如"始终排除测试账户"）和常见查询模式。

3. **要求 Claude A 创建技能**："创建一个技能来捕获我们刚刚使用的这个 BigQuery 分析模式。包括表模式、命名约定和关于过滤测试账户的规则。"

   <Tip>
     Claude 模型原生理解技能格式和结构。你不需要特殊的系统提示或"编写技能"技能来让 Claude 帮助创建技能。只需要求 Claude 创建技能，它将生成具有适当前言和主体内容的正确结构的 SKILL.md 内容。
   </Tip>

4. **审查简洁性**：检查 Claude A 是否添加了不必要的解释。问："删除关于胜率意味着什么的解释 - Claude 已经知道。"

5. **改进信息架构**：要求 Claude A 更有效地组织内容。例如："组织这个，使表模式在单独的文件中。我们以后可能会添加更多表。"

6. **在类似任务上测试**：在相关用例上使用 Claude B（加载了技能的新实例）。观察 Claude B 是否找到正确的信息、正确地应用规则并成功处理任务。

7. **根据观察迭代**：如果 Claude B 遇到困难或遗漏了某些内容，请带着具体内容返回 Claude A："当 Claude 使用此技能时，它忘记了按 Q4 过滤日期。我们应该添加一个关于日期过滤模式的部分吗？"

**迭代现有技能**：

当改进技能时，同样的分层模式继续。你交替进行：

* **与 Claude A 合作**（帮助你改进技能的专家）
* **使用 Claude B 测试**（使用技能执行实际工作的 agent）
* **观察 Claude B 的行为** 并将见解带回 Claude A

1. **在实际工作流中使用技能**：给 Claude B（加载了技能）实际任务，而不是测试场景

2. **观察 Claude B 的行为**：注意它在哪些地方挣扎、成功或做出意外选择

   **观察示例**："当我要求 Claude B 区域销售报告时，它编写了查询但忘记过滤掉测试账户，即使技能提到了此规则。"

3. **返回 Claude A 进行改进**：分享当前的 SKILL.md 并描述你观察到的内容。问："我注意到当我要求区域报告时 Claude B 忘记了过滤测试账户。技能提到了过滤，但也许它不够突出？"

4. **审查 Claude A 的建议**：Claude A 可能建议重新组织以使规则更突出，使用更强的语言如 "MUST filter" 而不是 "always filter"，或重组工作流部分。

5. **应用并测试更改**：使用 Claude A 的改进更新技能，然后在类似请求上使用 Claude B 再次测试

6. **根据使用重复**：遇到新场景时继续这个观察-改进-测试循环。每次迭代都根据实际 agent 行为改进技能，而不是假设。

**收集团队反馈**：

1. 与队友分享技能并观察他们的使用
2. 问：技能在预期时激活吗？指令清楚吗？缺少什么？
3. 合并反馈以解决你自己使用模式中的盲点

**为什么这种方法有效**：Claude A 理解 agent 需求，你提供领域专业知识，Claude B 通过实际使用揭示差距，迭代改进基于观察行为而不是假设。

### 观察 Claude 如何导航技能

当你迭代技能时，注意 Claude 实际上如何在实践中使用它们。注意以下事项：

* **意外的探索路径**：Claude 是否按你未预料的顺序读取文件？这可能表明你的结构不像你认为的那样直观
* **错过的连接**：Claude 是否无法遵循对重要文件的引用？你的链接可能需要更明确或更突出
* **过度依赖某些部分**：如果 Claude 重复读取同一个文件，请考虑该内容是否应该放在主 SKILL.md 中
* **被忽略的内容**：如果 Claude 从不访问捆绑的文件，它可能是不必要的或在主指令中信号不佳

根据这些观察进行迭代，而不是假设。技能元数据中的 `name` 和 `description` 特别关键。Claude 在决定是否针对当前任务触发技能时使用这些。确保它们清楚地描述了技能做什么以及何时应该使用它。

## 要避免的反模式

### 避免 Windows 风格路径

始终在文件路径中使用正斜杠，即使在 Windows 上：

* ✓ **好**：`scripts/helper.py`、`reference/guide.md`
* ✗ **避免**：`scripts\helper.py`、`reference\guide.md`

Unix 风格路径适用于所有平台，而 Windows 风格路径在 Unix 系统上会导致错误。

### 避免提供太多选择

除非必要，否则不要呈现多种方法：

````markdown  theme={null}
**坏示例：太多选择**（令人困惑）：
"You can use pypdf, or pdfplumber, or PyMuPDF, or pdf2image, or..."

**好示例：提供默认值**（带逃生口）：
"Use pdfplumber for text extraction:
```python
import pdfplumber
```

For scanned PDFs requiring OCR, use pdf2image with pytesseract instead."
````

## 高级：带可执行代码的技能

下面的部分侧重于包含可执行脚本的技能。如果你的技能仅使用 markdown 指令，请跳到[有效技能清单](#checklist-for-effective-skills)。

### 解决问题，而不是推卸

在为技能编写脚本时，处理错误情况而不是推卸给 Claude。

**好示例：明确处理错误**：

```python  theme={null}
def process_file(path):
    """Process a file, creating it if it doesn't exist."""
    try:
        with open(path) as f:
            return f.read()
    except FileNotFoundError:
        # Create file with default content instead of failing
        print(f"File {path} not found, creating default")
        with open(path, 'w') as f:
            f.write('')
        return ''
    except PermissionError:
        # Provide alternative instead of failing
        print(f"Cannot access {path}, using default")
        return ''
```

**坏示例：推卸给 Claude**：

```python  theme={null}
def process_file(path):
    # Just fail and let Claude figure it out
    return open(path).read()
```

配置参数也应该被证明合理并记录，以避免"巫毒常量"（Ousterhout 定律）。如果你不知道正确的值，Claude 将如何确定它？

**好示例：自文档化**：

```python  theme={null}
# HTTP requests typically complete within 30 seconds
# Longer timeout accounts for slow connections
REQUEST_TIMEOUT = 30

# Three retries balances reliability vs speed
# Most intermittent failures resolve by the second retry
MAX_RETRIES = 3
```

**坏示例：魔数**：

```python  theme={null}
TIMEOUT = 47  # Why 47?
RETRIES = 5   # Why 5?
```

### 提供实用脚本

即使 Claude 可以编写脚本，预制的脚本也有优势：

**实用脚本的好处**：

* 比生成的代码更可靠
* 节省 token（不需要在上下文中包含代码）
* 节省时间（不需要生成代码）
* 确保跨使用的一致性

<img src="https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-executable-scripts.png?fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=4bbc45f2c2e0bee9f2f0d5da669bad00" alt="在指令文件旁边捆绑可执行脚本" data-og-width="2048" width="2048" data-og-height="1154" height="1154" data-path="images/agent-skills-executable-scripts.png" data-optimize="true" data-opv="3" srcset="https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-executable-scripts.png?w=280&fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=9a04e6535a8467bfeea492e517de389f 280w, https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-executable-scripts.png?w=560&fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=e49333ad90141af17c0d7651cca7216b 560w, https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-executable-scripts.png?w=840&fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=954265a5df52223d6572b6214168c428 840w, https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-executable-scripts.png?w=1100&fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=2ff7a2d8f2a83ee8af132b29f10150fd 1100w, https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-executable-scripts.png?w=1650&fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=48ab96245e04077f4d15e9170e081cfb 1650w, https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-executable-scripts.png?w=2500w" />

上面的图显示了可执行脚本如何与指令文件一起工作。指令文件（forms.md）引用脚本，Claude 可以在不将其内容加载到上下文中的情况下执行它。只有脚本的输出消耗 token。

**重要区别**：在你的指令中明确说明 Claude 是否应该：

* **执行脚本**（最常见）："运行 `analyze_form.py` 提取字段"
* **将其作为参考阅读**（对于复杂逻辑）："有关字段提取算法，请参阅 `analyze_form.py`"

对于大多数实用脚本，优先选择执行，因为它更可靠和高效。有关脚本执行如何工作的详细信息，请参阅下面的[运行时环境](#runtime-environment)部分。

**示例**：

````markdown  theme={null}
## 实用脚本

**analyze_form.py**：从 PDF 提取所有表单字段

```bash
python scripts/analyze_form.py input.pdf > fields.json
```

输出格式：
```json
{
  "field_name": {"type": "text", "x": 100, "y": 200},
  "signature": {"type": "sig", "x": 150, "y": 500}
}
```

**validate_boxes.py**：检查重叠的边界框

```bash
python scripts/validate_boxes.py fields.json
# Returns: "OK" or lists conflicts
```

**fill_form.py**：将字段值应用于 PDF

```bash
python scripts/fill_form.py input.pdf fields.json output.pdf
```
````

### 使用视觉分析

当输入可以渲染为图像时，让 Claude 分析它们：

````markdown  theme={null}
## 表单布局分析

1. 将 PDF 转换为图像：
   ```bash
   python scripts/pdf_to_images.py form.pdf
   ```

2. 分析每个页面图像以识别表单字段
3. Claude 可以在视觉上看到字段位置和类型
````

<Note>
  在此示例中，你需要编写 `pdf_to_images.py` 脚本。
</Note>

Claude 的视觉能力有助于理解布局和结构。

### 创建可验证的中间输出

当 Claude 执行复杂的开放式任务时，它可能会出错。"计划-验证-执行"模式通过让 Claude 首先以结构化格式创建计划，然后使用脚本验证该计划再执行来及早捕获错误。

**示例**：想象一下，要求 Claude 根据电子表格更新 PDF 中的 50 个表单字段。如果没有验证，Claude 可能会引用不存在的字段、创建冲突的值、错过必需的字段或不正确地应用更新。

**解决方案**：使用上面显示的工作流模式（PDF 表单填写），但添加一个在应用更改之前经过验证的中间 `changes.json` 文件。工作流变为：分析 → **创建计划文件** → **验证计划** → 执行 → 验证。

**为什么此模式有效**：

* **及早捕获错误**：验证在应用更改之前发现问题
* **机器可验证**：脚本提供客观验证
* **可逆的规划**：Claude 可以迭代计划而不触及原件
* **清晰的调试**：错误消息指向具体问题

**何时使用**：批处理操作、破坏性更改、复杂验证规则、高风险操作。

**实施技巧**：使用具体的错误消息使验证脚本详细，例如"Field 'signature\_date' not found. Available fields: customer\_name, order\_total, signature\_date\_signed"以帮助 Claude 修复问题。

### 包依赖项

技能在具有平台特定限制的代码执行环境中运行：

* **claude.ai**：可以从 npm 和 PyPI 安装包并从 GitHub 仓库拉取
* **Anthropic API**：没有网络访问，没有运行时包安装

在你的 SKILL.md 中列出必需的包并验证它们在[代码执行工具文档](/en/docs/agents-and-tools/tool-use/code-execution-tool)中可用。

### 运行时环境

技能在具有文件系统访问、bash 命令和代码执行能力的代码执行环境中运行。有关此架构的概念性解释，请参阅概述中的[技能架构](/en/docs/agents-and-tools/agent-skills/overview#the-skills-architecture)。

**这如何影响你的编写**：

**Claude 如何访问技能**：

1. **元数据预加载**：在启动时，所有技能 YAML 前言中的名称和描述被加载到系统提示中
2. **文件按需读取**：Claude 在需要时使用 bash 读取工具从文件系统访问 SKILL.md 和其他文件
3. **脚本高效执行**：实用脚本可以通过 bash 执行而无需将其完整内容加载到上下文中。只有脚本的输出消耗 token
4. **大文件没有上下文惩罚**：参考文件、数据或文档在真正读取之前不消耗上下文 token

* **文件路径很重要**：Claude 像文件系统一样浏览你的技能目录。使用正斜杠（`reference/guide.md`），而不是反斜杠
* **描述性命名文件**：使用指示内容的名称：`form_validation_rules.md`，而不是 `doc2.md`
* **为发现而组织**：按领域或功能组织目录
  * 好：`reference/finance.md`、`reference/sales.md`
  * 差：`docs/file1.md`、`docs/file2.md`
* **捆绑全面的资源**：包括完整的 API 文档、大量示例、大型数据集；在访问之前没有上下文惩罚
* **确定性操作首选脚本**：编写 `validate_form.py` 而不是要求 Claude 生成验证代码
* **明确执行意图**：
  * "运行 `analyze_form.py` 提取字段"（执行）
  * "有关提取算法，请参阅 `analyze_form.py`"（作为参考阅读）
* **测试文件访问模式**：通过使用真实请求测试来验证 Claude 可以浏览你的目录结构

**示例**：

```
bigquery-skill/
├── SKILL.md（概述，指向参考文件）
└── reference/
    ├── finance.md（收入指标）
    ├── sales.md（管道数据）
    └── product.md（使用分析）
```

当用户询问收入时，Claude 读取 SKILL.md，看到对 `reference/finance.md` 的引用，并调用 bash 仅读取该文件。sales.md 和 product.md 文件保留在文件系统上，在需要之前消耗零上下文 token。这种基于文件系统的模型是启用渐进式披露的原因。Claude 可以浏览并选择性地加载每个任务所需的内容。

有关技术架构的完整详细信息，请参阅技能概述中的[技能如何工作](/en/docs/agents-and-tools/agent-skills/overview#how-skills-work)。

### MCP 工具引用

如果你的技能使用 MCP（模型上下文协议）工具，请始终使用完全限定的工具名称以避免"工具未找到"错误。

**格式**：`ServerName:tool_name`

**示例**：

```markdown  theme={null}
Use the BigQuery:bigquery_schema tool to retrieve table schemas.
Use the GitHub:create_issue tool to create issues.
```

其中：

* `BigQuery` 和 `GitHub` 是 MCP 服务器名称
* `bigquery_schema` 和 `create_issue` 是服务器内的工具名称

没有服务器前缀，Claude 可能无法找到工具，尤其是在有多个 MCP 服务器可用时。

### 避免假设工具已安装

不要假设包可用：

````markdown  theme={null}
**坏示例：假设安装**：
"Use the pdf library to process the file."

**好示例：明确依赖**：
"Install required package: `pip install pypdf`

Then use it:
```python
from pypdf import PdfReader
reader = PdfReader("file.pdf")
```"
````

## 技术说明

### YAML 前言要求

SKILL.md 前言需要 `name`（最多 64 个字符）和 `description`（最多 1024 个字符）字段。有关完整的结构详细信息，请参阅[技能概述](/en/docs/agents-and-tools/agent-skills/overview#skill-structure)。

### Token 预算

保持 SKILL.md 主体在 500 行以内以获得最佳性能。如果你内容超过此限制，请使用上面描述的渐进式披露模式将其拆分为单独的文件。有关架构详细信息，请参阅[技能概述](/en/docs/agents-and-tools/agent-skills/overview#how-skills-work)中的[技能如何工作](/en/docs/agents-and-tools/agent-skills/overview#how-skills-work)。

## 有效技能清单

在共享技能之前，验证：

### 核心质量

* [ ] 描述具体并包含关键术语
* [ ] 描述包括技能做什么和何时使用它
* [ ] SKILL.md 主体在 500 行以内
* [ ] 附加详细信息在单独的文件中（如果需要）
* [ ] 没有时间敏感信息（或在"旧模式"部分中）
* [ ] 全文使用一致的术语
* [ ] 示例是具体的，不是抽象的
* [ ] 文件引用是一层深
* [ ] 适当地使用渐进式披露
* [ ] 工作流有清晰的步骤

### 代码和脚本

* [ ] 脚本解决问题而不是推卸给 Claude
* [ ] 错误处理是明确和有帮助的
* [ ] 没有"巫毒常量"（所有值都有正当理由）
* [ ] 必需包在指令中列出并验证为可用
* [ ] 脚本有清晰的文档
* [ ] 没有 Windows 风格路径（全部正斜杠）
* [ ] 关键操作的验证/验证步骤
* [ ] 为质量关键任务包含反馈循环

### 测试

* [ ] 至少创建三个评估
* [ ] 用 Haiku、Sonnet 和 Opus 测试
* [ ] 用真实的使用场景测试
* [ ] 合并团队反馈（如果适用）

## 后续步骤

<CardGroup cols={2}>
  <Card title="开始使用 Agent Skills" icon="rocket" href="/en/docs/agents-and-tools/agent-skills/quickstart">
    创建你的第一个技能
  </Card>

  <Card title="在 Claude Code 中使用技能" icon="terminal" href="/en/docs/claude-code/skills">
    在 Claude Code 中创建和管理技能
  </Card>

  <Card title="通过 API 使用技能" icon="code" href="/en/api/skills-guide">
    以编程方式上传和使用技能
  </Card>
</CardGroup>