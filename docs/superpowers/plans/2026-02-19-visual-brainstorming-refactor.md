# 視覺化腦力激盪重構實作計畫

> **給 agentic workers：** 必須使用 superpowers:subagent-driven-development（若可用子代理）或 superpowers:executing-plans 來實作此計畫。步驟使用核取方塊（`- [ ]`）語法以追蹤。

**目標：** 將視覺化腦力激盪從阻塞式 TUI 回饋模型，重構為非阻塞的「瀏覽器顯示、終端指令」架構。

**架構：** 瀏覽器成為互動顯示；終端保持對話通道。伺服器把使用者事件寫到每個畫面的 `.events` 檔，Claude 在下一回合讀取。移除 `wait-for-feedback.sh` 與所有 `TaskOutput` 阻塞。

**技術堆疊：** Node.js（Express、ws、chokidar）、原生 HTML/CSS/JS

**規格：** `docs/superpowers/specs/2026-02-19-visual-brainstorming-refactor-design.md`

---

## 檔案對照

| 檔案 | 動作 | 責任 |
|------|------|------|
| `lib/brainstorm-server/index.js` | 修改 | 伺服器：新增 `.events` 檔寫入、新畫面時清除、取代 `wrapInFrame` |
| `lib/brainstorm-server/frame-template.html` | 修改 | 模板：移除回饋頁尾，加入內容占位與選擇指示器 |
| `lib/brainstorm-server/helper.js` | 修改 | 用戶端 JS：移除 send/feedback 函式，縮小為點擊擷取 + 指示器更新 |
| `lib/brainstorm-server/wait-for-feedback.sh` | 刪除 | 不再需要 |
| `skills/brainstorming/visual-companion.md` | 修改 | 技能指示：改寫為非阻塞流程 |
| `tests/brainstorm-server/server.test.js` | 修改 | 測試：更新模板結構與 helper.js API |

---

## 區塊 1：伺服器、模板、用戶端、測試、技能

### 任務 1：更新 `frame-template.html`

**檔案：**
- 修改：`lib/brainstorm-server/frame-template.html`

- [ ] **步驟 1：移除回饋頁尾 HTML**

用選擇指示器列取代 feedback-footer div（第 227-233 行）：

```html
  <div class="indicator-bar">
    <span id="indicator-text">Click an option above, then return to the terminal</span>
  </div>
```

同時將 `#claude-content` 內的預設內容（第 220-223 行）替換為內容占位：

```html
    <div id="claude-content">
      <!-- CONTENT -->
    </div>
```

- [ ] **步驟 2：以指示器列 CSS 取代回饋頁尾 CSS**

移除 `.feedback-footer`、`.feedback-footer label`、`.feedback-row`，以及 `.feedback-footer` 內 textarea/button 的樣式（第 82-112 行）。

新增指示器列 CSS：

```css
    .indicator-bar {
      background: var(--bg-secondary);
      border-top: 1px solid var(--border);
      padding: 0.5rem 1.5rem;
      flex-shrink: 0;
      text-align: center;
    }
    .indicator-bar span {
      font-size: 0.75rem;
      color: var(--text-secondary);
    }
    .indicator-bar .selected-text {
      color: var(--accent);
      font-weight: 500;
    }
```

- [ ] **步驟 3：確認模板可渲染**

執行測試套件，確認模板仍可載入：
```bash
cd /Users/drewritter/prime-rad/superpowers && node tests/brainstorm-server/server.test.js
```
預期：測試 1-5 應仍可通過。測試 6-8 可能失敗（預期 — 它們仍在驗證舊結構）。

- [ ] **步驟 4：提交**

```bash
git add lib/brainstorm-server/frame-template.html
git commit -m "Replace feedback footer with selection indicator bar in brainstorm template"
```

---

### 任務 2：更新 `index.js` — 內容注入與 `.events` 檔

**檔案：**
- 修改：`lib/brainstorm-server/index.js`

