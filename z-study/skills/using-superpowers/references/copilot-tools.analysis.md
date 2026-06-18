# copilot-tools.md 分析报告

## 基本信息

| 项目 | 内容 |
| --- | --- |
| 文档路径 | `skills/using-superpowers/references/copilot-tools.md` |
| 字数规模 | 约 43 行 / 1.8KB（**最小**的 references） |
| 文档类型 | 平台适配参考 / 工具映射表 |
| 主题 | Claude Code 工具名 → Copilot CLI 等价物的映射 |
| 核心定位 | Superpowers 在 Copilot CLI 上的"运行时翻译层" |
| 特殊内容 | 异步 shell 会话、SQLite-backed todos |
| 面向读者 | 在 Copilot CLI 上运行 Superpowers 技能的 AI 代理 |

---

## 一、文档目的

解决 Superpowers 在 GitHub Copilot CLI 上的**跨平台运行**问题。

具体目标：
1. **建立工具名映射**：14 个 Claude Code 工具 → Copilot CLI 等价物
2. **暴露独有特性**：异步 shell 会话（Copilot CLI 特有）
3. **暴露其他工具**：store_memory、report_intent、GitHub MCP 等
4. **标记无等价物**：WebSearch、EnterPlanMode（明确缺失）

定位上属于 **"Copilot CLI 平台运行时翻译层"**——是 `using-superpowers` 母技能的多平台扩展。

---

## 二、核心技术决策

### 决策 1：14 个工具映射

| Claude Code | Copilot CLI |
|-------------|-------------|
| Read | view |
| Write | create |
| Edit | edit |
| Bash | bash |
| Grep | grep |
| Glob | glob |
| Skill | skill |
| WebFetch | web_fetch |
| Task | task with agent_type |
| TodoWrite | sql with todos table |
| WebSearch | **无等价物** |
| EnterPlanMode/ExitPlanMode | **无等价物** |

**设计意图**：
- 大部分工具都有**直接对应**（Read/Write/Edit/Bash/Grep/Glob）
- 一些工具有**不同实现**（Task 用 `agent_type` 参数，TodoWrite 用 SQL）
- 一些工具**完全缺失**（WebSearch、EnterPlanMode）——**显式标记**

**关键洞察**：**显式标记"无等价物"**比"假装有"更诚实。

### 决策 2：异步 shell 会话是 Copilot CLI 独有

```text
bash with async: true   → 启动后台命令
write_bash              → 向异步会话发送输入
read_bash               → 读取输出
stop_bash               → 终止
list_bash               → 列出活跃会话
```

**精妙之处**：
- **没有直接 Claude Code 等价物**——这是 Copilot CLI 独有特性
- 5 个工具形成**完整异步会话生命周期**
- 文档**主动暴露**这个特性——让技能作者能利用

**这是"平台能力扩展"**——不只做"等价映射"，还**暴露平台特有能力**。

### 决策 3：TodoWrite 用 SQLite-backed `sql` 工具

```text
TodoWrite（任务跟踪）→ sql 带内建 todos 表
```

**精妙之处**：
- Copilot CLI 把 todos 存储在 **SQLite 数据库**
- 用 `sql` 工具查询和修改
- 比 Claude Code 的临时 `TodoWrite` 更**持久化**

**这是"持久化 vs 临时"的设计差异**。

### 决策 4：Task 工具用 `agent_type` 参数

```text
Task 工具（分派子代理）→ task 带 agent_type: "general-purpose" 或 "explore"
```

**设计意图**：
- 显式声明子代理**类型**（general-purpose / explore）
- 比 Claude Code 的"未指定类型"更结构化
- **可扩展**（新 agent_type 加新值）

### 决策 5：GitHub MCP 工具的原生集成

```text
GitHub MCP 工具（github-mcp-server-*）→ 原生 GitHub API 访问
```

**精妙之处**：
- Copilot CLI 是 **GitHub 产品**——原生 GitHub 集成是**理所当然**
- 通过 MCP（Model Context Protocol）暴露
- 比 Claude Code 的"间接集成"（需要 WebFetch）更**直接**

**这是"平台垂直整合的优势"**。

### 决策 6：WebSearch 无等价物的处理

```text
WebSearch → 无等价物 —— 使用 web_fetch 加搜索引擎 URL
```

