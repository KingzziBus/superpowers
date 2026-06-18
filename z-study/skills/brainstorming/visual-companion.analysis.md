# brainstorming/visual-companion.md 深度分析报告

> 本报告分析 superpowers 项目的"视觉伴侣"配套文档。它是 `brainstorming/SKILL.md` 中"视觉伴侣"部分的详细操作指南。

---

## 1. 基本信息

| 项 | 值 |
|----|----|
| **文件名** | `visual-companion.md` |
| **总行数** | 288 行 |
| **文档定位** | 视觉伴侣的详细操作指南 |
| **核心机制** | 本地 HTML 服务器 + 浏览器渲染 + JSON 事件流 |
| **平台支持** | Claude Code (Mac/Linux/Windows) + Codex + Gemini CLI |
| **核心创新** | "工具而非模式" + "per-question decision" |
| **关联技能** | `brainstorming`（使用场景）、`subagent-driven-development`（子代理模式） |

---

## 2. 目的与背景

### 2.1 文档的"工具配套"角色

`visual-companion.md` 在 superpowers 中扮演**"工具配套"**角色——它是 brainstorming 流程中"按需工具"的详细使用手册：

```
brainstorming 流程
    ↓
    "Offer Visual Companion"（提议使用）
    ↓
visual-companion.md ← 本文档（详细操作）
    ↓
HTML 服务器 + 浏览器
    ↓
视觉化头脑风暴
```

它把"提供视觉伴侣"这一句话的提议，**展开为 288 行的完整操作手册**。

### 2.2 三大核心命题

1. **"Use the browser when content IS visual, use the terminal when content is text"** —— 内容决定工具
2. **"Decide per-question, not per-session"** —— 细粒度决策
3. **"The server watches a directory for HTML files and serves the newest one"** —— 文件系统即状态

### 2.3 解决的具体问题

| 问题 | 本文档提供的方案 |
|------|------------------|
| 何时使用视觉伴侣 | 7 类视觉内容 + 5 类文本内容的明确区分 |
| 如何启动服务器 | 4 平台 + `--project-dir` 持久化 |
| 如何迭代视觉化 | 6 步循环（写 HTML / 提示用户 / 读事件 / 迭代） |
| 如何写内容 | 7 类 CSS 类（options、cards、mockup、split、pros-cons） |
| 如何清理 | `stop-server.sh` + 持久化策略 |

---

## 3. 核心技术决策

### 3.1 决策 1：内容决定工具（IS visual vs IS text）

**使用浏览器（IS visual）：**
- UI 模型图（线框图、布局、组件）
- 架构图（组件、数据流、关系）
- 并排视觉比较
- 设计润色
- 空间关系（状态机、流程图）

**使用终端（IS text）：**
- 需求和范围问题
- 概念性 A/B/C 选择
- 权衡列表
- 技术决策
- 澄清问题

**这一决策的精妙：**
- "A question *about* a UI topic is not automatically a visual question"
- "What does personality mean?" = 概念 → 终端
- "Which layout feels right?" = 视觉 → 浏览器

**这是"内容本身"而非"主题领域"决定工具**——很多 LLM 容易混淆这两者。

### 3.2 决策 2：按问题决策而非按会话决策

> "Decide per-question, not per-session."

**这一决策的洞察：**
- 用户接受伴侣 ≠ 每个问题都用
- 每次重新决定 = 避免 token 浪费
- 决策成本低（一个判断）

**与 superpowers 整体哲学一致：**
- "工具而非模式"
- "use 视情况" 而非 "use 总是"

### 3.3 决策 3：内容片段（Content Fragments）vs 完整文档

> "If your HTML file starts with `<!DOCTYPE` or `<html`, the server serves it as-is. Otherwise, the server automatically wraps your content in the frame template."

**两种模式的洞察：**
- **内容片段**（默认）：服务器自动包装（header、CSS、选择指示器）
- **完整文档**：完全控制页面

**默认片段的价值：**
- 降低 agent 的"页面构造"负担
- 统一主题和样式
- 减少 token 浪费

### 3.4 决策 4：文件系统即状态

> "The server watches a directory for HTML files and serves the newest one to the browser."

**这一架构的洞察：**
- 不需要数据库或复杂的会话管理
- 写入文件 = 推送屏幕
- 文件修改时间 = 顺序
- 删除/重命名 = 卸载

