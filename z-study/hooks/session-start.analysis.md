# session-start 深度分析报告

> 文件: `hooks/session-start`（58 行，~2.8 KB）
> 类型: SessionStart Hook 核心实现（bash 脚本，无扩展名）
> 角色: 把 using-superpowers 内容注入到 Claude/Cursor/Copilot 上下文的"心脏"

---

## 1. 基本信息

| 字段 | 值 |
|------|-----|
| 文件路径 | `hooks/session-start` |
| 文件大小 | 约 2.8 KB（58 行） |
| 文件类型 | Bash 脚本（shebang `#!/usr/bin/env bash`） |
| 平台目标 | 跨平台（Mac/Linux 原生 bash + Windows Git Bash via run-hook.cmd） |
| 调用方 | `hooks/run-hook.cmd session-start`（间接被 hooks.json 调用） |
| 读取的文件 | `${PLUGIN_ROOT}/skills/using-superpowers/SKILL.md` |
| 检测的旧路径 | `~/.config/superpowers/skills`（遗留技能目录） |
| 输出格式 | 3 种 JSON（Cursor / Claude Code / Copilot CLI） |
| 关键函数 | `escape_for_json()`（行 23-31） |
| 关键技巧 | `${s//old/new}` 一次性替换（性能优化） |
| Bug 规避 | `printf` 替代 heredoc（bash 5.3+ hang issue） |

---

## 2. 目的与背景

`session-start` 是 superpowers 插件的 **SessionStart Hook 核心实现**。它的工作是：

1. 读取 `skills/using-superpowers/SKILL.md` 的完整内容
2. 包装成"超重要"系统提示格式
3. 检测是否有遗留技能目录，发出警告
4. 按平台输出对应的 JSON 格式

**为什么这个脚本至关重要**：

- superpowers 库的核心理念是"教会 AI 如何系统地使用技能"
- 但 AI 模型**不会主动记得**它有这些技能
- 这个脚本**强制**在每个会话开始时把"你有 superpowers"+"如何使用"喂给模型
- 换句话说，**没有这个脚本，superpowers 就不工作**

**这 58 行代码是整个插件的"心脏"——所有技能都依赖它被 Claude/Cursor/Copilot 知道**。

---

## 3. 核心技术决策

### 3.1 为什么检测 `~/.config/superpowers/skills` 目录？

来自行 12-15 的代码：

```bash
legacy_skills_dir="${HOME}/.config/superpowers/skills"
if [ -d "$legacy_skills_dir" ]; then
    warning_message="\n\n<important-reminder>...⚠️ WARNING: Superpowers now uses Claude Code's skills system. Custom skills in ~/.config/superpowers/skills will not be read. Move custom skills to ~/.claude/skills instead. ..."
fi
```

**背景**：

- superpowers 早期版本（v3 之前）使用 `~/.config/superpowers/skills` 作为自定义技能目录
- Claude Code 引入了官方 skills 系统后，自定义技能应该放到 `~/.claude/skills`
- 但**用户的旧目录可能还在**——他们可能还依赖那个目录

**优雅迁移策略**：

- 不强制删除（避免破坏用户已有技能）
- 每次会话开始时**提醒**用户迁移
- 用 `<important-reminder>` 标签让 Claude 必须在首条回复中告知用户

**这体现了一种"用户友好"的工程哲学**——不强迁，但**每次都提醒**直到用户处理。

### 3.2 性能黑魔法：`${s//old/new}` 一次性替换

来自行 23-31：

```bash
escape_for_json() {
    local s="$1"
    s="${s//\\/\\\\}"     # \ → \\
    s="${s//\"/\\\"}"     # " → \"
    s="${s//$'\n'/\\n}"   # newline → \n
    s="${s//$'\r'/\\r}"   # CR → \r
    s="${s//$'\t'/\\t}"   # tab → \t
    printf '%s' "$s"
}
```

**为什么是 5 行而不是循环？**

