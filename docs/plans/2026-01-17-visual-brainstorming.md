# 視覺化腦力激盪輔助實作計畫

> **給 agentic workers：** 必須使用 superpowers:executing-plans 來逐步實作此計畫。

**目標：** 為 Claude 的腦力激盪工作階段提供瀏覽器視覺輔助 — 在終端對話旁展示 mockup、原型與互動式選擇。

**架構：** Claude 將 HTML 寫入暫存檔。本機 Node.js 伺服器監看該檔案並提供內容，同時自動注入 helper 函式庫。使用者互動透過 WebSocket 流向伺服器 stdout，而 Claude 會在背景任務輸出中看到。

**技術堆疊：** Node.js、Express、ws（WebSocket）、chokidar（檔案監看）

---

## 任務 1：建立伺服器基礎

**檔案：**
- 新增：`lib/brainstorm-server/index.js`
- 新增：`lib/brainstorm-server/package.json`

**步驟 1：建立 package.json**

```json
{
  "name": "brainstorm-server",
  "version": "1.0.0",
  "description": "Visual brainstorming companion server for Claude Code",
  "main": "index.js",
  "dependencies": {
    "chokidar": "^3.5.3",
    "express": "^4.18.2",
    "ws": "^8.14.2"
  }
}
```

**步驟 2：建立可啟動的最小伺服器**

```javascript
const express = require('express');
const http = require('http');
const WebSocket = require('ws');
const chokidar = require('chokidar');
const fs = require('fs');
const path = require('path');

const PORT = process.env.BRAINSTORM_PORT || 3333;
const SCREEN_FILE = process.env.BRAINSTORM_SCREEN || '/tmp/brainstorm/screen.html';
const SCREEN_DIR = path.dirname(SCREEN_FILE);

// Ensure screen directory exists
if (!fs.existsSync(SCREEN_DIR)) {
  fs.mkdirSync(SCREEN_DIR, { recursive: true });
}

// Create default screen if none exists
if (!fs.existsSync(SCREEN_FILE)) {
  fs.writeFileSync(SCREEN_FILE, `<!DOCTYPE html>