**这种"文件系统作为认知架构"与 superpowers 整体哲学一致：**
- writing-skills 强调"@-引用会强制加载"
- 这里强调"文件系统即状态"

### 3.5 决策 5：JSON 行（JSONL）作为事件流

```jsonl
{"type":"click","choice":"a","text":"Option A - Simple Layout","timestamp":1706000101}
{"type":"click","choice":"c","text":"Option C - Complex Grid","timestamp":1706000108}
{"type":"click","choice":"b","text":"Option B - Hybrid","timestamp":1706000115}
```

**这一设计的洞察：**
- 每次点击 = 一行 JSON
- 完整事件流显示**探索路径**（不是只最后一次）
- "The pattern of clicks can reveal hesitation or preferences worth asking about"

**关键洞察：探索路径是数据**——不只最后一次选择，整个过程都有信息。

### 3.6 决策 6：语义化文件名（Semantic Filenames）

> "Use semantic filenames: `platform.html`, `visual-style.html`, `layout.html`. Never reuse filenames — each screen gets a fresh file."

**这一决策的洞察：**
- 不用"screen1.html"、"output.html" 等通用名
- 不用"layout.html" 再用 "layout.html"（用 layout-v2.html）

**价值：**
- 文件名即文档
- 版本清晰
- 服务器按时间排序——新文件覆盖旧文件

### 3.7 决策 7：等待屏幕（Waiting Screen）

```html
<div style="display:flex;align-items:center;justify-content:center;min-height:60vh">
  <p class="subtitle">Continuing in terminal...</p>
</div>
```

**这一决策的洞察：**
- 当对话移到终端时，浏览器屏幕应该卸载
- 否则用户盯着"已解决的选择"看
- "waiting.html" 推送 = 卸载

**这是 UX 细节——但反映了"屏幕即对话状态"的设计哲学**。

### 3.8 决策 8：4 平台支持

| 平台 | 启动方式 |
|------|----------|
| Claude Code (macOS/Linux) | 后台模式（默认） |
| Claude Code (Windows) | run_in_background: true |
| Codex | 脚本自动检测 CODEX_CI |
| Gemini CLI | --foreground + is_background: true |

**这一决策的洞察：**
- 不同平台对后台进程的处理不同
- **脚本自动检测**平台约束
- 给出"4 个其他环境"的一般建议

**这是"跨平台即工程挑战"的具体体现。**

### 3.9 决策 9：项目级持久化 vs /tmp

> "Pass the project root as `--project-dir` so mockups persist in `.superpowers/brainstorm/` and survive server restarts. Without it, files go to `/tmp` and get cleaned up."

**两种模式的洞察：**
- **项目级**：文件持久化、跨重启、便于团队查看
- **/tmp**：临时、不持久、自动清理

**建议：项目级**——并提醒用户添加 `.superpowers/` 到 `.gitignore`。

**这是"默认安全 + 用户选择"的哲学**。

### 3.10 决策 10：30 分钟自动退出

> "The server auto-exits after 30 minutes of inactivity."

**这一决策的洞察：**
- 防止"僵尸服务器"
- 资源节约
- 用户应该用 `$STATE_DIR/server-info` 检查状态

**这种"自动清理 + 状态可见"是工程化设计**。

---

## 4. 文件结构剖析

```
visual-companion.md (288 行)
├── 标题 + 概述 (3 行)
├── 何时使用 (26 行)
│   ├── 使用浏览器 (7 行) - 5 类内容
│   └── 使用终端 (6 行) - 5 类内容
├── 它如何工作 (5 行) - 文件系统即状态
├── 启动会话 (52 行)
│   ├── 基础启动 (10 行)
│   ├── 查找连接信息 (3 行)
│   ├── 项目级 vs /tmp (3 行)
│   ├── 4 平台支持 (25 行)
│   └── URL 不可达的解决方案 (5 行)
├── 循环 (32 行) - 6 步流程
├── 编写内容片段 (29 行) - 最小示例
├── 可用的 CSS 类 (88 行)
│   ├── 选项 (18 行)
│   ├── 多选 (4 行)
│   ├── 卡片 (10 行)
│   ├── 模型图容器 (6 行)
│   ├── 分割视图 (4 行)
│   ├── 优/缺 (4 行)
│   ├── 模拟元素 (10 行)
│   └── 排版和章节 (6 行)
├── 浏览器事件格式 (13 行) - JSONL
├── 设计技巧 (9 行) - 7 原则
├── 文件命名 (6 行) - 3 规则
├── 清理 (5 行)
└── 参考 (3 行)
```