bash 的 `${var//pattern/replacement}` 是**单遍 C 级替换**——一次扫描整个字符串完成所有替换。注释说（行 20-22）：

> Each `${s//old/new}` is a single C-level pass - orders of magnitude faster than the character-by-character loop this replaces.

**性能对比**：

- **循环方式**（字符级）：O(n*m)，其中 n=字符串长度，m=替换次数
- **`${s//old/new}` 方式**：O(n)，但常数因子小得多

**实际影响**：

- `using-superpowers/SKILL.md` 内容约 200 行
- 循环方式可能耗时几十毫秒
- `${s//old/new}` 方式几乎瞬时（< 1ms）

**这是 bash 字符串处理的关键性能技巧**——99% 的 bash 脚本都写错。

### 3.3 为什么 3 种不同的 JSON 输出格式？

来自行 39-55：

```bash
if [ -n "${CURSOR_PLUGIN_ROOT:-}" ]; then
  # Cursor sets CURSOR_PLUGIN_ROOT (may also set CLAUDE_PLUGIN_ROOT)
  printf '{\n  "additional_context": "%s"\n}\n' "$session_context"
elif [ -n "${CLAUDE_PLUGIN_ROOT:-}" ] && [ -z "${COPILOT_CLI:-}" ]; then
  # Claude Code sets CLAUDE_PLUGIN_ROOT without COPILOT_CLI
  printf '{\n  "hookSpecificOutput": {\n    "hookEventName": "SessionStart",\n    "additionalContext": "%s"\n  }\n}\n' "$session_context"
else
  # Copilot CLI (sets COPILOT_CLI=1) or unknown platform — SDK standard format
  printf '{\n  "additionalContext": "%s"\n}\n' "$session_context"
fi
```

**3 种格式对比**：

| 平台 | 环境变量 | 字段路径 | 字段名风格 |
|------|---------|---------|-----------|
| **Cursor** | `CURSOR_PLUGIN_ROOT` | `additional_context` | snake_case（**Cursor 自己的私有约定**） |
| **Claude Code** | `CLAUDE_PLUGIN_ROOT` (且无 `COPILOT_CLI`) | `hookSpecificOutput.additionalContext` | camelCase + 嵌套（**Claude Code 私有约定**） |
| **Copilot CLI** | `COPILOT_CLI=1`（任意根变量） | `additionalContext` | camelCase 顶层（**SDK 标准**） |

**注释（行 41-42）解释了一个关键点**：

> Claude Code reads BOTH additional_context and hookSpecificOutput without deduplication, so we must emit only the field the current platform consumes.

**翻译**：Claude Code 实际上**会同时读取** `additional_context` 和 `hookSpecificOutput.additionalContext`，**不去重**。所以如果同时输出两个，注入的内容**会重复 2 倍**。

**这意味着什么**：

- 平台特化不是"美观"问题，是**功能正确性**问题
- Cursor 必须用 `additional_context`，Claude Code 必须用 `hookSpecificOutput.additionalContext`，**不能错**否则注入重复

### 3.4 为什么用 `printf` 而非 heredoc？

来自行 44-45 的注释：

> Uses printf instead of heredoc to work around bash 5.3+ heredoc hang.
> See: https://github.com/obra/superpowers/issues/571

**背景**：

- bash 5.3+（2024 年发布）在某种场景下 heredoc 会**永久挂起**
- 这个问题在 GitHub issue #571 中有详细报告
- `printf` 是同步输出，没有 heredoc 的潜在 hang 问题

**这是一个真实的 bug 规避**——superpowers 团队在生产中遇到 hang 问题后主动换用 printf。

### 3.5 为什么用 `${VAR:-}` 而非直接 `$VAR`？

行 46, 49 都用了 `${CURSOR_PLUGIN_ROOT:-}` 这种语法：

```bash
if [ -n "${CURSOR_PLUGIN_ROOT:-}" ]; then
```

