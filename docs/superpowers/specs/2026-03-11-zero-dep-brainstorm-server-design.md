# 零依賴腦力激盪伺服器

將 brainstorm companion server 的 vendored node_modules（express、ws、chokidar — 714 個受追蹤檔案）替換為單一零依賴 `server.js`，只使用 Node.js 內建模組。

## 動機

把 node_modules 直接放進 git repo 會產生供應鏈風險：凍結的依賴無法獲得安全修補，714 個第三方程式碼檔案在未審核下被提交，對 vendored 程式碼的修改也會看起來像正常提交。雖然實際風險不高（僅 localhost 的 dev server），但消除它很容易。

## 架構

單一 `server.js`（約 250-300 行），使用 `http`、`crypto`、`fs`、`path`。該檔案扮演兩個角色：

- **直接執行**（`node server.js`）：啟動 HTTP/WebSocket 伺服器
- **被 require**（`require('./server.js')`）：匯出 WebSocket 協定函式以便單元測試

### WebSocket 協定

只實作 RFC 6455 的文字幀：

**Handshake：**使用 SHA-1 + RFC 6455 magic GUID 從 client 的 `Sec-WebSocket-Key` 計算 `Sec-WebSocket-Accept`，回傳 101 Switching Protocols。

**幀解碼（client → server）：**處理三種遮罩長度編碼：
- 小：payload < 126 bytes
- 中：126-65535 bytes（16-bit extended）
- 大：> 65535 bytes（64-bit extended）

使用 4-byte mask key XOR 解碼 payload。回傳 `{ opcode, payload, bytesConsumed }`，若 buffer 不完整則回傳 `null`。拒絕未遮罩幀。

**幀編碼（server → client）：**不遮罩幀，使用同樣三種長度編碼。

**支援的 opcode：**TEXT（0x01）、CLOSE（0x08）、PING（0x09）、PONG（0x0A）。未識別 opcode 以狀態碼 1003（Unsupported Data）回傳 close 幀。

**刻意略過：**二進位幀、分段訊息、擴充（permessage-deflate）、子協定。這些對 localhost 間的小型 JSON 文字訊息不必要。擴充與子協定在 handshake 中協商 — 不宣告就不會啟用。

**Buffer 累積：**每個連線維護 buffer。`data` 事件時附加並重複呼叫 `decodeFrame`，直到回傳 null 或 buffer 為空。

### HTTP 伺服器

三條路由：

1. **`GET /`** — 依 mtime 提供 screen 目錄中最新的 `.html`。辨識完整文件與片段，片段包入 frame template 並注入 helper.js。回傳 `text/html`。若沒有 `.html`，回傳硬編碼等待頁（"Waiting for Claude to push a screen..."）並注入 helper.js。
2. **`GET /files/*`** — 從 screen 目錄提供靜態檔案，MIME type 由硬編碼副檔名表查找（html, css, js, png, jpg, gif, svg, json）。找不到則 404。
3. **其他** — 404。

WebSocket 升級由 HTTP 伺服器的 `'upgrade'` 事件處理，與一般 request handler 分離。

### 設定

環境變數（皆可選）：

- `BRAINSTORM_PORT` — 綁定的埠（預設：隨機高位埠 49152-65535）
- `BRAINSTORM_HOST` — 綁定介面（預設：`127.0.0.1`）
- `BRAINSTORM_URL_HOST` — 回傳 JSON 內 URL 的 hostname（預設：當 host 是 `127.0.0.1` 時為 `localhost`，否則同 host）
- `BRAINSTORM_DIR` — screen 目錄路徑（預設：`/tmp/brainstorm`）

### 啟動流程

1. 若 `SCREEN_DIR` 不存在則建立（`mkdirSync` recursive）
2. 從 `__dirname` 載入 frame template 與 helper.js
3. 以設定的 host/port 啟動 HTTP 伺服器
4. 對 `SCREEN_DIR` 啟動 `fs.watch`
5. 成功 listen 後在 stdout 輸出 `server-started` JSON：`{ type, port, host, url_host, url, screen_dir }`
6. 同樣 JSON 寫入 `SCREEN_DIR/.server-info`，讓 stdout 被隱藏（背景執行）時也能取得連線資訊

### 應用層 WebSocket 訊息

當 client 傳入 TEXT 幀：

1. 解析為 JSON。若解析失敗，寫入 stderr 並繼續。
2. 以 `{ source: 'user-event', ...event }` 輸出到 stdout。
3. 若事件包含 `choice` 屬性，將 JSON 附加到 `SCREEN_DIR/.events`（每事件一行）。

### 檔案監看

使用 `fs.watch(SCREEN_DIR)` 取代 chokidar。處理 HTML 檔案事件：

- 新檔（`rename` 事件且檔案存在）：若 `.events` 存在則刪除（`unlinkSync`），並以 JSON 輸出 `screen-added`
- 檔案變更（`change` 事件）：以 JSON 輸出 `screen-updated`（**不要**清除 `.events`）
- 兩者都要：傳送 `{ type: 'reload' }` 給所有連線的 WebSocket client

對每個檔名做約 100ms 的 debounce，以避免重複事件（macOS 與 Linux 常見）。

### 錯誤處理

- WebSocket client 的 JSON 格式錯誤：寫到 stderr 並繼續
- 未支援 opcode：以狀態 1003 close
- Client 斷線：從廣播集合移除
- `fs.watch` 錯誤：寫到 stderr 並繼續
- 無優雅關閉邏輯 — 由 shell 腳本以 SIGTERM 管理流程生命週期

## 變更內容

| 之前 | 之後 |
|---|---|
| `index.js` + `package.json` + `package-lock.json` + 714 `node_modules` 檔案 | `server.js`（單一檔案） |
| express, ws, chokidar 依賴 | 無 |
| 無靜態檔案服務 | `/files/*` 從 screen 目錄提供 |

## 不變內容

- `helper.js` — 不變
- `frame-template.html` — 不變
- `start-server.sh` — 一行更新：`index.js` → `server.js`
- `stop-server.sh` — 不變
- `visual-companion.md` — 不變
- 所有既有伺服器行為與外部合約

## 平台相容性

- `server.js` 只用跨平台 Node 內建模組
- `fs.watch` 在 macOS、Linux、Windows 的單層目錄上可靠
- Shell 腳本需要 bash（Windows 上需要 Git Bash，Claude Code 也要求）

## 測試

**單元測試**（`ws-protocol.test.js`）：透過 require `server.js` 匯出，直接測試 WebSocket 幀編碼/解碼、握手計算與協定邊界情況。

**整合測試**（`server.test.js`）：測試完整伺服器行為 — HTTP 服務、WebSocket 通訊、檔案監看、腦力激盪流程。使用 `ws` npm 套件作為測試用 client 依賴（不隨產品交付）。
