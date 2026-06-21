# Installing the weekly-report Skill

This document is written for LLM agents. Follow these steps to install the skill on the user's machine.

## What You Need

- OpenCode installed and configured
- Git (to clone) or `curl`/`wget` (to download)
- The skill is a single file: `SKILL.md`

## Platform-Agnostic Installation

### Step 1 — Locate the skills directory

OpenCode skills live under `~/.agents/skills/`. Create it if it doesn't exist:

**Linux / macOS:**
```bash
mkdir -p ~/.agents/skills/weekly-report
```

**Windows (PowerShell):**
```powershell
New-Item -ItemType Directory -Force -Path "$env:USERPROFILE\.agents\skills\weekly-report"
```

### Step 2 — Download SKILL.md

Choose one of these methods:

**Option A — Clone the repo (recommended, easy to update):**
```bash
git clone https://github.com/ChiefTechLabs/weekly-report.git ~/.agents/skills/weekly-report
```

**Option B — Direct download (no git needed):**

Linux / macOS:
```bash
curl -fsSL https://raw.githubusercontent.com/ChiefTechLabs/weekly-report/main/SKILL.md -o ~/.agents/skills/weekly-report/SKILL.md
```

Windows PowerShell:
```powershell
Invoke-WebRequest -Uri "https://raw.githubusercontent.com/ChiefTechLabs/weekly-report/main/SKILL.md" -OutFile "$env:USERPROFILE\.agents\skills\weekly-report\SKILL.md"
```

### Step 3 — Verify installation

Check the file exists and is readable:

**Linux / macOS:**
```bash
test -f ~/.agents/skills/weekly-report/SKILL.md && echo "Installed" || echo "Failed"
head -3 ~/.agents/skills/weekly-report/SKILL.md
```

**Windows PowerShell:**
```powershell
if (Test-Path "$env:USERPROFILE\.agents\skills\weekly-report\SKILL.md") {
    Write-Host "Installed"
    Get-Content "$env:USERPROFILE\.agents\skills\weekly-report\SKILL.md" -TotalCount 3
} else {
    Write-Host "Failed"
}
```

Expected output starts with:
```yaml
---
name: weekly-report
```

### Step 4 — Confirm skill is loaded

Restart OpenCode. The skill triggers when the user says any of:
- "写周报" / "上周总结" / "工作汇报"
- "weekly report" / "summarize last week"

You can also invoke it explicitly: `/weekly-report`

## Dependencies

The skill requires:

| Tool | Used For | Linux | Windows |
|------|----------|-------|---------|
| `sqlite3` | Query OpenCode session database | Pre-installed | Install via `winget install SQLite.SQLite` |
| `python3` | Parse boulder.json | Pre-installed | Install from python.org |
| `rg` (ripgrep) | Count plan checkboxes | `apt install ripgrep` | `winget install BurntSushi.ripgrep.MSVC` |

If any tool is missing, the skill falls back to alternatives (`grep`, `findstr`, `python` one-liners).

## Update

If installed via git clone:
```bash
cd ~/.agents/skills/weekly-report && git pull
```

If installed via direct download, re-run the curl / Invoke-WebRequest command.

## Uninstall

```bash
rm -rf ~/.agents/skills/weekly-report
```
