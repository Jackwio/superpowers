# 安裝 Codex 版 Superpowers

透過原生技能探索在 Codex 中啟用 superpowers 技能。只要 clone 並建立符號連結即可。

## 先決條件

- Git

## 安裝

1. **複製 superpowers 儲存庫：**
   ```bash
   git clone https://github.com/obra/superpowers.git ~/.codex/superpowers
   ```

2. **建立技能符號連結：**
   ```bash
   mkdir -p ~/.agents/skills
   ln -s ~/.codex/superpowers/skills ~/.agents/skills/superpowers
   ```

   **Windows（PowerShell）：**
   ```powershell
   New-Item -ItemType Directory -Force -Path "$env:USERPROFILE\.agents\skills"
   cmd /c mklink /J "$env:USERPROFILE\.agents\skills\superpowers" "$env:USERPROFILE\.codex\superpowers\skills"
   ```

3. **重新啟動 Codex**（關閉並重新啟動 CLI）以探索技能。

## 從舊版 bootstrap 遷移

若你是在原生技能探索推出前就安裝了 superpowers，請依序做：

1. **更新儲存庫：**
   ```bash
   cd ~/.codex/superpowers && git pull
   ```

2. **建立技能符號連結**（上方步驟 2）— 這是新的探索機制。

3. **從 `~/.codex/AGENTS.md` 移除舊的 bootstrap 區塊** — 任何引用 `superpowers-codex bootstrap` 的區塊都不再需要。

4. **重新啟動 Codex。**

## 驗證

```bash
ls -la ~/.agents/skills/superpowers
```

你應該會看到指向 superpowers 技能目錄的符號連結（Windows 上為 junction）。

## 更新

```bash
cd ~/.codex/superpowers && git pull
```

技能會透過符號連結即時更新。

## 解除安裝

```bash
rm ~/.agents/skills/superpowers
```

如有需要，可刪除複本：`rm -rf ~/.codex/superpowers`。
