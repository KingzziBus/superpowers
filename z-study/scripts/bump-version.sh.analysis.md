# bump-version.sh 深度分析报告

> 文件: `scripts/bump-version.sh`（221 行，~6.0 KB）
> 类型: 仓库级版本号管理工具（纯 bash + jq）
> 角色: superpowers 多文件版本号的"单一事实来源"守卫

---

## 1. 基本信息

| 字段 | 值 |
|------|-----|
| 文件路径 | `scripts/bump-version.sh` |
| 文件大小 | 约 6.0 KB（221 行） |
| 文件类型 | Bash 脚本（shebang `#!/usr/bin/env bash`） |
| 严格模式 | `set -euo pipefail` |
| 依赖 | `jq`（JSON 处理） |
| 配置文件 | `.version-bump.json`（仓库根） |
| 三种模式 | `<new-version>`（bump）/ `--check`（检查漂移）/ `--audit`（审计） |
| 退出码 | 0 = 成功，1 = 漂移/缺失/审计发现问题 |
| 平台 | Unix / Mac 原生 bash + Windows Git Bash |

---

## 2. 目的与背景

superpowers 是一个**跨多 AI Agent 平台**的插件库，同时支持：

- Claude Code（`.claude-plugin/plugin.json`）
- Cursor（`.cursor-plugin/plugin.json`）
- OpenAI Codex（`.codex-plugin/plugin.json`）
- OpenCode（`.opencode/plugin.json`）
- Gemini CLI（`gemini-extension.json`）

每个平台都有**自己的元数据清单文件**，而每个清单文件都有自己的 `version` 字段。如果手动同步这些版本号，会出现"drift"（漂移）—— 不同文件版本不一致。

**这个脚本的存在就是为了**：

1. **单一入口**：一个命令 bump 所有声明的文件
2. **漂移检测**：自动发现哪些文件版本不一致
3. **审计能力**：扫描整个仓库，找出**未声明的**版本号引用
4. **数据驱动**：版本号文件列表来自 `.version-bump.json`，**无需修改脚本就能扩展**

这是典型的**"data over code"** 设计哲学——把"哪些文件需要同步版本"这个信息从代码中抽离到配置中。

---

## 3. 核心技术决策

### 3.1 为什么用 jq 而非 sed/awk？

- **JSON 路径处理**：`field = "plugins.0.version"` 这种嵌套路径用 sed 极难处理
- **格式保留**：`jq "$jq_path = \"$value\""` 写回时保留缩进、引号风格
- **错误处理**：jq 在路径不存在时返回 `null`，易于检测
- **可移植性**：jq 是几乎所有 Unix 系统的标准工具

**替代方案对比**：

| 方案 | 优点 | 缺点 |
|------|------|------|
| sed 正则 | 简单 | 嵌套 JSON 易错、破坏格式 |
| python3 | 强大 | 多一层解释器、慢 |
| yq（YAML） | 简单 | 配置文件是 JSON 不是 YAML |
| **jq** | **快、准、保留格式** | **需要安装 jq** |

**jq 是"刚刚好"的选择**。

### 3.2 路径转换技巧：`field` → `jq path`

```bash
jq_path=$(echo "$field" | sed -E 's/\.([0-9]+)/[\1]/g' | sed 's/^/./' | sed 's/\.\././g')
```

**核心问题**：

- 配置文件用点路径（如 `plugins.0.version`）表示 JSON 路径
- 但 jq 期望的是 jq 路径（如 `.plugins[0].version`）
- 需要做"点路径 → jq 路径"的转换

**转换规则**：

1. `\.([0-9]+)` → `[\1]`：把 `.0` 转换为 `[0]`（数组索引）
2. `^` → `.`：在最前面加点（如 `version` → `.version`）
3. `..` → `.`：清理双重点（防止 `plugins.0.version` 变 `..plugins[0].version`）

**关键细节**：

- 第二步和第三步是 `sed 's/^/./'` 和 `sed 's/\.\././g'`
- 顺序很重要：先转换数字索引（`.0` → `[0]`），再加前导点（避免重复加点），最后清理双重点
- 这是一个**"小而美"的 sed 链**——3 个 sed 调用解决嵌套 JSON 路径问题

