# OpenCode 版 Superpowers

使用 Superpowers 搭配 [OpenCode.ai](https://opencode.ai) 的完整指南。

## 快速安裝

告訴 OpenCode：

```
Clone https://github.com/obra/superpowers to ~/.config/opencode/superpowers, then create directory ~/.config/opencode/plugins, then symlink ~/.config/opencode/superpowers/.opencode/plugins/superpowers.js to ~/.config/opencode/plugins/superpowers.js, then symlink ~/.config/opencode/superpowers/skills to ~/.config/opencode/skills/superpowers, then restart opencode.
```

## 手動安裝

### 先決條件

- 已安裝 [OpenCode.ai](https://opencode.ai)
- 已安裝 Git

### macOS / Linux

```bash
# 1. Install Superpowers (or update existing)
if [ -d ~/.config/opencode/superpowers ]; then
  cd ~/.config/opencode/superpowers && git pull
else
  git clone https://github.com/obra/superpowers.git ~/.config/opencode/superpowers
fi

# 2. Create directories
mkdir -p ~/.config/opencode/plugins ~/.config/opencode/skills

# 3. Remove old symlinks/directories if they exist
rm -f ~/.config/opencode/plugins/superpowers.js
rm -rf ~/.config/opencode/skills/superpowers

# 4. Create symlinks
ln -s ~/.config/opencode/superpowers/.opencode/plugins/superpowers.js ~/.config/opencode/plugins/superpowers.js
ln -s ~/.config/opencode/superpowers/skills ~/.config/opencode/skills/superpowers

# 5. Restart OpenCode
```

#### 驗證安裝

```bash
ls -l ~/.config/opencode/plugins/superpowers.js
ls -l ~/.config/opencode/skills/superpowers
```

兩者都應顯示指向 superpowers 目錄的符號連結。

### Windows

**先決條件：**
- 已安裝 Git
- 啟用**開發者模式**或具備**系統管理員權限**
  - Windows 10：設定 → 更新與安全性 → 針對開發人員
  - Windows 11：設定 → 系統 → 針對開發人員

請選擇你的 Shell：[Command Prompt](#command-prompt) | [PowerShell](#powershell) | [Git Bash](#git-bash)

#### Command Prompt

以系統管理員身分執行，或啟用開發者模式：

```cmd
:: 1. Install Superpowers
git clone https://github.com/obra/superpowers.git "%USERPROFILE%\.config\opencode\superpowers"

:: 2. Create directories
mkdir "%USERPROFILE%\.config\opencode\plugins" 2>nul
mkdir "%USERPROFILE%\.config\opencode\skills" 2>nul

:: 3. Remove existing links (safe for reinstalls)
del "%USERPROFILE%\.config\opencode\plugins\superpowers.js" 2>nul
rmdir "%USERPROFILE%\.config\opencode\skills\superpowers" 2>nul

:: 4. Create plugin symlink (requires Developer Mode or Admin)
mklink "%USERPROFILE%\.config\opencode\plugins\superpowers.js" "%USERPROFILE%\.config\opencode\superpowers\.opencode\plugins\superpowers.js"

:: 5. Create skills junction (works without special privileges)
mklink /J "%USERPROFILE%\.config\opencode\skills\superpowers" "%USERPROFILE%\.config\opencode\superpowers\skills"

:: 6. Restart OpenCode
```

#### PowerShell

以系統管理員身分執行，或啟用開發者模式：

```powershell
# 1. Install Superpowers
git clone https://github.com/obra/superpowers.git "$env:USERPROFILE\.config\opencode\superpowers"

# 2. Create directories
New-Item -ItemType Directory -Force -Path "$env:USERPROFILE\.config\opencode\plugins"
New-Item -ItemType Directory -Force -Path "$env:USERPROFILE\.config\opencode\skills"

# 3. Remove existing links (safe for reinstalls)
Remove-Item "$env:USERPROFILE\.config\opencode\plugins\superpowers.js" -Force -ErrorAction SilentlyContinue
Remove-Item "$env:USERPROFILE\.config\opencode\skills\superpowers" -Force -ErrorAction SilentlyContinue

# 4. Create plugin symlink (requires Developer Mode or Admin)
New-Item -ItemType SymbolicLink -Path "$env:USERPROFILE\.config\opencode\plugins\superpowers.js" -Target "$env:USERPROFILE\.config\opencode\superpowers\.opencode\plugins\superpowers.js"

# 5. Create skills junction (works without special privileges)
New-Item -ItemType Junction -Path "$env:USERPROFILE\.config\opencode\skills\superpowers" -Target "$env:USERPROFILE\.config\opencode\superpowers\skills"

# 6. Restart OpenCode
```

#### Git Bash

注意：Git Bash 內建的 `ln` 指令會複製檔案而不是建立符號連結。請改用 `cmd //c mklink`（`//c` 是 Git Bash 對應 `/c` 的語法）。

```bash
# 1. Install Superpowers
git clone https://github.com/obra/superpowers.git ~/.config/opencode/superpowers

# 2. Create directories
mkdir -p ~/.config/opencode/plugins ~/.config/opencode/skills

# 3. Remove existing links (safe for reinstalls)
rm -f ~/.config/opencode/plugins/superpowers.js 2>/dev/null
rm -rf ~/.config/opencode/skills/superpowers 2>/dev/null

# 4. Create plugin symlink (requires Developer Mode or Admin)
cmd //c "mklink \"$(cygpath -w ~/.config/opencode/plugins/superpowers.js)\" \"$(cygpath -w ~/.config/opencode/superpowers/.opencode/plugins/superpowers.js)\""

# 5. Create skills junction (works without special privileges)
cmd //c "mklink /J \"$(cygpath -w ~/.config/opencode/skills/superpowers)\" \"$(cygpath -w ~/.config/opencode/superpowers/skills)\""

# 6. Restart OpenCode
```

#### WSL 使用者

如果在 WSL 中執行 OpenCode，請改用 [macOS / Linux](#macos--linux) 的指示。

#### 驗證安裝

**Command Prompt：**
```cmd
dir /AL "%USERPROFILE%\.config\opencode\plugins"
dir /AL "%USERPROFILE%\.config\opencode\skills"
```

**PowerShell：**
```powershell
Get-ChildItem "$env:USERPROFILE\.config\opencode\plugins" | Where-Object { $_.LinkType }
Get-ChildItem "$env:USERPROFILE\.config\opencode\skills" | Where-Object { $_.LinkType }
```

在輸出中尋找 `<SYMLINK>` 或 `<JUNCTION>`。

#### Windows 疑難排解

**「You do not have sufficient privilege」錯誤：**
- 在 Windows 設定中啟用開發者模式，或
- 以滑鼠右鍵點選終端機 →「以系統管理員身分執行」

**「Cannot create a file when that file already exists」：**
- 先執行移除指令（步驟 3），再重試

**git clone 後符號連結無法運作：**
- 執行 `git config --global core.symlinks true` 並重新 clone

## 使用方式

### 尋找技能

使用 OpenCode 的原生 `skill` 工具列出所有可用技能：

```
use skill tool to list skills
```

### 載入技能

使用 OpenCode 的原生 `skill` 工具載入特定技能：

```
use skill tool to load superpowers/brainstorming
```

### 個人技能

在 `~/.config/opencode/skills/` 中建立你自己的技能：

```bash
mkdir -p ~/.config/opencode/skills/my-skill
```

建立 `~/.config/opencode/skills/my-skill/SKILL.md`：

```markdown
---
name: my-skill
description: Use when [condition] - [what it does]
---

# My Skill

[Your skill content here]
```

### 專案技能

在你的 OpenCode 專案中建立專案專用技能：

```bash
# In your OpenCode project
mkdir -p .opencode/skills/my-project-skill
```

建立 `.opencode/skills/my-project-skill/SKILL.md`：

```markdown
---
name: my-project-skill
description: Use when [condition] - [what it does]
---

# My Project Skill

[Your skill content here]
```

## 技能位置

OpenCode 會從以下位置探索技能：

1. **專案技能**（`.opencode/skills/`）- 最高優先序
2. **個人技能**（`~/.config/opencode/skills/`）
3. **Superpowers 技能**（`~/.config/opencode/skills/superpowers/`）- 透過符號連結

## 功能

### 自動上下文注入

外掛會透過 `experimental.chat.system.transform` hook 自動注入 superpowers 上下文。這會在每次請求時把「using-superpowers」技能內容加入系統提示。

### 原生技能整合

Superpowers 使用 OpenCode 的原生 `skill` 工具進行技能探索與載入。技能會被符號連結到 `~/.config/opencode/skills/superpowers/`，因此會與你的個人與專案技能並列顯示。

### 工具對應

為 Claude Code 撰寫的技能會自動適配 OpenCode。啟動流程提供以下對應指示：

- `TodoWrite` → `todowrite`
- 帶子代理的 `Task` → OpenCode 的 `@mention` 系統
- `Skill` 工具 → OpenCode 的原生 `skill` 工具
- 檔案操作 → OpenCode 原生工具

## 架構

### 外掛結構

**位置：**`~/.config/opencode/superpowers/.opencode/plugins/superpowers.js`

**元件：**
- 用於啟動注入的 `experimental.chat.system.transform` hook
- 讀取並注入「using-superpowers」技能內容

### 技能

**位置：**`~/.config/opencode/skills/superpowers/`（符號連結至 `~/.config/opencode/superpowers/skills/`）

技能由 OpenCode 的原生技能系統探索。每個技能都有包含 YAML frontmatter 的 `SKILL.md` 檔案。

## 更新

```bash
cd ~/.config/opencode/superpowers
git pull
```

重新啟動 OpenCode 以載入更新。

## 疑難排解

### 外掛無法載入

1. 確認外掛存在：`ls ~/.config/opencode/superpowers/.opencode/plugins/superpowers.js`
2. 檢查符號連結/junction：`ls -l ~/.config/opencode/plugins/`（macOS/Linux）或 `dir /AL %USERPROFILE%\.config\opencode\plugins`（Windows）
3. 檢查 OpenCode 日誌：`opencode run "test" --print-logs --log-level DEBUG`
4. 在日誌中尋找外掛載入訊息

### 找不到技能

1. 驗證技能符號連結：`ls -l ~/.config/opencode/skills/superpowers`（應指向 superpowers/skills/）
2. 使用 OpenCode 的 `skill` 工具列出可用技能
3. 檢查技能結構：每個技能都需要一個含有效 frontmatter 的 `SKILL.md` 檔案

### Windows：找不到模組錯誤

若在 Windows 看到 `Cannot find module`：
- **原因：**Git Bash 的 `ln -sf` 會複製檔案而不是建立符號連結
- **解法：**改用 `mklink /J` 目錄 junction（參考 Windows 安裝步驟）

### 啟動注入未出現

1. 確認 using-superpowers 技能存在：`ls ~/.config/opencode/superpowers/skills/using-superpowers/SKILL.md`
2. 確認 OpenCode 版本支援 `experimental.chat.system.transform` hook
3. 外掛變更後重新啟動 OpenCode

## 取得協助

- 回報問題：https://github.com/obra/superpowers/issues
- 主要文件：https://github.com/obra/superpowers
- OpenCode 文件：https://opencode.ai/docs/

## 測試

驗證你的安裝：

```bash
# Check plugin loads
opencode run --print-logs "hello" 2>&1 | grep -i superpowers

# Check skills are discoverable
opencode run "use skill tool to list all skills" 2>&1 | grep -i superpowers

# Check bootstrap injection
opencode run "what superpowers do you have?"
```

代理應該會提到擁有 superpowers，並能列出 `superpowers/` 的技能。
