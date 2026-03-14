# 零相依腦力激盪伺服器實作計畫

> **給 agentic workers：** 必須使用 superpowers:subagent-driven-development（若可用子代理）或 superpowers:executing-plans 來逐步實作此計畫。步驟使用核取方塊（`- [ ]`）語法以追蹤。

**目標：** 以單一零相依 `server.js`（僅使用 Node 內建）取代腦力激盪伺服器中隨附的 node_modules。

**架構：** 單一檔案，包含 WebSocket 協定（RFC 6455 文字框）、HTTP 伺服器（`http` 模組）與檔案監看（`fs.watch`）。在需要做單元測試時，作為模組匯出協定函式。

**技術堆疊：** 僅使用 Node.js 內建：`http`、`crypto`、`fs`、`path`

**規格：** `docs/superpowers/specs/2026-03-11-zero-dep-brainstorm-server-design.md`

**既有測試：** `tests/brainstorm-server/ws-protocol.test.js`（單元）、`tests/brainstorm-server/server.test.js`（整合）

---

## 檔案對照

- **新增：** `skills/brainstorming/scripts/server.js` — 零相依替代品
- **修改：** `skills/brainstorming/scripts/start-server.sh:94,100` — 將 `index.js` 改為 `server.js`
- **修改：** `.gitignore:6` — 移除 `!skills/brainstorming/scripts/node_modules/` 例外
- **刪除：** `skills/brainstorming/scripts/index.js`
- **刪除：** `skills/brainstorming/scripts/package.json`
- **刪除：** `skills/brainstorming/scripts/package-lock.json`
- **刪除：** `skills/brainstorming/scripts/node_modules/`（714 個檔案）
- **不變：** `skills/brainstorming/scripts/helper.js`、`skills/brainstorming/scripts/frame-template.html`、`skills/brainstorming/scripts/stop-server.sh`

---

## 區塊 1：WebSocket 協定層

### 任務 1：實作 WebSocket 協定輸出

**檔案：**
- 新增：`skills/brainstorming/scripts/server.js`
- 測試：`tests/brainstorm-server/ws-protocol.test.js`（已存在）

- [ ] **步驟 1：建立 server.js，加入 OPCODES 常數與 computeAcceptKey**

```js
const crypto = require('crypto');

const OPCODES = { TEXT: 0x01, CLOSE: 0x08, PING: 0x09, PONG: 0x0A };
const WS_MAGIC = '258EAFA5-E914-47DA-95CA-C5AB0DC85B11';

function computeAcceptKey(clientKey) {
  return crypto.createHash('sha1').update(clientKey + WS_MAGIC).digest('base64');
}
```

- [ ] **步驟 2：實作 encodeFrame**

伺服器送出的 frame 永不遮罩。共有三種長度編碼：
- payload < 126：2-byte header（FIN+opcode, length）
- 126-65535：4-byte header（FIN+opcode, 126, 16-bit length）
- &gt; 65535：10-byte header（FIN+opcode, 127, 64-bit length）

```js
function encodeFrame(opcode, payload) {
  const fin = 0x80;
  const len = payload.length;
  let header;

  if (len < 126) {
    header = Buffer.alloc(2);
    header[0] = fin | opcode;
    header[1] = len;
  } else if (len < 65536) {
    header = Buffer.alloc(4);
    header[0] = fin | opcode;
    header[1] = 126;
    header.writeUInt16BE(len, 2);
  } else {
    header = Buffer.alloc(10);
    header[0] = fin | opcode;
    header[1] = 127;
    header.writeBigUInt64BE(BigInt(len), 2);
  }

  return Buffer.concat([header, payload]);
}
```

- [ ] **步驟 3：實作 decodeFrame**

用戶端送出的 frame 必須遮罩。回傳 `{ opcode, payload, bytesConsumed }` 或在資料不足時回傳 `null`。若 frame 未遮罩則丟出錯誤。

```js
function decodeFrame(buffer) {
  if (buffer.length < 2) return null;

  const firstByte = buffer[0];
  const secondByte = buffer[1];
  const opcode = firstByte & 0x0F;
  const masked = (secondByte & 0x80) !== 0;
  let payloadLen = secondByte & 0x7F;
  let offset = 2;

  if (!masked) throw new Error('Client frames must be masked');

  if (payloadLen === 126) {
    if (buffer.length < 4) return null;
    payloadLen = buffer.readUInt16BE(2);
    offset = 4;
  } else if (payloadLen === 127) {
    if (buffer.length < 10) return null;
    payloadLen = Number(buffer.readBigUInt64BE(2));
    offset = 10;
  }

  const maskOffset = offset;
  const dataOffset = offset + 4;
  const totalLen = dataOffset + payloadLen;
  if (buffer.length < totalLen) return null;

  const mask = buffer.slice(maskOffset, dataOffset);
  const data = Buffer.alloc(payloadLen);
  for (let i = 0; i < payloadLen; i++) {
    data[i] = buffer[dataOffset + i] ^ mask[i % 4];
  }

  return { opcode, payload: data, bytesConsumed: totalLen };
}
```