### 3.3 临时文件 + mv 模式：原子写入

```bash
local tmp="${file}.tmp"
jq "$jq_path = \"$value\"" "$file" > "$tmp" && mv "$tmp" "$file"
```

**为什么不直接 `jq ... > "$file"`（in-place）**：

- jq 1.6 之前没有 `--inplace` 选项
- 即使有，**in-place 写入不是原子的**——如果脚本被 kill，写入一半的文件会损坏

**优势**：

- 先写临时文件，成功后再 mv（POSIX 保证 mv 在同一文件系统是原子的）
- 如果 jq 失败，原文件保持不变
- 失败时 tmp 文件残留，可手动恢复

**这是 bash 中写文件的"金标准"**。

### 3.4 `IFS=$'\t'` + process substitution 处理 TSV 数据

```bash
while IFS=$'\t' read -r path field; do
    ...
done < <(declared_files)
```

**`declared_files` 输出格式**：

```bash
jq -r '.files[] | "\(.path)\t\(.field)"' "$CONFIG"
```

输出形如：

```
.claude-plugin/plugin.json	version
.cursor-plugin/plugin.json	version
plugins.0.version	some_field
```

**`IFS=$'\t'` 的作用**：

- 拆分时只按 TAB 分割，不按空格分割
- `read -r` 保留反斜杠字面值（不解释转义）
- 路径中可能含空格（如 `My Documents/`），所以不能按空格分

**`< <(command)` vs `command | while`**：

- `command | while` 在子 shell 中运行 `while`，变量修改不传出来
- `< <(command)`（process substitution）让 while 在当前 shell 运行，**可以修改外部变量**

这是 bash 脚本中的"高级 IO"技巧。

### 3.5 漂移检测：`sort -u | wc -l`

```bash
local unique
unique=$(printf '%s\n' "${versions[@]}" | sort -u | wc -l | tr -d ' ')
if [[ "$unique" -gt 1 ]]; then
    echo "DRIFT DETECTED..."
```

**极简的漂移检测算法**：

1. 把所有版本号 sort + unique
2. 数行数（`wc -l`）
3. 如果 > 1，说明有不同版本

**为什么不用更复杂的数据结构（如 set）**：

- bash 没有内置 set
- 管道 + sort 是"接近 set"的最简方式
- 对几十个文件完全够用，性能不是瓶颈

**`tr -d ' '` 的细节**：

- macOS 的 `wc -l` 输出可能带前导空格（`       3`）
- Linux 是 `3`
- 删空格保证数字字符串干净

### 3.6 漂移统计：`uniq -c | sort -rn`

```bash
printf '%s\n' "${versions[@]}" | sort | uniq -c | sort -rn | while read -r count ver; do
    echo "  $ver ($count files)"
done
```

**这是一个"4 个命令链"**：

1. `printf` 输出所有版本（每行一个）
2. `sort` 排序（让相同版本相邻）
3. `uniq -c` 计数（输出 `<count> <version>`）
4. `sort -rn` 按数字倒序（最常见的版本在前）

**输出**：

```
  2.0.1 (4 files)
  2.0.0 (1 files)
```

**让用户一眼看到"哪个版本是多数派"**——多数派通常是"正确的"，少数派需要修复。

### 3.7 审计：从"声明列表"反推"未声明文件"

```bash
while IFS= read -r match; do
    local match_file
    match_file=$(echo "$match" | cut -d: -f1)
    local rel_path="${match_file#$REPO_ROOT/}"

    local is_declared=0
    for dp in "${declared_paths[@]}"; do
        if [[ "$rel_path" == "$dp" ]]; then
            is_declared=1
            break
        fi
    done

    if [[ "$is_declared" -eq 0 ]]; then
        echo "  $match"
    fi
done < <(grep -rn "${exclude_args[@]}" -F "$current_version" "$REPO_ROOT" 2>/dev/null || true)
```

**核心算法**：

1. `grep -rn -F "$current_version"` 在仓库中找**所有**包含当前版本字符串的文件
2. 逐个检查：是否在"声明列表"中？
3. 不在 → 未声明 → 报告