**含义**：如果 `CURSOR_PLUGIN_ROOT` 未设置，**用空字符串替代**而不是报错。

**为什么必要**：

- `set -euo pipefail` 在行 4 设置了（严格模式）
- 直接 `$CURSOR_PLUGIN_ROOT` 在未设置时会**触发 unbound variable 错误**并退出
- `${VAR:-}` 是 bash 的"安全解引用"惯用法

### 3.6 错误降级：技能文件不存在时如何处理？

行 18：

```bash
using_superpowers_content=$(cat "${PLUGIN_ROOT}/skills/using-superpowers/SKILL.md" 2>&1 || echo "Error reading using-superpowers skill")
```

**机制**：

- `cat ... 2>&1` 把 stderr 重定向到 stdout
- `|| echo "..."` 在 cat 失败时输出错误信息而不是空字符串

**效果**：

- 如果 `SKILL.md` 丢失，Claude 会看到 `"Error reading using-superpowers skill"`
- 不会因为文件不存在而**完全崩溃**（`set -e` 不会触发，因为 `||` 提供了 fallback）
- 用户会看到"插件坏了"而不是"hook 失败"的错误

**这是一个"软失败"模式**——比硬崩溃友好，但比"静默成功"诚实。

### 3.7 输出格式的"双 <EXTREMELY_IMPORTANT>" 包裹

行 35：

```bash
session_context="<EXTREMELY_IMPORTANT>\nYou have superpowers.\n\n**Below is the full content of your 'superpowers:using-superpowers' skill - your introduction to using skills. For all other skills, use the 'Skill' tool:**\n\n${using_superpowers_escaped}\n\n${warning_escaped}\n</EXTREMELY_IMPORTANT>"
```

**设计意图**：

- `<EXTREMELY_IMPORTANT>` 标签是**针对模型**的——告诉模型"这是非常重要，必须遵循"
- 标签外的"You have superpowers"是**人类可读**的提示（让用户看到日志时明白发生了什么）
- `**Below is the full content of your 'superpowers:using-superpowers' skill - your introduction to using skills. For all other skills, use the 'Skill' tool:**` 是**指令**，明确告诉模型：
  - 这是 using-superpowers 技能的**完整内容**（无需再调用 Skill 工具读一次）
  - 其他技能需要用 `Skill` 工具调用

**这是元指令（meta-instruction）**——教导模型如何**对待**这段内容。

---

## 4. 文件结构剖析

```
┌─ 头部 (行 1-9) ─────────────────────────────────────────┐
│ shebang + set -euo pipefail                             │
│ 解析 SCRIPT_DIR 和 PLUGIN_ROOT                          │
└──────────────────────────────────────────────────────────┘

┌─ 警告检测 (行 11-15) ────────────────────────────────────┐
│ 检测 ~/.config/superpowers/skills 目录                  │
│ 构建 warning_message                                    │
└──────────────────────────────────────────────────────────┘

┌─ 技能内容读取 (行 17-18) ────────────────────────────────┐
│ cat SKILL.md 2>&1 || echo "Error..."                    │
└──────────────────────────────────────────────────────────┘

┌─ JSON 转义函数 (行 20-31) ───────────────────────────────┐
│ 5 个 ${s//old/new} 一次性替换                            │
│ printf 输出 (而非 echo, 避免 -e 行为差异)                │
└──────────────────────────────────────────────────────────┘

┌─ 上下文构造 (行 33-35) ─────────────────────────────────┐
│ escape_for_json 两次调用                                │
│ 构造 <EXTREMELY_IMPORTANT>...</EXTREMELY_IMPORTANT>     │
└──────────────────────────────────────────────────────────┘

┌─ 平台分发输出 (行 37-55) ───────────────────────────────┐
│ if CURSOR_PLUGIN_ROOT → additional_context               │
│ elif CLAUDE_PLUGIN_ROOT && !COPILOT_CLI → hookSpecificOutput │
│ else → additionalContext (SDK standard)                  │
└──────────────────────────────────────────────────────────┘

┌─ 退出 (行 57) ─────────────────────────────────────────┐
│ exit 0                                                  │
└──────────────────────────────────────────────────────────┘
```

