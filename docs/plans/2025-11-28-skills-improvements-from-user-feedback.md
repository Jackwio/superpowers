# 來自使用者回饋的技能改進

**日期：** 2025-11-28
**狀態：** 草稿
**來源：** 兩個在真實開發情境中使用 superpowers 的 Claude 實例

---

## 執行摘要

兩個 Claude 實例從實際開發工作階段提供了詳細回饋。他們的回饋揭露了目前技能的**系統性缺口**，使得即便遵循技能也仍有可避免的 bug 出貨。

**關鍵洞見：** 這些是問題回報，而不只是解法提案。問題確實存在；解法需要謹慎評估。

**核心主題：**
1. **驗證缺口** - 我們驗證操作是否成功，但未驗證是否達成預期結果
2. **流程衛生** - 背景程序跨子代理累積並互相干擾
3. **情境最佳化** - 子代理收到過多不相關資訊
4. **缺乏自我反思** - 交接前沒有要求自我批判
5. **Mock 安全性** - Mock 可能與介面漂移卻未被偵測
6. **技能啟用** - 技能存在但未被閱讀/使用

---

## 已辨識問題

### 問題 1：設定變更驗證缺口

**發生情況：**
- 子代理測試「OpenAI 整合」
- 設定 `OPENAI_API_KEY` 環境變數
- 回應狀態為 200
- 回報「OpenAI 整合正常」
- **但** 回應包含 `"model": "claude-sonnet-4-20250514"` — 實際上仍在用 Anthropic

**根因：**
`verification-before-completion` 只檢查操作是否成功，未檢查結果是否反映預期設定變更。

**影響：** 高 — 整合測試的錯誤信心，bug 出到 production

**典型失敗模式：**
- 切換 LLM provider → 驗證狀態 200，但未檢查模型名稱
- 啟用 feature flag → 驗證無錯誤，但未確認功能啟用
- 更換環境 → 驗證部署成功，但未檢查環境變數

---

### 問題 2：背景程序累積

**發生情況：**
- 工作階段中派發多個子代理
- 每個都啟動背景伺服器
- 程序累積（4+ 伺服器仍在跑）
- 舊程序仍綁定連接埠
- 後續 E2E 測試連到舊伺服器且設定錯誤
- 測試結果混亂/不正確

**根因：**
子代理是無狀態的，不知道先前子代理啟動的程序。沒有清理流程。

**影響：** 中高 — 測試連到錯誤伺服器、誤判通過/失敗、除錯混亂

---

### 問題 3：子代理提示的情境膨脹

**發生情況：**
- 標準做法：提供完整計畫檔給子代理閱讀
- 實驗：只給任務 + 模式 + 檔案 + 驗證指令
- 結果：更快、更聚焦，單次完成率更高

**根因：**
子代理把 token 與注意力耗費在不相關的計畫段落。

**影響：** 中 — 執行變慢、失敗率上升

**有效做法：**
```
You are adding a single E2E test to packnplay's test suite.

**Your task:** Add `TestE2E_FeaturePrivilegedMode` to `pkg/runner/e2e_test.go`

**What to test:** A local devcontainer feature that requests `"privileged": true`
in its metadata should result in the container running with `--privileged` flag.

**Follow the exact pattern of TestE2E_FeatureOptionValidation** (at the end of the file)

**After writing, run:** `go test -v ./pkg/runner -run TestE2E_FeaturePrivilegedMode -timeout 5m`
```

---

### 問題 4：交接前缺乏自我反思

**發生情況：**
- 新增自我反思提示：「用新鮮的眼光回看你的工作，還能更好嗎？」
- 任務 5 的實作者發現測試失敗是實作 bug，而不是測試 bug
- 追查到第 99 行：`strings.Join(metadata.Entrypoint, " ")` 造成無效的 Docker 語法
- 沒有自我反思的話，可能只會回報「測試失敗」而不找根因

**根因：**
實作者在回報完成前不會自然停下來檢視自己的成果。

**影響：** 中 — 可以由實作者先發現的 bug 被交給審查者

---

### 問題 5：Mock-介面漂移

