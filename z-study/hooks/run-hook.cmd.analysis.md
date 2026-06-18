# run-hook.cmd 深度分析报告

> 文件: `hooks/run-hook.cmd`（47 行，~1.4 KB）
> 类型: 跨平台 Polyglot 包装器（同时兼容 Windows cmd 和 Unix bash）
> 角色: superpowers 插件的"跨平台 Hook 分发中心"

---

## 1. 基本信息

| 字段 | 值 |
|------|-----|
| 文件路径 | `hooks/run-hook.cmd` |
| 文件大小 | 约 1.4 KB（47 行） |
| 文件类型 | Polyglot 脚本（同时被 cmd.exe 和 bash 解释） |
| Windows 行为 | cmd.exe 解析 `: << 'CMDBLOCK' ... CMDBLOCK` 之前的批处理部分 |
| Unix 行为 | bash 跳过 `: << 'CMDBLOCK' ... CMDBLOCK` heredoc，执行最后 5 行 |
| 调用方 | `hooks/hooks.json` 和 `hooks/hooks-cursor.json` |
| 被调用方 | `hooks/session-start`（作为参数 `<script-name>` 传入） |
| 主要功能 | 在 Windows 上找 bash 并调用无扩展名 hook 脚本 |
| 参数协议 | `<script-name> [args...]`（最多 9 个显式参数） |

---

## 2. 目的与背景

`run-hook.cmd` 是 superpowers 插件的**跨平台 Hook 分发中心**。它的存在是为了解决一个具体的、棘手的工程问题：

**核心问题**：

1. Claude Code / Cursor 在 Windows 上需要调用 hook 脚本
2. hook 脚本本质是 bash（要做字符串处理、JSON 转义、文件读取）
3. Windows 默认不包含 bash
4. 即使包含 bash，Claude Code 在 Windows 上会自动给 `.sh` 文件加 `bash` 前缀，导致双重 bash 调用而失败

**解决方案**：写一个**双解释器脚本**（polyglot）：

- 在 Windows 上：被 `cmd.exe` 解析为批处理文件，找 bash 并调用真正的 hook 脚本
- 在 Unix 上：被 `bash` 解析为 shell 脚本，直接执行 hook 脚本
- **同一份文件** 兼容两个平台

文件名后缀 `.cmd` 是 Windows 习惯（让 Claude Code 知道这是可执行命令），但 bash 也会执行它（shebang 缺失但 bash 通过启发式识别）。

---

## 3. 核心技术决策

### 3.1 Polyglot 黑魔法：`: << 'CMDBLOCK' ... CMDBLOCK`

这是整个文件最精彩的部分。理解它需要知道两个解释器的特性：

- **cmd.exe 中**: `:` 是空命令（no-op），所以 `: << 'CMDBLOCK'` 之后的所有行都是合法 cmd 语法（被忽略）
- **bash 中**: `<<` 是 heredoc 操作符，`<<'CMDBLOCK' ... CMDBLOCK` 把整段内容作为字符串喂给 `:`（空命令），相当于注释

**效果**：
- 同一段代码（行 1-40）在 cmd 中是有效批处理，在 bash 中是 no-op heredoc
- 行 42-46 是 bash 专属代码，在 cmd 中会被忽略（被 `CMDBLOCK` 闭合后是 `exec` 语句，不符合 cmd 语法但 cmd 已经 exit 在 `exit /b 0` 之前）

这是 Unix 传统中的 **"self-extracting shell script"** 模式，被巧用于跨平台。

### 3.2 为什么用无扩展名的 hook 脚本（`session-start` 而非 `session-start.sh`）？

来自文件注释（行 7-9）：

> Hook scripts use extensionless filenames (e.g. "session-start" not "session-start.sh") so Claude Code's Windows auto-detection -- which prepends "bash" to any command containing .sh -- doesn't interfere.

**核心问题**：

- Claude Code 在 Windows 上的"自动检测"逻辑是：如果命令包含 `.sh`，自动在前面加 `bash`
- 如果 hook 命令是 `"bash session-start.sh"`，会被改写为 `"bash bash session-start.sh"`
- 双 `bash` 调用会失败（第一个 bash 把第二个当参数）

**解法**：

- 故意**不写 `.sh` 扩展名**（用 `session-start`）
- 通过 `run-hook.cmd` 间接调用，绕开自动检测
- 这是一个**利用 bug 缺陷**的工程应对

### 3.3 为什么按顺序探测 Git Bash 的多个标准路径？

