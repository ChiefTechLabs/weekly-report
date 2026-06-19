---
name: weekly-report
description: 生成 OpenCode 工作周报。当用户要求写周报、工作总结、weekly report、本周做了什么、上周总结、工作汇报，或任何形式的周期性工作回顾时使用。自动从 OpenCode SQLite 数据库提取 session 数据、扫描计划文件进度、检查活跃计划状态，按项目维度汇总生成结构化 markdown 报告。适配 Linux 和 Windows 双平台。
---

# Weekly Report Generator

为 OpenCode 用户生成结构化工作周报。

## 核心原则

OpenCode 内置的 `session_list` 工具默认只返回当前工作目录的 session，会漏掉大量其他项目的 session。**周报必须直接从 SQLite 数据库读取，这是不可替代的数据源**。

报告输出三部分：
1. 工作周报（按项目维度汇总已完成工作）
2. 未完成事项（所有开放计划的进度与预计完成时间）
3. 协调与帮助（跨项目依赖、资源需求、优先级建议、风险）

---

## 执行流程

### 第一步：平台检测与路径解析

先确定运行平台，再解析关键路径。

**Linux:**
```bash
DB_PATH="$HOME/.local/share/opencode/opencode.db"
HOME_DIR="$HOME"
```
查找工具：`find`，Shell：`bash`

**Windows:**
```powershell
$DB_PATH = "$env:LOCALAPPDATA\opencode\opencode.db"
# 如果 LOCALAPPDATA 下不存在，尝试 APPDATA
if (-not (Test-Path $DB_PATH)) {
    $DB_PATH = "$env:APPDATA\opencode\opencode.db"
}
$HOME_DIR = $env:USERPROFILE
```
查找工具：`Get-ChildItem -Recurse`，Shell：`PowerShell`

验证 DB 存在：
```bash
# Linux
test -f "$DB_PATH" && echo "ok" || echo "missing"
# Windows PowerShell
Test-Path $DB_PATH
```

### 第二步：确定报告日期范围

从用户输入中提取日期范围。如果没有指定：
- "上周" → 上周一 00:00 到上周日 23:59（以当天为基准计算）
- "本周" → 本周一 00:00 到今天
- 默认取上周

将日期转为 SQLite datetime 可比较的格式 `YYYY-MM-DD`。

### 第三步：查询 Session 数据（核心步骤）

这是最关键的一步。必须 JOIN 三张表来获取完整数据：

```sql
SELECT 
  COALESCE(pd.directory, p.worktree, '/') as project_dir,
  COUNT(DISTINCT s.id) as sessions,
  SUM(s.tokens_input) as total_tokens,
  ROUND(SUM(s.cost), 2) as total_cost,
  MIN(date(datetime(s.time_created/1000, 'unixepoch'))) as first_day,
  MAX(date(datetime(s.time_created/1000, 'unixepoch'))) as last_day
FROM session s
JOIN project p ON s.project_id = p.id
LEFT JOIN project_directory pd ON p.id = pd.project_id
WHERE date(datetime(s.time_created/1000, 'unixepoch')) >= '<start_date>'
  AND date(datetime(s.time_created/1000, 'unixepoch')) < '<end_date>'
GROUP BY project_dir
ORDER BY total_tokens DESC
```

这一步拿到按项目目录分组的 session 统计，形成项目清单。

然后对每个项目拉取详细 session 列表。只关注 `agent = 'Sisyphus - ultraworker'` 的主 session（它们代表用户的实际对话主题），过滤掉 `New session` 等空标题：

```sql
SELECT 
  date(datetime(s.time_created/1000, 'unixepoch')) as day,
  s.title,
  s.agent,
  s.tokens_input,
  ROUND(s.cost, 4) as cost
FROM session s
JOIN project p ON s.project_id = p.id
WHERE p.worktree = '<project_worktree>'
  AND date(datetime(s.time_created/1000, 'unixepoch')) >= '<start_date>'
  AND date(datetime(s.time_created/1000, 'unixepoch')) < '<end_date>'
  AND s.agent = 'Sisyphus - ultraworker'
  AND s.title NOT LIKE 'New session%'
ORDER BY s.time_created
```

**注意**：时间戳字段是毫秒级 Unix epoch（`time_created/1000`），不是秒级。

### 第四步：扫描计划文件

搜索所有 `.omo/plans/` 下的 `.md` 文件，统计每个计划的完成情况：

```bash
# Linux
find "$HOME_DIR" -path "*/.omo/plans/*.md" 2>/dev/null | while read f; do
  pending=$(rg -c '^\- \[ \]' "$f" 2>/dev/null || echo 0)
  completed=$(rg -c '^\- \[[xX]\]' "$f" 2>/dev/null || echo 0)
  echo "$f|$pending|$completed"
done

# Windows PowerShell
Get-ChildItem -Path "$HOME_DIR" -Recurse -Filter "*.md" -ErrorAction SilentlyContinue |
  Where-Object { $_.FullName -match '\.omo[\\/]plans[\\/]' } |
  ForEach-Object {
    $content = Get-Content $_.FullName -Raw
    $pending = ([regex]::Matches($content, '^- \[ \]')).Count
    $completed = ([regex]::Matches($content, '^- \[[xX]\]')).Count
    "$($_.FullName)|$pending|$completed"
  }
```

只关注 `pending > 0` 的计划（有未完成事项）。按项目目录分组，关联到第二步的项目清单。

### 第五步：检查活跃计划

读取各项目的 `boulder.json`，获取当前活跃计划：