**發生情況：**
```typescript
// Interface defines close()
interface PlatformAdapter {
  close(): Promise<void>;
}

// Code (BUGGY) calls cleanup()
await adapter.cleanup();

// Mock (MATCHES BUG) defines cleanup()
vi.mock('web-adapter', () => ({
  WebAdapter: vi.fn().mockImplementation(() => ({
    cleanup: vi.fn().mockResolvedValue(undefined),  // Wrong!
  })),
}));
```
- 測試通過
- 執行時崩潰：「adapter.cleanup is not a function」

**根因：**
Mock 依照錯誤的實作呼叫建立，而不是依介面定義建立。TypeScript 無法抓到 inline mock 的錯誤方法名稱。

**影響：** 高 — 測試給出錯誤信心，執行時崩潰

**為何 testing-anti-patterns 沒擋住：**
該技能涵蓋「測試 mock 行為」與「在不了解相依時 mock」，但沒有涵蓋「Mock 應依介面而非實作」的特定模式。

---

### 問題 6：程式碼審查者的檔案存取

**發生情況：**
- 派發 code reviewer 子代理
- 找不到測試檔：「The file doesn't appear to exist in the repository」
- 檔案其實存在
- Reviewer 不知道要先明確讀取檔案

**根因：**
Reviewer 的提示中沒有明確要求先讀檔。

**影響：** 低到中 — 審查失敗或不完整

---

### 問題 7：修正流程延遲

**發生情況：**
- 實作者在自我反思時發現 bug
- 實作者知道修法
- 目前流程：回報 → 我派 fixer → fixer 修 → 我驗證
- 額外往返增加延遲但沒增加價值

**根因：**
實作者已診斷仍被僵化地分隔為實作/修正角色。

**影響：** 低 — 延遲增加，但正確性不受影響

---

### 問題 8：技能未被閱讀

**發生情況：**
- `testing-anti-patterns` 技能已存在
- 人與子代理在寫測試前都沒閱讀
- 本來能避免部分問題（但不是全部，見問題 5）

**根因：**
沒有強制子代理閱讀相關技能。提示中沒有加入技能閱讀。

**影響：** 中 — 技能投資被浪費

---

## 建議改進

### 1. verification-before-completion：加入設定變更驗證

**新增段落：**

```markdown
## Verifying Configuration Changes

When testing changes to configuration, providers, feature flags, or environment:

**Don't just verify the operation succeeded. Verify the output reflects the intended change.**

### Common Failure Pattern

Operation succeeds because *some* valid config exists, but it's not the config you intended to test.

### Examples

| Change | Insufficient | Required |
|--------|-------------|----------|
| Switch LLM provider | Status 200 | Response contains expected model name |
| Enable feature flag | No errors | Feature behavior actually active |
| Change environment | Deploy succeeds | Logs/vars reference new environment |
| Set credentials | Auth succeeds | Authenticated user/context is correct |

### Gate Function

```
BEFORE claiming configuration change works:

1. IDENTIFY: What should be DIFFERENT after this change?
2. LOCATE: Where is that difference observable?
   - Response field (model name, user ID)
   - Log line (environment, provider)
   - Behavior (feature active/inactive)
3. RUN: Command that shows the observable difference
4. VERIFY: Output contains expected difference
5. ONLY THEN: Claim configuration change works

Red flags:
  - "Request succeeded" without checking content
  - Checking status code but not response body
  - Verifying no errors but not positive confirmation
```

**Why this works:**
Forces verification of INTENT, not just operation success.
```

---

### 2. subagent-driven-development：加入 E2E 測試的流程衛生

**新增段落：**

```markdown
## Process Hygiene for E2E Tests

When dispatching subagents that start services (servers, databases, message queues):

### Problem

Subagents are stateless - they don't know about processes started by previous subagents. Background processes persist and can interfere with later tests.

### Solution

**Before dispatching E2E test subagent, include cleanup in prompt:**

```
BEFORE starting any services:
1. Kill existing processes: pkill -f "<service-pattern>" 2>/dev/null || true
2. Wait for cleanup: sleep 1
3. Verify port free: lsof -i :<port> && echo "ERROR: Port still in use" || echo "Port free"