**结构特点：**
1. **何时使用占 9%**（26 行）—— 决策框架
2. **启动会话占 18%**（52 行）—— 多平台支持
3. **CSS 类占 31%**（88 行）—— 最大的技术参考
4. **循环占 11%**（32 行）—— 操作流程

---

## 5. 优点 / 亮点

### 5.1 "内容决定工具" 的清晰分类

7 类视觉 + 5 类文本的明确分类，让 agent 决策有依据。**"A question about a UI topic is not automatically a visual question"** 的反例说明特别有价值。

### 5.2 "per-question decision" 的细粒度

不是"开启就要一直用"——每个问题都重新决定。**避免沉没成本**。

### 5.3 内容片段 vs 完整文档的双模式

默认片段（低门槛）+ 完整文档（高控制）的双模式覆盖了不同需求。

### 5.4 文件系统即状态的简洁架构

不需要数据库——文件即状态。**与 LLM 的文件系统模型一致**。

### 5.5 JSONL 事件流的探索路径

不只是最后一次选择——**完整点击流是数据**。这反映了"过程 > 结果"的设计。

### 5.6 语义化文件名的"自文档化"

`platform.html`、`visual-style.html`、`layout.html` 比"screen1.html"更易理解。

### 5.7 等待屏幕的 UX 细节

当对话移到终端时推送"waiting.html"——**避免用户盯着陈旧内容**。

### 5.8 4 平台支持的工程化

不是"一招通用"——**平台特定性**被明确处理（macOS/Linux/Windows/Codex/Gemini）。

### 5.9 项目级持久化的"默认安全"

建议 `--project-dir`——文件持久化、跨重启。**用户不主动选择 = 项目级（更安全）**。

### 5.10 30 分钟自动退出的资源节约

防止僵尸服务器。**自动清理**是工程化设计。

### 5.11 88 行的 CSS 类参考

完整、可复用的 CSS 类清单（options、cards、mockup、split、pros-cons）。**这种"工具箱"是 LLM 友好的**——agent 可以直接用而不需"创造"。

### 5.12 7 条设计技巧的工程化

> "Scale fidelity to the question — wireframes for layout, polish for polish questions"

这种"按问题缩放保真度"是工程化思维。

### 5.13 真实内容的使用建议

> "Use real content when it matters — for a photography portfolio, use actual images (Unsplash). Placeholder content obscures design issues."

这种"避免占位符"的建议是 LLM 容易忽视的细节。

### 5.14 "2-4 options max" 的可证伪标准

不是"少一些"——是 "2-4 options" 的明确数量。**比"适度"更有用**。

---

## 6. 缺点 / 风险 / 局限

### 6.1 内容分类的"模糊地带"风险

> "Spatial relationships — state machines, flowcharts, entity relationships rendered as diagrams"

**问题：**
- "状态机"既可以用文字描述也可以用图表
- 哪些情况下"图表优于文字"？
- 边界模糊

### 6.2 "per-question decision" 的认知负担

**风险：**
- 每个问题都重新判断 = 持续认知负担
- agent 可能"懒得判断"而默认使用
- 与"减少决策疲劳"的总体目标冲突

### 6.3 CSS 类的"完整性"问题

**问题：**
- 88 行的 CSS 类是否覆盖所有需求？
- 如果需要复杂动画/交互怎么办？
- 缺少"自定义 CSS"的扩展机制

### 6.4 文件系统即状态的"可观测性"问题

**风险：**
- 文件很多时（多次迭代后）难以追踪
- 没有"会话视图"
- 依赖文件名和时间排序——脆弱

### 6.5 JSONL 事件流的"可读性"问题

**问题：**
- 大量点击 = 大量 JSON 行
- 没有"摘要视图"
- 文档没说"如何处理 100+ 行的 events"

### 6.6 4 平台支持的"维护负担"

**问题：**
- 4 平台 = 4 个测试场景
- 平台差异（macOS vs Windows vs Codex）可能演化
- 文档可能很快过时

### 6.7 "30 分钟自动退出" 的边界

**问题：**
- 30 分钟是硬编码
- 复杂设计可能需要 1 小时思考
- 没有"延长超时"的标准

### 6.8 URL 不可达的"复杂性"

```bash
scripts/start-server.sh \
  --project-dir /path/to/project \
  --host 0.0.0.0 \
  --url-host localhost
```

