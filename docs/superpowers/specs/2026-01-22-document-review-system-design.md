# 文件審查系統設計

## 概覽

在 superpowers 工作流程中新增兩個審查階段：

1. **規格文件審查** - 腦力激盪後、writing-plans 之前
2. **計畫文件審查** - writing-plans 後、實作前

兩者都遵循實作審查使用的迴圈模式。

## 規格文件審查者

**目的：**確認規格完整、一致，且可進入實作規劃。

**位置：**`skills/brainstorming/spec-document-reviewer-prompt.md`

**檢查重點：**

| 類別 | 要注意什麼 |
|----------|------------------|
| 完整性 | TODO、占位文字、「TBD」、未完成段落 |
| 覆蓋性 | 遺漏的錯誤處理、邊界情況、整合點 |
| 一致性 | 內部矛盾、相互衝突的需求 |
| 清晰度 | 模稜兩可的需求 |
| YAGNI | 未被要求的功能、過度工程 |

**輸出格式：**
```
## Spec Review

**Status:** Approved | Issues Found

**Issues (if any):**
- [Section X]: [issue] - [why it matters]

**Recommendations (advisory):**
- [suggestions that don't block approval]
```

**審查迴圈：**發現問題 → 腦力激盪代理修正 → 重新審查 → 重複直到通過。

**派發機制：**使用 Task 工具，並指定 `subagent_type: general-purpose`。審查者提示範本提供完整 prompt。由 brainstorming 技能的 controller 派發。

## 計畫文件審查者

**目的：**確認計畫完整、符合規格，且任務拆解正確。

**位置：**`skills/writing-plans/plan-document-reviewer-prompt.md`

**檢查重點：**

| 類別 | 要注意什麼 |
|----------|------------------|
| 完整性 | TODO、占位文字、未完成任務 |
| 規格對齊 | 計畫涵蓋規格需求，無範圍膨脹 |
| 任務拆解 | 任務具原子性、邊界清楚 |
| 任務語法 | 任務與步驟使用勾選框語法 |
| 區塊大小 | 每個區塊小於 1000 行 |

**區塊定義：**區塊是計畫文件中的一組邏輯任務，使用 `## Chunk N: <name>` 標題界定。writing-plans 技能會依邏輯階段（例如「Foundation」「Core Features」「Integration」）建立區塊。每個區塊應足夠自洽以便獨立審查。

**規格對齊驗證：**審查者同時取得：
1. 計畫文件（或當前區塊）
2. 參考規格文件的路徑

審查者會閱讀兩者並比對需求覆蓋。

**輸出格式：**與規格審查相同，但只針對當前區塊。

**審查流程（逐區塊）：**
1. writing-plans 建立區塊 N
2. Controller 派發 plan-document-reviewer，提供區塊 N 內容與規格路徑
3. 審查者閱讀區塊與規格並回傳裁定
4. 若有問題：writing-plans 代理修正區塊 N，回到步驟 2
5. 若通過：前往區塊 N+1
6. 重複直到所有區塊通過

**派發機制：**與規格審查相同 — 使用 Task 工具，`subagent_type: general-purpose`。

## 更新後流程

```
brainstorming -> spec -> SPEC REVIEW LOOP -> writing-plans -> plan -> PLAN REVIEW LOOP -> implementation
```

**規格審查迴圈：**
1. 規格完成
2. 派發審查者
3. 若有問題：修正 → 回到 2
4. 若通過：前進

**計畫審查迴圈：**
1. 區塊 N 完成
2. 派發區塊 N 審查者
3. 若有問題：修正 → 回到 2
4. 若通過：下一區塊或進入實作

## Markdown 任務語法

任務與步驟使用勾選框語法：

```markdown
- [ ] ### Task 1: Name

- [ ] **Step 1:** Description
  - File: path
  - Command: cmd
```

## 錯誤處理

**審查迴圈終止：**
- 無硬性迭代上限 — 直到審查者通過
- 若超過 5 次迭代，controller 應回報使用者請求指引
- 使用者可選擇：繼續迭代、在已知問題下通過，或中止

**歧見處理：**
- 審查者為建議性 — 會標記問題但不阻擋
- 若代理認為回饋不正確，應在修正中說明理由
- 若同一問題 3 次迭代仍無共識，回報使用者

**審查輸出格式錯誤：**
- Controller 應驗證審查輸出包含必要欄位（Status，必要時 Issues）
- 若格式錯誤，重新派發審查者並提醒期望格式
- 若 2 次仍錯誤，回報使用者

## 需變更檔案

**新檔案：**
- `skills/brainstorming/spec-document-reviewer-prompt.md`
- `skills/writing-plans/plan-document-reviewer-prompt.md`

**修改檔案：**
- `skills/brainstorming/SKILL.md` - 在規格寫完後加入審查迴圈
- `skills/writing-plans/SKILL.md` - 加入逐區塊審查迴圈，更新任務語法範例