- [ ] **步驟 4：在檔案底部加入模組匯出**

```js
module.exports = { computeAcceptKey, encodeFrame, decodeFrame, OPCODES };
```

- [ ] **步驟 5：執行單元測試**

執行：`cd tests/brainstorm-server && node ws-protocol.test.js`
預期：所有測試通過（握手、編碼、解碼、邊界與 edge cases）

- [ ] **步驟 6：提交**

```bash
git add skills/brainstorming/scripts/server.js
git commit -m "Add WebSocket protocol layer for zero-dep brainstorm server"
```

---

## 區塊 2：HTTP 伺服器與應用邏輯

### 任務 2：加入 HTTP 伺服器、檔案監看與 WebSocket 連線處理

**檔案：**
- 修改：`skills/brainstorming/scripts/server.js`
- 測試：`tests/brainstorm-server/server.test.js`（已存在）

- [ ] **步驟 1：在 server.js 頂部（require 之後）加入設定與常數**

```js
const http = require('http');
const fs = require('fs');
const path = require('path');

const PORT = process.env.BRAINSTORM_PORT || (49152 + Math.floor(Math.random() * 16383));
const HOST = process.env.BRAINSTORM_HOST || '127.0.0.1';
const URL_HOST = process.env.BRAINSTORM_URL_HOST || (HOST === '127.0.0.1' ? 'localhost' : HOST);
const SCREEN_DIR = process.env.BRAINSTORM_DIR || '/tmp/brainstorm';

const MIME_TYPES = {
  '.html': 'text/html', '.css': 'text/css', '.js': 'application/javascript',
  '.json': 'application/json', '.png': 'image/png', '.jpg': 'image/jpeg',
  '.jpeg': 'image/jpeg', '.gif': 'image/gif', '.svg': 'image/svg+xml'
};
```

- [ ] **步驟 2：加入 WAITING_PAGE、在模組層載入模板與輔助函式**

在模組層載入 `frameTemplate` 與 `helperInjection`，讓 `wrapInFrame` 與 `handleRequest` 可使用。它們只從 `__dirname`（scripts 目錄）讀取檔案，不論模組被 require 或直接執行都成立。