**问题：**
- 容器化/远程环境的具体配置没说
- 网络/防火墙问题没覆盖
- 可能存在"配置地狱"

### 6.9 "真实内容"建议的版权风险

> "for a photography portfolio, use actual images (Unsplash)"

**问题：**
- Unsplash 图片的许可？
- 实际客户的项目用什么图？
- 文档没说"占位符"和"真实内容"的法律边界

### 6.10 "工具而非模式" 的潜在滥用

**风险：**
- agent 可能"完全不用"伴侣（节省 token）
- 错过了"视觉化"的价值
- "per-question decision" 演变为"几乎不用"

### 6.11 跨会话的"上下文丢失"

**问题：**
- server 退出 = 事件流丢失？
- 没有"重连机制"
- 没有"会话归档"

### 6.12 "stop-server.sh" 的清理不完整

**问题：**
- 只清理了 server 进程
- 没有清理临时文件
- 没有清理浏览器状态

### 6.13 文档本身的"操作性"挑战

**问题：**
- 288 行 = 信息密度高
- 缺少"快速开始"指南
- 缺少"故障排除"指南

### 6.14 缺少"安全"考虑

**问题：**
- 本地服务器 = 安全风险？
- `--host 0.0.0.0` 暴露给网络？
- HTML 内容是否有 XSS 风险？

---

## 7. 关键洞察

### 7.1 洞察 1："内容决定工具"是认知清晰

不是"主题领域"（UI vs Backend）而是"内容本质"（visual vs text）。**这种精确分类**让 agent 决策有依据。

### 7.2 洞察 2："per-question decision" 是反沉没成本

不是"开启就要一直用"——每次重新决定。**避免决策疲劳的"承诺升级"**。

### 7.3 洞察 3：内容片段默认降低门槛

agent 不需要"构造完整页面"——只写内容。**降低创作成本**。

### 7.4 洞察 4：文件系统即状态的简洁

不需要数据库——文件即状态。**符合 LLM 的文件系统模型**。

### 7.5 洞察 5：探索路径是数据

不只是最后一次选择——完整点击流是数据。**过程 > 结果**。

### 7.6 洞察 6：等待屏幕是 UX 哲学

当对话移到终端时，浏览器屏幕应该卸载。**避免"陈旧内容"**。

### 7.7 洞察 7：平台特定性的诚实承认

4 平台 = 4 个配置。**不假装通用**——这与 anthropic-best-practices 的"平台特定性"一致。

### 7.8 洞察 8：项目级持久化的"默认安全"

不主动指定 = 项目级（更安全）。**用户偏好 = 更保守的选择**。

### 7.9 洞察 9：30 分钟自动退出是资源节约

防止僵尸服务器。**自动清理**是工程化设计。

### 7.10 洞察 10：CSS 工具箱的"创作民主化"

agent 不需要"创造 CSS"——只用预定义的类。**降低 CSS 知识门槛**。

### 7.11 洞察 11："真实内容"建议的反占位符

用真实图片而非占位符——**避免设计问题被掩盖**。

### 7.12 洞察 12："2-4 options max" 的可证伪标准

不是"少一些"——是 "2-4" 的明确数量。**比"适度"更有用**。

### 7.13 洞察 13：跨平台的工程化

不是"一招通用"——**承认平台差异、给具体方案**。

### 7.14 洞察 14：工具的"使用"是按需而非默认

与 superpowers 的"工具而非模式"哲学一致——**避免 token 浪费**。

---

## 8. 关联文档

### 8.1 内部关联（同目录）

| 文件 | 关系 |
|------|------|
| `SKILL.md` | **使用入口**——SKILL.md 在视觉伴侣部分引用本文档 |
| `spec-document-reviewer-prompt.md` | 独立工具——本文档不涉及规范审查 |

### 8.2 外部关联（其他技能）

| 技能 | 关系 |
|------|------|
| `brainstorming` | **使用场景**——本文档是 brainstorming 的"工具配套" |
| `subagent-driven-development` | 间接相关——子代理可派遣做视觉化 |
| `using-superpowers` | 元技能——本文档是其工具之一 |

### 8.3 上游文档

| 来源 | 关系 |
|------|------|
| 软件工程传统 | "文件即状态" 的简洁架构 |
| Web 设计原则 | CSS 类的语义化（options、cards） |
| 跨平台工程 | 4 平台支持的实现 |

### 8.4 配套脚本

