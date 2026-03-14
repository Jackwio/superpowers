# 測試反模式

**在以下情況載入此參考：**撰寫或修改測試、加入 mock，或想在 production code 加測試專用方法時。

## 概覽

測試必須驗證真實行為，而非 mock 行為。Mock 是用來隔離，不是被測試的主體。

**核心原則：**測程式碼做了什麼，不是 mock 做了什麼。

**嚴格 TDD 能避免這些反模式。**

## 鐵則

```
1. 絕不測 mock 行為
2. 絕不在 production class 加測試專用方法
3. 未理解相依前絕不 mock
```

## 反模式 1：測 mock 行為

**違規範例：**
```typescript
// ❌ BAD: Testing that the mock exists
test('renders sidebar', () => {
  render(<Page />);
  expect(screen.getByTestId('sidebar-mock')).toBeInTheDocument();
});
```

**為何錯：**
- 你在驗證 mock 有沒有存在，而不是元件是否正常
- mock 存在就通過，不存在就失敗
- 無法告訴你真實行為

**你的使用者的糾正：**「我們是在測 mock 的行為嗎？」

**修正：**
```typescript
// ✅ GOOD: Test real component or don't mock it
test('renders sidebar', () => {
  render(<Page />);  // Don't mock sidebar
  expect(screen.getByRole('navigation')).toBeInTheDocument();
});

// OR if sidebar must be mocked for isolation:
// Don't assert on the mock - test Page's behavior with sidebar present
```

### Gate Function

```
在對任何 mock 元素做斷言之前：
  問：「我是在測真實元件行為，還是只是在測 mock 存在？」

  若只是測 mock 存在：
    停止 — 刪掉斷言或取消 mock

  改測真實行為
```

## 反模式 2：在 production 加測試專用方法

**違規範例：**
```typescript
// ❌ BAD: destroy() only used in tests
class Session {
  async destroy() {  // Looks like production API!
    await this._workspaceManager?.destroyWorkspace(this.id);
    // ... cleanup
  }
}

// In tests
afterEach(() => session.destroy());
```

**為何錯：**
- 讓 production class 被測試專用程式碼污染
- 若不小心在 production 呼叫會危險
- 違反 YAGNI 與關注點分離
- 混淆物件生命週期與實體生命週期

**修正：**
```typescript
// ✅ GOOD: Test utilities handle test cleanup
// Session has no destroy() - it's stateless in production

// In test-utils/
export async function cleanupSession(session: Session) {
  const workspace = session.getWorkspaceInfo();
  if (workspace) {
    await workspaceManager.destroyWorkspace(workspace.id);
  }
}

// In tests
afterEach(() => cleanupSession(session));
```

### Gate Function

```
在新增任何 production class 方法之前：
  問：「這只在測試中使用嗎？」

  若是：
    停止 — 不要加
    放到 test utilities

  再問：「這個 class 真的擁有這個資源的生命週期嗎？」

  若否：
    停止 — 這個方法放錯 class
```

## 反模式 3：未理解相依就 mock

**違規範例：**
```typescript
// ❌ BAD: Mock breaks test logic
test('detects duplicate server', () => {
  // Mock prevents config write that test depends on!
  vi.mock('ToolCatalog', () => ({
    discoverAndCacheTools: vi.fn().mockResolvedValue(undefined)
  }));

  await addServer(config);
  await addServer(config);  // Should throw - but won't!
});
```

**為何錯：**
- mock 的方法有副作用（寫入 config），測試依賴該副作用
- 為了「安全」過度 mock 反而破壞行為
- 測試可能因錯誤原因通過或莫名失敗

**修正：**
```typescript
// ✅ GOOD: Mock at correct level
test('detects duplicate server', () => {
  // Mock the slow part, preserve behavior test needs
  vi.mock('MCPServerManager'); // Just mock slow server startup

  await addServer(config);  // Config written
  await addServer(config);  // Duplicate detected ✓
});
```

### Gate Function

