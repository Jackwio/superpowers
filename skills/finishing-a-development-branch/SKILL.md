---
name: finishing-a-development-branch
description: 當實作完成、測試通過，且需要決定如何整合成果時使用 — 透過提供合併、PR 或清理的結構化選項，引導完成開發工作
---

# 完成開發分支

## 概覽

透過清楚選項引導開發收尾，並依選擇處理流程。

**核心原則：**驗證測試 → 提出選項 → 執行選擇 → 清理。

**一開始要宣告：**「我正在使用 finishing-a-development-branch 技能完成這項工作。」

## 流程

### 步驟 1：驗證測試

**在提出選項前，先確認測試通過：**

```bash
# Run project's test suite
npm test / cargo test / pytest / go test ./...
```

**若測試失敗：**
```
Tests failing (<N> failures). Must fix before completing:

[Show failures]

Cannot proceed with merge/PR until tests pass.
```

停止，不要進入步驟 2。

**若測試通過：**繼續步驟 2。

### 步驟 2：判斷基底分支

```bash
# Try common base branches
git merge-base HEAD main 2>/dev/null || git merge-base HEAD master 2>/dev/null
```

或詢問：「這個分支是從 main 分出去的，對嗎？」

### 步驟 3：提出選項

**只提供這 4 個選項：**

```
Implementation complete. What would you like to do?

1. Merge back to <base-branch> locally
2. Push and create a Pull Request
3. Keep the branch as-is (I'll handle it later)
4. Discard this work

Which option?
```

**不要加說明** — 保持選項簡潔。

### 步驟 4：執行選擇

#### 選項 1：本地合併

```bash
# Switch to base branch
git checkout <base-branch>

# Pull latest
git pull

# Merge feature branch
git merge <feature-branch>

# Verify tests on merged result
<test command>

# If tests pass
git branch -d <feature-branch>
```

然後：清理 worktree（步驟 5）

#### 選項 2：推送並建立 PR

```bash
# Push branch
git push -u origin <feature-branch>

# Create PR
gh pr create --title "<title>" --body "$(cat <<'EOF'
## Summary
<2-3 bullets of what changed>

## Test Plan
- [ ] <verification steps>
EOF
)"
```

然後：清理 worktree（步驟 5）

#### 選項 3：保留現狀

回報：「保留分支 <name>。Worktree 保留於 <path>。」

**不要清理 worktree。**

#### 選項 4：丟棄

**先確認：**
```
This will permanently delete:
- Branch <name>
- All commits: <commit-list>
- Worktree at <path>

Type 'discard' to confirm.
```

等待使用者明確確認。

若確認：
```bash
git checkout <base-branch>
git branch -D <feature-branch>
```

然後：清理 worktree（步驟 5）

### 步驟 5：清理 Worktree

**針對選項 1、2、4：**

確認是否在 worktree：
```bash
git worktree list | grep $(git branch --show-current)
```

若是：
```bash
git worktree remove <worktree-path>
```

**選項 3：**保留 worktree。

## 快速對照

| 選項 | 合併 | 推送 | 保留 Worktree | 清理分支 |
|--------|-------|------|---------------|----------------|
| 1. 本地合併 | ✓ | - | - | ✓ |
| 2. 建立 PR | - | ✓ | ✓ | - |
| 3. 保留現狀 | - | - | ✓ | - |
| 4. 丟棄 | - | - | - | ✓（強制） |

## 常見錯誤

**跳過測試驗證**
- **問題：**合併壞掉的程式碼，建立失敗的 PR
- **修正：**在提供選項前一定要驗證測試

**開放式問題**
- **問題：**「我接下來要做什麼？」→ 含糊不清
- **修正：**只提供 4 個結構化選項

**自動清理 worktree**
- **問題：**在仍可能需要時移除 worktree（選項 2、3）
- **修正：**只在選項 1 與 4 清理

**丟棄未確認**
- **問題：**不小心刪掉成果
- **修正：**要求使用者輸入 "discard" 確認

## 紅旗

**絕不：**
- 在測試失敗時繼續
- 未驗證合併結果就合併
- 未確認就刪除成果
- 未明確要求就強制推送

**務必：**
- 在提供選項前驗證測試
- 只提供 4 個選項
- 選項 4 需輸入文字確認
- 只在選項 1 與 4 清理 worktree

## 整合

**呼叫來源：**
- **subagent-driven-development**（步驟 7）- 所有任務完成後
- **executing-plans**（步驟 5）- 所有批次完成後

**搭配：**
- **using-git-worktrees** - 清理該技能建立的 worktree