AFTER tests complete:
1. Kill the process you started
2. Verify cleanup: pgrep -f "<service-pattern>" || echo "Cleanup successful"
```

### Example

```
Task: Run E2E test of API server

Prompt includes:
"Before starting the server:
- Kill any existing servers: pkill -f 'node.*server.js' 2>/dev/null || true
- Verify port 3001 is free: lsof -i :3001 && exit 1 || echo 'Port available'

After tests:
- Kill the server you started
- Verify: pgrep -f 'node.*server.js' || echo 'Cleanup verified'"
```

### Why This Matters

- Stale processes serve requests with wrong config
- Port conflicts cause silent failures
- Process accumulation slows system
- Confusing test results (hitting wrong server)
```

**取捨分析：**
- 提示會更冗長
- 但能避免非常混亂的除錯
- 對 E2E 測試子代理值得加入

---

### 3. subagent-driven-development：加入精簡情境選項

**修改 Step 2：Execute Task with Subagent**

**修改前：**
```
Read that task carefully from [plan-file].
```

**修改後：**
```
## Context Approaches

**Full Plan (default):**
Use when tasks are complex or have dependencies:
```
Read Task N from [plan-file] carefully.
```

**Lean Context (for independent tasks):**
Use when task is standalone and pattern-based:
```
You are implementing: [1-2 sentence task description]

File to modify: [exact path]
Pattern to follow: [reference to existing function/test]
What to implement: [specific requirement]
Verification: [exact command to run]

[Do NOT include full plan file]
```

**Use lean context when:**
- Task follows existing pattern (add similar test, implement similar feature)
- Task is self-contained (doesn't need context from other tasks)
- Pattern reference is sufficient (e.g., "follow TestE2E_FeatureOptionValidation")

**Use full plan when:**
- Task has dependencies on other tasks
- Requires understanding of overall architecture
- Complex logic that needs context
```

**範例：**
```
Lean context prompt:

"You are adding a test for privileged mode in devcontainer features.

File: pkg/runner/e2e_test.go
Pattern: Follow TestE2E_FeatureOptionValidation (at end of file)
Test: Feature with `"privileged": true` in metadata results in `--privileged` flag
Verify: go test -v ./pkg/runner -run TestE2E_FeaturePrivilegedMode -timeout 5m

Report: Implementation, test results, any issues."
```

**Why this works:**
Reduces token usage, increases focus, faster completion when appropriate.

---

### 4. subagent-driven-development：加入自我反思步驟

**修改 Step 2：Execute Task with Subagent**

**加入以下提示模板：**

```
When done, BEFORE reporting back:

Take a step back and review your work with fresh eyes.

Ask yourself:
- Does this actually solve the task as specified?
- Are there edge cases I didn't consider?
- Did I follow the pattern correctly?
- If tests are failing, what's the ROOT CAUSE (implementation bug vs test bug)?
- What could be better about this implementation?

If you identify issues during this reflection, fix them now.

Then report:
- What you implemented
- Self-reflection findings (if any)
- Test results
- Files changed
```

**Why this works:**
Catches bugs implementer can find themselves before handoff. Documented case: identified entrypoint bug through self-reflection.

**取捨：**
每個任務大約多花 30 秒，但能在審查前先抓出問題。

---

### 5. requesting-code-review：加入明確讀檔要求

**修改 code-reviewer 範本：**

**在開頭加入：**

```markdown
## Files to Review

BEFORE analyzing, read these files:

1. [List specific files that changed in the diff]
2. [Files referenced by changes but not modified]

Use Read tool to load each file.

If you cannot find a file:
- Check exact path from diff
- Try alternate locations
- Report: "Cannot locate [path] - please verify file exists"

DO NOT proceed with review until you've read the actual code.
```

**Why this works:**
明確指示可避免「找不到檔案」問題。

---

### 6. testing-anti-patterns：加入 Mock-介面漂移反模式

**新增反模式 6：**