```js
const WAITING_PAGE = `<!DOCTYPE html>
<html>
<head><title>Brainstorm Companion</title>
<style>body { font-family: system-ui, sans-serif; padding: 2rem; max-width: 800px; margin: 0 auto; }
h1 { color: #333; } p { color: #666; }</style>
</head>
<body><h1>Brainstorm Companion</h1>
<p>Waiting for Claude to push a screen...</p></body></html>`;

const frameTemplate = fs.readFileSync(path.join(__dirname, 'frame-template.html'), 'utf-8');
const helperScript = fs.readFileSync(path.join(__dirname, 'helper.js'), 'utf-8');
const helperInjection = '<script>\n' + helperScript + '\n</script>';

function isFullDocument(html) {
  const trimmed = html.trimStart().toLowerCase();
  return trimmed.startsWith('<!doctype') || trimmed.startsWith('<html');
}

function wrapInFrame(content) {
  return frameTemplate.replace('<!-- CONTENT -->', content);
}

function getNewestScreen() {
  const files = fs.readdirSync(SCREEN_DIR)
    .filter(f => f.endsWith('.html'))
    .map(f => {
      const fp = path.join(SCREEN_DIR, f);
      return { path: fp, mtime: fs.statSync(fp).mtime.getTime() };
    })
    .sort((a, b) => b.mtime - a.mtime);
  return files.length > 0 ? files[0].path : null;
}
```

- [ ] **步驟 3：加入 HTTP request handler**

```js
function handleRequest(req, res) {
  if (req.method === 'GET' && req.url === '/') {
    const screenFile = getNewestScreen();
    let html = screenFile
      ? (raw => isFullDocument(raw) ? raw : wrapInFrame(raw))(fs.readFileSync(screenFile, 'utf-8'))
      : WAITING_PAGE;

    if (html.includes('</body>')) {
      html = html.replace('</body>', helperInjection + '\n</body>');
    } else {
      html += helperInjection;
    }

    res.writeHead(200, { 'Content-Type': 'text/html' });
    res.end(html);
  } else if (req.method === 'GET' && req.url.startsWith('/files/')) {
    const fileName = req.url.slice(7); // strip '/files/'
    const filePath = path.join(SCREEN_DIR, path.basename(fileName));
    if (!fs.existsSync(filePath)) {
      res.writeHead(404);
      res.end('Not found');
      return;
    }
    const ext = path.extname(filePath).toLowerCase();
    const contentType = MIME_TYPES[ext] || 'application/octet-stream';
    res.writeHead(200, { 'Content-Type': contentType });
    res.end(fs.readFileSync(filePath));
  } else {
    res.writeHead(404);
    res.end('Not found');
  }
}
```

- [ ] **步驟 4：加入 WebSocket 連線處理**

```js
const clients = new Set();

function handleUpgrade(req, socket) {
  const key = req.headers['sec-websocket-key'];
  if (!key) { socket.destroy(); return; }

  const accept = computeAcceptKey(key);
  socket.write(
    'HTTP/1.1 101 Switching Protocols\r\n' +
    'Upgrade: websocket\r\n' +
    'Connection: Upgrade\r\n' +
    'Sec-WebSocket-Accept: ' + accept + '\r\n\r\n'
  );

  let buffer = Buffer.alloc(0);
  clients.add(socket);

  socket.on('data', (chunk) => {
    buffer = Buffer.concat([buffer, chunk]);
    while (buffer.length > 0) {
      let result;
      try {
        result = decodeFrame(buffer);
      } catch (e) {
        socket.end(encodeFrame(OPCODES.CLOSE, Buffer.alloc(0)));
        clients.delete(socket);
        return;
      }
      if (!result) break;
      buffer = buffer.slice(result.bytesConsumed);

      switch (result.opcode) {
        case OPCODES.TEXT:
          handleMessage(result.payload.toString());
          break;
        case OPCODES.CLOSE:
          socket.end(encodeFrame(OPCODES.CLOSE, Buffer.alloc(0)));
          clients.delete(socket);
          return;
        case OPCODES.PING:
          socket.write(encodeFrame(OPCODES.PONG, result.payload));
          break;
        case OPCODES.PONG:
          break;
        default:
          // Unsupported opcode — close with 1003
          const closeBuf = Buffer.alloc(2);
          closeBuf.writeUInt16BE(1003);
          socket.end(encodeFrame(OPCODES.CLOSE, closeBuf));
          clients.delete(socket);
          return;
      }
    }
  });

  socket.on('close', () => clients.delete(socket));
  socket.on('error', () => clients.delete(socket));
}

function handleMessage(text) {
  let event;
  try {
    event = JSON.parse(text);
  } catch (e) {
    console.error('Failed to parse WebSocket message:', e.message);
    return;
  }
  console.log(JSON.stringify({ source: 'user-event', ...event }));
  if (event.choice) {
    const eventsFile = path.join(SCREEN_DIR, '.events');
    fs.appendFileSync(eventsFile, JSON.stringify(event) + '\n');
  }
}

function broadcast(msg) {
  const frame = encodeFrame(OPCODES.TEXT, Buffer.from(JSON.stringify(msg)));
  for (const socket of clients) {
    try { socket.write(frame); } catch (e) { clients.delete(socket); }
  }
}
```

- [ ] **步驟 5：加入防抖計時器 Map**

```js
const debounceTimers = new Map();
```

檔案監看邏輯內嵌於 `startServer`（步驟 6）中，以確保 watcher 生命週期與伺服器一致，並依規格加入 `error` handler。

- [ ] **步驟 6：加入 startServer 函式與條件式 main**

`frameTemplate` 與 `helperInjection` 已於模組層（步驟 2）定義。`startServer` 只需建立畫面目錄、啟動 HTTP 伺服器與 watcher，並輸出啟動資訊。

```js
function startServer() {
  if (!fs.existsSync(SCREEN_DIR)) fs.mkdirSync(SCREEN_DIR, { recursive: true });

  const server = http.createServer(handleRequest);
  server.on('upgrade', handleUpgrade);

  const watcher = fs.watch(SCREEN_DIR, (eventType, filename) => {
    if (!filename || !filename.endsWith('.html')) return;
    if (debounceTimers.has(filename)) clearTimeout(debounceTimers.get(filename));
    debounceTimers.set(filename, setTimeout(() => {
      debounceTimers.delete(filename);
      const filePath = path.join(SCREEN_DIR, filename);
      if (eventType === 'rename' && fs.existsSync(filePath)) {
        const eventsFile = path.join(SCREEN_DIR, '.events');
        if (fs.existsSync(eventsFile)) fs.unlinkSync(eventsFile);
        console.log(JSON.stringify({ type: 'screen-added', file: filePath }));
      } else if (eventType === 'change') {
        console.log(JSON.stringify({ type: 'screen-updated', file: filePath }));
      }
      broadcast({ type: 'reload' });
    }, 100));
  });
  watcher.on('error', (err) => console.error('fs.watch error:', err.message));

  server.listen(PORT, HOST, () => {
    const info = JSON.stringify({
      type: 'server-started', port: Number(PORT), host: HOST,
      url_host: URL_HOST, url: 'http://' + URL_HOST + ':' + PORT,
      screen_dir: SCREEN_DIR
    });
    console.log(info);
    fs.writeFileSync(path.join(SCREEN_DIR, '.server-info'), info + '\n');
  });
}

if (require.main === module) {
  startServer();
}
```

- [ ] **步驟 7：執行整合測試**

測試目錄已包含 `package.json`（依賴 `ws`）。如有需要先安裝，再執行測試。

執行：`cd tests/brainstorm-server && npm install && node server.test.js`
預期：所有測試通過

- [ ] **步驟 8：提交**

```bash
git add skills/brainstorming/scripts/server.js
git commit -m "Add HTTP server, WebSocket handling, and file watching to server.js"
```

---

## 區塊 3：替換與清理

### 任務 3：更新 start-server.sh 並移除舊檔案

**檔案：**
- 修改：`skills/brainstorming/scripts/start-server.sh:94,100`
- 修改：`.gitignore:6`
- 刪除：`skills/brainstorming/scripts/index.js`
- 刪除：`skills/brainstorming/scripts/package.json`
- 刪除：`skills/brainstorming/scripts/package-lock.json`
- 刪除：`skills/brainstorming/scripts/node_modules/`（整個目錄）

- [ ] **步驟 1：更新 start-server.sh — 將 `index.js` 改為 `server.js`**

需修改兩行：

第 94 行：`env BRAINSTORM_DIR="$SCREEN_DIR" BRAINSTORM_HOST="$BIND_HOST" BRAINSTORM_URL_HOST="$URL_HOST" node server.js`

第 100 行：`nohup env BRAINSTORM_DIR="$SCREEN_DIR" BRAINSTORM_HOST="$BIND_HOST" BRAINSTORM_URL_HOST="$URL_HOST" node server.js > "$LOG_FILE" 2>&1 &`

- [ ] **步驟 2：移除 gitignore 對 node_modules 的例外**

在 `.gitignore` 中刪除第 6 行：`!skills/brainstorming/scripts/node_modules/`

- [ ] **步驟 3：刪除舊檔案**

```bash
git rm skills/brainstorming/scripts/index.js
git rm skills/brainstorming/scripts/package.json
git rm skills/brainstorming/scripts/package-lock.json
git rm -r skills/brainstorming/scripts/node_modules/
```

- [ ] **步驟 4：執行兩個測試套件**

執行：`cd tests/brainstorm-server && node ws-protocol.test.js && node server.test.js`
預期：所有測試通過

- [ ] **步驟 5：提交**

```bash
git add skills/brainstorming/scripts/ .gitignore
git commit -m "Remove vendored node_modules, swap to zero-dep server.js"
```

### 任務 4：手動煙霧測試

- [ ] **步驟 1：手動啟動伺服器**

```bash
cd skills/brainstorming/scripts
BRAINSTORM_DIR=/tmp/brainstorm-smoke BRAINSTORM_PORT=9876 node server.js
```

預期：輸出包含 port 9876 的 `server-started` JSON

- [ ] **步驟 2：在瀏覽器開啟 http://localhost:9876**

預期：顯示「Waiting for Claude to push a screen...」的等待頁面

- [ ] **步驟 3：在 screen 目錄寫入 HTML 檔**

```bash
echo '<h2>Hello from smoke test</h2>' > /tmp/brainstorm-smoke/test.html
```

預期：瀏覽器重新載入並顯示被框架模板包裹的「Hello from smoke test」

- [ ] **步驟 4：確認 WebSocket 正常 — 查看瀏覽器 console**

開啟瀏覽器開發者工具。WebSocket 連線應顯示為 connected（console 無錯誤）。框架模板的狀態指示應顯示「Connected」。

- [ ] **步驟 5：Ctrl-C 停止伺服器並清理**

```bash
rm -rf /tmp/brainstorm-smoke
```