- [ ] **步驟 1：為 `.events` 檔寫入加入失敗測試**

在 `tests/brainstorm-server/server.test.js` 的 Test 4 區段後新增測試：送出包含 `choice` 的 WebSocket 事件，並驗證 `.events` 檔被寫入：

```javascript
    // Test: Choice events written to .events file
    console.log('Test: Choice events written to .events file');
    const ws3 = new WebSocket(`ws://localhost:${TEST_PORT}`);
    await new Promise(resolve => ws3.on('open', resolve));

    ws3.send(JSON.stringify({ type: 'click', choice: 'a', text: 'Option A' }));
    await sleep(300);

    const eventsFile = path.join(TEST_DIR, '.events');
    assert(fs.existsSync(eventsFile), '.events file should exist after choice click');
    const lines = fs.readFileSync(eventsFile, 'utf-8').trim().split('\n');
    const event = JSON.parse(lines[lines.length - 1]);
    assert.strictEqual(event.choice, 'a', 'Event should contain choice');
    assert.strictEqual(event.text, 'Option A', 'Event should contain text');
    ws3.close();
    console.log('  PASS');
```

- [ ] **步驟 2：執行測試確認失敗**

```bash
cd /Users/drewritter/prime-rad/superpowers && node tests/brainstorm-server/server.test.js
```
預期：新測試會失敗 — `.events` 尚不存在。

- [ ] **步驟 3：為 `.events` 新畫面清除加入失敗測試**

新增另一個測試：

```javascript
    // Test: .events cleared on new screen
    console.log('Test: .events cleared on new screen');
    // .events file should still exist from previous test
    assert(fs.existsSync(path.join(TEST_DIR, '.events')), '.events should exist before new screen');
    fs.writeFileSync(path.join(TEST_DIR, 'new-screen.html'), '<h2>New screen</h2>');
    await sleep(500);
    assert(!fs.existsSync(path.join(TEST_DIR, '.events')), '.events should be cleared after new screen');
    console.log('  PASS');
```

- [ ] **步驟 4：執行測試確認失敗**

```bash
cd /Users/drewritter/prime-rad/superpowers && node tests/brainstorm-server/server.test.js
```
預期：新測試會失敗 — `.events` 沒有在推送新畫面時清除。

- [ ] **步驟 5：在 `index.js` 實作 `.events` 檔寫入**

在 WebSocket `message` handler（`index.js` 第 74-77 行）中，於 `console.log` 後加入：

```javascript
    // Write user events to .events file for Claude to read
    if (event.choice) {
      const eventsFile = path.join(SCREEN_DIR, '.events');
      fs.appendFileSync(eventsFile, JSON.stringify(event) + '\n');
    }
```

在 chokidar `add` handler（第 104-111 行）加入 `.events` 清除：

```javascript
    if (filePath.endsWith('.html')) {
      // Clear events from previous screen
      const eventsFile = path.join(SCREEN_DIR, '.events');
      if (fs.existsSync(eventsFile)) fs.unlinkSync(eventsFile);

      console.log(JSON.stringify({ type: 'screen-added', file: filePath }));
      // ... existing reload broadcast
    }