Windows 上最常见的 bash 来源是 Git for Windows。它有多个可能的安装位置：

```batch
if exist "C:\Program Files\Git\bin\bash.exe" (
    "C:\Program Files\Git\bin\bash.exe" "%HOOK_DIR%%~1" %2 %3 %4 %5 %6 %7 %8 %9
    exit /b %ERRORLEVEL%
)
if exist "C:\Program Files (x86)\Git\bin\bash.exe" (
    "C:\Program Files (x86)\Git\bin\bash.exe" "%HOOK_DIR%%~1" %2 %3 %4 %5 %6 %7 %8 %9
    exit /b %ERRORLEVEL%
)
```

- 第一个路径：64-bit Git for Windows（标准 Program Files）
- 第二个路径：32-bit Git for Windows（x86 Program Files，少数老用户）
- 第三个：`where bash` 找 PATH 中的 bash（覆盖 MSYS2、Cygwin、Chocolatey 安装的 bash 等）

**为什么不直接用 `where bash`？**

- 性能：`where` 会扫描整个 PATH，第一次调用较慢
- 可靠性：标准 Git 路径的 64-bit/32-bit 探测是确定性的，避免 PATH 中有多个 bash 时的歧义
- 这种"硬编码 + 兜底"的模式在系统脚本中很常见

### 3.4 为什么用 `%2 %3 %4 %5 %6 %7 %8 %9` 而不是 `%*`？

- `cmd.exe` 的 `%*` 表示"所有参数"，但**在批处理中展开行为有坑**（空参数会消失，路径含空格会断开）
- 显式列 `%2` 到 `%9` 是**安全的展开方式**，且**最多 9 个参数**（cmd 限制）
- 当前 hook 用法只传 1 个参数（`<script-name>`），所以 `session-start` 收到的参数 = `%2 %3 ... %9` 是空的，无影响
- 这是**为未来扩展预留**的代码（如果以后 hook 需要传多个参数）

### 3.5 静默失败的哲学：`exit /b 0` 而非错误退出

文件最后一段（行 37-39）：

```batch
REM No bash found - exit silently rather than error
REM (plugin still works, just without SessionStart context injection)
exit /b 0
```

**这是关键设计决策**：

- 如果用户没装 bash，`run-hook.cmd` 不报错（`exit /b 1`）
- 而是**优雅降级**：插件仍加载，但 SessionStart 不注入上下文
- 用户体验：用户不会因为缺 bash 而被 Claude Code 报错（"Hook failed" 会很烦人）
- 代价：用户**不会知道** superpowers 没生效——但这是更可接受的代价

### 3.6 Unix 部分为什么只 5 行？

```bash
SCRIPT_DIR="$(cd "$(dirname "$0")" && pwd)"
SCRIPT_NAME="$1"
shift
exec bash "${SCRIPT_DIR}/${SCRIPT_NAME}" "$@"
```

- `SCRIPT_DIR`：解析 `run-hook.cmd` 自身所在目录
- `SCRIPT_NAME`：第一个参数是 hook 名（如 `session-start`）
- `shift`：把 `SCRIPT_NAME` 从参数列表移除
- `exec bash ...`：用 exec 替换当前进程为 bash，避免多一层 shell

**Unix 部分比 Windows 部分简单得多**——因为 Unix 系统必有 bash，无需探测。

---

## 4. 文件结构剖析

```
┌─ cmd.exe 解析区 (行 1-40) ─────────────────────────────┐
│ 1-2:   : << 'CMDBLOCK'                                  │
│        @echo off                                        │
│        (此块是 bash heredoc，被 bash 视为注释字符串)      │
│ ...                                                      │
│ 13-16: 参数检查                                          │
│ 18:    解析 HOOK_DIR                                     │
│ 20-35: 探测 Git Bash 三个路径                            │
│ 37-39: 静默失败出口                                       │
│ 40:    CMDBLOCK (闭合 heredoc)                           │
└─────────────────────────────────────────────────────────┘

┌─ bash 解析区 (行 42-46) ────────────────────────────────┐
│ 42:  # Unix: run the named script directly               │
│ 43:  SCRIPT_DIR="$(cd "$(dirname "$0")" && pwd)"         │
│ 44:  SCRIPT_NAME="$1"                                    │
│ 45:  shift                                                │
│ 46:  exec bash "${SCRIPT_DIR}/${SCRIPT_NAME}" "$@"       │
└─────────────────────────────────────────────────────────┘
```

**完整调用链**：

