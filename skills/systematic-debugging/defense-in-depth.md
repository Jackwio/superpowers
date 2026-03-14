# 深度防禦式驗證

## 概覽

當你修了一個由無效資料導致的 bug，在某一處加驗證看似夠了。但單點檢查可能被不同程式路徑、重構或 mock 繞過。

**核心原則：**在資料流經的每一層都驗證，讓這個 bug 從結構上不可能發生。

## 為何需要多層

單一驗證：「我們修好了 bug」
多層驗證：「我們讓 bug 變得不可能」

不同層會捕捉不同情況：
- 入口驗證抓大多數錯誤
- 商業邏輯抓邊界情況
- 環境防護避免情境特有的危險
- 除錯記錄在其他層失效時提供線索

## 四層防線

### 第 1 層：入口驗證
**目的：**在 API 邊界拒絕明顯無效輸入

```typescript
function createProject(name: string, workingDirectory: string) {
  if (!workingDirectory || workingDirectory.trim() === '') {
    throw new Error('workingDirectory cannot be empty');
  }
  if (!existsSync(workingDirectory)) {
    throw new Error(`workingDirectory does not exist: ${workingDirectory}`);
  }
  if (!statSync(workingDirectory).isDirectory()) {
    throw new Error(`workingDirectory is not a directory: ${workingDirectory}`);
  }
  // ... proceed
}
```

### 第 2 層：商業邏輯驗證
**目的：**確保資料對該操作是合理的

```typescript
function initializeWorkspace(projectDir: string, sessionId: string) {
  if (!projectDir) {
    throw new Error('projectDir required for workspace initialization');
  }
  // ... proceed
}
```

### 第 3 層：環境防護
**目的：**在特定情境避免危險操作

```typescript
async function gitInit(directory: string) {
  // In tests, refuse git init outside temp directories
  if (process.env.NODE_ENV === 'test') {
    const normalized = normalize(resolve(directory));
    const tmpDir = normalize(resolve(tmpdir()));

    if (!normalized.startsWith(tmpDir)) {
      throw new Error(
        `Refusing git init outside temp dir during tests: ${directory}`
      );
    }
  }
  // ... proceed
}
```

### 第 4 層：除錯紀錄
**目的：**捕捉鑑識所需脈絡

```typescript
async function gitInit(directory: string) {
  const stack = new Error().stack;
  logger.debug('About to git init', {
    directory,
    cwd: process.cwd(),
    stack,
  });
  // ... proceed
}
```

## 套用模式

當你發現 bug 時：

1. **追蹤資料流** — 無效值從哪裡來？在哪裡被使用？
2. **列出所有檢查點** — 列出資料流經的每個節點
3. **在每一層加驗證** — 入口、商業邏輯、環境、除錯
4. **測試每一層** — 嘗試繞過第 1 層，確認第 2 層能抓到

## 會話中的例子

Bug：空的 `projectDir` 造成 `git init` 在原始碼目錄執行

**資料流：**
1. 測試設定 → 空字串
2. `Project.create(name, '')`
3. `WorkspaceManager.createWorkspace('')`
4. `git init` 在 `process.cwd()` 執行

**新增四層：**
- 第 1 層：`Project.create()` 驗證非空/存在/可寫
- 第 2 層：`WorkspaceManager` 驗證 projectDir 非空
- 第 3 層：`WorktreeManager` 在測試時拒絕在 tmpdir 外 git init
- 第 4 層：git init 前加入 stack trace 記錄

**結果：**1847 個測試全部通過，bug 無法重現

## 關鍵洞見

四層都必要。測試過程中，每一層都攔下其他層漏掉的問題：
- 不同程式路徑繞過入口驗證
- Mock 繞過商業邏輯檢查
- 不同平台的邊界情況需要環境防護
- 除錯記錄揭示結構性誤用

**不要只停在一個驗證點。**在每一層都加檢查。
