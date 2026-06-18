# polyglot-hooks.md 分析报告

## 基本信息

| 项目 | 内容 |
| --- | --- |
| 文档路径 | `docs/windows/polyglot-hooks.md` |
| 字数规模 | 约 213 行 / 6KB |
| 文档类型 | 跨平台技术指南 / 工程实践文档 |
| 主题 | Claude Code 跨平台钩子的 polyglot 包装器技术 |
| 涉及平台 | Windows / macOS / Linux（重点 Windows） |
| 面向读者 | 需要为 Claude Code 编写跨平台插件的开发者 |

---

## 一、文档目的

解决一个具体的工程问题：**Claude Code 插件的钩子（hooks）如何在 Windows、macOS、Linux 三大平台上无缝运行**。

具体目标：
1. **揭示痛点**：CMD 无法直接执行 `.sh` 脚本；`bash` 不在 CMD 的 PATH 中；路径分隔符不兼容
2. **给出方案**：用 polyglot 脚本（一段代码在多个解释器中都合法）作为"统一入口"
3. **避免踩坑**：通过"纯 bash"写法规避 sed/awk 等不一定存在的工具
4. **故障排查**：5 类常见错误的诊断与解决

定位上属于 **"跨平台基础设施层"** 的工程文档——不是给最终用户看的，而是给**插件作者**看的。

---

## 二、核心技术决策

### 决策 1：使用 polyglot 而不是分发多套脚本

**Polyglot 思路**：

```cmd
: << 'CMDBLOCK'        ← 既是 CMD label，也是 bash heredoc 起始
@echo off               ← 只对 CMD 有效
...Windows only code...
CMDBLOCK                ← 既是 CMD 字符串，也是 bash heredoc 终止

# Unix shell runs from here
...Unix only code...
```

**为什么这个方案优雅**：

| 对比方案 | 缺点 |
| --- | --- |
| 分发 `session-start.cmd` + `session-start.sh` 两份 | hooks.json 要写两套、平台判断逻辑复杂 |
| 用 WSL 跑 Windows 钩子 | 性能差、配置复杂、需要 WSL 安装 |
| 用 PowerShell 统一 | 跨平台问题没解决，macOS/Linux 默认没有 PowerShell |
| **Polyglot 单一文件** | 同一份代码，两个解释器各自只看自己的部分 |

**技术本质**：利用了"两个解释器对相同语法的不同解释"——
- CMD 把 `: << 'CMDBLOCK'` 看成 `:` label（无效） + ` << 'CMDBLOCK'`（被忽略）
- bash 把 `: << 'CMDBLOCK'` 看成 `: no-op` + `<< 'CMDBLOCK' heredoc`

### 决策 2：用 `.cmd` 扩展名（而非 `.sh`）作为入口

**理由**：
- Windows 资源管理器默认把 `.cmd` 识别为可执行文件（双击能跑）
- `hooks.json` 中的 `command` 字段不需要平台判断
- 同一份文件在 macOS/Linux 上加可执行权限后也能跑

### 决策 3：用 Git Bash 作为 Windows 端的 Unix 兼容层

依赖 **Git for Windows** 自带的：
- `bash.exe`：bash 解释器
- `cygpath`：Windows ↔ Unix 路径转换
- `dirname`、`pwd` 等 Unix 工具

**为什么不用 WSL/Cygwin/MSYS2**：
- Git for Windows 是开发者机器的标配（覆盖率最高）
- 安装包小（约 50MB）
- 不需要额外配置

**代价**：必须在脚本中硬编码 `C:\Program Files\Git\bin\bash.exe` 路径，假设用户没有改 Git 安装位置。

### 决策 4：`cygpath -u` 做路径转换

```bash
"$(cygpath -u \"$CLAUDE_PLUGIN_ROOT\")"
```

**作用**：`C:\Program Files\plugin` → `/c/Program Files/plugin`

**为什么必须**：
- bash 拿到 `${CLAUDE_PLUGIN_ROOT}` 时，是 Windows 路径
- 但 bash 内部 `cd`、`source`、`./script.sh` 都期望 Unix 路径
- 直接拼接会产生混合路径（如 `/c/Program Files/hooks/foo.sh`），可能不工作

### 决策 5：`-l` (login shell) 标志