```

- [ ] **步驟 6：以註解占位注入取代 `wrapInFrame`**

替換 `wrapInFrame` 函式（`index.js` 第 27-32 行）：

```javascript
function wrapInFrame(content) {
  return frameTemplate.replace('<!-- CONTENT -->', content);
}
```

- [ ] **步驟 7：執行所有測試**

```bash
cd /Users/drewritter/prime-rad/superpowers && node tests/brainstorm-server/server.test.js
```
預期：新的 `.events` 測試通過。其餘測試可能仍會因舊斷言而失敗（在任務 4 修正）。

- [ ] **步驟 8：提交**

```bash
git add lib/brainstorm-server/index.js tests/brainstorm-server/server.test.js
git commit -m "Add .events file writing and comment-based content injection to brainstorm server"
```

---

### 任務 3：簡化 `helper.js`

**檔案：**
- 修改：`lib/brainstorm-server/helper.js`

- [ ] **步驟 1：移除 `sendToClaude` 函式**

刪除 `sendToClaude` 函式（第 92-106 行）— 包含函式內容與頁面接管 HTML。

- [ ] **步驟 2：移除 `window.send` 函式**

刪除 `window.send` 函式（第 120-129 行）— 它與已移除的 Send 按鈕相關。

- [ ] **步驟 3：移除表單送出與輸入變更處理器**

刪除表單送出 handler（第 57-71 行）與輸入變更 handler（第 73-89 行），包含 `inputTimeout` 變數。

- [ ] **步驟 4：移除 `pageshow` 事件監聽器**

刪除之前新增的 `pageshow` listener（已無 textarea 可清）。

- [ ] **步驟 5：將點擊 handler 縮小為只處理 `[data-choice]`**

用較精簡版本取代點擊 handler（第 36-55 行）：

```javascript
  // Capture clicks on choice elements
  document.addEventListener('click', (e) => {
    const target = e.target.closest('[data-choice]');
    if (!target) return;

    sendEvent({
      type: 'click',
      text: target.textContent.trim(),
      choice: target.dataset.choice,
      id: target.id || null
    });
  });
```

- [ ] **步驟 6：在選擇點擊後更新指示器列**

在 click handler 內 `sendEvent` 後加入：

```javascript
    // Update indicator bar
    const indicator = document.getElementById('indicator-text');
    if (indicator) {
      const label = target.querySelector('h3, .content h3, .card-body h3')?.textContent?.trim() || target.dataset.choice;
      indicator.innerHTML = '<span class="selected-text">' + label + ' selected</span> — return to terminal to continue';
    }
```

- [ ] **步驟 7：從 `window.brainstorm` API 移除 `sendToClaude`**

更新 `window.brainstorm` 物件（第 132-136 行），移除 `sendToClaude`：

```javascript
  window.brainstorm = {
    send: sendEvent,
    choice: (value, metadata = {}) => sendEvent({ type: 'choice', value, ...metadata })
  };
```

- [ ] **步驟 8：執行測試**

```bash
cd /Users/drewritter/prime-rad/superpowers && node tests/brainstorm-server/server.test.js
```

- [ ] **步驟 9：提交**

```bash
git add lib/brainstorm-server/helper.js
git commit -m "Simplify helper.js: remove feedback functions, narrow to choice capture + indicator"
```

---

### 任務 4：更新測試以符合新結構

**檔案：**
- 修改：`tests/brainstorm-server/server.test.js`

**注意：** 以下行號來自「原始」檔案。任務 2 已在檔案前段插入新測試，實際行號會偏移。請用 `console.log` 標籤（例如「Test 5:」、「Test 6:」）定位。

- [ ] **步驟 1：更新 Test 5（完整文件斷言）**

找到 Test 5 的斷言 `!fullRes.body.includes('feedback-footer')`。改成：完整文件不應該有指示器列（它們是原樣提供）：

```javascript
    assert(!fullRes.body.includes('indicator-bar') || fullDoc.includes('indicator-bar'),
      'Should not wrap full documents in frame template');
```

- [ ] **步驟 2：更新 Test 6（片段包裝）**

第 125 行：將 `feedback-footer` 斷言改成指示器列斷言：

```javascript
    assert(fragRes.body.includes('indicator-bar'), 'Fragment should get indicator bar from frame');
```

同時驗證內容占位已被替換（片段內容出現，占位註解不存在）：

```javascript
    assert(!fragRes.body.includes('<!-- CONTENT -->'), 'Content placeholder should be replaced');
```

- [ ] **步驟 3：更新 Test 7（helper.js API）**

第 140-142 行：更新斷言以反映新的 API 介面：

```javascript
    assert(helperContent.includes('toggleSelect'), 'helper.js should define toggleSelect');
    assert(helperContent.includes('sendEvent'), 'helper.js should define sendEvent');
    assert(helperContent.includes('selectedChoice'), 'helper.js should track selectedChoice');
    assert(helperContent.includes('brainstorm'), 'helper.js should expose brainstorm API');
    assert(!helperContent.includes('sendToClaude'), 'helper.js should not contain sendToClaude');
