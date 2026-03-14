---
name: requesting-code-review
description: 在完成任務、實作主要功能、或合併前使用，以驗證成果符合需求
---

# 請求程式碼審查

派出 superpowers:code-reviewer 子代理，在問題擴大前先抓出來。審查者會拿到精準的評估脈絡 — **絕不**提供你的會話歷史。這能讓審查者聚焦在成果本身，而非你的思考過程，也保留你的脈絡以便繼續工作。

**核心原則：**早審查、常審查。

## 何時請求審查

**必須：**
- subagent-driven development 的每個任務之後
- 完成主要功能之後
- 合併到 main 之前

**可選但很有價值：**
- 卡住時（換個視角）
- 重構前（建立基準）
- 修複雜 bug 之後

## 如何請求

**1. 取得 git SHA：**
```bash
BASE_SHA=$(git rev-parse HEAD~1)  # or origin/main
HEAD_SHA=$(git rev-parse HEAD)
```

**2. 派出 code-reviewer 子代理：**

使用 Task 工具，指定 superpowers:code-reviewer 類型，並填入 `code-reviewer.md` 範本

**占位符：**
- `{WHAT_WAS_IMPLEMENTED}` - 你剛完成的內容
- `{PLAN_OR_REQUIREMENTS}` - 需求或計畫內容
- `{BASE_SHA}` - 起始提交
- `{HEAD_SHA}` - 結束提交
- `{DESCRIPTION}` - 簡短摘要

**3. 依回饋處理：**
- 立即修正 Critical
- 在繼續前修正 Important
- Minor 記下稍後處理
- 若審查者錯誤，以理由反駁

## 範例

```
[剛完成任務 2：加入驗證函式]

你：讓我在繼續前請求程式碼審查。

BASE_SHA=$(git log --oneline | grep "Task 1" | head -1 | awk '{print $1}')
HEAD_SHA=$(git rev-parse HEAD)

[派出 superpowers:code-reviewer 子代理]
  WHAT_WAS_IMPLEMENTED: Verification and repair functions for conversation index
  PLAN_OR_REQUIREMENTS: Task 2 from docs/superpowers/plans/deployment-plan.md
  BASE_SHA: a7981ec
  HEAD_SHA: 3df7661
  DESCRIPTION: Added verifyIndex() and repairIndex() with 4 issue types

[子代理回報]:
  Strengths: Clean architecture, real tests
  Issues:
    Important: Missing progress indicators
    Minor: Magic number (100) for reporting interval
  Assessment: Ready to proceed

你：[修正進度指示]
[繼續任務 3]
```

## 與工作流程整合

**Subagent-Driven Development：**
- 每個任務後審查
- 在問題累積前先抓出
- 修完再進下一個任務

**Executing Plans：**
- 每批次（3 個任務）後審查
- 收回饋、修正、繼續

**臨時開發：**
- 合併前審查
- 卡住時審查

## 紅旗

**絕不：**
- 因為「很簡單」就跳過審查
- 忽略 Critical 問題
- 未修正 Important 就繼續
- 對合理的技術回饋爭辯

**若審查者錯誤：**
- 以技術理由反駁
- 提供可證明運作的程式碼/測試
- 請求釐清

範本位置：requesting-code-review/code-reviewer.md