```bash
"C:\Program Files\Git\bin\bash.exe" -l -c "..."
```

**作用**：让 bash 作为 login shell 启动，从而读取 `.bash_profile`、设置完整的 PATH

**为什么必须**：默认情况下，从 CMD 启动的 bash 不会自动获得 PATH 中的 Unix 工具（`/usr/bin/cygpath` 等），导致 `cygpath: command not found`。

### 决策 6：纯 bash 内建实现 JSON 转义

```bash
escape_for_json() {
    local input="$1"
    local output=""
    local i char
    for (( i=0; i<${#input}; i++ )); do
        char="${input:$i:1}"
        case "$char" in
            $'\\') output+='\\' ;;
            '"') output+='\"' ;;
            ...
        esac
    done
    printf '%s' "$output"
}
```

**为什么**：避免依赖 `sed`/`awk`，因为：
- Git Bash 默认带它们，但 PATH 可能没设好
- 多管道调用性能差
- 不同平台 BSD sed vs GNU sed 行为有差异

**代价**：代码量变大，复杂度从"一行管道"变成"20 行循环"。但换来了**零外部依赖**。

### 决策 7：通用 `run-hook.cmd` 模式

把"polyglot 包装器"参数化：

```cmd
"${CLAUDE_PLUGIN_ROOT}/hooks/run-hook.cmd" session-start.sh
"${CLAUDE_PLUGIN_ROOT}/hooks/run-hook.cmd" validate-bash.sh
```

**好处**：
- 一个包装器服务所有钩子
- 新增钩子不需要重复写 polyglot 包装
- 维护成本集中在 1 个文件

---

## 三、文件结构

| 章节 | 内容 | 工程价值 |
| --- | --- | --- |
| 1. The Problem | 4 个跨平台挑战 | 让读者意识到问题的复杂度 |
| 2. The Solution | polyglot 包装器 + 双向解析说明 | 核心方案 |
| 3. File Structure | 3 个文件的协作关系 | 直观展示架构 |
| 4. Requirements | 平台依赖 | 部署前提 |
| 5. Writing Cross-Platform Scripts | Do/Don't 清单 + JSON 转义示例 | 实战指导 |
| 6. Reusable Wrapper Pattern | 通用 run-hook.cmd | 进阶抽象 |
| 7. Troubleshooting | 5 类错误诊断 | 救援指南 |
| 8. Related Issues | 3 个 GitHub issue 链接 | 问题溯源 |

**结构特点**：从"问题→方案→实现→优化→排错"是一个**完整的工程闭环**。

---

## 四、优点

### 1. 工程美学：polyglot 本身的优雅
- 用最少的代码（一个 `: << 'CMDBLOCK'`）解决了"双解释器入口"问题
- 没有任何 hack、没有任何环境变量判断
- 一旦理解，方案就**自然涌现**

### 2. 实战导向：不是 toy example
- 提供的 JSON 转义函数是**真实场景中会用到**的
- 5 类故障排查来自**真实 GitHub issue**

### 3. 进阶与基础并重
- 基础用户：直接复制 polyglot 模板
- 进阶用户：通用 run-hook.cmd 模式可扩展

### 4. 暴露失败模式
- "Script opens in text editor" 这类错误非常具体、非常有"亲历感"
- 不回避"硬编码 `C:\Program Files\Git\bin\bash.exe`"这个**已知限制**

### 5. 与上游 issue 联动
- 末尾的 3 个 GitHub issue 链接，让文档具备**演进能力**——上游修好后，可以更新文档

---

## 五、缺点与风险

### 1. 硬编码 Git 安装路径

```bash
"C:\Program Files\Git\bin\bash.exe"
```