| 脚本 | 关系 |
|------|------|
| `scripts/frame-template.html` | CSS 框架 |
| `scripts/helper.js` | 客户端 JavaScript |
| `scripts/server.cjs` | Node.js 服务器 |
| `scripts/start-server.sh` | 启动脚本（4 平台） |
| `scripts/stop-server.sh` | 清理脚本 |

---

## 9. 状态评价

### 9.1 完成度：★★★★★（5/5）

- 何时使用（7+5 分类）
- 启动会话（4 平台）
- 6 步循环
- 7 类 CSS 类
- 事件流解析
- 清理

### 9.2 一致性：★★★★★（5/5）

- 与 brainstorming "工具而非模式" 哲学一致
- 与 superpowers 跨平台工程化一致
- 与文件系统状态架构一致

### 9.3 可操作性：★★★★（4/5）

- 88 行 CSS 类是完整工具箱
- 4 平台启动指南
- **扣分点**：缺少"快速开始"、故障排除

### 9.4 创新度：★★★★★（5/5）

- "内容决定工具" 的认知清晰
- "per-question decision" 的细粒度
- 文件系统即状态的简洁
- 探索路径是数据的哲学

### 9.5 整体评价：**9.4 / 10**

这是一份**理论清晰、操作完整、跨平台、UX 精细**的工具配套文档。它把"提供视觉伴侣"一句话的提议，展开为 288 行的完整操作手册。

**唯一扣分点：** 安全考虑、故障排除指南缺失。

---

## 10. 给读者的启示

### 10.1 对技能作者

1. **"工具而非模式"** 是资源节约的智慧
2. **"per-question decision"** 避免沉没成本
3. **内容决定工具** 是认知清晰
4. **文件系统即状态** 是简洁架构
5. **跨平台工程** 是不假装通用

### 10.2 对 AI 工程师

1. **内容分类（visual vs text）** 是决策框架的范例
2. **JSONL 事件流** 是过程数据的存储格式
3. **88 行 CSS 类** 是 LLM 友好的工具箱
4. **等待屏幕** 是 UX 哲学
5. **30 分钟自动退出** 是资源节约

### 10.3 对 superpowers 用户

1. **本文档** 是 brainstorming 视觉伴侣的完整指南
2. **7+5 分类** 决定何时使用
3. **4 平台支持** 覆盖主要环境
4. **CSS 类** 是即用即得的工具
5. **JSONL 事件流** 包含完整探索路径

### 10.4 对更广泛的提示工程

1. **"per-question decision"** 是 LLM 工具使用的细粒度哲学
2. **"内容决定工具"** 是工具选择的清晰标准
3. **文件系统即状态** 是 LLM 友好的状态管理
4. **跨平台工程** 是不假装通用的工程态度
5. **工具箱（CSS 类）** 是 LLM 创作民主化

---

## 11. 后续工作 / 改进方向

### 11.1 应补充的内容

1. **快速开始指南**——5 分钟上手版
2. **故障排除指南**——常见问题和解决方案
3. **安全考虑**——本地服务器、暴露网络的风险
4. **会话归档**——跨会话的上下文保存
5. **扩展机制**——自定义 CSS / 动画

### 11.2 应展开的讨论

1. **"工具而非模式" 的边界**——什么时候**应该**默认使用
2. **内容分类的灰色地带**——状态机/流程图的"本质"
3. **30 分钟超时的灵活性**——复杂项目的处理
4. **跨会话的体验**——重新进入会话时如何"恢复"
5. **多模态扩展**——视频、音频的支持

### 11.3 元层面

本文档是 superpowers "工具化"哲学的具体体现。建议：
- 收集"何时使用伴侣" 的实际数据
- 跟踪 token 节省 vs 价值的对比
- 定期更新 CSS 类清单

### 11.4 哲学层面

最大的开放问题是：**"工具按需使用" vs "工具默认使用"的平衡**？

当前立场：按需使用（per-question decision）。
**问题：**
- 某些场景下，视觉化应该默认（如 UI 项目）
- "per-question decision" 演变为"几乎不用"的风险
- 工具使用与 token 经济的平衡

**未来可能：**
- 按项目类型调整（UI 项目默认开启，API 项目默认关闭）
- 学习 agent 的"使用模式"——智能推荐
- 工具的"使用率"指标化

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

所有论断都基于 visual-companion.md 文本和其在 superpowers 生态中的角色。**不做超出文档声明的推论**。
