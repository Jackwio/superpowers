# Gemini CLI 工具對應

技能使用 Claude Code 的工具名稱。當你在技能中看到它們時，請使用你平台上的對應工具：

| 技能引用 | Gemini CLI 對應 |
|-----------------|----------------------|
| `Read`（讀檔） | `read_file` |
| `Write`（建立檔案） | `write_file` |
| `Edit`（編輯檔案） | `replace` |
| `Bash`（執行指令） | `run_shell_command` |
| `Grep`（搜尋檔案內容） | `grep_search` |
| `Glob`（依名稱搜尋檔案） | `glob` |
| `TodoWrite`（任務追蹤） | `write_todos` |
| `Skill` 工具（呼叫技能） | `activate_skill` |
| `WebSearch` | `google_web_search` |
| `WebFetch` | `web_fetch` |
| `Task` 工具（派出子代理） | 無對應 — Gemini CLI 不支援子代理 |

## 不支援子代理

Gemini CLI 沒有 Claude Code 的 `Task` 工具對應。依賴子代理派發的技能（`subagent-driven-development`、`dispatching-parallel-agents`）會改用 `executing-plans` 以單一會話執行。

## Gemini CLI 其他工具

以下工具在 Gemini CLI 可用，但 Claude Code 沒有對應：

| 工具 | 用途 |
|------|---------|
| `list_directory` | 列出檔案與子目錄 |
| `save_memory` | 跨會話將事實保存到 GEMINI.md |
| `ask_user` | 向使用者請求結構化輸入 |
| `tracker_create_task` | 完整任務管理（建立、更新、列出、視覺化） |
| `enter_plan_mode` / `exit_plan_mode` | 切換到只讀研究模式後再修改 |
