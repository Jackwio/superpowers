# 視覺輔助指南

以瀏覽器為基礎的視覺化腦力激盪輔助工具，用來展示 mockup、圖表與選項。

## 何時使用

以「每個問題」為單位判斷，而不是整個會話。測試標準：**使用者會不會看圖比看文字更容易理解？**

**當內容本身是視覺化時，使用瀏覽器：**

- **UI mockup** — 線框圖、版面配置、導覽結構、元件設計
- **架構圖** — 系統元件、資料流、關係圖
- **視覺並排比較** — 比較兩種版面、兩種配色、兩種設計方向
- **設計精修** — 問題關於外觀、間距、視覺層級
- **空間關係** — 狀態機、流程圖、以圖表呈現的實體關係

**當內容是文字或表格時，使用終端機：**

- **需求與範圍問題** —「X 是什麼意思？」「哪些功能在範圍內？」
- **概念性的 A/B/C 選擇** — 以文字描述的作法選擇
- **取捨清單** — 優缺點、比較表
- **技術決策** — API 設計、資料建模、架構方案選擇
- **澄清問題** — 回答是文字而非視覺偏好

「關於 UI 的問題」不一定就是視覺問題。「你想要哪種類型的精靈？」是概念問題 — 用終端機。「哪一種精靈版面看起來對？」是視覺問題 — 用瀏覽器。

## 如何運作

伺服器會監看某個目錄的 HTML 檔案，並把最新的檔案提供給瀏覽器。你寫 HTML 內容，使用者在瀏覽器看到並可點擊選項。點選會被記錄到 `.events` 檔案中，你在下一回合讀取。

**內容片段 vs 完整文件：**如果你的 HTML 檔案以 `<!DOCTYPE` 或 `<html` 開頭，伺服器就原樣提供（只注入 helper script）。否則，伺服器會自動把你的內容包進 frame template — 加上標頭、CSS 主題、選取指示器與互動基礎設施。**預設請寫內容片段。**只有在需要完整掌控頁面時才寫完整文件。

## 啟動會話

```bash
# Start server with persistence (mockups saved to project)
scripts/start-server.sh --project-dir /path/to/project

# Returns: {"type":"server-started","port":52341,"url":"http://localhost:52341",
#           "screen_dir":"/path/to/project/.superpowers/brainstorm/12345-1706000000"}
```

從回傳中保存 `screen_dir`，並請使用者打開 URL。

**找到連線資訊：**伺服器會把啟動 JSON 寫入 `$SCREEN_DIR/.server-info`。如果你在背景啟動伺服器且沒抓到 stdout，讀取該檔案取得 URL 與 port。使用 `--project-dir` 時，可在 `<project>/.superpowers/brainstorm/` 找到會話目錄。

**注意：**請把專案根目錄當作 `--project-dir`，讓 mockup 存在 `.superpowers/brainstorm/` 中並在重啟後保留。未指定則寫入 `/tmp` 並會被清理。提醒使用者將 `.superpowers/` 加入 `.gitignore`（若尚未加入）。

**依平台啟動伺服器：**

**Claude Code：**
```bash
# Default mode works — the script backgrounds the server itself
scripts/start-server.sh --project-dir /path/to/project
```

**Codex：**
```bash
# Codex reaps background processes. The script auto-detects CODEX_CI and
# switches to foreground mode. Run it normally — no extra flags needed.
scripts/start-server.sh --project-dir /path/to/project
```

**Gemini CLI：**
```bash
# Use --foreground and set is_background: true on your shell tool call
# so the process survives across turns
scripts/start-server.sh --project-dir /path/to/project --foreground
```

**其他環境：**伺服器必須在對話回合間持續在背景運作。若你的環境會回收背景行程，請使用 `--foreground` 並透過平台的背景執行機制啟動指令。

若瀏覽器無法存取 URL（常見於遠端或容器化環境），請綁定非 loopback host：

```bash
scripts/start-server.sh \
  --project-dir /path/to/project \
  --host 0.0.0.0 \
  --url-host localhost
```

使用 `--url-host` 來控制回傳 URL JSON 中顯示的主機名稱。

## 操作循環

1. **確認伺服器仍在運作**，然後**寫入 HTML** 到 `screen_dir` 的新檔案：
   - 每次寫入前檢查 `$SCREEN_DIR/.server-info` 是否存在。若不存在（或 `.server-stopped` 存在），代表伺服器已關閉 — 先用 `start-server.sh` 重新啟動再繼續。伺服器會在 30 分鐘無活動後自動退出。
   - 使用語意化檔名：`platform.html`、`visual-style.html`、`layout.html`
   - **不要重用檔名** — 每個畫面都要新檔案
   - 使用 Write 工具 — **不要用 cat/heredoc**（會把雜訊輸出到終端）
   - 伺服器會自動提供最新檔案

2. **告知使用者並結束你的回合：**
   - 提醒他們 URL（每一步都要提醒，不只第一次）
   - 用簡短文字說明畫面內容（例如「正在顯示首頁的 3 種版面」）
   - 請他們在終端機回覆：「看完告訴我你的想法，需要的話可以點選選項。」

3. **下一回合** — 在使用者於終端機回覆後：
   - 若 `$SCREEN_DIR/.events` 存在就讀取 — 其中包含使用者在瀏覽器的互動（點擊、選取）JSON Lines
   - 與使用者的終端文字一起整合判斷
   - 終端訊息是主要回饋；`.events` 提供結構化互動資料

4. **迭代或前進** — 若回饋需要調整當前畫面，寫入新檔案（例如 `layout-v2.html`）。只有當這一步被確認後才進入下一個問題。

