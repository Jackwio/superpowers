# This is a Chinese translation of the original project.

Original project:
https://github.com/obra/superpowers

License: MIT
Copyright (c) 2025 Jesse Vincent

# Superpowers

Superpowers 是一套完整的軟體開發工作流程，專為你的程式代理打造，建立在可組合的「技能」以及一組初始指令之上，確保你的代理會使用這些技能。

## 如何運作

從你啟動程式代理的那一刻開始。當它看到你要開始打造東西時，它*不會*直接跳進去寫程式碼，而是先退一步，問你真正想完成什麼。

一旦它從對話中梳理出規格，就會以足夠短的小段呈現給你，讓你真的看得完、消化得了。

在你確認設計之後，你的代理會整理出一份實作計畫，清楚到連一位熱情但品味不佳、缺乏判斷、沒有專案脈絡、而且抗拒測試的初級工程師都能照著走。它強調真正的紅/綠 TDD、YAGNI（You Aren't Gonna Need It），以及 DRY。

接著，當你說「go」，它會啟動 *subagent-driven-development* 流程，讓代理逐一完成工程任務，檢查並審查其成果，再繼續往前。Claude 一次能自主工作幾個小時而不偏離你制定的計畫，並不罕見。

還有更多細節，但核心就是這些。因為技能會自動觸發，你不需要做任何特別的事。你的程式代理就是有 Superpowers。


## 贊助

如果 Superpowers 幫你做了能賺錢的事，而且你也願意，我會非常感激你考慮[贊助我的開源工作](https://github.com/sponsors/obra)。

謝謝！

- Jesse


## 安裝

**注意：**安裝方式會因平台而異。Claude Code 或 Cursor 內建外掛市集。Codex 與 OpenCode 需要手動設定。

### Claude Code 官方市集

Superpowers 可透過[官方 Claude 外掛市集](https://claude.com/plugins/superpowers)取得。

從 Claude 市集安裝外掛：

```bash
/plugin install superpowers@claude-plugins-official
```

### Claude Code（透過外掛市集）

在 Claude Code 中，先註冊市集：

```bash
/plugin marketplace add obra/superpowers-marketplace
```

接著從這個市集安裝外掛：

```bash
/plugin install superpowers@superpowers-marketplace
```

### Cursor（透過外掛市集）

在 Cursor Agent 對話中，從市集安裝：

```text
/add-plugin superpowers
```

或在外掛市集中搜尋「superpowers」。

### Codex

告訴 Codex：

```
Fetch and follow instructions from https://raw.githubusercontent.com/obra/superpowers/refs/heads/main/.codex/INSTALL.md
```

**詳細文件：**[docs/README.codex.md](docs/README.codex.md)

### OpenCode

告訴 OpenCode：

```
Fetch and follow instructions from https://raw.githubusercontent.com/obra/superpowers/refs/heads/main/.opencode/INSTALL.md
```

**詳細文件：**[docs/README.opencode.md](docs/README.opencode.md)

### Gemini CLI

```bash
gemini extensions install https://github.com/obra/superpowers
```

更新方式：

```bash
gemini extensions update superpowers
```

### 驗證安裝

在你選擇的平台開啟新對話，提出會觸發技能的請求（例如：「幫我規劃這個功能」或「我們來除錯這個問題」）。代理應該會自動呼叫相關的 Superpowers 技能。

## 基本工作流程

1. **brainstorming** - 在寫程式碼之前啟動。透過提問精煉粗略想法，探索替代方案，分段呈現設計以便確認。會保存設計文件。

2. **using-git-worktrees** - 設計核准後啟動。在新分支建立隔離的工作空間，執行專案設定，驗證測試基準為乾淨狀態。

3. **writing-plans** - 在設計核准後啟動。把工作拆成小任務（每個 2-5 分鐘）。每個任務都包含精確的檔案路徑、完整程式碼、驗證步驟。

4. **subagent-driven-development** 或 **executing-plans** - 有計畫時啟動。每個任務派出新的子代理，進行兩階段審查（規格符合度、再來是程式碼品質），或以批次執行並保留人工檢查點。

5. **test-driven-development** - 實作期間啟動。強制 RED-GREEN-REFACTOR：先寫會失敗的測試、確認失敗，再寫最小程式碼、確認通過，然後提交。會刪除在測試之前寫的程式碼。

6. **requesting-code-review** - 任務之間啟動。對照計畫審查，依嚴重度回報問題。重大問題會阻擋進度。

7. **finishing-a-development-branch** - 任務完成時啟動。驗證測試、提供選項（合併/PR/保留/丟棄），清理工作樹。

**代理會在任何任務前檢查相關技能。** 這是強制流程，不是建議。

## 內容概覽

### 技能庫

**測試**
- **test-driven-development** - RED-GREEN-REFACTOR 循環（包含測試反模式參考）

**除錯**
- **systematic-debugging** - 4 階段根因流程（包含 root-cause-tracing、defense-in-depth、condition-based-waiting 技術）
- **verification-before-completion** - 確保真的修好了

**協作**
- **brainstorming** - 蘇格拉底式設計精煉
- **writing-plans** - 詳細實作計畫
- **executing-plans** - 具檢查點的批次執行
- **dispatching-parallel-agents** - 併發子代理流程
- **requesting-code-review** - 預先審查檢查清單
- **receiving-code-review** - 回應回饋
- **using-git-worktrees** - 平行開發分支
- **finishing-a-development-branch** - 合併/PR 決策流程
- **subagent-driven-development** - 具兩階段審查的快速迭代（規格符合度、再來是程式碼品質）

**後設**
- **writing-skills** - 依最佳實務建立新技能（包含測試方法論）
- **using-superpowers** - 技能系統簡介

## 理念

- **Test-Driven Development** - 永遠先寫測試
- **Systematic over ad-hoc** - 流程勝過猜測
- **Complexity reduction** - 以簡化為首要目標
- **Evidence over claims** - 宣告成功前先驗證

更多閱讀：[Superpowers for Claude Code](https://blog.fsck.com/2025/10/09/superpowers/)

## 參與貢獻

技能直接放在這個儲存庫中。若要貢獻：

1. Fork 這個儲存庫
2. 為你的技能建立分支
3. 依照 `writing-skills` 技能建立並測試新技能
4. 提交 PR

完整指南請見 `skills/writing-skills/SKILL.md`。

## 更新

更新外掛時技能會自動更新：

```bash
/plugin update superpowers
```

## 授權

MIT License - 詳情請見 LICENSE 檔案

## 支援

- **Issues**: https://github.com/obra/superpowers/issues
- **Marketplace**: https://github.com/obra/superpowers-marketplace