```

- [ ] **步驟 4：以指示器列測試取代 Test 8（sendToClaude 主題）**

替換 Test 8（第 145-149 行）— `sendToClaude` 已不存在。改測指示器列：

```javascript
    // Test 8: Indicator bar uses CSS variables (theme support)
    console.log('Test 8: Indicator bar uses CSS variables');
    const templateContent = fs.readFileSync(
      path.join(__dirname, '../../lib/brainstorm-server/frame-template.html'), 'utf-8'
    );
    assert(templateContent.includes('indicator-bar'), 'Template should have indicator bar');
    assert(templateContent.includes('indicator-text'), 'Template should have indicator text element');
    console.log('  PASS');
```

- [ ] **步驟 5：執行完整測試套件**

```bash
cd /Users/drewritter/prime-rad/superpowers && node tests/brainstorm-server/server.test.js
```
預期：所有測試通過。

- [ ] **步驟 6：提交**

```bash
git add tests/brainstorm-server/server.test.js
git commit -m "Update brainstorm server tests for new template structure and helper.js API"
```

---

### 任務 5：刪除 `wait-for-feedback.sh`

**檔案：**
- 刪除：`lib/brainstorm-server/wait-for-feedback.sh`

- [ ] **步驟 1：確認沒有其他檔案引用 `wait-for-feedback.sh`**

搜尋整個程式碼庫：
```bash
grep -r "wait-for-feedback" /Users/drewritter/prime-rad/superpowers/ --include="*.js" --include="*.md" --include="*.sh" --include="*.json"
```

預期引用：僅 `visual-companion.md`（在任務 6 會改寫）以及可能的 release notes（歷史，保留）。

- [ ] **步驟 2：刪除檔案**

```bash
rm lib/brainstorm-server/wait-for-feedback.sh
```

- [ ] **步驟 3：執行測試確認無誤**

```bash
cd /Users/drewritter/prime-rad/superpowers && node tests/brainstorm-server/server.test.js
```
預期：所有測試通過（沒有測試引用此檔案）。

- [ ] **步驟 4：提交**

```bash
git add -u lib/brainstorm-server/wait-for-feedback.sh
git commit -m "Delete wait-for-feedback.sh: replaced by .events file"
```

---

### 任務 6：改寫 `visual-companion.md`

**檔案：**
- 修改：`skills/brainstorming/visual-companion.md`

- [ ] **步驟 1：更新「How It Works」描述（第 18 行）**

將「以 JSON 接收回饋」的句子替換為：

```markdown
The server watches a directory for HTML files and serves the newest one to the browser. You write HTML content, the user sees it in their browser and can click to select options. Selections are recorded to a `.events` file that you read on your next turn.
```

- [ ] **步驟 2：更新片段描述（第 20 行）**

移除對「feedback footer」的描述：

```markdown
**Content fragments vs full documents:** If your HTML file starts with `<!DOCTYPE` or `<html`, the server serves it as-is (just injects the helper script). Otherwise, the server automatically wraps your content in the frame template — adding the header, CSS theme, selection indicator, and all interactive infrastructure. **Write content fragments by default.** Only write full documents when you need complete control over the page.
```

- [ ] **步驟 3：改寫「The Loop」章節（第 36-61 行）**

以以下內容取代整段「The Loop」：

```markdown
## The Loop

1. **Write HTML** to a new file in `screen_dir`:
   - Use semantic filenames: `platform.html`, `visual-style.html`, `layout.html`
   - **Never reuse filenames** — each screen gets a fresh file
   - Use Write tool — **never use cat/heredoc** (dumps noise into terminal)
   - Server automatically serves the newest file

