---
name: using-git-worktrees
description: 在開始需要與目前工作區隔離的功能開發，或在執行實作計畫前使用 — 以智慧目錄選擇與安全驗證建立隔離的 git worktree
---

# 使用 Git Worktrees

## 概覽

Git worktree 會建立共享同一個 repo 的隔離工作區，讓你能同時在多個分支上工作而不用切換。

**核心原則：**系統化目錄選擇 + 安全驗證 = 可靠隔離。

**一開始要宣告：**「我正在使用 using-git-worktrees 技能建立隔離工作區。」

## 目錄選擇流程

依下列優先順序：

### 1. 檢查既有目錄

```bash
# Check in priority order
ls -d .worktrees 2>/dev/null     # Preferred (hidden)
ls -d worktrees 2>/dev/null      # Alternative
```

**若存在：**使用該目錄。若兩者都存在，優先使用 `.worktrees`。

### 2. 檢查 CLAUDE.md

```bash
grep -i "worktree.*director" CLAUDE.md 2>/dev/null
```

**若指定偏好：**不需詢問，直接使用。

### 3. 詢問使用者

若沒有目錄，也沒有 CLAUDE.md 偏好：

```
No worktree directory found. Where should I create worktrees?

1. .worktrees/ (project-local, hidden)
2. ~/.config/superpowers/worktrees/<project-name>/ (global location)

Which would you prefer?
```

## 安全驗證

### 專案內目錄（.worktrees 或 worktrees）

**建立 worktree 前必須確認目錄已被忽略：**

```bash
# Check if directory is ignored (respects local, global, and system gitignore)
git check-ignore -q .worktrees 2>/dev/null || git check-ignore -q worktrees 2>/dev/null
```

**若未被忽略：**

依 Jesse 的規則「壞掉的東西立刻修」：
1. 將適當行加入 .gitignore
2. 提交變更
3. 再建立 worktree

**為何重要：**避免意外把 worktree 內容提交到 repo。

### 全域目錄（~/.config/superpowers/worktrees）

不需要 .gitignore 驗證 — 完全在專案外。

## 建立步驟

### 1. 取得專案名稱

```bash
project=$(basename "$(git rev-parse --show-toplevel)")
```

### 2. 建立 Worktree

```bash
# Determine full path
case $LOCATION in
  .worktrees|worktrees)
    path="$LOCATION/$BRANCH_NAME"
    ;;
  ~/.config/superpowers/worktrees/*)
    path="~/.config/superpowers/worktrees/$project/$BRANCH_NAME"
    ;;
esac

# Create worktree with new branch
git worktree add "$path" -b "$BRANCH_NAME"
cd "$path"
```

### 3. 執行專案設定

自動偵測並執行適當設定：

```bash
# Node.js
if [ -f package.json ]; then npm install; fi

# Rust
if [ -f Cargo.toml ]; then cargo build; fi

# Python
if [ -f requirements.txt ]; then pip install -r requirements.txt; fi
if [ -f pyproject.toml ]; then poetry install; fi

# Go
if [ -f go.mod ]; then go mod download; fi
```

### 4. 驗證乾淨基準

執行測試以確保 worktree 一開始是乾淨狀態：

```bash
# Examples - use project-appropriate command
npm test
cargo test
pytest
go test ./...
```

**若測試失敗：**回報失敗並詢問是否要繼續或調查。

**若測試通過：**回報可開始。

### 5. 回報位置

```
Worktree ready at <full-path>
Tests passing (<N> tests, 0 failures)
Ready to implement <feature-name>
```

## 快速對照

| 情況 | 動作 |
|-----------|--------|
| 存在 `.worktrees/` | 使用它（並驗證忽略） |
| 存在 `worktrees/` | 使用它（並驗證忽略） |
| 兩者都存在 | 使用 `.worktrees/` |
| 都不存在 | 檢查 CLAUDE.md → 詢問使用者 |
| 目錄未被忽略 | 加入 .gitignore + 提交 |
| 基準測試失敗 | 回報失敗 + 詢問 |
| 無 package.json/Cargo.toml | 略過安裝依賴 |

## 常見錯誤

### 跳過忽略驗證

- **問題：**worktree 內容被追蹤，污染 git status
- **修正：**建立專案內 worktree 前一定要 `git check-ignore`

### 假設目錄位置

- **問題：**造成不一致，違反專案慣例
- **修正：**依優先序：既有 > CLAUDE.md > 詢問

### 在測試失敗仍繼續

- **問題：**無法分辨新 bug 或既有問題
- **修正：**回報失敗，取得明確允許再繼續

### 硬編設定指令

- **問題：**不同工具的專案會壞掉
- **修正：**依專案檔案自動偵測（package.json 等）

## 範例流程

```
You: I'm using the using-git-worktrees skill to set up an isolated workspace.

[Check .worktrees/ - exists]
[Verify ignored - git check-ignore confirms .worktrees/ is ignored]
[Create worktree: git worktree add .worktrees/auth -b feature/auth]
[Run npm install]
[Run npm test - 47 passing]

Worktree ready at /Users/jesse/myproject/.worktrees/auth
Tests passing (47 tests, 0 failures)
Ready to implement auth feature
```

## 紅旗

**絕不：**
- 未驗證忽略就建立 worktree（專案內）
- 跳過基準測試驗證
- 測試失敗未詢問就繼續
- 在不明確時假設目錄位置