```
hooks.json
  └─ command: run-hook.cmd session-start
       └─ cmd.exe 解析 run-hook.cmd
            ├─ 行 1-40 (cmd 视为批处理)
            │   └─ 探测 bash
            │       └─ bash session-start
            │            └─ 输出 JSON 到 stdout
            └─ 行 42-46 (cmd 视为无效批处理, 跳过)
```

---

## 5. 优点亮点

- **Polyglot 黑魔法**: 同一份文件兼容两个平台，无需写两份
- **注释丰富**: 17 行里有 6 行注释，解释每个设计决策
- **优雅降级**: 没装 bash 不报错，插件仍可加载
- **路径探测穷尽**: Git Bash (64-bit) → Git Bash (32-bit) → PATH 中 bash
- **参数处理健壮**: 显式列 `%2-%9` 而非 `%*`，避免引号问题
- **`exec` 优化**: Unix 部分用 `exec` 避免多一层 shell
- **`%~dp0` 解析自身目录**: 不依赖工作目录
- **错误码透传**: `exit /b %ERRORLEVEL%` 把 bash 退出码传给 cmd

---

## 6. 缺点风险局限

- **不显式指定 bash 版本**: 如果系统 bash 太老（如 bash 3.x on macOS），可能有兼容性问题
- **参数限制 9 个**: 如果未来 hook 传超过 9 个参数，会丢失。建议改为 `shift` 循环
- **`%2` 到 `%9` 在空参数时会留空**: 可能导致 `bash "" ""` 多余空格
- **Git Bash 路径硬编码**: 如果 Git 装在自定义位置会找不到（但有 `where bash` 兜底）
- **`set -euo pipefail` 缺失**: 这是 run-hook.cmd 包装器，但 session-start 内部会自己设置
- **没有 `set -e` 等错误处理**: cmd 部分如果命令失败，仍会继续（虽然后面有 `exit /b %ERRORLEVEL%`）
- **WSA / WSL 检测缺失**: Windows 11 的 WSL 提供了 bash，但 `where bash` 不一定能找到（取决于 PATH 配置）
- **macOS 默认 bash 是 3.2 (2007)**: 可能缺新特性（虽不致命）

---

## 7. 关键洞察

1. **Polyglot 是 Unix 文化的活化石**: 自解压 shell 脚本（heredoc + 平台特定代码）是 1980 年代就有的技巧，superpowers 在 2024 年还在用——证明简单的工具能跨时代

2. **"利用 bug 做 feature" 的极致**: Claude Code 的 ".sh 自动加 bash" bug 是个**有用的特性**——它"强制"开发者走 `run-hook.cmd` 抽象，进而获得跨平台能力。如果没有这个 bug，开发者可能直接写 `bash session-start.sh`，Windows 上反而坏掉

3. **优雅降级是工程美学的体现**: 缺 bash 时不报错是**对用户的尊重**——没人喜欢看到一个"hook failed"的红字导致插件不工作

4. **注释是"决策考古学"**: 行 7-9 的注释解释了**为什么**用无扩展名——这是 3 个月后的维护者最需要的信息。**没有这个注释，未来某天有人会"优化"成 `.sh` 扩展名，然后 bug 复发**

5. **三层路径探测哲学**: 硬编码 → 启发式 → fallback。这是"确定性优先，灵活兜底"的工程模式

6. **参数处理的"够用就好"原则**: 9 个参数限制是 cmd 的硬约束，作者没尝试绕过（`shift` + 循环会引入复杂度）。**少做反而是美**

7. **`%ERRORLEVEL%` 是 cmd 时代的 `exit code`**: 这是 Windows batch 与 Unix shell 的"语义桥"

8. **`exec` 在 Unix 中是性能优化**: 避免多一层 shell 进程

---

## 8. 关联文档

