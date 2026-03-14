# Superpowers 發行說明

## v5.0.2 (2026-03-11)

### 零相依腦力激盪伺服器

**移除所有隨附的 node_modules — server.js 現在完全自足**

- 以使用內建 `http`、`fs`、`crypto` 模組的零相依 Node.js 伺服器取代 Express/Chokidar/WebSocket 相依
- 移除約 1,200 行隨附的 `node_modules/`、`package.json` 與 `package-lock.json`
- 自製 WebSocket 協定實作（RFC 6455 框架、ping/pong、正確的關閉握手）
- 以原生 `fs.watch()` 檔案監看取代 Chokidar
- 完整測試套件：HTTP 服務、WebSocket 協定、檔案監看與整合測試

### 腦力激盪伺服器穩定性

- **閒置 30 分鐘後自動退出** — 沒有客戶端連線時伺服器會關閉，避免孤兒程序
- **擁有者程序追蹤** — 伺服器監控父 harness PID，當擁有的工作階段死亡時退出
- **存活檢查** — 技能在重用既有實例前確認伺服器可回應
- **編碼修正** — 提供正確的 `<meta charset="utf-8">` 於提供的 HTML 頁面

### 子代理情境隔離

- 所有委派類技能（brainstorming、dispatching-parallel-agents、requesting-code-review、subagent-driven-development、writing-plans）現在都包含情境隔離原則
- 子代理只接收所需的情境，避免上下文視窗污染

## v5.0.1 (2026-03-10)

### Agentskills 相容

**brainstorm-server 移入技能目錄**