**问题**：
- 如果用户把 Git 装到 `D:\Git\` 就会失败
- 如果用 Portable Git（绿色版）也会失败
- 文档说"if Git is installed elsewhere, the wrapper needs modification"——但没给**自动检测**的方法

**改进方向**：
- 用 `where bash` 或 `where git` 探测
- 或者读取 `HKEY_LOCAL_MACHINE\SOFTWARE\GitForWindows` 注册表

### 2. 缺少 Windows 原生 PowerShell 方案

文档假设 `bash.exe` 一定可用，但有些企业环境：
- 没有 Git for Windows
- 有 WSL 但配置复杂
- 强制使用 PowerShell

**缺失场景**：纯 PowerShell 的 polyglot 包装器（PowerShell 也支持 here-string，可以做类似的双语兼容）。

### 3. cygpath 依赖 Git Bash

如果未来 Git for Windows 把 cygpath 移除或重命名，包装器就崩。

**风险等级**：低（cygpath 是 Git Bash 的核心工具，20 年没变过）。

### 4. JSON 转义函数不完整

文档给的 `escape_for_json` 只处理了 `\\`、`"`、`\n`、`\r`、`\t`，但 JSON 还可能需要：
- `\b`（退格）
- `\f`（换页）
- Unicode 转义（`\u00ff`）

**影响**：如果传给 Claude 的内容包含特殊字符，可能产生 invalid JSON。

### 5. 没有 macOS/Linux 端的"反向 polyglot"提示

文档聚焦在 Windows→bash 的翻译，但没说：
- macOS 的 bash 版本（bash 3.2 vs bash 5.x）的差异
- `/bin/sh` 实际是 dash 还是 bash

### 6. 缺少测试方法

**这是一个重要的"工程债"**：
- 文档没说怎么测试 polyglot 包装器在两个平台都正确解析
- 没有 CI 流程验证
- 故障排查是"用户发现问题→看文档"的事后流程，没有"事前验证"

### 7. 不适合 GUI 调试

- polyglot 包装器的"双解释器"性质，使得 IDE 调试工具很难跟入
- 出错时栈追踪信息混乱（CMD 的错？bash 的错？cygpath 的错？）

---

## 六、关键洞察

### 洞察 1：Polyglot 是"用语言差异性换代码统一性"

```
     ┌────────────┐
     │  source.cmd │
     └─────┬───────┘
           │
   ┌───────┴───────┐
   ↓               ↓
Windows         Unix
reads as        reads as
batch script    bash script
   ↓               ↓
bash.exe         bash
```

**本质**：**同一段字节序列在不同解释器中触发完全不同的执行路径**。这在编译器理论中叫"语法歧义"（syntactic ambiguity），但在这里被有意利用为"统一入口"。

### 洞察 2：CMD label + bash heredoc 是"对偶语法"

| 元素 | CMD 视角 | bash 视角 |
| --- | --- | --- |
| `:` | label 起始 | 空操作 (no-op) |
| `<< 'CMDBLOCK'` | 重定向操作符（被忽略） | heredoc 起始 |
| `'CMDBLOCK'` | 引号字符串 | 引号字符串（防止变量展开） |
| `CMDBLOCK` | 字符串字面量 | heredoc 结束符 |

**这种"对偶性"不是偶然**——shell 语法在历史上就是各种解释器互相借鉴的，自然形成了大量兼容的语法片段。

### 洞察 3：Git Bash 是 Windows 上事实标准的"伪 Unix"

- 比 Cygwin 轻量
- 比 WSL 简单
- 比 MSYS2 易装
- 几乎所有"在 Windows 上跑 Unix 工具"的需求都通过它

**结论**：Superpowers 选择依赖 Git Bash 是**最务实的选择**。

### 洞察 4：故障排查 5 条是"经验沉淀"

| 错误 | 原因 | 修复 |
| --- | --- | --- |
| `bash is not recognized` | Git 路径不对 | 改硬编码 |
| `cygpath: command not found` | 缺 `-l` | 加 `-l` |
| 路径中 `\/` | 没转换 | 用 `cygpath -u` |
| 脚本在编辑器中打开 | hooks.json 指向 .sh | 改 .cmd |
| 终端能跑钩子不行 | 钩子环境特殊 | 模拟 env 变量 |

每一条都是**真实踩过坑**才能写出的——这不是凭空设计，是实战总结。

### 洞察 5：通用 run-hook.cmd 是"框架化"

从"每个钩子一个 polyglot 文件"进化到"一个 polyglot 文件 + N 个 sh 文件"，本质上是**从代码复用 → 模式抽象**的飞跃。

这是 Unix 哲学（small tools, compose）的体现。

---

## 七、关联文档