2. **Tell user what to expect and end your turn:**
   - Remind them of the URL (every step, not just first)
   - Give a brief text summary of what's on screen (e.g., "Showing 3 layout options for the homepage")
   - Ask them to respond in the terminal: "Take a look and let me know what you think. Click to select an option if you'd like."

3. **On your next turn** — after the user responds in the terminal:
   - Read `$SCREEN_DIR/.events` if it exists — this contains the user's browser interactions (clicks, selections) as JSON lines
   - Merge with the user's terminal text to get the full picture
   - The terminal message is the primary feedback; `.events` provides structured interaction data

4. **Iterate or advance** — if feedback changes current screen, write a new file (e.g., `layout-v2.html`). Only move to the next question when the current step is validated.

5. Repeat until done.
```

- [ ] **步驟 4：取代「User Feedback Format」章節（第 165-174 行）**

替換為：

```markdown
## Browser Events Format

When the user clicks options in the browser, their interactions are recorded to `$SCREEN_DIR/.events` (one JSON object per line). The file is cleared automatically when you push a new screen.

```jsonl
{"type":"click","choice":"a","text":"Option A - Simple Layout","timestamp":1706000101}
{"type":"click","choice":"c","text":"Option C - Complex Grid","timestamp":1706000108}
{"type":"click","choice":"b","text":"Option B - Hybrid","timestamp":1706000115}
```

The full event stream shows the user's exploration path — they may click multiple options before settling. The last `choice` event is typically the final selection, but the pattern of clicks can reveal hesitation or preferences worth asking about.

If `.events` doesn't exist, the user didn't interact with the browser — use only their terminal text.
```

- [ ] **步驟 5：更新「Writing Content Fragments」描述（第 65 行）**

移除「feedback footer」參照：

```markdown
Write just the content that goes inside the page. The server wraps it in the frame template automatically (header, theme CSS, selection indicator, and all interactive infrastructure).
```

- [ ] **步驟 6：更新 Reference 章節（第 200-203 行）**

移除 helper.js 的「JS API」描述，保留路徑參照：

```markdown
## Reference

- Frame template (CSS reference): `${CLAUDE_PLUGIN_ROOT}/lib/brainstorm-server/frame-template.html`
- Helper script (client-side): `${CLAUDE_PLUGIN_ROOT}/lib/brainstorm-server/helper.js`
```

- [ ] **步驟 7：提交**

```bash
git add skills/brainstorming/visual-companion.md
git commit -m "Rewrite visual-companion.md for non-blocking browser-displays-terminal-commands flow"
```

---

### 任務 7：最終驗證

- [ ] **步驟 1：執行完整測試套件**

```bash
cd /Users/drewritter/prime-rad/superpowers && node tests/brainstorm-server/server.test.js
```
預期：所有測試通過。

- [ ] **步驟 2：手動煙霧測試**

手動啟動伺服器並確認流程端到端正常：

```bash
cd /Users/drewritter/prime-rad/superpowers && lib/brainstorm-server/start-server.sh --project-dir /tmp/brainstorm-smoke-test
```

寫入測試片段、在瀏覽器開啟、點擊選項，確認 `.events` 寫入並確認指示器列更新。然後停止伺服器：

```bash
lib/brainstorm-server/stop-server.sh <screen_dir from start output>
```

- [ ] **步驟 3：確認沒有殘留參照**

```bash
grep -r "wait-for-feedback\|sendToClaude\|feedback-footer\|send-to-claude\|TaskOutput.*block.*true" /Users/drewritter/prime-rad/superpowers/ --include="*.js" --include="*.md" --include="*.sh" --include="*.html" | grep -v node_modules | grep -v RELEASE-NOTES | grep -v "\.md:.*spec\|plan"
```

預期：除了 release notes 與 spec/plan 文件（歷史）外沒有任何結果。

- [ ] **步驟 4：如有需要進行最終提交**

```bash
git status
# Review untracked/modified files, stage specific files as needed, commit if clean
```