**`grep -F` 的作用**：

- `-F` = fixed string（字面字符串匹配）
- 不解释正则元字符
- 版本号里的 `.` 不会被当作"任意字符"
- 更快、更准

**为什么用 `cut -d: -f1`**：

- `grep -n` 输出 `path:lineno:content`
- 第一个 `:` 前的就是文件路径
- 注意：路径里**不应该有 `:`**（在 Windows 上可能有，但 Git Bash 通常用 `/`）

**`${match_file#$REPO_ROOT/}` 字符串剥离**：

- `#` 表示"从前缀剥离"
- 把绝对路径转换为相对路径（与声明列表的格式一致）

**这是一个 O(n*m) 算法**（n = grep 结果数，m = 声明文件数）：

- 对几十到几百个文件完全够用
- 如果声明列表有几千个文件，可改为 hashmap 优化（awk/sort 外部工具）

### 3.8 `cmd_audit` 调用 `cmd_check || true`

```bash
cmd_audit() {
    cmd_check || true
    ...
}
```

**为什么不传播 `cmd_check` 的退出码**：

- `cmd_check` 在发现漂移时返回 1
- 但 `cmd_audit` 的语义是"扫描并报告"，不是"是否漂移"
- 即便有漂移，audit 也要继续扫描仓库找未声明文件
- `|| true` 显式吞掉非零退出码

**这是一个"语义正确的错误处理"**。

### 3.9 `cmd_bump` 后自动跑 audit

```bash
cmd_bump() {
    ...
    cmd_audit
}
```

**为什么 bump 后必跑 audit**：

- 防止"漏 bump 某个未声明文件"
- 自动化安全网：每次发布都验证
- 输出 "Review the above files — if they should be bumped, add them to .version-bump.json" 引导用户修复

**这是"流程自动化的最佳实践"**——把"发布后必须做"的事嵌入发布命令本身。

### 3.10 简化的 Semver 验证

```bash
if ! echo "$new_version" | grep -qE '^[0-9]+\.[0-9]+\.[0-9]+'; then
    echo "error: '$new_version' doesn't look like a version (expected X.Y.Z)" >&2
    exit 1
fi
```

**正则解析**：

- `^[0-9]+\.[0-9]+\.[0-9]+`：X.Y.Z 格式
- **不**支持 semver 的预发布版本（`1.0.0-rc1`）或构建元数据（`1.0.0+build.1`）
- **不**验证 major/minor/patch 数值（如 `999.0.0` 也会通过）

**这是一个"够用就好"的验证**：

- 严格 semver 验证会很长
- 大多数项目用 `1.0.0` 基础格式
- 简单正则过滤明显错误即可

### 3.11 `--audit` 中"确定当前版本"算法

```bash
current_version=$(
    while IFS=$'\t' read -r path field; do
        local fullpath="$REPO_ROOT/$path"
        [[ -f "$fullpath" ]] && read_json_field "$fullpath" "$field"
    done < <(declared_files) | sort | uniq -c | sort -rn | head -1 | awk '{print $2}'
)
```

**算法**：

1. 读取所有声明文件的当前版本
2. 排序 + 计数（与漂移检测相同的 sort | uniq -c）
3. 取出现次数最多的（`head -1`）
4. 用 `awk '{print $2}'` 提取版本号（第二列）

**为什么用"最常见"作为"当前版本"**：

- 通常所有声明文件应该同步到相同版本
- 如果有漂移，多数派就是"最新发布的版本"
- 这与 bump 命令的目标一致

**如果全部文件都不同**（极端情况）：

- `head -1` 选第一个
- 但 audit 会报告"未声明文件包含 X、Y、Z"——用户需要手动判断

---

## 4. 文件结构剖析

