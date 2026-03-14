---
name: writing-plans
description: 當你有規格或需求且任務為多步驟時使用，在動到程式碼之前必須使用
---

# 撰寫計畫

## 概覽

撰寫完整的實作計畫，假設工程師對程式碼庫毫無脈絡且品味堪憂。記錄他們需要知道的一切：每個任務要動哪些檔案、要寫哪些程式碼、要跑哪些測試、需要看哪些文件、如何測試。把整個計畫拆成小而清楚的任務。DRY。YAGNI。TDD。頻繁提交。

假設他是技術熟練的工程師，但對我們的工具與領域幾乎不了解。也假設他對測試設計不夠熟練。

**一開始要宣告：**「我正在使用 writing-plans 技能建立實作計畫。」

**脈絡：**應在專用 worktree 中執行（由 brainstorming 技能建立）。

**計畫存放：**`docs/superpowers/plans/YYYY-MM-DD-<feature-name>.md`
- （若使用者指定存放位置，則以使用者偏好為準）

## 範圍檢查

若規格涵蓋多個獨立子系統，應該在 brainstorming 階段拆成子專案規格。若未拆分，請建議拆成多個計畫 — 每個子系統一份。每份計畫都應能獨立產出可運作、可測試的軟體。

## 檔案結構

在定義任務前，先規劃哪些檔案會新增或修改，以及每個檔案的責任。此處會鎖定拆分決策。

- 設計清楚邊界與介面的單元。每個檔案只負責一件事。
- 你對可一次掌握的程式碼推理最好，檔案越聚焦修改越可靠。偏好小而聚焦的檔案，避免過大的檔案做太多事。
- 會一起改動的檔案應該放在一起。依責任拆分，不依技術層拆分。
- 在既有程式碼庫中遵循既有模式。若程式碼庫本來就有大型檔案，不要單方面重構 — 但若你要改的檔案已經難以維護，在計畫中加入拆分是合理的。

此結構會影響任務拆解。每個任務應產出可獨立理解的變更。

## 小步任務粒度

**每一步是一個行動（2-5 分鐘）：**
- 「寫失敗測試」— 一步
- 「跑一次確認失敗」— 一步
- 「寫最小實作讓測試通過」— 一步
- 「跑測試確認通過」— 一步
- 「提交」— 一步

## 計畫文件標頭

**每份計畫必須以此標頭開始：**

```markdown
# [Feature Name] Implementation Plan

> **For agentic workers:** REQUIRED: Use superpowers:subagent-driven-development (if subagents available) or superpowers:executing-plans to implement this plan. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** [One sentence describing what this builds]

**Architecture:** [2-3 sentences about approach]

**Tech Stack:** [Key technologies/libraries]

---
```

## 任務結構

````markdown
### Task N: [Component Name]

**Files:**
- Create: `exact/path/to/file.py`
- Modify: `exact/path/to/existing.py:123-145`
- Test: `tests/exact/path/to/test.py`

- [ ] **Step 1: Write the failing test**

```python
def test_specific_behavior():
    result = function(input)
    assert result == expected
```

- [ ] **Step 2: Run test to verify it fails**

Run: `pytest tests/path/test.py::test_name -v`
Expected: FAIL with "function not defined"

- [ ] **Step 3: Write minimal implementation**

```python
def function(input):
    return expected
```

- [ ] **Step 4: Run test to verify it passes**

Run: `pytest tests/path/test.py::test_name -v`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add tests/path/test.py src/path/file.py
git commit -m "feat: add specific feature"
```
````

## 記住
- 永遠使用精確檔案路徑
- 計畫中要有完整程式碼（不要只寫「加入驗證」）
- 指令要精確，且附預期輸出
- 以 @ 語法引用相關技能
- DRY、YAGNI、TDD、頻繁提交

## 計畫審查迴圈

每完成一個計畫區塊後：

1. 派出 plan-document-reviewer 子代理（見 plan-document-reviewer-prompt.md），並提供精準審查脈絡 — 絕不提供你的會話歷史。這讓審查者聚焦在計畫本身，而非你的思考過程。
   - 提供：區塊內容、規格文件路徑
2. 若 ❌ 發現問題：
   - 修正該區塊問題
   - 重新派發該區塊審查
   - 重複直到 ✅ 通過
3. 若 ✅ 通過：進入下一區塊（或若最後一塊則交接執行）

**區塊邊界：**使用 `## Chunk N: <name>` 標題區分區塊。每個區塊 ≤1000 行且邏輯自洽。

**審查迴圈指引：**
- 寫計畫的同一個代理負責修正（保留脈絡）
- 若迴圈超過 5 次，回報使用者請求指引
- 審查為建議性質 — 若你認為回饋不正確，需說明理由

## 執行交接

儲存計畫後：

**「計畫完成並存至 `docs/superpowers/plans/<filename>.md`。可以開始執行嗎？」**

**執行路徑取決於平台能力：**

**若平台支援子代理（Claude Code 等）：**
- **必須：**使用 superpowers:subagent-driven-development
- 不要提供選擇 — 子代理驅動是標準做法
- 每任務一個新子代理 + 兩階段審查

**若平台不支援子代理：**
- 以 superpowers:executing-plans 在當前會話執行
- 批次執行並保留審查檢查點
