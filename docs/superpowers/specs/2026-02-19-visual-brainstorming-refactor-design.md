# 視覺腦力激盪重構：瀏覽器顯示、終端指令

**日期：**2026-02-19
**狀態：**Approved
**範圍：**`lib/brainstorm-server/`、`skills/brainstorming/visual-companion.md`、`tests/brainstorm-server/`

## 問題

在視覺腦力激盪時，Claude 會以背景任務執行 `wait-for-feedback.sh`，並在 `TaskOutput(block=true, timeout=600s)` 上阻塞。這會完全佔用 TUI — 使用者在視覺腦力激盪期間無法對 Claude 打字。瀏覽器變成唯一輸入管道。

Claude Code 的執行模型是回合制。在單一回合內，Claude 無法同時聽兩個管道。阻塞式 `TaskOutput` 是錯誤的原語 — 它模擬了平台不支援的事件驅動行為。

## 設計

### 核心模型

**瀏覽器 = 互動顯示。**展示 mockup，讓使用者點選選項。選取會記錄在伺服器端。

**終端機 = 對話管道。**永遠不被阻塞，永遠可用。使用者在此與 Claude 對話。

### 迴圈

1. Claude 寫入 HTML 檔到會話目錄
2. 伺服器透過 chokidar 偵測並推送 WebSocket reload 到瀏覽器（不變）
3. Claude 結束回合 — 告訴使用者去看瀏覽器並在終端回覆
4. 使用者看瀏覽器、可選擇點選選項，然後在終端輸入回饋
5. 下一回合 Claude 讀取 `$SCREEN_DIR/.events` 的瀏覽器互動串流（點擊、選取），與終端文字合併
6. 迭代或前進

不再需要背景任務。不再阻塞 `TaskOutput`。不再需要輪詢腳本。

### 重要刪除：`wait-for-feedback.sh`

完整刪除。它的目的是串接「伺服器把事件寫到 stdout」與「Claude 需要接收事件」。新的 `.events` 檔取代它 — 伺服器直接寫入使用者互動事件，Claude 透過平台提供的檔案讀取機制讀取。

### 重要新增：`.events` 檔（每畫面事件串流）

伺服器把所有使用者互動事件寫入 `$SCREEN_DIR/.events`，每行一個 JSON 物件。這提供 Claude 當前畫面的完整互動串流 — 不只是最後選擇，還包含使用者的探索路徑（點了 A、再點 B、最後選 C）。

使用者探索選項後的檔案內容範例：

```jsonl
{"type":"click","choice":"a","text":"Option A - Preset-First Wizard","timestamp":1706000101}
{"type":"click","choice":"c","text":"Option C - Manual Config","timestamp":1706000108}
{"type":"click","choice":"b","text":"Option B - Hybrid Approach","timestamp":1706000115}
```

- 在同一畫面中以附加方式寫入。每個使用者事件都新增一行。
- 當 chokidar 偵測到新的 HTML 檔（新畫面），會刪除檔案以清除舊事件，避免殘留。
- 若 Claude 讀取時檔案不存在，代表沒有瀏覽器互動 — Claude 只用終端文字。
- 檔案只包含使用者事件（`click` 等） — 不含伺服器生命週期事件（`server-started`、`screen-added`）。保持小而聚焦。
- Claude 可讀整段串流了解探索模式，或只看最後的 `choice` 事件作為最終選擇。

## 依檔案變更

### `index.js`（server）

**A. 將使用者事件寫入 `.events` 檔。**

在 WebSocket `message` handler 中，在把事件輸出到 stdout 後，使用 `fs.appendFileSync` 將事件以 JSON 行附加到 `$SCREEN_DIR/.events`。只寫使用者互動事件（`source: 'user-event'`），不要寫伺服器生命週期事件。

**B. 新畫面時清除 `.events`。**

在 chokidar `add` handler（偵測到新 `.html`）中，若 `$SCREEN_DIR/.events` 存在就刪除。這是明確的「新畫面」信號 — 比清除 GET `/` 更準確，因為 GET 每次 reload 都會觸發。

**C. 更換 `wrapInFrame` 的內容注入方式。**

目前 regex 以 `<div class="feedback-footer">` 為錨點，但該區塊會移除。改成使用註解占位：移除 `#claude-content` 內的預設文字（`<h2>Visual Brainstorming</h2>` 與副標題），改成單一 `<!-- CONTENT -->` 標記。注入方式改為 `frameTemplate.replace('<!-- CONTENT -->', content)`。更簡單，也不會因模板格式變更而破壞。

### `frame-template.html`（UI frame）

**移除：**
- `feedback-footer` div（textarea、Send 按鈕、label、`.feedback-row`）
- 相關 CSS（`.feedback-footer`、`.feedback-footer label`、`.feedback-row`、其中的 textarea 與 button 樣式）