```
┌─ 头部 (行 1-20) ─────────────────────────────────────────┐
│ shebang + 文档注释 + set -euo pipefail                    │
│ 解析 SCRIPT_DIR / REPO_ROOT / CONFIG                      │
│ 验证 .version-bump.json 存在                              │
└──────────────────────────────────────────────────────────┘

┌─ 辅助函数 (行 22-52) ────────────────────────────────────┐
│ read_json_field  - 读取嵌套 JSON 字段                    │
│ write_json_field - 原子写入嵌套 JSON 字段                 │
│ declared_files   - 输出所有声明文件 (TSV 格式)           │
│ audit_excludes   - 输出审计排除模式                       │
└──────────────────────────────────────────────────────────┘

┌─ 命令实现 (行 54-194) ──────────────────────────────────┐
│ cmd_check  - 检查所有声明文件的当前版本 + 漂移检测       │
│ cmd_audit  - 调用 check + 扫描仓库找未声明引用           │
│ cmd_bump   - bump 所有声明文件 + 自动跑 audit            │
└──────────────────────────────────────────────────────────┘

┌─ 主入口 (行 196-221) ───────────────────────────────────┐
│ case "$1" 分发                                            │
│ --check / --audit / <version> / --help / --* / *         │
└──────────────────────────────────────────────────────────┘
```

**完整数据流**：

```
.version-bump.json
  ↓
  ┌─ declared_files() ────┐
  │ 输出 path<TAB>field    │
  └────────────────────────┘
       ↓
  ┌─ read_json_field() ──────┐
  │ 用 jq 读嵌套字段          │
  └──────────────────────────┘
       ↓
  ┌─ write_json_field() ─────┐
  │ 用 jq 原子写入 (tmp + mv) │
  └──────────────────────────┘
       ↓
  ┌─ grep -rn -F "$version" ──────┐
  │ 扫描仓库找未声明版本引用      │
  └──────────────────────────────┘
       ↓
  ┌─ 与 declared_paths 比对 ──────┐
  │ 报告未声明的引用              │
  └──────────────────────────────┘
```

---

## 5. 优点亮点

- **数据驱动**：版本号文件列表来自 `.version-bump.json`，**无需改脚本**就能扩展
- **三重职责**：`bump` / `check` / `audit` 一个脚本搞定
- **jq 的优雅使用**：嵌套 JSON 路径、原子写入、格式保留
- **漂移检测**：自动发现不一致
- **未声明审计**：扫描整个仓库找"漏网之鱼"
- **简化的 semver 验证**：基础格式检查
- **`< <(...)` 高级 IO**：让 while 在当前 shell 运行
- **`|| true` 语义正确**：audit 不传播 check 的退出码
- **`bump` 后自动跑 `audit`**：安全网
- **纯 bash + jq**：依赖极少
- **`set -euo pipefail`**：严格模式

---

## 6. 缺点风险局限

- **jq 必需**：没装 jq 会失败（但这是合理依赖）
- **路径转换仅支持数字索引**：`plugins.0.version` 有效，但 `plugins[abc].version` 无效（不常见）
- **审计是 O(n*m)**：对超大仓库可能慢
- **漂移检测不报告"哪个文件错了"**：只说"有几个版本不一致"
- **不写 changelog**：不修改 CHANGELOG.md 或 RELEASE-NOTES.md
- **不支持 semver 预发布/构建元数据**：`1.0.0-rc1` 会失败
- **不创建 git tag**：bump 后不自动 `git tag v1.0.0`
- **不推送/不发 release**：发布流程需要其他工具
- **`--audit` 排除模式依赖 .gitignore 风格**：用 `git ls-files --others --ignored`，但需要仓库有 `.gitignore`
- **不验证版本号大小**：`0.0.0` 到 `999.999.999` 都能通过
- **跨平台时区问题**：`--audit` 不输出时间戳，调试时无时间信息
- **`declare -a` 显式但非必需**：bash 4+ 之后更现代

---

## 7. 关键洞察

1. **"data over code" 是配置驱动的精髓**：版本号文件列表在 `.version-bump.json` 而非脚本中。这是"可演化配置"的典范

2. **三层职责（bump/check/audit）的设计哲学**：一个工具的多个 subcommand，每个职责清晰，符合 Unix "do one thing well" 但**组合**为完整工作流

3. **jq 是 JSON 处理的"金标准"**：原子写入、嵌套路径、格式保留——`bump-version.sh` 展示了 jq 在 bash 脚本中的最佳实践

4. **路径转换的 sed 链很精巧**：`\.([0-9]+)` → `[\1]` 是关键点。这个 3 sed 链解决"dot path to jq path"问题

