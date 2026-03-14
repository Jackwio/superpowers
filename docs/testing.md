# 測試 Superpowers 技能

本文件說明如何測試 Superpowers 技能，特別是像 `subagent-driven-development` 這類複雜技能的整合測試。

## 概覽

涉及子代理、工作流程與複雜互動的技能測試，需要以無頭模式執行實際的 Claude Code 會話，並透過會話逐字稿驗證其行為。

## 測試結構

```
tests/
├── claude-code/
│   ├── test-helpers.sh                    # Shared test utilities
│   ├── test-subagent-driven-development-integration.sh
│   ├── analyze-token-usage.py             # Token analysis tool
│   └── run-skill-tests.sh                 # Test runner (if exists)
```

## 執行測試

### 整合測試

整合測試會用實際技能執行真實的 Claude Code 會話：

```bash
# Run the subagent-driven-development integration test
cd tests/claude-code
./test-subagent-driven-development-integration.sh
```

**注意：**整合測試可能需要 10-30 分鐘，因為它們會透過多個子代理執行真實的實作計畫。

### 需求

- 必須從 **superpowers 外掛目錄** 執行（不能從暫存目錄）
- 必須已安裝 Claude Code，且 `claude` 指令可用
- 必須啟用本機開發市集：在 `~/.claude/settings.json` 中設定 `"superpowers@superpowers-dev": true`

## 整合測試：subagent-driven-development

### 測試內容

整合測試會驗證 `subagent-driven-development` 技能是否正確：

1. **計畫載入**：一開始只讀一次計畫
2. **完整任務文字**：提供子代理完整任務描述（不讓它們去讀檔）
3. **自我審查**：確保子代理在回報前進行自我審查
4. **審查順序**：先進行規格符合度審查，再進行程式碼品質審查
5. **審查迴圈**：發現問題時會啟動審查迴圈
6. **獨立驗證**：規格審查者會獨立閱讀程式碼，不信任實作者的回報

### 運作方式

1. **準備**：建立一個含最小實作計畫的暫時 Node.js 專案
2. **執行**：以無頭模式執行 Claude Code 並啟用技能
3. **驗證**：解析會話逐字稿（`.jsonl` 檔案）以確認：
   - Skill 工具被呼叫
   - 子代理被派出（Task 工具）
   - 使用 TodoWrite 追蹤
   - 已建立實作檔案
   - 測試通過
   - Git 提交顯示正確流程
4. **Token 分析**：顯示各子代理的 token 使用量拆解

### 測試輸出

```
========================================
 Integration Test: subagent-driven-development
========================================

Test project: /tmp/tmp.xyz123

=== Verification Tests ===

Test 1: Skill tool invoked...
  [PASS] subagent-driven-development skill was invoked

Test 2: Subagents dispatched...
  [PASS] 7 subagents dispatched

Test 3: Task tracking...
  [PASS] TodoWrite used 5 time(s)

Test 6: Implementation verification...
  [PASS] src/math.js created
  [PASS] add function exists
  [PASS] multiply function exists
  [PASS] test/math.test.js created
  [PASS] Tests pass

Test 7: Git commit history...
  [PASS] Multiple commits created (3 total)

Test 8: No extra features added...
  [PASS] No extra features added

=========================================
 Token Usage Analysis
=========================================

Usage Breakdown:
----------------------------------------------------------------------------------------------------
Agent           Description                          Msgs      Input     Output      Cache     Cost
----------------------------------------------------------------------------------------------------
main            Main session (coordinator)             34         27      3,996  1,213,703 $   4.09
3380c209        implementing Task 1: Create Add Function     1          2        787     24,989 $   0.09
34b00fde        implementing Task 2: Create Multiply Function     1          4        644     25,114 $   0.09
3801a732        reviewing whether an implementation matches...   1          5        703     25,742 $   0.09
4c142934        doing a final code review...                    1          6        854     25,319 $   0.09
5f017a42        a code reviewer. Review Task 2...               1          6        504     22,949 $   0.08
a6b7fbe4        a code reviewer. Review Task 1...               1          6        515     22,534 $   0.08
f15837c0        reviewing whether an implementation matches...   1          6        416     22,485 $   0.07
----------------------------------------------------------------------------------------------------

TOTALS:
  Total messages:         41
  Input tokens:           62
  Output tokens:          8,419
  Cache creation tokens:  132,742
  Cache read tokens:      1,382,835

  Total input (incl cache): 1,515,639
  Total tokens:             1,524,058

  Estimated cost: $4.67
  (at $3/$15 per M tokens for input/output)

========================================
 Test Summary
========================================

STATUS: PASSED
```

## Token 分析工具

### 使用方式

分析任何 Claude Code 會話的 token 使用量：

```bash
python3 tests/claude-code/analyze-token-usage.py ~/.claude/projects/<project-dir>/<session-id>.jsonl
```

### 尋找會話檔案

會話逐字稿儲存在 `~/.claude/projects/`，其目錄名稱會包含工作目錄路徑編碼：