**完整数据流**：

```
PLUGIN_ROOT/skills/using-superpowers/SKILL.md
  ↓
cat → using_superpowers_content (200+ 行)
  ↓
escape_for_json → 转义后内容
  ↓
session_context = <EXTREMELY_IMPORTANT>...</EXTREMELY_IMPORTANT>
  ↓
平台检测 (CURSOR_PLUGIN_ROOT / CLAUDE_PLUGIN_ROOT / COPILOT_CLI)
  ↓
printf 输出 JSON → stdout
  ↓
Claude Code / Cursor / Copilot CLI 解析 → 注入到上下文
```

---

## 5. 优点亮点

- **极简而完整**: 58 行实现完整的跨平台 SessionStart 逻辑
- **性能优化**: `${s//old/new}` 一次性替换，比循环快"几个数量级"
- **3 平台支持**: Cursor + Claude Code + Copilot CLI 自动检测
- **优雅降级**: 多种错误场景下不崩溃
- **元指令设计**: `<EXTREMELY_IMPORTANT>` 标签 + 人类可读说明
- **Bug 规避**: `printf` 替代 heredoc（bash 5.3+ hang）
- **路径探测**: 软失败（`|| echo "..."`）而非硬退出
- **安全解引用**: `${VAR:-}` 避免 unbound 错误
- **结构化输出**: 3 种 JSON 格式各自正确
- **平台约定严格遵守**: Cursor 用 snake_case，Claude Code 用嵌套 camelCase

---

## 6. 缺点风险局限