- 依照 [agentskills.io](https://agentskills.io) 規範，`lib/brainstorm-server/` → `skills/brainstorming/scripts/`
- 所有 `${CLAUDE_PLUGIN_ROOT}/lib/brainstorm-server/` 參照已改為相對 `scripts/` 路徑
- 技能現在完全跨平台可攜 — 不需要平台特定環境變數即可定位腳本
- `lib/` 目錄已移除（原本是最後剩下的內容）

### 新功能

**Gemini CLI 擴充**

- 透過 repo 根目錄的 `gemini-extension.json` 與 `GEMINI.md` 提供原生 Gemini CLI 擴充支援
- `GEMINI.md` 在工作階段開始時 @imports `using-superpowers` 技能與工具對照表
- Gemini CLI 工具對照參考（`skills/using-superpowers/references/gemini-tools.md`）— 將 Claude Code 工具名稱（Read、Write、Edit、Bash 等）轉為 Gemini CLI 對應項（read_file、write_file、replace 等）
- 記錄 Gemini CLI 限制：不支援子代理，技能改回 `executing-plans`
- 擴充根目錄設在 repo 根目錄以提升跨平台相容（避免 Windows 符號連結問題）
- README 新增安裝說明

### 改進

**多平台腦力激盪伺服器啟動**

- `visual-companion.md` 提供各平台啟動說明：Claude Code（預設模式）、Codex（透過 `CODEX_CI` 自動前景）、Gemini CLI（`--foreground` 搭配 `is_background`），以及其他環境的備援方式
- 伺服器現在會把啟動 JSON 寫到 `$SCREEN_DIR/.server-info`，即使背景執行導致 stdout 被隱藏，代理也能找到 URL 與連接埠

**腦力激盪伺服器相依已打包**

- `node_modules` 已隨 repo 一起打包，讓腦力激盪伺服器在全新外掛安裝後即可運作，不需執行階段依賴 `npm`
- 從打包相依移除 `fsevents`（僅 macOS 的原生二進位；chokidar 沒有它也能優雅退回）
- 若缺少 `node_modules`，改以 `npm install` 進行備援自動安裝

**OpenCode 工具對照修正**

- `TodoWrite` → `todowrite`（先前錯誤對應為 `update_plan`）；已依 OpenCode 原始碼驗證

### 錯誤修正

**Windows/Linux：單引號會破壞 SessionStart hook** (#577, #529, #644, PR #585)

- hooks.json 內 `${CLAUDE_PLUGIN_ROOT}` 兩側的單引號在 Windows 失效（cmd.exe 不把單引號視為路徑界定符），在 Linux 也失效（單引號會阻止變數展開）
- 修正：改成跳脫的雙引號 — 可在 macOS bash、Windows cmd.exe、Windows Git Bash、Linux 皆正常運作，且路徑含空白也沒問題
- 已在 Windows 11（NT 10.0.26200.0）搭配 Claude Code 2.1.72 與 Git for Windows 驗證

**腦力激盪規格審查迴圈被跳過** (#677)

- 規格審查迴圈（派發 spec-document-reviewer 子代理並反覆直到核准）只出現在「設計完成後」的文字敘述，卻缺少於清單與流程圖
- 因為代理更依賴流程圖與清單，導致規格審查步驟完全被跳過
- 已將第 7 步（規格審查迴圈）加入清單，並在 dot 圖新增對應節點
- 以 `claude --plugin-dir` 與 `claude-session-driver` 測試：worker 現在能正確派發 reviewer

**Cursor 安裝指令** (PR #676)

- 修正 README 的 Cursor 安裝指令：`/plugin-add` → `/add-plugin`（已由 Cursor 2.5 發佈公告確認）

**腦力激盪中的使用者審核關卡** (#565)

- 在規格完成與 writing-plans 交接之間新增明確的使用者審核步驟
- 使用者必須先核准規格，才會開始實作規劃
- 清單、流程與文字敘述皆已更新

**Session-start hook 只在每個平台發送一次情境**

- Hook 現在會判斷是在 Claude Code 或其他平台
- Claude Code 輸出 `hookSpecificOutput`，其他平台輸出 `additional_context` — 避免重複注入情境

**Token 分析腳本的 lint 修正**

- `tests/claude-code/analyze-token-usage.py` 中 `except:` → `except Exception:`

### 維護

**移除死碼**

- 刪除 `lib/skills-core.js` 及其測試（`tests/opencode/test-skills-core.js`）— 自 2026 年 2 月起已未使用
- 從 `tests/opencode/test-plugin-loading.sh` 移除 skills-core 存在性檢查

### 社群

- @karuturi — Claude Code 官方市集安裝說明 (PR #610)
- @mvanhorn — session-start hook 雙重輸出修正、OpenCode 工具對照修正
- @daniel-graham — bare except 的 lint 修正
- PR #585 作者 — Windows/Linux hooks 引號修正

---

## v5.0.0 (2026-03-09)

### 破壞性變更

**規格與計畫目錄重構**

- 規格（腦力激盪輸出）現在存到 `docs/superpowers/specs/YYYY-MM-DD-<topic>-design.md`
- 計畫（writing-plans 輸出）現在存到 `docs/superpowers/plans/YYYY-MM-DD-<feature-name>.md`
- 使用者對規格/計畫位置的偏好設定可覆寫這些預設值
- 所有內部技能參照、測試檔與範例路徑都已更新以符合
- 遷移：若需要，可將既有檔案自 `docs/plans/` 移到新位置

**支援子代理的平台必須使用子代理驅動開發**

writing-plans 不再提供子代理驅動與 executing-plans 的選項。在具備子代理能力的 harness（Claude Code、Codex）上，必須使用 subagent-driven-development。executing-plans 保留給不支援子代理的平台，並會提示使用者 Superpowers 在支援子代理的平台上效果更好。

**Executing-plans 不再分批執行**

移除「執行 3 個任務再停下來審查」的模式。計畫現在會持續執行，只有遇到阻塞才會停止。

**斜線指令已棄用**

`/brainstorm`、`/write-plan`、`/execute-plan` 現在會顯示棄用提示，導向對應技能。指令將在下一個主要版本移除。

### 新功能

**視覺化腦力激盪輔助**

可選的瀏覽器輔助工具，用於腦力激盪工作階段。當主題會受益於視覺呈現時，brainstorming 技能會在終端對話旁提供瀏覽器視窗，展示 mockup、圖表、比較等內容。

- `lib/brainstorm-server/` — WebSocket 伺服器，含瀏覽器輔助函式庫、工作階段管理腳本，以及深/淺色主題框架模板（「Superpowers Brainstorming」含 GitHub 連結）
- `skills/brainstorming/visual-companion.md` — 以漸進揭露方式說明伺服器流程、畫面編寫與回饋蒐集
- brainstorming 技能在流程中新增視覺輔助的決策點：完成專案背景探索後，判斷接下來的問題是否需要視覺內容，並在獨立訊息中提供輔助
- 逐題判斷：即使接受了輔助，每個問題仍會評估瀏覽器或終端哪個更合適
- 整合測試位於 `tests/brainstorm-server/`

**文件審查系統**

以子代理派發實作規格與計畫文件的自動審查迴圈：

- `skills/brainstorming/spec-document-reviewer-prompt.md` — 審查者檢查完整性、一致性、架構與 YAGNI
- `skills/writing-plans/plan-document-reviewer-prompt.md` — 審查者檢查規格一致性、任務拆解、檔案結構與檔案大小
- Brainstorming 在撰寫設計文件後派發規格審查者
- Writing-plans 在每個章節後加入以區塊為單位的計畫審查迴圈
- 審查迴圈會重複直到核准，或在 5 次迭代後升級處理
- 端到端測試位於 `tests/claude-code/test-document-review-system.sh`
- 設計規格與實作計畫位於 `docs/superpowers/`

**跨技能流程的架構指引**

在 brainstorming、writing-plans 與 subagent-driven-development 中加入「隔離式設計」與「檔案大小意識」指引：

- **Brainstorming** — 新增章節：「為隔離與清晰度而設計」（明確邊界、良好介面、可獨立測試單元）與「在既有程式碼庫工作」（遵循既有模式，只做精準改動）
- **Writing-plans** — 新增「檔案結構」章節：在定義任務前先規劃檔案與責任。新增「範圍檢查」保險：捕捉應在 brainstorming 拆解的多子系統規格
- **SDD 實作者** — 新增「程式碼組織」章節（遵循計畫的檔案結構、回報檔案膨脹疑慮）與「當你超出負荷時」的升級指引
- **SDD 程式碼品質審查者** — 現在會檢查架構、單元拆解、計畫符合性與檔案成長
- **規格/計畫審查者** — 審查準則新增架構與檔案大小
- **範圍評估** — Brainstorming 現在會判斷專案是否過大而不適合單一規格。多子系統需求會在早期被標記並拆成子專案，每個都有自己的規格 → 計畫 → 實作循環

**子代理驅動開發改進**

- **模型選擇** — 依任務類型選擇模型能力：機械式實作用便宜模型、整合用標準模型、架構與審查用高能力模型
- **實作者狀態協定** — 子代理現在回報 DONE、DONE_WITH_CONCERNS、BLOCKED 或 NEEDS_CONTEXT。控制者依狀態處理：補充情境重新派發、升級模型能力、拆分任務或升級給人類處理

### 改進

**指令優先序階層**

在 using-superpowers 中加入明確的優先序：

1. 使用者明確指示（CLAUDE.md、AGENTS.md、直接要求）— 最高優先
2. Superpowers 技能 — 覆寫預設系統行為
3. 預設系統提示 — 最低優先

若 CLAUDE.md 或 AGENTS.md 表示「不要用 TDD」，而技能說「一律使用 TDD」，則以使用者指示為準。

**SUBAGENT-STOP 關卡**

在 using-superpowers 中新增 `<SUBAGENT-STOP>` 區塊。被派發特定任務的子代理會跳過技能，不再觸發 1% 規則與完整技能流程。

**多平台改進**

- Codex 工具對照移至漸進揭露參考檔（`references/codex-tools.md`）
- 新增 Platform Adaptation 指引，讓非 Claude Code 平台找到對應工具
- 計畫標頭改以「agentic workers」稱呼，不再特指「Claude」
- `docs/README.codex.md` 記錄協作功能需求

**Writing-plans 範本更新**

- 計畫步驟改用核取方塊語法（`- [ ] **Step N:**`）以利進度追蹤
- 計畫標頭同時提及 subagent-driven-development 與 executing-plans，並依平台導向

---

## v4.3.1 (2026-02-21)

### 新增

**Cursor 支援**

Superpowers 現在可搭配 Cursor 的外掛系統使用。包含 `.cursor-plugin/plugin.json` 清單檔，以及 README 內 Cursor 專用安裝說明。SessionStart hook 的輸出現在除了既有的 `hookSpecificOutput.additionalContext`，還會包含 `additional_context` 欄位，以符合 Cursor hook 相容性。

### 修正

**Windows：恢復 polyglot wrapper 以確保 hook 穩定執行（#518、#504、#491、#487、#466、#440）**

Claude Code 在 Windows 上的 `.sh` 自動偵測會在 hook 指令前加上 `bash`，導致執行失敗。修正方式如下：

- 將 `session-start.sh` 改名為 `session-start`（無副檔名）避免自動偵測干擾
- 恢復 `run-hook.cmd` polyglot wrapper，支援多位置 bash 探測（標準 Git for Windows 路徑，再以 PATH 兜底）
- 找不到 bash 時靜默退出而非報錯
- 在 Unix 上 wrapper 直接用 `exec bash` 執行腳本
- 使用 POSIX 安全的 `dirname "$0"` 解析路徑（可在 dash/sh 運作，不限於 bash）

此修正解決 Windows 上路徑含空白、缺少 WSL、MSYS 下 `set -euo pipefail` 脆弱性，以及反斜線被破壞的 SessionStart 失敗問題。

## v4.3.0 (2026-02-12)

此修正大幅提升 superpowers 技能的相容性，並降低 Claude 誤入原生計畫模式的機率。

### 變更

**brainstorming 技能現在會強制流程，而非僅描述**

模型過去會跳過設計階段，直接進入 frontend-design 等實作技能，或將整個腦力激盪流程縮成單一文字區塊。此版本加入硬性關卡、必填清單與 graphviz 流程圖，以強制遵循：

- `<HARD-GATE>`：在設計呈現且使用者核准前，禁止使用實作技能、產出程式碼或腳手架
- 必須建立並依序完成的明確清單（6 項）
- Graphviz 流程圖，唯一有效終點是 `writing-plans`
- 針對「這太簡單不需要設計」的反模式提醒 — 這正是模型跳過流程的合理化理由
- 設計章節篇幅依章節複雜度調整，而非依專案複雜度

**using-superpowers 工作流程圖攔截 EnterPlanMode**

在技能流程圖中新增 `EnterPlanMode` 攔截。模型準備進入 Claude 原生計畫模式時，會先檢查是否已進行 brainstorming，並改為走 brainstorming 流程。計畫模式不再被使用。

### 修正

**SessionStart hook 現在同步執行**

hooks.json 中的 `async: true` 改為 `async: false`。若以 async 執行，hook 可能在模型第一輪之前尚未完成，導致 using-superpowers 指令未能進入上下文。

## v4.2.0 (2026-02-05)

### 破壞性變更

**Codex：以原生技能發現取代 bootstrap CLI**

已移除 `superpowers-codex` bootstrap CLI、Windows `.cmd` wrapper，以及相關 bootstrap 內容檔。Codex 現在使用原生技能發現機制（`~/.agents/skills/superpowers/` 符號連結），不再需要舊的 `use_skill`/`find_skills` CLI 工具。

安裝流程變為單純 clone + symlink（已於 INSTALL.md 記錄）。不再需要 Node.js 相依。舊的 `~/.codex/skills/` 路徑已棄用。

### 修正

**Windows：修正 Claude Code 2.1.x hook 執行問題（#331）**

Claude Code 2.1.x 變更了 Windows 上的 hook 執行方式：自動偵測 `.sh` 檔並在命令前加上 `bash`。這會破壞 polyglot wrapper 模式，因為 `bash "run-hook.cmd" session-start.sh` 會把 `.cmd` 當成 bash 腳本執行。

修正：hooks.json 現在直接呼叫 session-start.sh。Claude Code 2.1.x 會自動處理 bash 呼叫。另新增 .gitattributes 以強制 shell scripts 使用 LF 行尾（修正 Windows checkout 的 CRLF 問題）。

**Windows：SessionStart hook 以 async 執行以避免終端凍結（#404、#413、#414、#419）**

同步執行的 SessionStart hook 會阻擋 TUI 進入 raw mode，導致 Windows 上鍵盤輸入凍結。改為 async 可避免凍結，同時仍能注入 superpowers 情境。

**Windows：修正 `escape_for_json` O(n^2) 效能**

逐字元回圈使用 `${input:$i:1}` 在 bash 中是 O(n^2)（因為子字串拷貝成本），在 Windows Git Bash 需 60+ 秒。改為 bash 參數替換（`${s//old/new}`），每個模式只跑一次 C-level pass — 在 macOS 快 7 倍，Windows 更是大幅改善。

**Codex：修正 Windows/PowerShell 呼叫（#285、#243）**

- Windows 不遵守 shebang，直接執行無副檔名 `superpowers-codex` 會跳出「以何種程式開啟」對話框。所有呼叫現在改以前置 `node`
- 修正 Windows 上 `~/` 路徑展開 — PowerShell 不會在傳給 `node` 的參數中展開 `~`。改用 `$HOME`，在 bash 與 PowerShell 都能正確展開

**Codex：修正安裝程式的路徑解析**

改用 `fileURLToPath()` 取代手動 URL pathname 解析，以正確處理含空白與特殊字元的路徑。

**Codex：修正 writing-skills 中的舊 skills 路徑**

將 `~/.codex/skills/`（已棄用）更新為 `~/.agents/skills/` 以符合原生發現。

### 改進

**實作前必須進行 worktree 隔離**

將 `using-git-worktrees` 列為 `subagent-driven-development` 與 `executing-plans` 的必需技能。實作流程現在明確要求先建立隔離的 worktree，避免直接在 main 上作業。

**主分支保護改為需明確同意**

不再完全禁止在 main 分支上作業，而是要求使用者明確同意。更彈性，同時仍能確保使用者了解影響。

**簡化安裝驗證**

移除 `/help` 指令檢查與特定斜線指令列表。技能主要透過描述需求觸發，而非執行特定指令。

**Codex：釐清 bootstrap 中的子代理工具對照**

改善文件，說明 Codex 工具如何對應 Claude Code 以支援子代理流程。

### 測試

- 新增子代理驅動開發的 worktree 需求測試
- 新增主分支紅旗警告測試
- 修正技能辨識測試斷言的大小寫敏感問題

---

## v4.1.1 (2026-01-23)

### 修正

**OpenCode：依官方文件統一使用 `plugins/` 目錄（#343）**

OpenCode 官方文件使用 `~/.config/opencode/plugins/`（複數）。先前文件使用 `plugin/`（單數）。雖然 OpenCode 同時支援兩者，但我們已統一採用官方慣例以避免混淆。

變更內容：
- 將 repo 結構中的 `.opencode/plugin/` 重新命名為 `.opencode/plugins/`
- 更新所有安裝文件（INSTALL.md、README.opencode.md）跨平台說明
- 更新測試腳本以一致

**OpenCode：修正 symlink 指示（#339、#342）**

- 在 `ln -s` 前明確加入 `rm`（修正重裝時「檔案已存在」錯誤）
- 新增 INSTALL.md 中缺漏的技能 symlink 步驟
- 由已棄用的 `use_skill`/`find_skills` 改為原生 `skill` 工具參照

---

## v4.1.0 (2026-01-23)

### 破壞性變更

**OpenCode：改用原生技能系統**

Superpowers for OpenCode 現在使用 OpenCode 原生 `skill` 工具，而非自訂的 `use_skill`/`find_skills` 工具。整合更乾淨，且能與 OpenCode 的原生技能發現機制配合。

**需要遷移：** 必須將技能 symlink 到 `~/.config/opencode/skills/superpowers/`（請見更新後的安裝文件）。

### 修正

**OpenCode：修正工作階段開始時的 agent 重置問題（#226）**

先前以 `session.prompt({ noReply: true })` 注入 bootstrap 的方式，會讓 OpenCode 在第一則訊息時把選定的 agent 重置為「build」。現在改用 `experimental.chat.system.transform` hook，直接修改 system prompt，避免副作用。

**OpenCode：修正 Windows 安裝（#232）**

- 移除對 `skills-core.js` 的相依（避免檔案被複製而非 symlink 時的相對匯入失敗）
- 新增 cmd.exe、PowerShell、Git Bash 的完整 Windows 安裝文件
- 針對各平台記錄正確的 symlink 與 junction 使用方式

**Claude Code：修正 Claude Code 2.1.x 的 Windows hook 執行**

Claude Code 2.1.x 變更了 Windows 上的 hook 執行方式：自動偵測 `.sh` 檔並在命令前加上 `bash `。這會破壞 polyglot wrapper 模式，因為 `bash "run-hook.cmd" session-start.sh` 會把 `.cmd` 當成 bash 腳本執行。

修正：hooks.json 現在直接呼叫 session-start.sh。Claude Code 2.1.x 會自動處理 bash 呼叫。另新增 .gitattributes 以強制 shell scripts 使用 LF 行尾（修正 Windows checkout 的 CRLF 問題）。

---

## v4.0.3 (2025-12-26)

### 改進

**強化 using-superpowers 對明確技能請求的處理**

修正 Claude 在使用者明確要求技能名稱時仍跳過技能的失敗模式（例如「subagent-driven-development, please」）。Claude 會覺得「我知道這是什麼」而直接動手，不去載入技能。

變更內容：
- 更新「The Rule」為「Invoke relevant or requested skills」而非「Check for skills」— 強調主動呼叫而不是被動檢查
- 新增「BEFORE any response or action」— 原本只寫「response」，導致 Claude 有時先動作再回覆
- 加入「即便呼叫錯技能也沒關係」的安撫 — 降低猶豫
- 新增紅旗：「I know what that means」→ 了解概念 ≠ 使用技能

**新增明確技能請求測試**

新增測試套件 `tests/explicit-skill-requests/`，驗證 Claude 在使用者點名技能時會正確呼叫。包含單回合與多回合測試情境。

## v4.0.2 (2025-12-23)

### 修正

**斜線指令現在僅限使用者**

為三個斜線指令（`/brainstorm`、`/execute-plan`、`/write-plan`）加入 `disable-model-invocation: true`。Claude 無法透過 Skill 工具呼叫這些指令 — 只能由使用者手動執行。底層技能（`superpowers:brainstorming`、`superpowers:executing-plans`、`superpowers:writing-plans`）仍可由 Claude 自主呼叫。此變更避免 Claude 呼叫指令但其實只是轉向技能而造成混淆。

## v4.0.1 (2025-12-23)

### 修正

**釐清 Claude Code 中的技能存取方式**

修正一個讓人困惑的模式：Claude 透過 Skill 工具呼叫技能後，又嘗試用 Read 去讀技能檔。using-superpowers 現在明確指出 Skill 工具會直接載入技能內容 — 不需要讀檔。

- 在 `using-superpowers` 新增「How to Access Skills」章節
- 指令文字由「read the skill」改為「invoke the skill」
- 斜線指令更新為完整限定技能名稱（例如 `superpowers:brainstorming`）

**在 receiving-code-review 加入 GitHub 討論串回覆指引**（感謝 @ralphbean）

新增提醒：應該在原始討論串內回覆 inline review 意見，而不是以頂層 PR 註解回覆。

**在 writing-skills 加入「以自動化取代文件」指引**（感謝 @EthanJStark）

新增指引：機械式約束應以自動化處理，而非寫進文件 — 技能應保留給需要判斷的事項。

## v4.0.0 (2025-12-17)

### 新功能

**子代理驅動開發的雙階段程式碼審查**

子代理流程在每個任務後分成兩個審查階段：

1. **規格符合性審查** - 懷疑論審查者確認實作完全符合規格。可抓出缺漏需求與過度建置。不會相信實作者回報 — 會直接讀程式碼。

2. **程式碼品質審查** - 只在規格符合性通過後執行。審查乾淨程式碼、測試覆蓋與可維護性。

這可避免常見失敗模式：程式寫得很好但不符合需求。審查是迴圈，不是一次性 — 審查者發現問題後由實作者修正，再回到審查。

其他子代理流程改進：
- 控制者提供完整任務文字給工作者（而非只給檔案參照）
- 工作者可在開始前與進行中提出澄清問題
- 回報完成前加入自我審查清單
- 計畫只在起始讀取一次，摘錄至 TodoWrite

在 `skills/subagent-driven-development/` 新增提示範本：
- `implementer-prompt.md` - 含自我審查清單，鼓勵提問
- `spec-reviewer-prompt.md` - 懷疑式需求驗證
- `code-quality-reviewer-prompt.md` - 標準程式碼審查

**除錯技術與工具整合**

`systematic-debugging` 現在打包支援技術與工具：
- `root-cause-tracing.md` - 逆向追蹤 bug 來源
- `defense-in-depth.md` - 在多層加入驗證
- `condition-based-waiting.md` - 以條件輪詢取代任意 timeout
- `find-polluter.sh` - 用於找出污染測試的二分法腳本
- `condition-based-waiting-example.ts` - 來自真實除錯工作階段的完整實作

**測試反模式參考**

`test-driven-development` 新增 `testing-anti-patterns.md`，涵蓋：
- 測試 mock 行為而非真實行為
- 在 production class 加入測試專用方法
- 在不了解相依的情況下進行 mock
- 不完整的 mock 隱藏結構性假設

**技能測試基礎設施**

新增三套測試框架以驗證技能行為：

`tests/skill-triggering/` - 驗證技能能被非明確命名的提示觸發。測試 6 項技能以確保僅靠描述就能觸發。

`tests/claude-code/` - 使用 `claude -p` 的整合測試（無頭測試）。透過 session transcript（JSONL）分析來驗證技能使用。包含 `analyze-token-usage.py` 以追蹤成本。

`tests/subagent-driven-dev/` - 端到端流程驗證，包含兩個完整測試專案：
- `go-fractals/` - CLI 工具，含 Sierpinski/Mandelbrot（10 個任務）
- `svelte-todo/` - CRUD 應用，含 localStorage 與 Playwright（12 個任務）

### 重大變更

**DOT 流程圖作為可執行規格**

以 DOT/GraphViz 流程圖重寫關鍵技能，並作為權威流程定義；文字敘述改為輔助。

**The Description Trap**（記錄於 `writing-skills`）：

發現技能描述會覆蓋流程圖內容，當描述內含流程摘要時，Claude 會跟隨短描述而非讀取詳細流程圖。修正方式：描述只能是觸發條件（「Use when X」），不含流程細節。

**using-superpowers 中的技能優先順序**

當多個技能適用時，流程型技能（brainstorming、debugging）明確優先於實作技能。「Build X」會先觸發 brainstorming，再進入領域技能。

**強化 brainstorming 觸發條件**

描述改為命令式：「在任何創作工作之前你都 MUST 使用此技能—建立功能、建置元件、新增功能或修改行為。」

### 破壞性變更

**技能整併** - 六個獨立技能已合併：
- `root-cause-tracing`、`defense-in-depth`、`condition-based-waiting` → 合併至 `systematic-debugging/`
- `testing-skills-with-subagents` → 合併至 `writing-skills/`
- `testing-anti-patterns` → 合併至 `test-driven-development/`
- `sharing-skills` 移除（已淘汰）

### 其他改進

- **render-graphs.js** - 用於擷取技能中的 DOT 圖並轉成 SVG
- **Rationalizations table** in using-superpowers - 以可掃描格式加入新條目：「I need more context first」、「Let me explore first」、「This feels productive」
- **docs/testing.md** - Claude Code 整合測試的技能測試指南

---

## v3.6.2 (2025-12-03)

### 修正

- **Linux 相容性**：修正 polyglot hook wrapper（`run-hook.cmd`）以符合 POSIX 語法
  - 將第 16 行的 bash 專用 `${BASH_SOURCE[0]:-$0}` 改為標準 `$0`
  - 解決 Ubuntu/Debian 上 `/bin/sh` 為 dash 時的「Bad substitution」錯誤
  - 修正 #141

---

## v3.5.1 (2025-11-24)

### 變更

- **OpenCode Bootstrap 重構**：從 `chat.message` hook 改為 `session.created` 事件注入 bootstrap
  - 現在在 session 建立時透過 `session.prompt()`（`noReply: true`）注入
  - 明確告知模型 using-superpowers 已載入，以避免重複載入技能
  - bootstrap 內容生成整併至共用的 `getBootstrapContent()` helper
  - 單一實作更乾淨（移除備援模式）

---

## v3.5.0 (2025-11-23)

### 新增

- **OpenCode 支援**：OpenCode.ai 原生 JavaScript 外掛
  - 自訂工具：`use_skill` 與 `find_skills`
  - 透過訊息插入模式讓技能在上下文壓縮後仍可保留
  - 透過 chat.message hook 自動注入情境
  - 在 session.compacted 事件後自動重新注入
  - 三層技能優先序：專案 > 個人 > superpowers
  - 支援專案本地技能（`.opencode/skills/`）
  - 共享核心模組（`lib/skills-core.js`）供 Codex 重用
  - 具備隔離性的自動化測試套件（`tests/opencode/`）
  - 平台專屬文件（`docs/README.opencode.md`、`docs/README.codex.md`）

### 變更

- **Codex 實作重構**：改為使用共享的 `lib/skills-core.js` ES module
  - 消除 Codex 與 OpenCode 之間的程式碼重複
  - 技能發現與解析的單一真實來源
  - Codex 已能透過 Node.js interop 載入 ES modules

- **文件改進**：重寫 README 以清楚說明問題與解法
  - 移除重複章節與矛盾資訊
  - 新增完整流程描述（brainstorm → plan → execute → finish）
  - 簡化各平台安裝說明
  - 強調技能檢查協議，而非「自動啟用」的宣稱

---

## v3.4.1 (2025-10-31)

### 改進

- 最佳化 superpowers bootstrap 以消除重複技能執行。`using-superpowers` 技能內容現在直接提供於工作階段情境中，並明確指示 Skill 工具僅用於其他技能。這可降低負擔並避免代理明明已從 session start 取得內容卻仍手動執行 `using-superpowers` 的混淆迴圈。

## v3.4.0 (2025-10-30)

### 改進

- 簡化 `brainstorming` 技能，回歸原本的對話式願景。移除沉重的 6 階段流程與正式清單，改為自然對話：一次問一題，再以 200-300 字章節呈現設計並確認。保留文件與實作交接功能。

## v3.3.1 (2025-10-28)

### 改進

- 更新 `brainstorming` 技能，要求在提問前先進行自主偵查，鼓勵以建議導向的決策，並避免代理把優先順序回拋給人類。
- 依照 Strunk 的《寫作風格的要素》原則改善 `brainstorming` 技能的寫作品質（刪除不必要詞語、負向轉正向、改善並列結構）。

### 錯誤修正

- 釐清 `writing-skills` 指引，指出正確的代理個人技能目錄（Claude Code 為 `~/.claude/skills`，Codex 為 `~/.codex/skills`）。

## v3.3.0 (2025-10-28)

### 新功能

**Codex 實驗性支援**
- 新增統一的 `superpowers-codex` 腳本，包含 bootstrap/use-skill/find-skills 指令
- 跨平台 Node.js 實作（Windows、macOS、Linux 皆可）
- 技能命名空間：superpowers 技能使用 `superpowers:skill-name`，個人技能使用 `skill-name`
- 同名時個人技能優先於 superpowers 技能
- 乾淨的技能顯示：僅顯示名稱/描述，不含原始 frontmatter
- 有用情境：顯示每個技能的支援檔案目錄
- Codex 工具對照：TodoWrite→update_plan、子代理→手動備援等
- 精簡的 AGENTS.md bootstrap 整合以自動啟動
- 完整的 Codex 安裝指南與 bootstrap 指示

**與 Claude Code 整合的關鍵差異：**
- 單一統一腳本，而非分離的工具
- Codex 專屬的工具替代系統
- 簡化子代理處理（以手動作業取代委派）
- 更新用語：「Superpowers skills」取代「Core skills」

### 新增檔案
- `.codex/INSTALL.md` - Codex 使用者安裝指南
- `.codex/superpowers-bootstrap.md` - 含 Codex 適配的 bootstrap 指示
- `.codex/superpowers-codex` - 統一 Node.js 執行檔，提供完整功能

**注意：** Codex 支援仍為實驗性。整合提供核心 superpowers 功能，但可能需要依使用者回饋進一步調整。

## v3.2.3 (2025-10-23)

### 改進

**using-superpowers 技能改用 Skill 工具（不再用 Read 工具）**
- 將技能呼叫指示由 Read 工具改為 Skill 工具
- 更新描述：「using Read tool」→「using Skill tool」
- 更新步驟 3：「Use the Read tool」→「Use the Skill tool to read and run」
- 更新合理化列表：「Read the current version」→「Run the current version」

Skill 工具才是 Claude Code 呼叫技能的正確機制。此更新修正 bootstrap 指示，讓代理使用正確工具。

## v3.2.2 (2025-10-21)

### 改進

**強化 using-superpowers 技能以防代理合理化**
- 新增 EXTREMELY-IMPORTANT 區塊，使用絕對語氣強制技能檢查
  - 「即使只有 1% 機率適用，也 MUST 讀取」
  - 「你沒有選擇。不能合理化逃避。」
- 新增 MANDATORY FIRST RESPONSE PROTOCOL 清單
  - 5 步驟流程，代理在任何回應前必須完成
  - 明確「未遵循即失敗」後果
- 新增 Common Rationalizations 章節，列出 8 種特定迴避模式
  - 「這只是個簡單問題」→ 錯
  - 「我可以先快速看檔案」→ 錯
  - 「先蒐集資訊」→ 錯
  - 以及另外 5 種常見模式

此變更針對觀察到的代理行為：即使有明確指示仍會合理化跳過技能。強硬語氣與預先反駁讓不遵循更困難。

### 檔案變更
- 更新：`skills/using-superpowers/SKILL.md` - 新增三層強制機制以防跳過技能

## v3.2.1 (2025-10-20)

### 新功能

**外掛新增 code reviewer agent**
- 新增 `superpowers:code-reviewer` agent 至外掛 `agents/` 目錄
- Agent 依計畫與程式碼標準提供系統化審查
- 過去需要使用者自行設定個人 agent
- 所有技能參照更新為 `superpowers:code-reviewer`
- 修正 #55

### 檔案變更
- 新增：`agents/code-reviewer.md` - Agent 定義與審查清單、輸出格式
- 更新：`skills/requesting-code-review/SKILL.md` - 更新 `superpowers:code-reviewer` 參照
- 更新：`skills/subagent-driven-development/SKILL.md` - 更新 `superpowers:code-reviewer` 參照

## v3.2.0 (2025-10-18)

### 新功能

**腦力激盪流程中的設計文件**
- 在 brainstorming 技能中加入第 4 階段：設計文件
- 設計文件在實作前寫入 `docs/plans/YYYY-MM-DD-<topic>-design.md`
- 恢復原本腦力激盪指令在技能轉換後遺失的功能
- 設計文件在 worktree 設定與實作規劃前完成
- 以子代理測試確認在時間壓力下仍能遵循

### 破壞性變更

**技能參照命名空間標準化**
- 所有內部技能參照均使用 `superpowers:` 命名空間前綴
- 更新格式：`superpowers:test-driven-development`（原先是 `test-driven-development`）
- 影響所有 REQUIRED SUB-SKILL、RECOMMENDED SUB-SKILL 與 REQUIRED BACKGROUND 參照
- 與 Skill 工具呼叫方式一致
- 更新檔案：brainstorming、executing-plans、subagent-driven-development、systematic-debugging、testing-skills-with-subagents、writing-plans、writing-skills

### 改進

**設計 vs 實作計畫命名**
- 設計文件使用 `-design.md` 後綴，避免檔名衝突
- 實作計畫延用既有 `YYYY-MM-DD-<feature-name>.md` 格式
- 兩者都存於 `docs/plans/` 目錄，命名區隔清楚

## v3.1.1 (2025-10-17)

### 錯誤修正

- **修正 README 指令語法** (#44) - 更新所有指令參照為正確的命名空間語法（`/superpowers:brainstorm` 而非 `/brainstorm`）。外掛提供的指令會由 Claude Code 自動加上命名空間，以避免外掛間衝突。

## v3.1.0 (2025-10-17)

### 破壞性變更

**技能名稱統一為小寫**
- 所有技能 frontmatter 的 `name:` 欄位統一為小寫 kebab-case，與目錄名稱一致
- 例如：`brainstorming`、`test-driven-development`、`using-git-worktrees`
- 所有技能公告與交叉參照更新為小寫格式
- 確保目錄、frontmatter 與文件命名一致

### 新功能

**強化 brainstorming 技能**
- 新增 Quick Reference 表，顯示階段、活動與工具使用
- 新增可複製的工作流程清單以追蹤進度
- 新增決策流程圖，用於判斷何時回到較早階段
- 新增 AskUserQuestion 工具指引，並提供具體範例
- 新增「Question Patterns」章節，說明何時使用結構化或開放式問題
- 重構 Key Principles 為可掃描表格

**Anthropic best practices 整合**
- 新增 `skills/writing-skills/anthropic-best-practices.md` - Anthropic 官方技能撰寫指南
- 在 writing-skills SKILL.md 中引用以提供完整指引
- 提供漸進揭露、工作流程與評估的模式

### 改進

**技能交叉參照更清楚**
- 所有技能參照改用明確需求標記：
  - `**REQUIRED BACKGROUND:**` - 必須理解的前置
  - `**REQUIRED SUB-SKILL:**` - 工作流程必用技能
  - `**Complementary skills:**` - 選用但有助的相關技能
- 移除舊路徑格式（`skills/collaboration/X` → 只寫 `X`）
- 更新 Integration 章節的關聯分類（Required vs Complementary）
- 更新交叉參照文件以符合最佳實務

**與 Anthropic best practices 對齊**
- 修正文法與語氣（全為第三人稱）
- 新增 Quick Reference 表以便掃描
- 新增 Claude 可複製追蹤的工作流程清單
- 在非直覺決策點適當使用流程圖
- 改善可掃描表格格式
- 所有技能皆遠低於 500 行建議上限

### 錯誤修正

- **重新加入遺漏的指令轉向** - 恢復 `commands/brainstorm.md` 與 `commands/write-plan.md`，這些檔案在 v3.0 遷移中被誤刪
- 修正 `defense-in-depth` 名稱不一致（原為 `Defense-in-Depth-Validation`）
- 修正 `receiving-code-review` 名稱不一致（原為 `Code-Review-Reception`）
- 修正 `commands/brainstorm.md` 參照的技能名稱
- 移除對不存在的相關技能參照

### 文件

**writing-skills 改進**
- 更新交叉參照指引，加入明確需求標記
- 新增 Anthropic 官方最佳實務參考
- 改善範例，示範正確技能參照格式

## v3.0.1 (2025-10-16)

### 變更

我們現在使用 Anthropic 的第一方 skills 系統！

## v2.0.2 (2025-10-12)

### 錯誤修正

- **修正本地 skills repo 超前 upstream 時的錯誤警告** - 初始化腳本會錯誤顯示「New skills available from upstream」即使本地分支領先 upstream。邏輯已修正，可正確區分三種 git 狀態：本地落後（應更新）、本地超前（不警告）、分歧（需警告）。

## v2.0.1 (2025-10-12)

### 錯誤修正

- **修正外掛情境下的 session-start hook 執行問題** (#8, PR #9) - Hook 會因「Plugin hook error」而靜默失敗，導致技能情境未載入。修正方式：
  - 在 BASH_SOURCE 未綁定時使用 `${BASH_SOURCE[0]:-$0}` 兜底（適用於 Claude Code 的執行情境）
  - 使用 `|| true` 以避免過濾狀態旗標時 grep 無結果造成失敗

---

# Superpowers v2.0.0 發行說明

## 概覽

Superpowers v2.0 透過重大架構調整，讓技能更易於使用、維護與社群共創。

重點變更是 **技能倉庫分離**：所有技能、腳本與文件已從外掛移至獨立倉庫（[obra/superpowers-skills](https://github.com/obra/superpowers-skills)）。這讓 superpowers 從單體外掛轉為輕量 shim，負責管理本機技能倉庫的 clone。技能會在 session start 自動更新。使用者可 fork 並透過標準 git 流程貢獻改進。技能庫可獨立於外掛版本演進。

除了基礎設施，此版本新增九個聚焦於問題解決、研究與架構的技能。我們重寫核心 **using-skills** 文件，使用命令式語氣與更清楚的結構，讓 Claude 更容易理解何時與如何使用技能。**find-skills** 現在輸出可直接貼入 Read 工具的路徑，降低技能發現流程阻力。

使用者體驗是無縫的：外掛會自動處理 clone、fork 與更新。貢獻者會發現新的架構讓改進與分享技能變得非常容易。此版本為社群快速演進技能奠定基礎。

## 破壞性變更

### 技能倉庫分離

**最大變更：** 技能不再放在外掛內，而是移到獨立倉庫 [obra/superpowers-skills](https://github.com/obra/superpowers-skills)。

**對你而言代表什麼：**

- **首次安裝：** 外掛會自動將技能 clone 到 `~/.config/superpowers/skills/`
- **Fork：** 在設定過程中會提供 fork 技能倉庫的選項（若安裝 `gh`）
- **更新：** 技能在每次 session start 自動更新（可快轉合併時）
- **貢獻：** 在分支上工作、於本地提交、向 upstream 提 PR
- **不再有 shadowing：** 舊的雙層系統（個人/核心）已以單一倉庫分支流程取代

**遷移：**

若已有既有安裝：
1. 舊的 `~/.config/superpowers/.git` 會備份至 `~/.config/superpowers/.git.bak`
2. 舊技能會備份至 `~/.config/superpowers/skills.bak`
3. 在 `~/.config/superpowers/skills/` 建立新的 obra/superpowers-skills clone

### 移除功能

- **個人 superpowers 覆蓋系統** - 以 git 分支工作流程取代
- **setup-personal-superpowers hook** - 以 initialize-skills.sh 取代

## 新功能

### 技能倉庫基礎設施

**自動 Clone 與設定**（`lib/initialize-skills.sh`）
- 首次執行時 clone obra/superpowers-skills
- 若有 GitHub CLI，提供建立 fork 的選項
- 正確設定 upstream/origin remote
- 處理舊安裝的遷移

**自動更新**
- 每次 session start 從追蹤 remote 拉取
- 能快轉時自動合併
- 分支分歧時提示需手動同步
- 使用 pulling-updates-from-skills-repository 技能進行手動同步

### 新技能

**問題解決技能**（`skills/problem-solving/`）
- **collision-zone-thinking** - 強迫不相關概念碰撞以產生新洞見
- **inversion-exercise** - 反轉假設以揭露隱藏限制
- **meta-pattern-recognition** - 跨領域尋找通用原則
- **scale-game** - 以極端情境測試以暴露根本真相
- **simplification-cascades** - 找到能同時刪除多個組件的洞見
- **when-stuck** - 將問題派送至合適的解題技巧

**研究技能**（`skills/research/`）
- **tracing-knowledge-lineages** - 理解想法如何隨時間演進

**架構技能**（`skills/architecture/`）
- **preserving-productive-tensions** - 保留多個有效方案，避免過早收斂

### 技能改進

**using-skills（原 getting-started）**
- getting-started 改名為 using-skills
- 以命令式語氣全面改寫（v4.0.0）
- 將關鍵規則前置
- 為所有流程加入「Why」說明
- 參照中一律包含 /SKILL.md 後綴
- 更清楚區分硬性規則與彈性模式

**writing-skills**
- 交叉參照指引從 using-skills 移出
- 新增 token 效率章節（字數目標）
- 改善 CSO（Claude Search Optimization）指引

**sharing-skills**
- 更新為新的分支與 PR 流程（v2.0.0）
- 移除個人/核心分拆參照

**pulling-updates-from-skills-repository**（新增）
- 完整的 upstream 同步流程
- 取代舊的「updating-skills」技能

### 工具改進

**find-skills**
- 現在輸出含 /SKILL.md 後綴的完整路徑
- 路徑可直接被 Read 工具使用
- 更新說明文字

**skill-run**
- 從 scripts/ 移到 skills/using-skills/
- 改善文件

### 外掛基礎設施

**Session Start Hook**
- 現在從技能倉庫位置載入
- 在 session start 顯示完整技能清單
- 顯示技能位置資訊
- 顯示更新狀態（已成功更新 / 落後 upstream）
- 將「技能落後」警告移到輸出末尾

**環境變數**
- `SUPERPOWERS_SKILLS_ROOT` 設定為 `~/.config/superpowers/skills`
- 全部路徑一致使用

## 錯誤修正

- 修正 fork 時重複新增 upstream remote
- 修正 find-skills 輸出重複「skills/」前綴
- 移除 session-start 中過時的 setup-personal-superpowers 呼叫
- 修正 hooks 與 commands 中的路徑參照

## 文件

### README
- 更新為新的技能倉庫架構
- 顯著標示 superpowers-skills 倉庫連結
- 更新自動更新說明
- 修正技能名稱與參照
- 更新 Meta 技能清單

### 測試文件
- 新增完整測試清單（`docs/TESTING-CHECKLIST.md`）
- 建立本地市集設定供測試
- 記錄手動測試情境

## 技術細節

### 檔案變更

**新增：**
- `lib/initialize-skills.sh` - 技能倉庫初始化與自動更新
- `docs/TESTING-CHECKLIST.md` - 手動測試情境
- `.claude-plugin/marketplace.json` - 本地測試設定

**移除：**
- `skills/` 目錄（82 檔）- 已移至 obra/superpowers-skills
- `scripts/` 目錄 - 已移至 obra/superpowers-skills/skills/using-skills/
- `hooks/setup-personal-superpowers.sh` - 已淘汰

**修改：**
- `hooks/session-start.sh` - 從 ~/.config/superpowers/skills 載入技能
- `commands/brainstorm.md` - 路徑更新為 SUPERPOWERS_SKILLS_ROOT
- `commands/write-plan.md` - 路徑更新為 SUPERPOWERS_SKILLS_ROOT
- `commands/execute-plan.md` - 路徑更新為 SUPERPOWERS_SKILLS_ROOT
- `README.md` - 針對新架構全面改寫

### Commit 歷史

此版本包含：
- 20+ 筆提交以完成技能倉庫分離
- PR #1：受 Amplifier 啟發的問題解決與研究技能
- PR #2：個人 superpowers 覆蓋系統（後續已替換）
- 多次技能精修與文件改進

## 升級說明

### 全新安裝

```bash
# 在 Claude Code 中
/plugin marketplace add obra/superpowers-marketplace
/plugin install superpowers@superpowers-marketplace
```

外掛會自動處理所有流程。

### 從 v1.x 升級

1. **備份你的個人技能**（如有）：
   ```bash
   cp -r ~/.config/superpowers/skills ~/superpowers-skills-backup
   ```

2. **更新外掛：**
   ```bash
   /plugin update superpowers
   ```

3. **下一次 session start：**
   - 舊安裝會自動備份
   - 會 clone 全新的技能倉庫
   - 若安裝 GitHub CLI，會提供 fork 選項

4. **遷移個人技能**（如有）：
   - 在本地技能倉庫建立分支
   - 從備份複製你的個人技能
   - 提交並推送至你的 fork
   - 可考慮提交 PR 回饋社群

## 接下來

### 給使用者

- 探索新的問題解決技能
- 嘗試以分支為基礎的技能改進流程
- 將技能貢獻回社群

### 給貢獻者

- 技能倉庫現在位於 https://github.com/obra/superpowers-skills
- Fork → Branch → PR 流程
- 參考 skills/meta/writing-skills/SKILL.md 的 TDD 文件撰寫方法

## 已知問題

目前沒有。

## 致謝

- 受 Amplifier 模式啟發的問題解決技能
- 社群貢獻與回饋
- 技能成效的大量測試與迭代

---

**完整變更紀錄：** https://github.com/obra/superpowers/compare/dd013f6...main
**技能倉庫：** https://github.com/obra/superpowers-skills
**問題追蹤：** https://github.com/obra/superpowers/issues