```markdown
## Anti-Pattern 6: Mocks Derived from Implementation

**The violation:**
```typescript
// Code (BUGGY) calls cleanup()
await adapter.cleanup();

// Mock (MATCHES BUG) has cleanup()
const mock = {
  cleanup: vi.fn().mockResolvedValue(undefined)
};

// Interface (CORRECT) defines close()
interface PlatformAdapter {
  close(): Promise<void>;
}
```

**Why this is wrong:**
- Mock encodes the bug into the test
- TypeScript can't catch inline mocks with wrong method names
- Test passes because both code and mock are wrong
- Runtime crashes when real object is used

**The fix:**
```typescript
// ✅ GOOD: Derive mock from interface

// Step 1: Open interface definition (PlatformAdapter)
// Step 2: List methods defined there (close, initialize, etc.)
// Step 3: Mock EXACTLY those methods

const mock = {
  initialize: vi.fn().mockResolvedValue(undefined),
  close: vi.fn().mockResolvedValue(undefined),  // From interface!
};

// Now test FAILS because code calls cleanup() which doesn't exist
// That failure reveals the bug BEFORE runtime
```

### Gate Function

```
BEFORE writing any mock:

  1. STOP - Do NOT look at the code under test yet
  2. FIND: The interface/type definition for the dependency
  3. READ: The interface file
  4. LIST: Methods defined in the interface
  5. MOCK: ONLY those methods with EXACTLY those names
  6. DO NOT: Look at what your code calls

  IF your test fails because code calls something not in mock:
    ✅ GOOD - The test found a bug in your code
    Fix the code to call the correct interface method
    NOT the mock

  Red flags:
    - "I'll mock what the code calls"
    - Copying method names from implementation
    - Mock written without reading interface
    - "The test is failing so I'll add this method to the mock"
```

**Detection:**

When you see runtime error "X is not a function" and tests pass:
1. Check if X is mocked
2. Compare mock methods to interface methods
3. Look for method name mismatches
```

**Why this works:**
Directly addresses the failure pattern from feedback.

---

### 7. subagent-driven-development：要求測試子代理閱讀技能

**在涉及測試的任務提示中加入：**

```markdown
BEFORE writing any tests:

1. Read testing-anti-patterns skill:
   Use Skill tool: superpowers:testing-anti-patterns

2. Apply gate functions from that skill when:
   - Writing mocks
   - Adding methods to production classes
   - Mocking dependencies

This is NOT optional. Tests that violate anti-patterns will be rejected in review.
```

**Why this works:**
確保技能真的被使用，而不只是存在。

**取捨：**
每個任務會多花一些時間，但能避免整類 bug。

---

### 8. subagent-driven-development：允許實作者修正自我發現的問題

**修改 Step 2：**

**目前：**
```
Subagent reports back with summary of work.
```

**建議：**
```
Subagent performs self-reflection, then:

IF self-reflection identifies fixable issues:
  1. Fix the issues
  2. Re-run verification
  3. Report: "Initial implementation + self-reflection fix"

ELSE:
  Report: "Implementation complete"

Include in report:
- Self-reflection findings
- Whether fixes were applied
- Final verification results
```

**Why this works:**
當實作者已知道修法時，可減少延遲。案例中若允許，能省掉一次往返。

**取捨：**
提示稍微更複雜，但端到端更快。

---

## 實作計畫

### Phase 1：高影響、低風險（先做）

1. **verification-before-completion：設定變更驗證**
   - 清楚新增，不改動既有內容
   - 解決高影響問題（測試錯誤信心）
   - 檔案：`skills/verification-before-completion/SKILL.md`

2. **testing-anti-patterns：Mock-介面漂移**
   - 新增反模式，不修改既有內容
   - 解決高影響問題（執行時崩潰）
   - 檔案：`skills/testing-anti-patterns/SKILL.md`

3. **requesting-code-review：明確讀檔**
   - 簡單新增範本
   - 解決具體問題（審查者找不到檔案）
   - 檔案：`skills/requesting-code-review/SKILL.md`

### Phase 2：中度變更（需仔細測試）