<html>
<head>
  <title>Brainstorm Companion</title>
  <style>
    body { font-family: system-ui, sans-serif; padding: 2rem; max-width: 800px; margin: 0 auto; }
    h1 { color: #333; }
    p { color: #666; }
  </style>
</head>
<body>
  <h1>Brainstorm Companion</h1>
  <p>Waiting for Claude to push a screen...</p>
</body>
</html>`);
}

const app = express();
const server = http.createServer(app);
const wss = new WebSocket.Server({ server });

// Track connected browsers for reload notifications
const clients = new Set();

wss.on('connection', (ws) => {
  clients.add(ws);
  ws.on('close', () => clients.delete(ws));

  ws.on('message', (data) => {
    // User interaction event - write to stdout for Claude
    const event = JSON.parse(data.toString());
    console.log(JSON.stringify({ type: 'user-event', ...event }));
  });
});

// Serve current screen with helper.js injected
app.get('/', (req, res) => {
  let html = fs.readFileSync(SCREEN_FILE, 'utf-8');

  // Inject helper script before </body>
  const helperScript = fs.readFileSync(path.join(__dirname, 'helper.js'), 'utf-8');
  const injection = `<script>\n${helperScript}\n</script>`;

  if (html.includes('</body>')) {
    html = html.replace('</body>', `${injection}\n</body>`);
  } else {
    html += injection;
  }

  res.type('html').send(html);
});

// Watch for screen file changes
chokidar.watch(SCREEN_FILE).on('change', () => {
  console.log(JSON.stringify({ type: 'screen-updated', file: SCREEN_FILE }));
  // Notify all browsers to reload
  clients.forEach(ws => {
    if (ws.readyState === WebSocket.OPEN) {
      ws.send(JSON.stringify({ type: 'reload' }));
    }
  });
});

server.listen(PORT, '127.0.0.1', () => {
  console.log(JSON.stringify({ type: 'server-started', port: PORT, url: `http://localhost:${PORT}` }));
});
```

**步驟 3：執行 npm install**

執行：`cd lib/brainstorm-server && npm install`
預期：相依套件已安裝

**步驟 4：測試伺服器可啟動**

執行：`cd lib/brainstorm-server && timeout 3 node index.js || true`
預期：看到包含 `server-started` 與連接埠資訊的 JSON

**步驟 5：提交**

```bash
git add lib/brainstorm-server/
git commit -m "feat: add brainstorm server foundation"
```

---

## 任務 2：建立 Helper 函式庫

**檔案：**
- 新增：`lib/brainstorm-server/helper.js`

**步驟 1：建立 helper.js 並自動擷取事件**

```javascript
(function() {
  const WS_URL = 'ws://' + window.location.host;
  let ws = null;
  let eventQueue = [];

  function connect() {
    ws = new WebSocket(WS_URL);

    ws.onopen = () => {
      // Send any queued events
      eventQueue.forEach(e => ws.send(JSON.stringify(e)));
      eventQueue = [];
    };

    ws.onmessage = (msg) => {
      const data = JSON.parse(msg.data);
      if (data.type === 'reload') {
        window.location.reload();
      }
    };

    ws.onclose = () => {
      // Reconnect after 1 second
      setTimeout(connect, 1000);
    };
  }

  function send(event) {
    event.timestamp = Date.now();
    if (ws && ws.readyState === WebSocket.OPEN) {
      ws.send(JSON.stringify(event));
    } else {
      eventQueue.push(event);
    }
  }

  // Auto-capture clicks on interactive elements
  document.addEventListener('click', (e) => {
    const target = e.target.closest('button, a, [data-choice], [role="button"], input[type="submit"]');
    if (!target) return;

    // Don't capture regular link navigation
    if (target.tagName === 'A' && !target.dataset.choice) return;

    e.preventDefault();

    send({
      type: 'click',
      text: target.textContent.trim(),
      choice: target.dataset.choice || null,
      id: target.id || null,
      className: target.className || null
    });
  });

  // Auto-capture form submissions
  document.addEventListener('submit', (e) => {
    e.preventDefault();
    const form = e.target;
    const formData = new FormData(form);
    const data = {};
    formData.forEach((value, key) => { data[key] = value; });

    send({
      type: 'submit',
      formId: form.id || null,
      formName: form.name || null,
      data: data
    });
  });

  // Auto-capture input changes (debounced)
  let inputTimeout = null;
  document.addEventListener('input', (e) => {
    const target = e.target;
    if (!target.matches('input, textarea, select')) return;

    clearTimeout(inputTimeout);
    inputTimeout = setTimeout(() => {
      send({
        type: 'input',
        name: target.name || null,
        id: target.id || null,
        value: target.value,
        inputType: target.type || target.tagName.toLowerCase()
      });
    }, 500); // 500ms debounce
  });

  // Expose for explicit use if needed
  window.brainstorm = {
    send: send,
    choice: (value, metadata = {}) => send({ type: 'choice', value, ...metadata })
  };

  connect();
})();
```

**步驟 2：驗證 helper.js 語法正確**

執行：`node -c lib/brainstorm-server/helper.js`
預期：沒有語法錯誤

**步驟 3：提交**

```bash
git add lib/brainstorm-server/helper.js
git commit -m "feat: add browser helper library for event capture"
```

---

## 任務 3：為伺服器撰寫測試

**檔案：**
- 新增：`tests/brainstorm-server/server.test.js`
- 新增：`tests/brainstorm-server/package.json`

**步驟 1：建立測試用 package.json**

```json
{
  "name": "brainstorm-server-tests",
  "version": "1.0.0",
  "scripts": {
    "test": "node server.test.js"
  }
}
```

**步驟 2：撰寫伺服器測試**

```javascript
const { spawn } = require('child_process');
const http = require('http');
const WebSocket = require('ws');
const fs = require('fs');
const path = require('path');
const assert = require('assert');

