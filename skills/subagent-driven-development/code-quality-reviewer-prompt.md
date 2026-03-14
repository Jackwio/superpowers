# 程式碼品質審查提示範本

在派出程式碼品質審查子代理時使用此範本。

**目的：**確認實作品質良好（乾淨、有測試、可維護）

**僅在規格符合度審查通過後才派出。**

```
Task tool (superpowers:code-reviewer):
  Use template at requesting-code-review/code-reviewer.md

  WHAT_WAS_IMPLEMENTED: [from implementer's report]
  PLAN_OR_REQUIREMENTS: Task N from [plan-file]
  BASE_SHA: [commit before task]
  HEAD_SHA: [current commit]
  DESCRIPTION: [task summary]
```

**除了標準的程式碼品質考量外，審查者還應檢查：**
- 每個檔案是否只有一個清楚責任，且介面明確？
- 單元是否拆分到可以獨立理解與測試？
- 實作是否遵循計畫中的檔案結構？
- 這次實作是否建立了已經很大的新檔案，或讓既有檔案明顯膨脹？（不要標記既有檔案大小，請聚焦於此變更的貢獻。）

**程式碼審查者回傳：**優點、問題（Critical/Important/Minor）、評估