**新增：**
- 在 `#claude-content` 內加入 `<!-- CONTENT -->` 占位，取代預設文字
- 在原 footer 的位置加入選取提示條，兩種狀態：
  - 預設：「Click an option above, then return to the terminal」
  - 選取後：「Option B selected — return to terminal to continue」
- 提示條的 CSS（視覺權重與現有 header 類似，低調）

**保持不變：**
- 標頭列（"Brainstorm Companion" 標題與連線狀態）
- `.main` 包裹與 `#claude-content` 容器
- 所有元件 CSS（`.options`、`.cards`、`.mockup`、`.split`、`.pros-cons`、placeholder、mock 元件）
- 深/淺色主題變數與 media query

### `helper.js`（前端腳本）

**移除：**
- `sendToClaude()` 函式與「Sent to Claude」頁面切換
- `window.send()` 函式（綁定已移除的 Send 按鈕）
- 表單送出 handler — 沒有 textarea 就沒有用途，還會產生日誌噪音
- input change handler — 同理
- `pageshow` 事件監聽（原本修 textarea 保留問題，現在已無）

**保留：**
- WebSocket 連線、重連、事件佇列
- Reload handler（伺服器推送時 `window.location.reload()`）
- `window.toggleSelect()` 用於選取高亮
- `window.selectedChoice` 追蹤
- `window.brainstorm.send()` 與 `window.brainstorm.choice()` — 與移除的 `window.send()` 不同。它們呼叫 `sendEvent` 透過 WebSocket 記錄事件。對自訂完整頁面仍有用。

**縮小範圍：**
- 點擊 handler：只捕捉 `[data-choice]` 點擊，不再抓所有按鈕/連結。過去廣泛捕捉是因為瀏覽器也是回饋管道；現在只需選取追蹤。

**新增：**
- 在 `data-choice` 點擊時更新選取提示條文字，顯示目前選擇。

**從 `window.brainstorm` API 移除：**
- `brainstorm.sendToClaude` — 不再存在

### `visual-companion.md`（技能指引）

**重寫「The Loop」章節**為上述的非阻塞流程。移除所有關於以下內容的參考：
- `wait-for-feedback.sh`
- `TaskOutput` 阻塞
- timeout/retry 邏輯（600s timeout、30 分鐘上限）
- 描述 `send-to-claude` JSON 的「User Feedback Format」

**改為：**
- 新迴圈（寫 HTML → 結束回合 → 使用者在終端回覆 → 讀 `.events` → 迭代）
- `.events` 檔案格式文件
- 指引：終端訊息是主要回饋；`.events` 提供瀏覽器互動串流以補充脈絡

**保留：**
- 伺服器啟動/停止指引
- 內容片段 vs 完整文件的說明
- CSS 類別與可用元件參考
- 設計建議（依問題調整 fidelity、每頁 2-4 個選項等）

### `wait-for-feedback.sh`

**完整刪除。**

### `tests/brainstorm-server/server.test.js`

需要更新的測試：
- 斷言 fragment 回應包含 `feedback-footer` — 改成斷言選取提示條或 `<!-- CONTENT -->`
- 斷言 `helper.js` 含 `send` — 更新以符合縮小 API
- 斷言 `sendToClaude` 使用 CSS 變數 — 移除（函式不再存在）

## 平台相容性

伺服器端程式（`index.js`、`helper.js`、`frame-template.html`）完全平台無關 — 純 Node.js 與瀏覽器 JavaScript。沒有 Claude Code 專用內容。已證實可在 Codex 透過背景終端互動運作。

技能指引（`visual-companion.md`）是平台適配層。每個平台的 Claude 以其工具啟動伺服器、讀 `.events` 等。非阻塞模型不依賴任何平台特定的阻塞原語，因此可跨平台運作。

## 這能帶來什麼

- **TUI 永遠可用**於視覺腦力激盪期間
- **混合輸入** — 在瀏覽器點選 + 在終端輸入，自然合併
- **優雅降級** — 瀏覽器掛了或使用者沒開啟？終端仍可用
- **更簡單架構** — 無背景任務、無輪詢腳本、無 timeout 管理
- **跨平台** — 同一套伺服器程式可用於 Claude Code、Codex 與未來平台

## 取捨

- **純瀏覽器回饋流程** — 使用者必須回到終端才能繼續。選取提示條會引導，但比舊版的點選 Send 後等待多了一步。
- **瀏覽器內文字回饋** — textarea 移除，所有文字回饋都改在終端。這是刻意的 — 終端是更好的文字輸入管道。
- **瀏覽器 Send 後即時回應** — 舊系統會在使用者點擊 Send 的瞬間回應。現在使用者要切回終端，會有些微時間差。實務上只有幾秒，且使用者可在終端補充更多脈絡。