```bash
# Example for /Users/jesse/Documents/GitHub/superpowers/superpowers
SESSION_DIR="$HOME/.claude/projects/-Users-jesse-Documents-GitHub-superpowers-superpowers"

# Find recent sessions
ls -lt "$SESSION_DIR"/*.jsonl | head -5
```

### 顯示內容

- **主會話使用量**：協調者（你或主 Claude 實例）的 token 使用量
- **每個子代理拆解**：每次 Task 呼叫包含：
  - 代理 ID
  - 描述（從提示中擷取）
  - 訊息數量
  - 輸入/輸出 token
  - 快取使用量
  - 成本估算
- **總計**：整體 token 使用量與成本估算

### 了解輸出

- **高快取讀取**：好事，代表提示快取運作正常
- **主會話輸入 token 偏高**：正常，協調者需要完整脈絡
- **子代理成本相近**：正常，每個任務的複雜度類似
- **每個任務成本**：常見範圍為每個子代理 $0.05-$0.15，取決於任務

## 疑難排解

### 技能無法載入

**問題**：在執行無頭測試時找不到技能

**解法**：
1. 確認你是從 superpowers 目錄執行：`cd /path/to/superpowers && tests/...`
2. 檢查 `~/.claude/settings.json` 的 `enabledPlugins` 是否包含 `"superpowers@superpowers-dev": true`
3. 確認 `skills/` 目錄中存在技能

### 權限錯誤

**問題**：Claude 被阻擋寫入檔案或存取目錄

**解法**：
1. 使用 `--permission-mode bypassPermissions` 參數
2. 使用 `--add-dir /path/to/temp/dir` 授權測試目錄存取
3. 檢查測試目錄的檔案權限

### 測試逾時

**問題**：測試時間過長而逾時

**解法**：
1. 增加逾時時間：`timeout 1800 claude ...`（30 分鐘）
2. 檢查技能邏輯是否有無限迴圈
3. 檢視子代理任務複雜度

### 找不到會話檔案

**問題**：測試執行後找不到會話逐字稿

**解法**：
1. 檢查 `~/.claude/projects/` 下是否是正確的專案目錄
2. 使用 `find ~/.claude/projects -name "*.jsonl" -mmin -60` 尋找近期會話
3. 確認測試實際有執行（檢查測試輸出是否有錯誤）

## 撰寫新的整合測試

### 範本

```bash
#!/usr/bin/env bash
set -euo pipefail

SCRIPT_DIR="$(cd "$(dirname "$0")" && pwd)"
source "$SCRIPT_DIR/test-helpers.sh"

# Create test project
TEST_PROJECT=$(create_test_project)
trap "cleanup_test_project $TEST_PROJECT" EXIT

# Set up test files...
cd "$TEST_PROJECT"

# Run Claude with skill
PROMPT="Your test prompt here"
cd "$SCRIPT_DIR/../.." && timeout 1800 claude -p "$PROMPT" \
  --allowed-tools=all \
  --add-dir "$TEST_PROJECT" \
  --permission-mode bypassPermissions \
  2>&1 | tee output.txt

# Find and analyze session
WORKING_DIR_ESCAPED=$(echo "$SCRIPT_DIR/../.." | sed 's/\//-/g' | sed 's/^-//')
SESSION_DIR="$HOME/.claude/projects/$WORKING_DIR_ESCAPED"
SESSION_FILE=$(find "$SESSION_DIR" -name "*.jsonl" -type f -mmin -60 | sort -r | head -1)

# Verify behavior by parsing session transcript
if grep -q '"name":"Skill".*"skill":"your-skill-name"' "$SESSION_FILE"; then
    echo "[PASS] Skill was invoked"
fi

# Show token analysis
python3 "$SCRIPT_DIR/analyze-token-usage.py" "$SESSION_FILE"
```

### 最佳實務

1. **務必清理**：使用 trap 清理暫存目錄
2. **解析逐字稿**：不要 grep 使用者輸出，請解析 `.jsonl` 會話檔
3. **授權權限**：使用 `--permission-mode bypassPermissions` 與 `--add-dir`
4. **從外掛目錄執行**：技能只會在 superpowers 目錄中載入
5. **顯示 token 使用量**：務必附上 token 分析，以利成本可見性
6. **測試真實行為**：驗證實際檔案建立、測試通過、提交完成

## 會話逐字稿格式

會話逐字稿是 JSONL（JSON Lines）檔案，每行是一個 JSON 物件，代表一則訊息或工具結果。

### 關鍵欄位

```json
{
  "type": "assistant",
  "message": {
    "content": [...],
    "usage": {
      "input_tokens": 27,
      "output_tokens": 3996,
      "cache_read_input_tokens": 1213703
    }
  }
}
```

### 工具結果

```json
{
  "type": "user",
  "toolUseResult": {
    "agentId": "3380c209",
    "usage": {
      "input_tokens": 2,
      "output_tokens": 787,
      "cache_read_input_tokens": 24989
    },
    "prompt": "You are implementing Task 1...",
    "content": [{"type": "text", "text": "..."}]
  }
}
```

`agentId` 欄位會對應到子代理會話，而 `usage` 欄位包含該子代理呼叫的 token 使用量。
