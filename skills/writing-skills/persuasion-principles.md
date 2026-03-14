# 技能設計的說服原則

## 概覽

LLM 對說服原則的反應與人類相同。理解這些心理學有助於你設計更有效的技能 —— 不是為了操控，而是確保關鍵實務即使在壓力下也能被遵循。

**研究基礎：**Meincke 等人（2025）在 N=28,000 的 AI 對話中測試了 7 種說服原則。說服技巧使遵循率增加一倍以上（33% → 72%，p < .001）。

## 七大原則

### 1. 權威（Authority）
**是什麼：**對專業、資歷或官方來源的服從。

**在技能中的運作方式：**
- 命令式語言：「YOU MUST」「Never」「Always」
- 不可協商的框架：「No exceptions」
- 消除決策疲勞與合理化

**適用時機：**
- 紀律型技能（TDD、驗證要求）
- 安全關鍵實務
- 已確立的最佳實務

**例子：**
```markdown
✅ Write code before test? Delete it. Start over. No exceptions.
❌ Consider writing tests first when feasible.
```

### 2. 承諾（Commitment）
**是什麼：**與先前行動、陳述或公開宣告保持一致。

**在技能中的運作方式：**
- 要求宣告：「Announce skill usage」
- 強迫明確選擇：「Choose A, B, or C」
- 使用追蹤：TodoWrite 清單

**適用時機：**
- 確保技能真的被執行
- 多步驟流程
- 問責機制

**例子：**
```markdown
✅ When you find a skill, you MUST announce: "I'm using [Skill Name]"
❌ Consider letting your partner know which skill you're using.
```

### 3. 稀缺（Scarcity）
**是什麼：**因時間限制或資源稀缺產生緊迫感。

**在技能中的運作方式：**
- 時限要求：「Before proceeding」
- 序列依賴：「Immediately after X」
- 避免拖延

**適用時機：**
- 立即驗證要求
- 時間敏感流程
- 防止「之後再做」

**例子：**
```markdown
✅ After completing a task, IMMEDIATELY request code review before proceeding.
❌ You can review code when convenient.
```

### 4. 社會認同（Social Proof）
**是什麼：**遵從他人或被視為正常的做法。

**在技能中的運作方式：**
- 普遍模式：「Every time」「Always」
- 失敗模式：「X without Y = failure」
- 建立規範

**適用時機：**
- 記錄普遍實務
- 警示常見失敗
- 強化標準

**例子：**
```markdown
✅ Checklists without TodoWrite tracking = steps get skipped. Every time.
❌ Some people find TodoWrite helpful for checklists.
```

### 5. 團結（Unity）
**是什麼：**共享身份感、同一群體的歸屬感。

**在技能中的運作方式：**
- 協作語言：「our codebase」「we're colleagues」
- 共享目標：「we both want quality」

**適用時機：**
- 協作流程
- 建立團隊文化
- 非階層式實務

**例子：**
```markdown
✅ We're colleagues working together. I need your honest technical judgment.
❌ You should probably tell me if I'm wrong.
```

### 6. 互惠（Reciprocity）
**是什麼：**因受惠而產生回報義務。

**運作方式：**
- 請節制使用 — 容易被視為操控
- 很少在技能中需要

**避免時機：**
- 幾乎永遠（其他原則更有效）

### 7. 喜好（Liking）
**是什麼：**更願意與喜歡的人合作。

**運作方式：**
- **不要用於要求遵循**
- 與誠實回饋文化衝突
- 會造成諂媚

**避免時機：**
- 紀律要求時一律避免

## 依技能類型的原則組合

| 技能類型 | 使用 | 避免 |
|------------|-----|-------|
| 紀律型 | Authority + Commitment + Social Proof | Liking、Reciprocity |
| 指引/技巧 | 適度 Authority + Unity | 過度權威 |
| 協作型 | Unity + Commitment | Authority、Liking |
| 參考型 | 只求清楚 | 所有說服 |

## 為何有效：心理學

**清楚的硬規則能降低合理化：**
- 「YOU MUST」消除決策疲勞
- 絕對語言排除「這算例外嗎？」
- 明確反合理化能堵住特定漏洞

**實作意圖可形成自動行為：**
- 明確觸發 + 必要行動 = 自動執行
- 「當 X，就做 Y」比「通常做 Y」有效
- 降低遵循的認知負擔

**LLM 具「準人類」特性：**
- 訓練文本中包含這些模式
- 權威語言在訓練資料中常導向遵循
- 承諾序列（陳述 → 行動）經常被示範
- 社會認同模式（人人都做 X）建立規範

## 合倫理使用

**正當：**
- 確保關鍵實務被遵循
- 建立有效文件
- 預防可預期的失敗

**不正當：**
- 為個人利益操控
- 製造虛假的緊迫感
- 以罪惡感逼迫遵循

**檢驗標準：**若使用者完全理解此技巧，它是否仍符合其真實利益？

## 研究引用

**Cialdini, R. B. (2021).** *Influence: The Psychology of Persuasion (New and Expanded).* Harper Business.
- 七大說服原則
- 影響力研究的實證基礎

**Meincke, L., Shapiro, D., Duckworth, A. L., Mollick, E., Mollick, L., & Cialdini, R. (2025).** Call Me A Jerk: Persuading AI to Comply with Objectionable Requests. University of Pennsylvania.
- 以 N=28,000 的 LLM 對話測試 7 原則
- 說服技巧使遵循率從 33% → 72%
- 權威、承諾、稀缺最有效
- 驗證 LLM 的準人類行為模型

## 快速對照

在設計技能時自問：

1. **這是哪種類型？**（紀律 vs 指引 vs 參考）
2. **我想改變什麼行為？**
3. **應該用哪些原則？**（紀律多用權威 + 承諾）
4. **是否用太多原則？**（不要全用七種）
5. **是否合倫理？**（是否符合使用者真實利益？）