- **`COPILOT_CLI` 检测依赖环境变量**: 如果未来 GitHub Copilot CLI 改变变量名，平台检测会失效
- **`<EXTREMELY_IMPORTANT>` 是 "prompt injection" 模式**: 模型可能开始忽略这些"超重要"标记（标签疲劳）。长期看可能需要新策略
- **未指定 `timeout`**: 如果 `cat SKILL.md` 卡住（如文件锁），整个会话会卡
- **未压缩输出**: SKILL.md 内容（200+ 行）原样注入，未做摘要或截断
- **无日志输出**: 整个过程静默运行，调试困难
- **未使用 `jq` 构造 JSON**: 而是手写 `printf` 字符串，脆弱（如果内容含特殊字符可能破坏 JSON）
- **`printf '%s'` 中 `\` 转义可能有边缘 case**: 如 emoji 字符或 Unicode 转义
- **未实现重试机制**: 如果 `cat` 失败一次，整个会话会注入错误消息
- **Windows 路径含空格未处理**: `${PLUGIN_ROOT}/skills/using-superpowers/SKILL.md` 没加双引号
- **未版本化输出格式**: 如果未来需要修改 JSON 字段名，需要协调所有平台

---

## 7. 关键洞察

1. **session-start 是 superpowers 的"心脏"**: 没有它，所有技能都是死的。它是"元技能加载机制"的实现

2. **平台差异的"3 倍成本"**: 同一逻辑需要 3 种 JSON 输出。这是 Hook 机制跨平台的真实代价

3. **Claude Code 字段重复 bug**: Claude Code 读 `additional_context` 和 `hookSpecificOutput.additionalContext` **不去重**——这是 Claude Code 平台的设计 bug（或者 feature），superpowers 必须主动规避

4. **性能黑魔法隐藏在大文件处理中**: `${s//old/new}` 是 bash 字符串处理的"性能快捷方式"，但 99% 的 bash 教程不教

5. **`<EXTREMELY_IMPORTANT>` 是 "prompt 工程的元层"**: 这是 superpowers 主动"操纵"模型的优先级——属于 prompt injection 的"良性使用"

6. **软失败 vs 硬失败的哲学**: 技能文件不存在时输出错误信息而非崩溃。这体现"对用户诚实但不破坏工作流"的工程哲学

7. **`${VAR:-}` 是 bash 严格模式下的必备**: 与 `set -euo pipefail` 配合使用，避免 unbound 错误

8. **Bug 规避是"工程现实主义"**: bash 5.3+ 的 heredoc hang bug 让 superpowers 主动换 printf——这是"读 issue tracker 并修复"的典范

9. **人类可读 + 模型可读的双重设计**: `<EXTREMELY_IMPORTANT>` 外层是给用户的，内层是给模型的

10. **`printf` vs `echo` 的微妙差异**: 在严格 POSIX 中，echo 行为差异很大（`-e` 是否解释转义）。`printf` 是一致的选择

---

## 8. 关联文档

| 文档/文件 | 关系 |
|----------|------|
| [`hooks/hooks.json`](file://e:/WorkStation/MyProject/superpowers/hooks/hooks.json) | 调用 `run-hook.cmd session-start`（间接调用本文件） |
| [`hooks/hooks-cursor.json`](file://e:/WorkStation/MyProject/superpowers/hooks/hooks-cursor.json) | Cursor 平台入口（间接调用本文件） |
| [`hooks/run-hook.cmd`](file://e:/WorkStation/MyProject/superpowers/hooks/run-hook.cmd) | 跨平台包装器（直接调用本文件） |
| [`skills/using-superpowers/SKILL.md`](file://e:/WorkStation/MyProject/superpowers/skills/using-superpowers/SKILL.md) | 本文件读取并注入的核心内容 |
| [`docs/windows/polyglot-hooks.md`](file://e:/WorkStation/MyProject/superpowers/docs/windows/polyglot-hooks.md) | 解释跨平台 Hook 机制 |
| https://github.com/obra/superpowers/issues/571 | bash 5.3+ heredoc hang issue（printf 替代的原因） |
| `~/.config/superpowers/skills` | 旧版技能目录（本文件检测并警告） |
| `~/.claude/skills` | 新版技能目录（本文件建议迁移到） |

---

## 9. 状态评价

| 维度 | 评分 | 说明 |
|------|------|------|
| 跨平台兼容性 | 9/10 | 3 平台支持，平台检测完整 |
| 健壮性 | 8/10 | 多种软失败机制，但缺 timeout |
| 性能 | 9/10 | `${s//old/new}` 优化显著 |
| 可读性 | 8/10 | 注释充分，但 printf 字符串可读性差 |
| 可维护性 | 7/10 | 平台特化逻辑分散在 if/elif/else |
| 错误处理 | 8/10 | 软失败友好，无结构化日志 |
| 文档化 | 9/10 | 关键决策都有注释 |
| 安全性 | 7/10 | JSON 输出未用 jq 验证，可能有边缘 case |
| 教育价值 | 10/10 | 涵盖 bash 性能技巧、跨平台、prompt 工程 |

**总体评价**: 8.5/10 - 一个精心设计的、承载关键功能的脚本。

**改进优先级**:
1. **高**: 使用 `jq` 构造 JSON 而非 `printf`（更安全）
2. **中**: 添加结构化日志（`logger` 或 `echo` 到 stderr）
3. **中**: 添加 `timeout` 防止 cat 卡死
4. **低**: 探索 `<EXTREMELY_IMPORTANT>` 标签疲劳的缓解策略
5. **低**: 路径含空格处理（加双引号）

---

## 10. 给读者的启示

- **"心脏"代码应最小且可读**: 58 行是合适的——少一行可能缺功能，多一行可能难懂
- **平台差异是真实成本**: 同一逻辑 3 份实现是"必须"而非"冗余"
- **性能优化藏在细节中**: `${s//old/new}` 是 bash 字符串处理的"性能快捷方式"
- **Bug 跟踪是工程现实主义**: 读 GitHub issue 并主动修复是高质量工程的体现
- **软失败比硬崩溃友好**: 让插件仍能加载 + 输出错误信息 > 直接崩溃
- **元指令是 prompt 工程的"超能力"**: `<EXTREMELY_IMPORTANT>` 标签是 superpowers 主动操纵模型优先级
- **平台特化是"必须"而非"应该"**: Cursor / Claude Code / Copilot CLI 的字段名差异不是"美观"问题
- **静默运行 vs 结构化日志**: 静默运行在生产中更难调试，结构化日志是必要的观测能力
- **`jq` 应该是 JSON 构造的标准工具**: 手写 `printf` 字符串是脆弱的
- **"prompt injection" 也能良性使用**: superpowers 用它确保模型"记住"自己的能力

---

## 11. 后续工作

### 11.1 短期改进（2-3 小时）

- [ ] 使用 `jq` 构造 JSON：`jq -n --arg ctx "$session_context" '{...}'` 而非 printf
- [ ] 添加 `timeout 5 cat ...` 防止卡死
- [ ] 添加结构化日志（每次输出到 stderr：`[session-start] platform=claude-code, time=123ms`）
- [ ] 路径加双引号：`cat "${PLUGIN_ROOT}/skills/using-superpowers/SKILL.md"`

### 11.2 中期改进（半天）

- [ ] 探索 `<EXTREMELY_IMPORTANT>` 标签疲劳的缓解策略（如随机化标签、添加警告）
- [ ] 添加单元测试：模拟不同环境变量，验证输出 JSON 格式
- [ ] 提取平台检测为独立函数：`detect_platform()` 返回 `cursor|claude-code|copilot-cli|unknown`
- [ ] 添加 `--dry-run` 模式：输出 JSON 而不实际注入（用于调试）

### 11.3 长期演进

- [ ] 考虑用 Python 重写（统一跨平台 + 内置 JSON 库）
- [ ] 探索"懒注入"模式：只在用户首次调用 Skill 工具前注入
- [ ] 设计 `<EXTREMELY_IMPORTANT>` 标签的"可发现性"测试（验证模型是否真的遵循）
- [ ] 实现"上下文压缩感知"：检测到 compact 后重新注入关键提示

### 11.4 风险预警

- ⚠️ Claude Code 平台可能改变字段重复行为（`additional_context` + `hookSpecificOutput`）——需要跟踪平台更新
- ⚠️ Copilot CLI 升级可能改变 `COPILOT_CLI` 环境变量名
- ⚠️ bash 5.3+ 的 heredoc hang bug 可能升级或变化——printf 是当前缓解
- ⚠️ `<EXTREMELY_IMPORTANT>` 标签可能被模型"标签疲劳"忽略——需要长期监测
- ⚠️ 如果 `using-superpowers/SKILL.md` 内容膨胀（> 1000 行），每次注入会显著增加 token 成本

### 11.5 教育价值挖掘

- [ ] 在 `docs/` 下补充 `docs/session-start-deep-dive.md` 详细讲解
- [ ] 录制视频讲解 bash 性能技巧（`${s//old/new}`）
- [ ] 编写"为什么用 printf 不用 heredoc" 的博客文章
- [ ] 演示"prompt injection 良性使用"的最佳实践

### 11.6 监控/观测

- [ ] 每次 SessionStart 输出耗时、token 数量、警告状态
- [ ] 设计"健康检查"命令：`session-start --health` 输出诊断信息
- [ ] 实现"上下文预览"模式：`session-start --preview` 输出即将注入的内容（用户可见）

### 11.7 跨平台演进

- [ ] 添加 Gemini CLI 平台支持（需了解其 Hook 机制）
- [ ] 添加 Zed / Continue.dev 等其他 AI 编辑器支持
- [ ] 探索"统一平台 SDK"——让所有平台用同一 JSON 格式

---
