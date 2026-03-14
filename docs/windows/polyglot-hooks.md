# Claude Code 的跨平台多語 Hook

Claude Code 外掛需要能在 Windows、macOS 與 Linux 上運作的 hooks。本文件說明讓這件事可行的多語包裝器技巧。

## 問題

Claude Code 會透過系統的預設 shell 執行 hook 指令：
- **Windows**：CMD.exe
- **macOS/Linux**：bash 或 sh

這會帶來幾個挑戰：

1. **腳本執行**：Windows CMD 無法直接執行 `.sh` 檔案 — 它會嘗試用文字編輯器開啟
2. **路徑格式**：Windows 使用反斜線（`C:\path`），Unix 使用正斜線（`/path`）
3. **環境變數**：`$VAR` 語法在 CMD 中無法使用
4. **PATH 中沒有 `bash`**：即使安裝了 Git Bash，CMD 執行時 PATH 也沒有 `bash`

## 解法：多語 `.cmd` 包裝器

多語腳本可以同時是多種語言的合法語法。我們的包裝器同時是 CMD 與 bash 的合法語法：

```cmd
: << 'CMDBLOCK'
@echo off
"C:\Program Files\Git\bin\bash.exe" -l -c "\"$(cygpath -u \"$CLAUDE_PLUGIN_ROOT\")/hooks/session-start.sh\""
exit /b
CMDBLOCK

# Unix shell runs from here
"${CLAUDE_PLUGIN_ROOT}/hooks/session-start.sh"
```

### 運作方式

#### 在 Windows（CMD.exe）

1. `: << 'CMDBLOCK'` - CMD 會把 `:` 視為標籤（像 `:label`），並忽略 `<< 'CMDBLOCK'`
2. `@echo off` - 抑制指令回顯
3. bash.exe 指令會以以下參數執行：
   - `-l`（登入 shell）以取得包含 Unix 工具的正確 PATH
   - `cygpath -u` 會把 Windows 路徑轉成 Unix 格式（`C:\foo` → `/c/foo`）
4. `exit /b` - 結束批次腳本，CMD 在此停止
5. `CMDBLOCK` 之後的內容不會被 CMD 觸及

#### 在 Unix（bash/sh）

1. `: << 'CMDBLOCK'` - `:` 是 no-op，`<< 'CMDBLOCK'` 啟動 heredoc
2. 直到 `CMDBLOCK` 之前的內容都會被 heredoc 吃掉（忽略）
3. `# Unix shell runs from here` - 註解
4. 腳本直接用 Unix 路徑執行

## 檔案結構

```
hooks/
├── hooks.json           # Points to the .cmd wrapper
├── session-start.cmd    # Polyglot wrapper (cross-platform entry point)
└── session-start.sh     # Actual hook logic (bash script)
```

### hooks.json

```json
{
  "hooks": {
    "SessionStart": [
      {
        "matcher": "startup|resume|clear|compact",
        "hooks": [
          {
            "type": "command",
            "command": "\"${CLAUDE_PLUGIN_ROOT}/hooks/session-start.cmd\""
          }
        ]
      }
    ]
  }
}
```

注意：必須加上引號，因為 `${CLAUDE_PLUGIN_ROOT}` 在 Windows 上可能包含空白（例如 `C:\Program Files\...`）。

## 需求

### Windows
- **Git for Windows** 必須已安裝（提供 `bash.exe` 與 `cygpath`）
- 預設安裝路徑：`C:\Program Files\Git\bin\bash.exe`
- 若 Git 安裝在其他位置，需要修改包裝器

### Unix（macOS/Linux）
- 標準 bash 或 sh shell
- `.cmd` 檔案必須有可執行權限（`chmod +x`）

## 撰寫跨平台 Hook 腳本

實際的 hook 邏輯放在 `.sh` 檔案中。為了確保在 Windows（透過 Git Bash）上能正常運作：

### 建議：
- 盡量使用純 bash 內建功能
- 使用 `$(command)` 取代反引號
- 對所有變數展開加上引號：`"$VAR"`
- 使用 `printf` 或 heredoc 輸出

### 避免：
- 不在 PATH 中的外部指令（sed、awk、grep）
- 若必須使用，Git Bash 有提供，但請確保 PATH 設定正確（使用 `bash -l`）

### 範例：不使用 sed/awk 的 JSON 逸出

不要這樣做：
```bash
escaped=$(echo "$content" | sed 's/\\/\\\\/g' | sed 's/"/\\"/g' | awk '{printf "%s\\n", $0}')
```

改用純 bash：
```bash
escape_for_json() {
    local input="$1"
    local output=""
    local i char
    for (( i=0; i<${#input}; i++ )); do
        char="${input:$i:1}"
        case "$char" in
            $'\\') output+='\\' ;;
            '"') output+='\"' ;;
            $'\n') output+='\n' ;;
            $'\r') output+='\r' ;;
            $'\t') output+='\t' ;;
            *) output+="$char" ;;
        esac
    done
    printf '%s' "$output"
}
```

## 可重複使用的包裝器模式

若外掛有多個 hook，可以建立接受腳本名稱作為參數的通用包裝器：

### run-hook.cmd
```cmd
: << 'CMDBLOCK'
@echo off
set "SCRIPT_DIR=%~dp0"
set "SCRIPT_NAME=%~1"
"C:\Program Files\Git\bin\bash.exe" -l -c "cd \"$(cygpath -u \"%SCRIPT_DIR%\")\" && \"./%SCRIPT_NAME%\""
exit /b
CMDBLOCK

# Unix shell runs from here
SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]:-$0}")" && pwd)"
SCRIPT_NAME="$1"
shift
"${SCRIPT_DIR}/${SCRIPT_NAME}" "$@"
```

### hooks.json 使用可重複包裝器
```json
{
  "hooks": {
    "SessionStart": [
      {
        "matcher": "startup",
        "hooks": [
          {
            "type": "command",
            "command": "\"${CLAUDE_PLUGIN_ROOT}/hooks/run-hook.cmd\" session-start.sh"
          }
        ]
      }
    ],
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "\"${CLAUDE_PLUGIN_ROOT}/hooks/run-hook.cmd\" validate-bash.sh"
          }
        ]
      }
    ]
  }
}
```

## 疑難排解

### "bash is not recognized"
CMD 找不到 bash。包裝器使用完整路徑 `C:\Program Files\Git\bin\bash.exe`。若 Git 安裝在其他位置，請更新路徑。

### "cygpath: command not found" 或 "dirname: command not found"
Bash 沒有以登入 shell 執行。請確認使用 `-l` 參數。

### 路徑中出現奇怪的 `\/`
`${CLAUDE_PLUGIN_ROOT}` 展開成以反斜線結尾的 Windows 路徑，然後又接上 `/hooks/...`。請用 `cygpath` 轉換完整路徑。

### 腳本在文字編輯器中開啟而非執行
hooks.json 直接指向 `.sh` 檔案。請改指向 `.cmd` 包裝器。