4. **subagent-driven-development：流程衛生**
   - 新增段落，不改流程
   - 解決中高影響（測試可靠性）
   - 檔案：`skills/subagent-driven-development/SKILL.md`

5. **subagent-driven-development：自我反思**
   - 變更提示範本（風險較高）
   - 但可捕捉已知 bug
   - 檔案：`skills/subagent-driven-development/SKILL.md`

6. **subagent-driven-development：技能閱讀要求**
   - 增加提示負擔
   - 但確保技能被使用
   - 檔案：`skills/subagent-driven-development/SKILL.md`

### Phase 3：最佳化（先驗證）

7. **subagent-driven-development：精簡情境選項**
   - 增加複雜度（兩種方式）
   - 需要驗證不會造成混淆
   - 檔案：`skills/subagent-driven-development/SKILL.md`

8. **subagent-driven-development：允許實作者修正**
   - 改變流程（風險較高）
   - 是最佳化，而非修 bug
   - 檔案：`skills/subagent-driven-development/SKILL.md`

---

## 未解問題

1. **精簡情境方式：**
   - 是否應成為模式型任務的預設？
   - 如何判斷要用哪種方式？
   - 太精簡導致漏掉重要情境的風險？

2. **自我反思：**
   - 會不會讓簡單任務變慢很多？
   - 是否只套用在複雜任務？
   - 如何避免「反思疲勞」變成形式？

3. **流程衛生：**
   - 應該放在 subagent-driven-development 還是另立技能？
   - 是否適用於 E2E 以外的流程？
   - 若程序本就該持續存在（dev server），該如何處理？

4. **技能閱讀強制：**
   - 是否要要求所有子代理都讀相關技能？
   - 如何避免提示過長？
   - 是否會過度文件化而失焦？

---

## 成功指標

如何判斷這些改進有效？

1. **設定驗證：**
   - 不再出現「測試通過但用了錯誤設定」
   - Jesse 不再說「這其實沒有測到你以為的內容」

2. **流程衛生：**
   - 不再出現「測試打到錯誤伺服器」
   - E2E 測試過程沒有連接埠衝突錯誤

3. **Mock-介面漂移：**
   - 不再出現「測試通過但執行時因缺少方法崩潰」
   - Mock 方法名稱與介面不再不一致

4. **自我反思：**
   - 可量化：實作者回報有自我反思內容
   - 質化：進入 code review 的 bug 變少

5. **技能閱讀：**
   - 子代理回報提到技能關卡函式
   - code review 中的反模式違規變少

---

## 風險與緩解

### 風險：提示膨脹
**問題：** 新增要求過多導致提示難以消化
**緩解：**
- 分階段導入（不要一次全加）
- 部分新增改為條件式（E2E 衛生只適用 E2E 測試）
- 針對不同任務類型建立範本

### 風險：分析癱瘓
**問題：** 太多反思/驗證會拖慢執行
**緩解：**
- 關卡函式要快速（秒級而非分鐘）
- 精簡情境先採 opt-in
- 監控任務完成時間

### 風險：錯誤安全感
**問題：** 遵循清單不保證正確
**緩解：**
- 強調關卡函式是最低要求，而非上限
- 保留「需用判斷」的語氣
- 文件化：技能只涵蓋常見失敗，不涵蓋所有失敗

### 風險：技能分歧
**問題：** 不同技能給出衝突建議
**緩解：**
- 跨技能一致性審查
- 在 Integration 章節說明技能互動
- 上線前以真實情境測試

---

## 建議

**立即執行 Phase 1：**
- verification-before-completion：設定變更驗證
- testing-anti-patterns：Mock-介面漂移
- requesting-code-review：明確讀檔

**在與 Jesse 測試 Phase 2 後再定稿：**
- 蒐集自我反思影響的回饋
- 驗證流程衛生作法
- 確認技能閱讀要求值得投入負擔

**Phase 3 先暫緩，待驗證後再推：**
- 精簡情境需要真實測試
- 實作者修正流程需謹慎評估

這些改動對應使用者所回報的真實問題，同時降低把技能改壞的風險。
