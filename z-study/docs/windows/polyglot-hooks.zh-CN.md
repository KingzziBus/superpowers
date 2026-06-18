# Claude Code 的跨平台 Polyglot 钩子

Claude Code 插件需要能在 Windows、macOS 和 Linux 上运行的钩子。本文解释让这一切成为可能的 polyglot（多语言兼容）包装器技术。

## 问题背景

Claude Code 通过系统默认 shell 来执行钩子命令：

- **Windows**：CMD.exe
- **macOS/Linux**：bash 或 sh

这带来几个挑战：

1. **脚本执行**：Windows CMD 无法直接执行 `.sh` 文件——它会尝试在文本编辑器中打开
2. **路径格式**：Windows 使用反斜杠（`C:\path`），Unix 使用正斜杠（`/path`）
3. **环境变量**：`$VAR` 语法在 CMD 中无效
4. **PATH 中没有 `bash`**：即使安装了 Git Bash，CMD 运行时 `bash` 也不在 PATH 里

## 解决方案：Polyglot `.cmd` 包装器

Polyglot 脚本是同时在多种语言中合法的语法。我们的包装器在 CMD 和 bash 中都合法：

```cmd
: << 'CMDBLOCK'
@echo off
"C:\Program Files\Git\bin\bash.exe" -l -c "\"$(cygpath -u \"$CLAUDE_PLUGIN_ROOT\")/hooks/session-start.sh\""
exit /b
CMDBLOCK

# Unix shell runs from here
"${CLAUDE_PLUGIN_ROOT}/hooks/session-start.sh"
```

### 工作原理

#### Windows (CMD.exe) 下

1. `: << 'CMDBLOCK'` - CMD 把 `:` 当作 label（标签），如 `:label`，并忽略 `<< 'CMDBLOCK'`
2. `@echo off` - 关闭命令回显
3. bash.exe 命令携带以下参数运行：
   - `-l`（login shell）以获得包含 Unix 工具的完整 PATH
   - `cygpath -u` 把 Windows 路径转换为 Unix 格式（`C:\foo` → `/c/foo`）
4. `exit /b` - 退出批处理脚本，CMD 在此终止
5. `CMDBLOCK` 之后的所有内容，CMD 永远不会执行到

#### Unix (bash/sh) 下

1. `: << 'CMDBLOCK'` - `:` 是个空操作（no-op），`<< 'CMDBLOCK'` 开启一个 heredoc
2. 直到 `CMDBLOCK` 之前的所有内容都会被 heredoc 消耗（忽略掉）
3. `# Unix shell runs from here` - 注释
4. 脚本使用 Unix 路径直接运行

## 文件结构

```
hooks/
├── hooks.json           # 指向 .cmd 包装器
├── session-start.cmd    # Polyglot 包装器（跨平台入口点）
└── session-start.sh     # 实际的钩子逻辑（bash 脚本）
```

### hooks.json

```json
{
  "hooks": {
    "SessionStart": [
      {
        "matcher": "startup|resume|clear|compact",
        "hooks": [
          {
            "type": "command",
            "command": "\"${CLAUDE_PLUGIN_ROOT}/hooks/session-start.cmd\""
          }
        ]
      }
    ]
  }
}
```

注意：路径必须加引号，因为 `${CLAUDE_PLUGIN_ROOT}` 在 Windows 上可能包含空格（如 `C:\Program Files\...`）。

## 依赖要求

### Windows
- 必须安装 **Git for Windows**（提供 `bash.exe` 和 `cygpath`）
- 默认安装路径：`C:\Program Files\Git\bin\bash.exe`
- 如果 Git 安装在别的位置，包装器需要相应修改

### Unix (macOS/Linux)
- 标准 bash 或 sh shell
- `.cmd` 文件必须有可执行权限（`chmod +x`）

## 编写跨平台钩子脚本

实际的钩子逻辑放在 `.sh` 文件中。为了确保它在 Windows 上（通过 Git Bash）也能工作：

### 推荐做法
- 尽量使用纯 bash 内建命令
- 使用 `$(command)` 而非反引号
- 引用所有变量展开：`"$VAR"`
- 使用 `printf` 或 here-doc 来输出

### 避免
- 使用可能在 PATH 中找不到的外部命令（sed、awk、grep）
- 如果一定要用，它们在 Git Bash 中可用，但要确保 PATH 设置正确（使用 `bash -l`）

### 示例：不用 sed/awk 实现 JSON 转义

不要这样：

```bash
escaped=$(echo "$content" | sed 's/\\/\\\\/g' | sed 's/"/\\"/g' | awk '{printf "%s\\n", $0}')
```

而是使用纯 bash：

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
            $'\n') output+='\n' ;;
            $'\r') output+='\r' ;;
            $'\t') output+='\t' ;;
            *) output+="$char" ;;
        esac
    done
    printf '%s' "$output"
}
```

## 可复用包装器模式

对于有多个钩子的插件，可以创建一个接受脚本名参数的通用包装器：

### run-hook.cmd

```cmd
: << 'CMDBLOCK'
@echo off
set "SCRIPT_DIR=%~dp0"
set "SCRIPT_NAME=%~1"
"C:\Program Files\Git\bin\bash.exe" -l -c "cd \"$(cygpath -u \"%SCRIPT_DIR%\")\" && \"./%SCRIPT_NAME%\""
exit /b
CMDBLOCK

# Unix shell runs from here
SCRIPT_DIR="$(cd "$(dirname "$0")" && pwd)"
SCRIPT_NAME="$1"
shift
"${SCRIPT_DIR}/${SCRIPT_NAME}" "$@"
```

### 使用可复用包装器的 hooks.json

```json
{
  "hooks": {
    "SessionStart": [
      {
        "matcher": "startup",
        "hooks": [
          {
            "type": "command",
            "command": "\"${CLAUDE_PLUGIN_ROOT}/hooks/run-hook.cmd\" session-start.sh"
          }
        ]
      }
    ],
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "\"${CLAUDE_PLUGIN_ROOT}/hooks/run-hook.cmd\" validate-bash.sh"
          }
        ]
      }
    ]
  }
}
```

## 故障排查

### "bash is not recognized"（bash 无法识别）

CMD 找不到 bash。包装器使用的是完整路径 `C:\Program Files\Git\bin\bash.exe`。如果 Git 安装在别的位置，请更新路径。

### "cygpath: command not found" 或 "dirname: command not found"

bash 没有作为 login shell 运行。确保使用了 `-l` 参数。

### 路径中出现了奇怪的 `\/`

`${CLAUDE_PLUGIN_ROOT}` 展开成了以反斜杠结尾的 Windows 路径，然后 `/hooks/...` 被追加了。使用 `cygpath` 来转换整个路径。

### 脚本在文本编辑器中打开而不是运行

`hooks.json` 直接指向了 `.sh` 文件。改为指向 `.cmd` 包装器。

### 在终端中能跑但作为钩子不工作

Claude Code 运行钩子的方式可能不同。可以通过模拟钩子环境来测试：

```powershell
$env:CLAUDE_PLUGIN_ROOT = "C:\path\to\plugin"
cmd /c "C:\path\to\plugin\hooks\session-start.cmd"
```

## 相关问题

- [anthropics/claude-code#9758](https://github.com/anthropics/claude-code/issues/9758) - Windows 上 .sh 脚本在编辑器中打开
- [anthropics/claude-code#3417](https://github.com/anthropics/claude-code/issues/3417) - Windows 上钩子不工作
- [anthropics/claude-code#6023](https://github.com/anthropics/claude-code/issues/6023) - 找不到 CLAUDE_PROJECT_DIR
