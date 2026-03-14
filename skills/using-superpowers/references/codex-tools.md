# Codex 工具對應

技能使用 Claude Code 的工具名稱。當你在技能中看到它們時，請使用你平台上的對應工具：

| 技能引用 | Codex 對應 |
|-----------------|------------------|
| `Task` 工具（派出子代理） | `spawn_agent` |
| 多次 `Task` 呼叫（平行） | 多次 `spawn_agent` 呼叫 |
| Task 回傳結果 | `wait` |
| Task 自動完成 | 使用 `close_agent` 釋放插槽 |
| `TodoWrite`（任務追蹤） | `update_plan` |
| `Skill` 工具（呼叫技能） | 技能原生載入 — 直接遵循指示 |
| `Read`、`Write`、`Edit`（檔案） | 使用原生檔案工具 |
| `Bash`（執行指令） | 使用原生 shell 工具 |

## 子代理派發需要啟用 collab

在你的 Codex 設定（`~/.codex/config.toml`）中加入：

```toml
[features]
collab = true
```

這會啟用 `spawn_agent`、`wait`、`close_agent`，以供 `dispatching-parallel-agents` 與 `subagent-driven-development` 等技能使用。