5. **回到終端時卸載** — 當下一步不需要瀏覽器（例如澄清問題、取捨討論）時，推送等待畫面以清除過期內容：

   ```html
   <!-- filename: waiting.html (or waiting-2.html, etc.) -->
   <div style="display:flex;align-items:center;justify-content:center;min-height:60vh">
     <p class="subtitle">Continuing in terminal...</p>
   </div>
   ```

   這能避免使用者盯著已完成的選擇，而對話已經往前。當下一個視覺問題出現時，如常推送新內容檔案。

6. 重複直到完成。

## 撰寫內容片段

只寫會放進頁面內的內容即可。伺服器會自動用 frame template 包裹（標頭、主題 CSS、選取指示器與互動基礎設施）。

**最小範例：**

```html
<h2>Which layout works better?</h2>
<p class="subtitle">Consider readability and visual hierarchy</p>

<div class="options">
  <div class="option" data-choice="a" onclick="toggleSelect(this)">
    <div class="letter">A</div>
    <div class="content">
      <h3>Single Column</h3>
      <p>Clean, focused reading experience</p>
    </div>
  </div>
  <div class="option" data-choice="b" onclick="toggleSelect(this)">
    <div class="letter">B</div>
    <div class="content">
      <h3>Two Column</h3>
      <p>Sidebar navigation with main content</p>
    </div>
  </div>
</div>
```

就這樣。不需要 `<html>`、CSS 或 `<script>`。伺服器會提供。

## 可用 CSS 類別

frame template 會提供以下 CSS 類別供你的內容使用：

### Options（A/B/C 選擇）

```html
<div class="options">
  <div class="option" data-choice="a" onclick="toggleSelect(this)">
    <div class="letter">A</div>
    <div class="content">
      <h3>Title</h3>
      <p>Description</p>
    </div>
  </div>
</div>
```

**多選：**在容器上加入 `data-multiselect` 讓使用者可選多個選項。每次點擊會切換選取狀態，指示條會顯示數量。

```html
<div class="options" data-multiselect>
  <!-- same option markup — users can select/deselect multiple -->
</div>
```

### Cards（視覺設計）

```html
<div class="cards">
  <div class="card" data-choice="design1" onclick="toggleSelect(this)">
    <div class="card-image"><!-- mockup content --></div>
    <div class="card-body">
      <h3>Name</h3>
      <p>Description</p>
    </div>
  </div>
</div>
```

### Mockup 容器

```html
<div class="mockup">
  <div class="mockup-header">Preview: Dashboard Layout</div>
  <div class="mockup-body"><!-- your mockup HTML --></div>
</div>
```

### 分割檢視（左右並排）

```html
<div class="split">
  <div class="mockup"><!-- left --></div>
  <div class="mockup"><!-- right --></div>
</div>
```

### 優缺點

```html
<div class="pros-cons">
  <div class="pros"><h4>Pros</h4><ul><li>Benefit</li></ul></div>
  <div class="cons"><h4>Cons</h4><ul><li>Drawback</li></ul></div>
</div>
```

### Mock 元件（線框圖積木）

```html
<div class="mock-nav">Logo | Home | About | Contact</div>
<div style="display: flex;">
  <div class="mock-sidebar">Navigation</div>
  <div class="mock-content">Main content area</div>
</div>
<button class="mock-button">Action Button</button>
<input class="mock-input" placeholder="Input field">
<div class="placeholder">Placeholder area</div>
```

### 排版與區塊

- `h2` — 頁面標題
- `h3` — 區段標題
- `.subtitle` — 標題下方的次要文字
- `.section` — 具有下方間距的內容區塊
- `.label` — 小型大寫標籤文字

## 瀏覽器事件格式

使用者在瀏覽器點擊選項時，互動會記錄到 `$SCREEN_DIR/.events`（每行一個 JSON 物件）。當你推送新畫面時，該檔案會自動清除。

```jsonl
{"type":"click","choice":"a","text":"Option A - Simple Layout","timestamp":1706000101}
{"type":"click","choice":"c","text":"Option C - Complex Grid","timestamp":1706000108}
{"type":"click","choice":"b","text":"Option B - Hybrid","timestamp":1706000115}
```

完整事件串流能顯示使用者的探索路徑 — 他們可能在定案前點選多個選項。最後一個 `choice` 事件通常是最終選擇，但點擊的模式可能透露猶豫或偏好，值得追問。

若 `.events` 不存在，代表使用者沒有在瀏覽器互動 — 請只使用其終端文字回覆。

## 設計建議

- **依問題調整 fidelity** — 版面用線框，精修問題用更完整視覺
- **每頁說清楚問題** —「哪種版面更專業？」而不是只說「選一個」
- **先迭代再前進** — 回饋改變當前畫面時，先改版
- **每頁 2-4 個選項**
- **需要時用真實內容** — 攝影作品集就用真實圖片（Unsplash）。占位內容會遮蔽設計問題。
- **保持 mockup 簡單** — 聚焦版面與結構，不追求像素完美

## 檔案命名

- 使用語意化名稱：`platform.html`、`visual-style.html`、`layout.html`
- 不要重用檔名 — 每個畫面必須是新檔案
- 迭代時：加版本後綴，如 `layout-v2.html`、`layout-v3.html`
- 伺服器會依修改時間提供最新檔案

## 清理

```bash
scripts/stop-server.sh $SCREEN_DIR
```

若會話使用了 `--project-dir`，mockup 檔案會保留在 `.superpowers/brainstorm/` 以便日後參考。只有 `/tmp` 會話會在停止時被刪除。

## 參考

- Frame template（CSS 參考）：`scripts/frame-template.html`
- Helper script（前端）：`scripts/helper.js`