5. **`< <(command)` 是 bash 高级 IO 的代表**：让 while 在当前 shell 运行，能修改外部变量——比 `command | while` 强大得多

6. **`|| true` 是"语义正确的错误吞咽"**：audit 不想被 check 的退出码中断，但 `set -e` 默认会因为 check 返回 1 而退出脚本

7. **`bump` 自动跑 `audit` 是"安全网自动化"**：把"发布后必做的事"嵌入发布命令——防呆设计

8. **漂移检测的"最常见"原则**：用 sort | uniq -c | sort -rn | head -1 找多数派，假设多数派是"正确的"——这是一种**统计意义上的真相判断**

9. **简化的 semver 验证体现"够用就好"**：不追求完整 semver 规范，只挡掉明显错误

10. **"路径前缀剥离" `${var#prefix}` 是 bash 的小技巧**：把绝对路径转相对路径，避免硬编码 `$REPO_ROOT/` 长度

11. **`grep -F` 是"字面字符串匹配"的正确选择**：版本号里的 `.` 不应该被解释为正则元字符

12. **`audit` 不传播 `check` 的退出码是"职责分离"**：audit 是"扫描报告"，不是"是否成功"

---

## 8. 关联文档

| 文档/文件 | 关系 |
|----------|------|
| [`.version-bump.json`](file://e:/WorkStation/MyProject/superpowers/.version-bump.json) | 本脚本读取的配置文件（包含 files 和 audit.exclude） |
| [`scripts/sync-to-codex-plugin.sh`](file://e:/WorkStation/MyProject/superpowers/scripts/sync-to-codex-plugin.sh) | 兄弟脚本（同步到 Codex 插件） |
| [`.claude-plugin/plugin.json`](file://e:/WorkStation/MyProject/superpowers/.claude-plugin/plugin.json) | 被管理的版本号文件之一 |
| [`.cursor-plugin/plugin.json`](file://e:/WorkStation/MyProject/superpowers/.cursor-plugin/plugin.json) | 被管理的版本号文件之一 |
| [`.codex-plugin/plugin.json`](file://e:/WorkStation/MyProject/superpowers/.codex-plugin/plugin.json) | 被管理的版本号文件之一 |
| [`.opencode/INSTALL.md`](file://e:/WorkStation/MyProject/superpowers/.opencode/INSTALL.md) | OpenCode 安装文档 |
| `gemini-extension.json` | Gemini 平台清单（被管理） |
| `package.json` | Node.js 清单（被管理） |
| `jq` 官方文档 | 外部参考（路径语法） |
| `RELEASE-NOTES.md` | 发布说明（本脚本不写，但应保持同步） |

---

## 9. 状态评价

| 维度 | 评分 | 说明 |
|------|------|------|
| 完整性 | 9/10 | 覆盖 bump/check/audit 三职责 |
| 正确性 | 9/10 | 严格模式 + 原子写入 + 错误处理 |
| 可维护性 | 9/10 | 函数拆分清晰，配置驱动 |
| 可读性 | 9/10 | 注释充分，命令名清晰 |
| 性能 | 8/10 | 对中等仓库够用，超大仓库 O(n*m) 是瓶颈 |
| 健壮性 | 9/10 | 多种错误处理，审计安全网 |
| 可移植性 | 9/10 | 仅依赖 jq（Unix/Mac/Windows Git Bash 都有） |
| 文档化 | 8/10 | 头部有 Usage 注释，但缺少 --help 长文 |
| 教育价值 | 10/10 | bash + jq 最佳实践范例 |

**总体评价**: 9/10 - 一个精心设计的、生产就绪的版本管理工具。

**改进优先级**:

1. **中**: 漂移检测报告"哪个文件是错的版本"（当前只说"有几个版本"）
2. **中**: 支持 semver 预发布版本（`1.0.0-rc1`）
3. **中**: bump 后自动 `git add .` + `git tag v1.0.0`（可选行为）
4. **低**: 性能优化（声明列表改为 hashmap 查找）
5. **低**: 添加 `--rollback` 子命令（撤销上一次 bump）
6. **低**: 与 RELEASE-NOTES.md 联动（提醒用户更新）

---

## 10. 给读者的启示

- **"data over code" 适用于所有"列表类"配置**：版本号文件、忽略文件、依赖文件——把"是什么"从代码抽离到配置

- **jq 是 bash 处理 JSON 的"必备武器"**：本脚本展示了 jq 在嵌套路径、原子写入、格式保留上的最佳实践

- **三层 subcommand 是 Unix 哲学的扩展**：bump/check/audit 三个职责独立，组合为完整工作流

- **`bump` 自动跑 `audit` 是"防呆设计"**：把"必须做"嵌入"应该做"

- **漂移检测是"配置同步"的通用模式**：版本号漂移、配置漂移、状态漂移——都可以用"sort | uniq -c" 检测

- **`< <(command)` 比 `command | while` 更强大**：让循环在当前 shell 运行，能修改外部变量

- **`|| true` 是"语义正确的错误吞咽"**：当上层不想被下层退出码影响时

- **路径转换的 sed 链体现"小而美"**：3 个 sed 调用解决嵌套 JSON 路径问题——比写一个 50 行的 Python 解析器更优雅

- **`grep -F` 比 `grep` 更准**：字面字符串匹配，避免正则元字符误伤

- **"最常见 = 真相" 是统计意义上的判断**：在配置漂移场景下，多数派通常是"正确"版本

- **原子写入是 bash 写文件的"金标准"**：tmp + mv 保证失败时原文件不损坏

- **简化的验证比过度验证友好**：`^[0-9]+\.[0-9]+\.[0-9]+` 挡掉明显错误，但不追求完整规范

---

## 11. 后续工作

### 11.1 短期改进（2-3 小时）

- [ ] 漂移检测增强：报告"哪个文件是错的版本"（不仅说有差异）
- [ ] 支持 semver 预发布版本：`1.0.0-rc1`、`1.0.0+build.1`
- [ ] `--dry-run` 模式：显示将要修改的内容但不真改
- [ ] `--rollback` 模式：从 `.git` 历史恢复上一个版本

### 11.2 中期改进（半天）

- [ ] bump 后可选 `git add .` + `git tag v1.0.0`
- [ ] 与 RELEASE-NOTES.md 联动（提醒更新）
- [ ] 性能优化：声明列表改为 hashmap（awk/sort）
- [ ] 添加 `bump-version.sh init`：扫描仓库自动生成 `.version-bump.json`
- [ ] 添加单元测试（每种模式独立验证）

### 11.3 长期演进

- [ ] 探索 `bump-version.sh validate`：在 CI 中跑 `--check`，失败时阻止 merge
- [ ] 跨仓库协调：支持 monorepo 多个 .version-bump.json
- [ ] 集成 changelog 工具（如 `git-cliff`）
- [ ] 发布 Slack/Discord 通知

### 11.4 风险预警

- ⚠️ jq 版本兼容性：旧版 jq（如 1.4）不支持某些路径语法
- ⚠️ `.version-bump.json` 配置错误会导致脚本静默失败（声明不存在的文件）
- ⚠️ macOS 默认 BSD `grep` 与 Linux GNU `grep` 行为差异
- ⚠️ 路径中含 `:` 字符在 Windows 上可能导致 `cut -d:` 出错
- ⚠️ 大仓库的 audit 性能（O(n*m)）

### 11.5 教育价值挖掘

- [ ] 在 `docs/` 下补充 `docs/version-management.md` 详细讲解
- [ ] 录制视频讲解 jq + bash 最佳实践
- [ ] 写一篇博客文章："为什么 data over code 是配置管理的未来"
- [ ] 添加 `--explain` 模式：显示每个步骤在做什么（教学用）

### 11.6 监控/可观测

- [ ] 输出每次 bump 的耗时、修改文件数
- [ ] 结构化日志：可被 CI 收集
- [ ] `--json` 输出模式：供其他工具消费

### 11.7 跨脚本协同

- [ ] 与 `sync-to-codex-plugin.sh` 协同：bump 后自动触发同步
- [ ] 与 CI 集成：merge 前自动跑 `--check`

---