| 文档/文件 | 关系 |
|----------|------|
| [`hooks/hooks.json`](file://e:/WorkStation/MyProject/superpowers/hooks/hooks.json) | 调用本文件的 Claude Code 配置 |
| [`hooks/hooks-cursor.json`](file://e:/WorkStation/MyProject/superpowers/hooks/hooks-cursor.json) | 调用本文件的 Cursor 配置 |
| [`hooks/session-start`](file://e:/WorkStation/MyProject/superpowers/hooks/session-start) | 本文件分发的目标脚本 |
| [`docs/windows/polyglot-hooks.md`](file://e:/WorkStation/MyProject/superpowers/docs/windows/polyglot-hooks.md) | 解释 Polyglot Hook 机制的设计文档 |
| Git for Windows 文档 | 外部参考（Git Bash 路径） |
| Bash 3.2 vs 5.x 兼容性表 | 外部参考（macOS 默认 bash 限制） |

---

## 9. 状态评价

| 维度 | 评分 | 说明 |
|------|------|------|
| 跨平台兼容性 | 10/10 | Polyglot 技巧成熟稳定 |
| 健壮性 | 9/10 | 路径探测穷尽，静默失败优雅 |
| 可读性 | 9/10 | 注释充分，结构清晰 |
| 可维护性 | 8/10 | Polyglot 技巧对新手不友好，需注释 |
| 性能 | 8/10 | 启动多一层 cmd/bash，但不可避免 |
| 错误处理 | 7/10 | 静默失败好，但没有结构化错误日志 |
| 文档化 | 10/10 | 17 行里 6 行注释，比例极高 |
| 教育价值 | 10/10 | 教科书级别的 Polyglot 脚本 |

**总体评价**: 9.5/10 - 工程美学的典范。

**改进优先级**:
1. **低**: WSL bash 路径探测（如 `/usr/bin/bash` 通过 WSL 暴露）
2. **低**: 将 9 个参数限制改为 `shift` 循环
3. **低**: 添加 WSA（Windows Subsystem for Android）bash 路径
4. **极低**: 添加 macOS 特定 bash 提示（默认 3.2 可能不兼容某些 bash 5+ 特性）

---

## 10. 给读者的启示

- **Polyglot 是个被低估的技巧**: 不必为跨平台写两份代码
- **Heredoc 是 Unix 的"瑞士军刀"**: 几乎所有 scripting 场景都能用
- **"利用 bug 做 feature" 是高级工程能力**: 看到限制时不抱怨，把它变工具
- **优雅降级比"完美"更重要**: 不报错让插件仍可用，比"严格失败"更友好
- **注释是给未来的自己/同事的情书**: 解释**为什么**比解释**是什么**更珍贵
- **三层探测哲学**: 硬编码 → 启发式 → fallback，这是"实用主义"的工程模式
- **`exec` 是 shell 性能优化**: 避免多一层进程
- **保持简单**: 9 个参数限制是 cmd 的硬约束，但**不绕过**它也是种智慧
- **Windows 也有"Unix 之魂"**: `cmd.exe` + Git Bash 让 Windows 也能享受 Unix 工具链
- **跨平台不是"运行相同代码"**: 而是"在每个平台用最合适的方式实现同一目标"

---

## 11. 后续工作

### 11.1 短期改进（2 小时）

- [ ] 添加 WSL bash 路径探测（`\\wsl$\...\bin\bash.exe`）
- [ ] 将 9 个参数限制改为 `shift` + `for` 循环
- [ ] 添加简单单元测试：模拟不同 PATH 环境，验证 bash 探测顺序

### 11.2 中期改进（半天）

- [ ] 设计 `run-hook.ps1` PowerShell 版本（如果 cmd 兼容性成为问题）
- [ ] 探索"静态分析工具"检测 Polyglot 脚本的语法正确性
- [ ] 添加 macOS 特定警告（bash 3.2 vs 5+ 兼容性）

### 11.3 长期演进

- [ ] 考虑用 Python 重写（如果有 `python3` 可用），统一跨平台
- [ ] 探索 Node.js 单文件可执行方案（pkg + npm）
- [ ] 抽象"path detection"为独立模块，让所有 hook 复用

### 11.4 风险预警

- ⚠️ Git for Windows 改变安装路径会让前两个 `if exist` 失效（兜底 `where bash` 救场）
- ⚠️ 未来 cmd.exe 升级可能改变 heredoc 解析（虽然极不可能）
- ⚠️ Windows ARM 设备可能把 Git 装到 `C:\Program Files (Arm)\Git\`——需要新路径
- ⚠️ Claude Code 升级可能改变 ".sh 自动加 bash" 行为，需要重新评估

### 11.5 教育价值挖掘

- [ ] 在 `docs/` 下补充 `docs/polyglot-scripting.md` 讲解这种技巧
- [ ] 在 README 中加一个"如何添加新平台"小节，演示如何扩展示例
- [ ] 录制一个 5 分钟视频讲解这个文件的工作原理

### 11.6 跨平台演进

- [ ] 添加 `run-hook.ps1` 作为 PowerShell 用户的选择
- [ ] 探索 `bun run` 作为统一跨平台入口
- [ ] 评估 WSL2 优先策略（Windows 11 默认开启 WSL）

---