**关键洞察**：
- 不假装有等价物
- 给**降级方案**（web_fetch + 搜索 URL）
- 用户可选择

**这是"诚实标记 + 提供降级"原则**。

### 决策 7：EnterPlanMode/ExitPlanMode 无等价物的处理

```text
EnterPlanMode / ExitPlanMode → 无等价物 —— 保持在主会话中
```

**精妙之处**：
- 标记为"无等价物"
- 给**默认行为**（保持在主会话）
- 技能仍然能用，但工作流不切换

**这是"无降级，保持现状"原则**。

---

## 三、文件结构

| 章节 | 内容 | 工程价值 |
| --- | --- | --- |
| 工具映射表 | 14 个工具 | 核心 |
| 异步 shell 会话 | 5 个工具 | 平台特有 |
| 其他 Copilot CLI 工具 | 5 个工具 | 平台特有 |

**结构特点**：**1 个核心表 + 2 个平台特有表**——简洁、聚焦。

---

## 四、优点

### 1. 工具映射表最完整

14 个工具映射——比 codex-tools.md（9 个）更全。

### 2. 显式标记"无等价物"

`WebSearch`、`EnterPlanMode` 明确标记为"无"——比假装有更诚实。

### 3. 主动暴露平台特有能力

异步 shell 会话、GitHub MCP 集成——不只做"等价映射"。

### 4. SQLite-backed todos 解释清楚

`sql` 带内建 `todos` 表——比 Claude Code 的临时存储更**持久化**。

### 5. Task 用 `agent_type` 参数

比 Claude Code 的"未指定类型"更**结构化**。

---

## 五、缺点与风险

### 1. 没有"功能差异"说明

例如 `view` 和 `Read` 真的**完全等价**吗？有没有"哪些功能 view 不支持"？

**缺失**：功能差异对比。

### 2. 异步 shell 的"使用场景"没说

5 个异步工具——但**什么时候应该用**？

**缺失**：使用场景指南。

### 3. SQL todos 的"操作语法"没给

`sql with todos table`——具体怎么查？怎么改？

**缺失**：SQL 语法示例。

### 4. GitHub MCP 工具列得不全

只说"原生 GitHub API 访问"——但**支持哪些具体操作**？

**缺失**：完整的 GitHub MCP 工具列表。

### 5. 缺"性能特性"说明

Copilot CLI 的 `bash` 工具和 Claude Code 的 Bash 工具——**性能一样**吗？

**缺失**：性能基准。

### 6. 缺"使用限制"说明

异步 shell 有什么限制？SQL 有大小限制吗？

**缺失**：使用限制。

---

## 六、关键洞察

### 洞察 1：这是 Superpowers 跨平台战略在 Copilot CLI 上的实现

**核心模式**：
- 技能用 **Claude Code 工具名**为标准
- 每个平台提供**映射文档**
- 翻译在**运行**时进行

**Copilot CLI 特殊性**：
- 是 GitHub 的产品——**GitHub 集成最深**
- 有**异步 shell 会话**——Claude Code 没有
- 用 **SQLite 持久化** todos——比 Claude Code 临时

### 洞察 2："无等价物"的诚实标记

**WebSearch、EnterPlanMode** 明确标记为"无"。

**为什么不假装**：
- 假装有 → 技能作者误用 → 失败
- 显式标记 → 技能作者用降级方案 → 可用

**这是"工程诚实"原则**。

### 洞察 3：异步 shell 是 Copilot CLI 的"独有优势"

```text
bash with async: true
```

**关键洞察**：
- 5 个工具形成**完整异步会话生命周期**
- 让技能作者能写"长跑命令"——而不阻塞主会话
- **平台能力扩展**——不只做等价映射

**启示**：跨平台适配不是"做减法"（只保留等价功能），**也可以"做加法"**（暴露平台特有能力）。

### 洞察 4：SQLite-backed todos 是"持久化优势"

**关键洞察**：
- Claude Code 的 `TodoWrite` 是**临时**的（会话结束就丢）
- Copilot CLI 的 `todos` 表是**持久化**的（跨会话保留）
- 对**长任务**特别有用

**这是"平台能力的差异化"**——不是劣势，是**优势**。

### 洞察 5：GitHub MCP 工具是"垂直整合优势"

**关键洞察**：
- Copilot CLI = GitHub 产品
- GitHub 集成 = **最深**
- 通过 MCP 暴露 → **标准化**