```bash
# Linux
for f in $(find "$HOME_DIR" -name "boulder.json" -path "*/.omo/*" 2>/dev/null); do
  python3 -c "import json; d=json.load(open('$f')); print(f'{d.get(\"plan_name\",\"?\")}|{d.get(\"started_at\",\"?\")}')" 2>/dev/null
done

# Windows PowerShell
Get-ChildItem -Path "$HOME_DIR" -Recurse -Filter "boulder.json" -ErrorAction SilentlyContinue |
  Where-Object { $_.FullName -match '\.omo[\\/]boulder\.json$' } |
  ForEach-Object {
    $json = Get-Content $_.FullName -Raw | ConvertFrom-Json
    "$($json.plan_name)|$($json.started_at)"
  }
```

活跃计划标记 `[进行中]`，优先排在未完成事项列表前面。

### 第六步：按项目归类汇总

将第四步和第五步的未完成计划按所属项目（从文件路径提取项目目录）归类。

识别项目名称的启发式规则：
- 路径末尾的目录名（如 `/geo` → Geo 内容平台）
- 排除 `.omo`、`.ws`、`vendor` 等前缀
- 如果多个目录属于同一项目（如 `piper_ros` 和 `OMRobot` 可能同属机器人项目），根据业务语义合并

### 第七步：生成报告文件

在用户指定的输出目录下创建三个 markdown 文件。默认路径：
- Linux: `$HOME/Documents/week-report/<YYYY-MM-DD_YYYY-MM-DD>/`
- Windows: `$env:USERPROFILE\Documents\week-report\<YYYY-MM-DD_YYYY-MM-DD>\`

**文件 1：`01-工作周报.md`**

```markdown
# 工作周报（<开始日期> — <结束日期>）

> 统计口径：<N> sessions / ~<M>M tokens / ~$<cost>

## 一、<项目名>
<sessions> sessions / <tokens> tokens / $<cost> | <日期范围>

**已完成**
- <具体工作项>（每项一行，用动词开头）

**执行方式**：<如有特殊流程，简要说明>

## 二、...

---

## 按项目汇总
| 项目 | Sessions | Tokens | 成本 |
|------|----------|--------|------|
| ... | ... | ... | ... |
| **合计** | **...** | **...** | **...** |
```

**文件 2：`02-未完成事项.md`**

每个项目一个表格，活跃计划排在前面加 `▶` 标记：

```markdown
## <项目名>
| 事项 | 计划文件 | 进度 | 预计完成 | 说明 |
|------|---------|------|---------|------|
| ▶ <事项> | `<文件名>.md` | <done>/<total> | **<日期>** | <说明> |
| <事项> | ... | ... | <日期> | ... |

---

## 本周预期完成清单
| 日期 | 事项 | 项目 |
|------|------|------|
| ... | ... | ... |
```

预计完成时间根据进度和计划规模估算：
- 进度 > 80% 且 pending < 10 → 1-2 天
- 进度 50-80% → 3-5 天
- 进度 < 50% 且 total > 80 → 1-2 周
- 进度 0% → 标注「待启动」

**文件 3：`03-协调与帮助.md`**

```markdown
## 一、跨项目依赖
| 依赖关系 | 阻塞方 | 被阻塞方 | 影响范围 |
|---------|--------|---------|---------|
| ... | ...（预计完成日期） | ...（计划文件） | ... |

## 二、硬件/环境资源
| 资源 | 需求方 | 用途 | 状态 |
|------|--------|------|------|
| ... | ... | ... | 未确认 / 已预约 |

## 三、需决策的优先级
（按项目列出推荐推进顺序，标注需确认的决策点）

## 四、对外协调事项
| 事项 | 协调对象 | 内容 |
|------|---------|------|

## 五、风险提示
（项目积压、硬件依赖、范围不明等风险）
```

### 第八步：确认与说明

生成后向用户展示：
- 统计总览（项目数、session 数、token 总量、成本）
- 输出路径
- 提示用户 `session_list` 工具默认限制（解释为什么直接查 DB）
- 如有被排除的项目（用户指定的），明确告知

---

## 输出质量标准

- **具体而非泛泛**：不要写「修复了一些 bug」，写「修复登录后返回登录页的循环跳转」
- **量化**：每个项目标注 session 数、token 数、成本
- **诚实**：未完成事项如实标注进度，不要美化
- **结构清晰**：三个文件各司其职，不要混在一起
- **预计时间基于证据**：参考计划规模（pending 数量）和用户历史推进速度，不拍脑袋

## 常见陷阱

- **不要用 `session_list` 工具**：它默认只返回当前工作目录的 session，会漏掉其他项目。始终直接查 DB。
- **不要用 `session_read` 逐个读 session**：太慢且消耗大量上下文。从 title 字段就能获取足够的信息。
- **时间戳是毫秒**：SQLite 中 `time_created` 除以 1000 才是 Unix 秒。
- **过滤空 session**：大量 `New session` 标题的 session 是 agent 自动创建的占位 session，应排除。
- **计划文件只统计顶层 checkbox**：`- [ ]` 和 `- [x]`（不区分大小写），不要统计嵌套列表或代码块中的。
- **Windows 路径反斜杠**：在 SQL 查询和输出路径中统一用正斜杠 `/` 或双反斜杠 `\\`。

## 依赖性

- **sqlite3**：Linux 通常预装，Windows 可能需要从 https://sqlite.org/download.html 下载或通过 `choco install sqlite` / `winget install SQLite.SQLite` 安装
- **Python 3**：用于读取 boulder.json（Linux 和 Windows 均预装或易于安装）
- **rg (ripgrep)**：用于高效统计 checkbox（可选，fallback 到 grep/findstr）