```
在 mock 任何方法之前：
  停止 — 先不要 mock

  1. 問：「真實方法有哪些副作用？」
  2. 問：「這個測試是否依賴其中任何副作用？」
  3. 問：「我是否完全理解測試需要什麼？」

  若依賴副作用：
    在更低層 mock（實際慢/外部操作）
    或使用保留必要行為的 test double
    不要 mock 測試依賴的高階方法

  若不確定測試依賴什麼：
    先用真實實作跑一次測試
    觀察到底需要什麼
    再在正確層級加最小 mock

  紅旗：
    -「先 mock 以防萬一」
    -「可能會慢，先 mock」
    - 未理解相依鏈就 mock
```

## 反模式 4：不完整的 mock

**違規範例：**
```typescript
// ❌ BAD: Partial mock - only fields you think you need
const mockResponse = {
  status: 'success',
  data: { userId: '123', name: 'Alice' }
  // Missing: metadata that downstream code uses
};

// Later: breaks when code accesses response.metadata.requestId
```

**為何錯：**
- **不完整的 mock 隱藏了結構假設** — 你只 mock 了你知道的欄位
- **下游程式碼可能依賴你沒提供的欄位** — 靜默失敗
- **測試通過但整合失敗** — mock 不完整，真實 API 完整
- **虛假信心** — 測試無法證明真實行為

**鐵則：**mock 必須完整鏡像真實資料結構，而不是只包含你當下測試用到的欄位。

**修正：**
```typescript
// ✅ GOOD: Mirror real API completeness
const mockResponse = {
  status: 'success',
  data: { userId: '123', name: 'Alice' },
  metadata: { requestId: 'req-789', timestamp: 1234567890 }
  // All fields real API returns
};
```

### Gate Function

```
在建立 mock 回應前：
  檢查：「真實 API 回應包含哪些欄位？」

  行動：
    1. 從文件/範例查看真實回應
    2. 包含系統下游可能使用的所有欄位
    3. 驗證 mock 完整符合真實 schema

  重要：
    如果你要做 mock，就必須理解整個結構
    不完整 mock 會在依賴欄位時靜默失敗

  若不確定：包含所有已文件化欄位
```

## 反模式 5：把整合測試當作事後補

**違規範例：**
```
✅ Implementation complete
❌ No tests written
"Ready for testing"
```

**為何錯：**
- 測試是實作的一部分，不是可選的後續
- TDD 本來就會抓到這種情況
- 沒有測試不能說完成

**修正：**
```
TDD cycle:
1. Write failing test
2. Implement to pass
3. Refactor
4. THEN claim complete
```

## 當 mock 變得太複雜

**警訊：**
- mock 設定比測試邏輯還長
- mock 一堆才能讓測試通過
- mock 缺少真實元件的方法
- 測試因 mock 變動而失敗

**你的使用者問題：**「這裡真的需要 mock 嗎？」

**考慮：**用真實元件的整合測試，往往比複雜 mock 更簡單

## TDD 能避免這些反模式

**TDD 有效的原因：**
1. **先寫測試** → 迫使你思考自己在測什麼
2. **看它失敗** → 確認測試真正在測行為，不是測 mock
3. **最小實作** → 不會混進測試專用方法
4. **真實相依** → 在 mock 前先看測試真正需要什麼

**若你在測 mock 行為，你已違反 TDD** — 你在沒有先看真實程式碼失敗的情況下加了 mock。

## 快速對照

| 反模式 | 修正 |
|--------------|-----|
| 對 mock 元素斷言 | 測真實元件或取消 mock |
| production 內的測試專用方法 | 移到 test utilities |
| 未理解相依就 mock | 先理解相依，再最小 mock |
| 不完整 mock | 完整鏡像真實 API |
| 把測試當事後補 | TDD — 先測試 |
| mock 過度複雜 | 考慮整合測試 |

## 紅旗

- 斷言檢查 `*-mock` 測試 ID
- 只有測試檔才呼叫的方法
- mock 設定超過測試的 50%
- 移除 mock 測試就失敗
- 無法說明為何需要 mock
- 「先 mock 以防萬一」

## 底線

**mock 是用來隔離的工具，不是被測的對象。**

若 TDD 顯示你在測 mock 行為，那就做錯了。

修正：測真實行為，或重新思考為何要 mock。