| 文档 | 关系 |
| --- | --- |
| `hooks/hooks.json` | 实际项目中的 hook 配置，文档中所有示例都基于它 |
| `hooks/hooks-cursor.json` | Cursor 平台的 hook 配置（也用了类似的 polyglot 思路） |
| `hooks/session-start` | 文档中的 `session-start.sh` 在仓库中的实际位置 |
| `hooks/run-hook.cmd` | 文档"可复用包装器"模式在仓库中的实际产物 |
| Anthropic 官方 Claude Code 文档 | hooks 的官方 API 文档 |
| `docs/plans/` 下的设计文档 | polyglot 思路的可能演化记录 |

---

## 八、状态评价

| 维度 | 评分 | 说明 |
| --- | --- | --- |
| **问题定义** | ⭐⭐⭐⭐⭐ (5/5) | 4 个跨平台挑战清晰、具体 |
| **方案创新性** | ⭐⭐⭐⭐⭐ (5/5) | Polyglot 思路优雅、工程美感强 |
| **可操作性** | ⭐⭐⭐⭐ (4/5) | 模板可复制，但硬编码 Git 路径降低可移植性 |
| **完整性** | ⭐⭐⭐⭐ (4/5) | 覆盖了正向流程，但缺测试、缺 PowerShell 方案 |
| **跨平台平衡** | ⭐⭐⭐ (3/5) | 重点 Windows，macOS/Linux 描述相对薄弱 |
| **可维护性** | ⭐⭐⭐⭐ (4/5) | 通用 run-hook 模式降低维护成本 |
| **演进能力** | ⭐⭐⭐⭐ (4/5) | 关联了 GitHub issue，能跟随上游修复 |

**总体评价**：**工程实践文档的典范**。把一个真实的工程难题（跨平台 shell 兼容）讲得清晰、有美感、有解决方案。硬编码 Git 路径和缺 PowerShell 方案是明显短板，但不影响它在"Windows 用户群"中的价值。

---

## 九、给读者的启示

### 如果你正在为 Claude Code 写插件
- **必读**：第 2 节（polyglot 模板）、第 3 节（文件结构）、第 7 节（故障排查）
- **必做**：把 `cygpath -u` + `-l` 这两个细节照搬
- **必改**：把硬编码的 Git 路径改为适合你用户群的探测方式

### 如果你正在做其他跨平台工具
- **借鉴**：
  - "同一份代码，多个解释器"的 polyglot 思路
  - "纯 bash 内建"避免外部依赖的哲学
  - "通用包装器"模式（一个 polyglot 入口 + N 个具体脚本）
- **警惕**：
  - 硬编码路径的可移植性问题
  - 没有 CI 验证的 polyglot 代码可能在某个解释器升级时崩

### 如果你维护 Superpowers
- **改进建议 1**：增加 PowerShell 变体（满足企业用户）
- **改进建议 2**：自动探测 Git for Windows 安装路径
- **改进建议 3**：补充 `escape_for_json` 的完整 Unicode 处理
- **改进建议 4**：在 `tests/` 下加一个 polyglot 包装器的 CI 测试

### 如果你只是 Windows 上遇到 Claude Code 钩子跑不起来
- **直接翻到第 7 节**（故障排查）—— 5 条对应 5 个最常见症状

---

## 十、后续工作

按重要性排序：

1. **高优先级**：把硬编码的 `C:\Program Files\Git\bin\bash.exe` 改为自动探测（用 `where bash`、`%PROGRAMFILES%`、注册表等）
2. **高优先级**：补全 `escape_for_json` 的 Unicode 处理
3. **中优先级**：补充 PowerShell 用户的替代方案
4. **中优先级**：增加 CI 测试（GitHub Actions 多平台）验证 polyglot 在两个解释器中都正确解析
5. **低优先级**：写一篇"为什么 polyglot 这么设计"的原理性文章（可以放在 `docs/superpowers/specs/`）
6. **可选**：探索用 PowerShell 也能识别的 polyglot 语法（如 here-string）来支持更多平台

---

**总结**：这份文档是**"工程美学"与"实战经验"的完美结合**。Polyglot 包装器本身是软件工程中"用语言差异性换代码统一性"的精彩案例；而 5 条故障排查则凝聚了**无数次踩坑后的真知灼见**。对于任何需要写跨平台 shell 脚本的人来说，这都是必读教材。