const SERVER_PATH = path.join(__dirname, '../../lib/brainstorm-server/index.js');
const TEST_PORT = 3334;
const TEST_SCREEN = '/tmp/brainstorm-test/screen.html';

// Clean up test directory
function cleanup() {
  if (fs.existsSync(path.dirname(TEST_SCREEN))) {
    fs.rmSync(path.dirname(TEST_SCREEN), { recursive: true });
  }
}

async function sleep(ms) {
  return new Promise(resolve => setTimeout(resolve, ms));
}

async function fetch(url) {
  return new Promise((resolve, reject) => {
    http.get(url, (res) => {
      let data = '';
      res.on('data', chunk => data += chunk);
      res.on('end', () => resolve({ status: res.statusCode, body: data }));
    }).on('error', reject);
  });
}

async function runTests() {
  cleanup();

  // Start server
  const server = spawn('node', [SERVER_PATH], {
    env: { ...process.env, BRAINSTORM_PORT: TEST_PORT, BRAINSTORM_SCREEN: TEST_SCREEN }
  });

  let stdout = '';
  server.stdout.on('data', (data) => { stdout += data.toString(); });
  server.stderr.on('data', (data) => { console.error('Server stderr:', data.toString()); });

  await sleep(1000); // Wait for server to start

  try {
    // Test 1: Server starts and outputs JSON
    console.log('Test 1: Server startup message');
    assert(stdout.includes('server-started'), 'Should output server-started');
    assert(stdout.includes(TEST_PORT.toString()), 'Should include port');
    console.log('  PASS');

    // Test 2: GET / returns HTML with helper injected
    console.log('Test 2: Serves HTML with helper injected');
    const res = await fetch(`http://localhost:${TEST_PORT}/`);
    assert.strictEqual(res.status, 200);
    assert(res.body.includes('brainstorm'), 'Should include brainstorm content');
    assert(res.body.includes('WebSocket'), 'Should have helper.js injected');
    console.log('  PASS');

    // Test 3: WebSocket connection and event relay
    console.log('Test 3: WebSocket relays events to stdout');
    stdout = ''; // Reset stdout capture
    const ws = new WebSocket(`ws://localhost:${TEST_PORT}`);
    await new Promise(resolve => ws.on('open', resolve));

    ws.send(JSON.stringify({ type: 'click', text: 'Test Button' }));
    await sleep(100);

    assert(stdout.includes('user-event'), 'Should relay user events');
    assert(stdout.includes('Test Button'), 'Should include event data');
    ws.close();
    console.log('  PASS');

    // Test 4: File change triggers reload notification
    console.log('Test 4: File change notifies browsers');
    const ws2 = new WebSocket(`ws://localhost:${TEST_PORT}`);
    await new Promise(resolve => ws2.on('open', resolve));

    let gotReload = false;
    ws2.on('message', (data) => {
      const msg = JSON.parse(data.toString());
      if (msg.type === 'reload') gotReload = true;
    });

    // Modify the screen file
    fs.writeFileSync(TEST_SCREEN, '<html><body>Updated</body></html>');
    await sleep(500);

    assert(gotReload, 'Should send reload message on file change');
    ws2.close();
    console.log('  PASS');

    console.log('\nAll tests passed!');

  } finally {
    server.kill();
    cleanup();
  }
}

runTests().catch(err => {
  console.error('Test failed:', err);
  process.exit(1);
});
```

**步驟 3：執行測試**

執行：`cd tests/brainstorm-server && npm install ws && node server.test.js`
預期：所有測試通過

**步驟 4：提交**

```bash
git add tests/brainstorm-server/
git commit -m "test: add brainstorm server integration tests"
```

---

## 任務 4：將視覺輔助加入 brainstorming 技能

**檔案：**
- 修改：`skills/brainstorming/SKILL.md`
- 新增：`skills/brainstorming/visual-companion.md`（支援文件）

**步驟 1：建立支援文件**

建立 `skills/brainstorming/visual-companion.md`：

```markdown
# Visual Companion Reference

## Starting the Server

Run as a background job:

```bash
node ${PLUGIN_ROOT}/lib/brainstorm-server/index.js
```