**这是"产品垂直整合的优势"**——Copilot CLI 在 GitHub 场景下比 Claude Code 强。

### 洞察 6：与 codex-tools 的对比

| 维度 | codex-tools | copilot-tools |
|------|-------------|---------------|
| 工具数 | 9 | 14 |
| Legacy 说明 | 有 | 无 |
| 沙箱降级 | 有 | 无 |
| 平台特有 | 多代理配置 | 异步 shell、GitHub MCP |
| 无等价物标记 | 0 | 2 |

**启示**：不同平台的"特殊点"不同——**没有统一模板**。

---

## 七、关联文档

| 文档 | 关系 |
| --- | --- |
| `skills/using-superpowers/SKILL.md` | 母技能 |
| `skills/using-superpowers/references/codex-tools.md` | 平行：另一个平台映射 |
| `skills/using-superpowers/references/gemini-tools.md` | 平行：另一个平台映射 |
| GitHub Copilot CLI 官方文档 | 上游 |
| MCP（Model Context Protocol）规范 | 上游 |

---

## 八、状态评价

| 维度 | 评分 | 说明 |
| --- | --- | --- |
| **翻译完整性** | ⭐⭐⭐⭐ (4/5) | 14 个工具，比 codex 全 |
| **平台特有暴露** | ⭐⭐⭐⭐⭐ (5/5) | 主动暴露异步 shell、GitHub MCP |
| **诚实性** | ⭐⭐⭐⭐⭐ (5/5) | 显式标记"无等价物" |
| **可操作性** | ⭐⭐⭐ (3/5) | 缺 SQL 语法、异步 shell 使用场景 |
| **降级方案** | ⭐⭐⭐ (3/5) | 只对 WebSearch 给降级 |
| **功能差异** | ⭐⭐ (2/5) | 没说工具语义是否真的等价 |

**总体评价**：**Copilot CLI 平台映射的实用参考**。14 个工具映射 + 平台特有能力暴露 + 显式无等价物标记构成了对 GitHub 编程工具的完整支持。**异步 shell、SQLite todos、GitHub MCP** 是亮点。**功能差异、SQL 语法、使用场景** 是不足。

---

## 九、给读者的启示

### 如果你在 Copilot CLI 上用 Superpowers
- **遇到 `Read`/`Write`/`Bash`**：用 `view`/`create`/`bash`
- **遇到 `TodoWrite`**：用 `sql` with todos table（持久化！）
- **遇到 `Task`**：用 `task` with `agent_type`
- **遇到 `WebSearch`/`EnterPlanMode`**：用降级方案或保持现状
- **异步长跑命令**：用 `bash` with `async: true`
- **GitHub 操作**：用 `github-mcp-server-*` 工具

### 如果你写新技能
- **始终用 Claude Code 工具名**——不要写 Copilot CLI 特定工具
- 利用**平台特有能力**——异步 shell、SQLite todos

### 如果你扩展到 Copilot for VS Code 等
- **模仿本文件结构**：工具映射 + 平台特有 + 显式无等价物

### 如果你维护 Superpowers
- **改进建议 1**：补充 SQL todos 的具体语法
- **改进建议 2**：补充异步 shell 的使用场景
- **改进建议 3**：补充 GitHub MCP 工具的完整列表
- **改进建议 4**：增加工具功能差异对比
- **改进建议 5**：增加 Copilot CLI 的性能特性说明

---

## 十、后续工作

按重要性排序：

1. **中优先级**：补充 SQL todos 的具体语法示例
2. **中优先级**：补充异步 shell 的使用场景
3. **中优先级**：增加工具功能差异对比
4. **低优先级**：增加 Copilot CLI 的性能基准
5. **低优先级**：增加使用限制说明

---

**总结**：这个 43 行的文件是 **"Copilot CLI 平台映射的精炼参考"**。14 个工具映射 + 异步 shell 暴露 + SQLite todos 暴露 + GitHub MCP 集成 + 显式无等价物标记构成完整支持。

最显著的优点是**"无等价物"的诚实标记、平台特有能力的主动暴露**；最显著的不足是**SQL 语法、使用场景、功能差异** 3 个方面可以提升。

对在 Copilot CLI 上使用 Superpowers 的任何人来说，**这是必读必用的参考**。其"做加法"（暴露平台特有）的设计哲学值得所有跨平台工具学习。
