# Codex 版 Superpowers

透過原生技能探索使用 Superpowers 與 OpenAI Codex 的指南。

## 快速安裝

告訴 Codex：

```
Fetch and follow instructions from https://raw.githubusercontent.com/obra/superpowers/refs/heads/main/.codex/INSTALL.md
```

## 手動安裝

### 先決條件

- OpenAI Codex CLI
- Git

### 步驟

1. 複製儲存庫：
   ```bash
   git clone https://github.com/obra/superpowers.git ~/.codex/superpowers
   ```

2. 建立技能符號連結：
   ```bash
   mkdir -p ~/.agents/skills
   ln -s ~/.codex/superpowers/skills ~/.agents/skills/superpowers
   ```

3. 重新啟動 Codex。

4. **子代理技能**（可選）：像 `dispatching-parallel-agents` 與 `subagent-driven-development` 這類技能需要 Codex 的 collab 功能。請在 Codex 設定中加入：
   ```toml
   [features]
   collab = true
   ```

### Windows

使用 junction 取代符號連結（不需要開發者模式）：

```powershell
New-Item -ItemType Directory -Force -Path "$env:USERPROFILE\.agents\skills"
cmd /c mklink /J "$env:USERPROFILE\.agents\skills\superpowers" "$env:USERPROFILE\.codex\superpowers\skills"
```

## 如何運作

Codex 具備原生技能探索能力 — 它在啟動時掃描 `~/.agents/skills/`，解析 SKILL.md 的 frontmatter，並按需載入技能。Superpowers 的技能會透過單一符號連結被看見：

```
~/.agents/skills/superpowers/ → ~/.codex/superpowers/skills/
```

`using-superpowers` 技能會自動被發現並強制技能使用紀律 — 不需要額外設定。

## 使用方式

技能會自動被發現。Codex 在以下情況會啟用它們：
- 你提到技能名稱（例如「use brainstorming」）
- 任務符合技能的描述
- `using-superpowers` 技能指示 Codex 使用某個技能

### 個人技能

在 `~/.agents/skills/` 中建立你自己的技能：

```bash
mkdir -p ~/.agents/skills/my-skill
```

建立 `~/.agents/skills/my-skill/SKILL.md`：

```markdown
---
name: my-skill
description: Use when [condition] - [what it does]
---

# My Skill

[Your skill content here]
```

`description` 欄位是 Codex 用來判斷何時自動啟用技能的依據 — 請把它寫成清楚的觸發條件。

## 更新

```bash
cd ~/.codex/superpowers && git pull
```

技能會透過符號連結即時更新。

## 解除安裝

```bash
rm ~/.agents/skills/superpowers
```

**Windows（PowerShell）：**
```powershell
Remove-Item "$env:USERPROFILE\.agents\skills\superpowers"
```

如有需要，可刪除複本：`rm -rf ~/.codex/superpowers`（Windows：`Remove-Item -Recurse -Force "$env:USERPROFILE\.codex\superpowers"`）。

## 疑難排解

### 技能沒有出現

1. 驗證符號連結：`ls -la ~/.agents/skills/superpowers`
2. 檢查技能是否存在：`ls ~/.codex/superpowers/skills`
3. 重新啟動 Codex — 技能是在啟動時被發現的

### Windows junction 問題

Junction 通常不需要特殊權限即可運作。若建立失敗，請嘗試以系統管理員身分執行 PowerShell。

## 取得協助

- 回報問題：https://github.com/obra/superpowers/issues
- 主要文件：https://github.com/obra/superpowers