Tell the user: "I've started a visual companion at http://localhost:3333 - open it in a browser."

## Pushing Screens

Write HTML to `/tmp/brainstorm/screen.html`. The server watches this file and auto-refreshes the browser.

## Reading User Responses

Check the background task output for JSON events:

```json
{"type":"user-event","type":"click","text":"Option A","choice":"optionA","timestamp":1234567890}
{"type":"user-event","type":"submit","data":{"notes":"My feedback"},"timestamp":1234567891}
```

Event types:
- **click**: User clicked button or `data-choice` element
- **submit**: User submitted form (includes all form data)
- **input**: User typed in field (debounced 500ms)

## HTML Patterns

### Choice Cards

```html
<div class="options">
  <button data-choice="optionA">
    <h3>Option A</h3>
    <p>Description</p>
  </button>
  <button data-choice="optionB">
    <h3>Option B</h3>
    <p>Description</p>
  </button>
</div>
```

### Interactive Mockup

```html
<div class="mockup">
  <header data-choice="header">App Header</header>
  <nav data-choice="nav">Navigation</nav>
  <main data-choice="main">Content</main>
</div>
```

### Form with Notes

```html
<form>
  <label>Priority: <input type="range" name="priority" min="1" max="5"></label>
  <textarea name="notes" placeholder="Additional thoughts..."></textarea>
  <button type="submit">Submit</button>
</form>
```

### Explicit JavaScript

```html
<button onclick="brainstorm.choice('custom', {extra: 'data'})">Custom</button>
```
```

**步驟 2：在 brainstorming 技能加入視覺輔助段落**

加入在 `skills/brainstorming/SKILL.md` 的「Key Principles」之後：

```markdown

## Visual Companion (Optional)

When brainstorming involves visual elements - UI mockups, wireframes, interactive prototypes - use the browser-based visual companion.

**When to use:**
- Presenting UI/UX options that benefit from visual comparison
- Showing wireframes or layout options
- Gathering structured feedback (ratings, forms)
- Prototyping click interactions

**How it works:**
1. Start the server as a background job
2. Tell user to open http://localhost:3333
3. Write HTML to `/tmp/brainstorm/screen.html` (auto-refreshes)
4. Check background task output for user interactions

The terminal remains the primary conversation interface. The browser is a visual aid.

**Reference:** See `visual-companion.md` in this skill directory for HTML patterns and API details.
```

**步驟 3：確認編輯結果**

執行：`grep -A5 "Visual Companion" skills/brainstorming/SKILL.md`
預期：顯示新增段落

**步驟 4：提交**

```bash
git add skills/brainstorming/
git commit -m "feat: add visual companion to brainstorming skill"
```

---

## 任務 5：將伺服器加入外掛忽略清單（可選清理）

**檔案：**
- 檢查 `.gitignore` 是否需要排除 lib/brainstorm-server 的 node_modules

**步驟 1：檢查現有 gitignore**

執行：`cat .gitignore 2>/dev/null || echo "No .gitignore"`

**步驟 2：必要時加入 node_modules**

若尚未存在，加入：
```
lib/brainstorm-server/node_modules/
```

**步驟 3：若有修改則提交**

```bash
git add .gitignore
git commit -m "chore: ignore brainstorm-server node_modules"
```

---

## 摘要

完成所有任務後：

1. **伺服器** 位於 `lib/brainstorm-server/` - 監看 HTML 檔並轉送事件的 Node.js 伺服器
2. **Helper 函式庫** 自動注入 - 擷取點擊、表單、輸入
3. **測試** 位於 `tests/brainstorm-server/` - 驗證伺服器行為
4. **brainstorming 技能** 已更新加入視覺輔助段落與 `visual-companion.md` 參考文件

**使用方式：**
1. 以背景工作啟動伺服器：`node lib/brainstorm-server/index.js &`
2. 提醒使用者開啟 `http://localhost:3333`
3. 將 HTML 寫入 `/tmp/brainstorm/screen.html`
4. 從任務輸出查看使用者事件
