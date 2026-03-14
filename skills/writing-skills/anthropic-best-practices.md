# 技能撰寫最佳實務

> 學習如何撰寫 Claude 能成功發現並使用的有效技能。

好的技能應該精簡、結構清楚，並經過真實使用測試。本指南提供實用的撰寫決策，幫助你寫出 Claude 能有效發現並使用的技能。

關於技能運作的概念背景，請見 [技能概覽](/en/docs/agents-and-tools/agent-skills/overview)。

## 核心原則

### 精簡是關鍵

[context window](https://platform.claude.com/docs/en/build-with-claude/context-windows) 是公共資源。你的技能與 Claude 需要知道的其他內容共享 context window，包括：

* 系統提示
* 對話歷史
* 其他技能的中繼資料
* 你的實際請求

技能中的每個 token 不一定立刻有成本。啟動時只會預載所有技能的中繼資料（name 與 description）。Claude 只有在技能變得相關時才會讀取 SKILL.md，並僅在需要時讀取其他檔案。然而，SKILL.md 的精簡仍然重要：一旦 Claude 載入它，每個 token 都要與對話歷史與其他脈絡競爭。

**預設假設**：Claude 已經很聰明

只加入 Claude 不會已知的資訊。對每段內容自我挑戰：

* 「Claude 真的需要這段解釋嗎？」
* 「我能假設 Claude 已經知道嗎？」
* 「這段文字值得它的 token 成本嗎？」

**好例子：精簡**（約 50 tokens）：

````markdown  theme={null}
## Extract PDF text

Use pdfplumber for text extraction:

```python
import pdfplumber

with pdfplumber.open("file.pdf") as pdf:
    text = pdf.pages[0].extract_text()
```
````

**壞例子：太冗長**（約 150 tokens）：

```markdown  theme={null}
## Extract PDF text

PDF (Portable Document Format) files are a common file format that contains
text, images, and other content. To extract text from a PDF, you'll need to
use a library. There are many libraries available for PDF processing, but we
recommend pdfplumber because it's easy to use and handles most cases well.
First, you'll need to install it using pip. Then you can use the code below...
```

精簡版假設 Claude 知道 PDF 是什麼，也知道如何使用函式庫。

### 設定適當的自由度

讓指示的具體程度匹配任務的脆弱性與變動性。

**高自由度**（文字型指示）：

適用於：

* 有多種有效作法
* 決策取決於情境
* 以啟發式方式引導

範例：

```markdown  theme={null}
## Code review process

1. Analyze the code structure and organization
2. Check for potential bugs or edge cases
3. Suggest improvements for readability and maintainability
4. Verify adherence to project conventions
```

**中自由度**（偽碼或可參數化腳本）：

適用於：

* 有偏好的模式
* 可接受部分變化
* 設定會影響行為

範例：

````markdown  theme={null}
## Generate report

Use this template and customize as needed:

```python
def generate_report(data, format="markdown", include_charts=True):
    # Process data
    # Generate output in specified format
    # Optionally include visualizations
```
````

**低自由度**（特定腳本、少或無參數）：

適用於：

* 操作脆弱且容易出錯
* 一致性很關鍵
* 必須遵循特定順序

範例：

````markdown  theme={null}
## Database migration

Run exactly this script:

```bash
python scripts/migrate.py --verify --backup
```

Do not modify the command or add additional flags.
````

**比喻：**把 Claude 想成在路徑上探索的機器人：

* **兩側是懸崖的窄橋**：只有一條安全路徑。提供明確護欄與精準指示（低自由度）。例：必須按序執行的資料庫遷移。
* **無危險的開闊平原**：多條路徑都能成功。給大方向，讓 Claude 找最佳路徑（高自由度）。例：需依情境決定的 code review。

### 用你計畫使用的所有模型測試

技能是加在模型上的內容，效果取決於底層模型。請用你計畫使用的所有模型測試。

**不同模型的測試考量：**

* **Claude Haiku**（快速、經濟）：技能提供的指引是否足夠？
* **Claude Sonnet**（平衡）：技能是否清楚且有效率？
* **Claude Opus**（強推理）：技能是否避免過度解釋？

對 Opus 完美的指引，對 Haiku 可能需要更多細節。若你計畫跨模型使用技能，請讓指示對所有模型都有效。

## 技能結構

<Note>
  **YAML Frontmatter**：SKILL.md frontmatter 支援兩個欄位：

  * `name` - 技能的人類可讀名稱（最多 64 字元）
  * `description` - 技能做什麼與何時使用的一行描述（最多 1024 字元）

  更完整的技能結構細節，請見 [Skills overview](/en/docs/agents-and-tools/agent-skills/overview#skill-structure)。
</Note>

### 命名慣例

使用一致的命名模式，讓技能更容易引用與討論。我們建議用**動名詞**（動詞 + -ing）作為技能名稱，因為它清楚描述技能提供的活動或能力。

**良好命名範例（動名詞）：**

* "Processing PDFs"
* "Analyzing spreadsheets"
* "Managing databases"
* "Testing code"
* "Writing documentation"

**可接受的替代：**

* 名詞片語："PDF Processing"、"Spreadsheet Analysis"
* 動作導向："Process PDFs"、"Analyze Spreadsheets"

**避免：**

* 含糊名稱："Helper"、"Utils"、"Tools"
* 過度泛化："Documents"、"Data"、"Files"
* 在技能集合中使用不一致的模式

一致命名讓你更容易：

* 在文件或對話中引用技能
* 一眼理解技能用途
* 組織與搜尋多個技能
* 維持專業且一致的技能庫

### 撰寫有效描述

`description` 欄位負責技能可發現性，應包含技能做什麼以及何時使用。

<Warning>
  **一定要用第三人稱**。description 會被注入系統提示，視角不一致會造成可發現性問題。

  * **好：**"Processes Excel files and generates reports"
  * **避免：**"I can help you process Excel files"
  * **避免：**"You can use this to process Excel files"
</Warning>

**請具體並包含關鍵字**。description 應包含技能做什麼，以及使用它的明確觸發條件/情境。

每個技能只有一個 description。這個欄位對技能選擇至關重要：Claude 可能面對 100+ 個技能，會依 description 選出合適的技能。你的 description 必須提供足夠細節讓 Claude 知道何時選它，而 SKILL.md 的其餘內容提供實作細節。

有效範例：

**PDF Processing skill：**

```yaml  theme={null}
description: Extract text and tables from PDF files, fill forms, merge documents. Use when working with PDF files or when the user mentions PDFs, forms, or document extraction.
```

**Excel Analysis skill：**

```yaml  theme={null}
description: Analyze Excel spreadsheets, create pivot tables, generate charts. Use when analyzing Excel files, spreadsheets, tabular data, or .xlsx files.
```

**Git Commit Helper skill：**

```yaml  theme={null}
description: Generate descriptive commit messages by analyzing git diffs. Use when the user asks for help writing commit messages or reviewing staged changes.
```

避免以下這類模糊描述：

```yaml  theme={null}
description: Helps with documents
```

```yaml  theme={null}
description: Processes data
```

```yaml  theme={null}
description: Does stuff with files
```

### 漸進揭露模式

SKILL.md 作為概覽，指引 Claude 在需要時前往更細的資料，就像 onboarding 指南的目錄。關於漸進揭露的說明，請見概覽中的 [How Skills work](/en/docs/agents-and-tools/agent-skills/overview#how-skills-work)。

**實務指引：**

* 為最佳效能，SKILL.md 內文保持在 500 行內
* 接近這個限制時把內容拆到獨立檔案
* 使用以下模式組織指示、程式碼與資源

#### 視覺化概覽：由簡到繁

基本技能只包含一個 SKILL.md，內含中繼資料與指示：

<img src="https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-simple-file.png?fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=87782ff239b297d9a9e8e1b72ed72db9" alt="Simple SKILL.md file showing YAML frontmatter and markdown body" data-og-width="2048" width="2048" data-og-height="1153" height="1153" data-path="images/agent-skills-simple-file.png" data-optimize="true" data-opv="3" srcset="https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-simple-file.png?w=280&fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=c61cc33b6f5855809907f7fda94cd80e 280w, https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-simple-file.png?w=560&fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=90d2c0c1c76b36e8d485f49e0810dbfd 560w, https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-simple-file.png?w=840&fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=ad17d231ac7b0bea7e5b4d58fb4aeabb 840w, https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-simple-file.png?w=1100&fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=f5d0a7a3c668435bb0aee9a3a8f8c329 1100w, https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-simple-file.png?w=1650&fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=0e927c1af9de5799cfe557d12249f6e6 1650w, https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-simple-file.png?w=2500&fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=46bbb1a51dd4c8202a470ac8c80a893d 2500w" />

當技能成長後，可把內容打包成 Claude 只在需要時載入的額外檔案：

<img src="https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-bundling-content.png?fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=a5e0aa41e3d53985a7e3e43668a33ea3" alt="Bundling additional reference files like reference.md and forms.md." data-og-width="2048" width="2048" data-og-height="1327" height="1327" data-path="images/agent-skills-bundling-content.png" data-optimize="true" data-opv="3" srcset="https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-bundling-content.png?w=280&fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=f8a0e73783e99b4a643d79eac86b70a2 280w, https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-bundling-content.png?w=560&fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=dc510a2a9d3f14359416b706f067904a 560w, https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-bundling-content.png?w=840&fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=82cd6286c966303f7dd914c28170e385 840w, https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-bundling-content.png?w=1100&fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=56f3be36c77e4fe4b523df209a6824c6 1100w, https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-bundling-content.png?w=1650&fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=d22b5161b2075656417d56f41a74f3dd 1650w, https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-bundling-content.png?w=2500&fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=3dd4bdd6850ffcc96c6c45fcb0acd6eb 2500w" />

完整技能目錄結構可能如下：

```
pdf/
├── SKILL.md              # Main instructions (loaded when triggered)
├── FORMS.md              # Form-filling guide (loaded as needed)
├── reference.md          # API reference (loaded as needed)
├── examples.md           # Usage examples (loaded as needed)
└── scripts/
    ├── analyze_form.py   # Utility script (executed, not loaded)
    ├── fill_form.py      # Form filling script
    └── validate.py       # Validation script
```

#### 模式 1：高層指南 + 參考連結

````markdown  theme={null}
---
name: PDF Processing
description: Extracts text and tables from PDF files, fills forms, and merges documents. Use when working with PDF files or when the user mentions PDFs, forms, or document extraction.
---

# PDF Processing

## Quick start

Extract text with pdfplumber:
```python
import pdfplumber
with pdfplumber.open("file.pdf") as pdf:
    text = pdf.pages[0].extract_text()
```

## Advanced features

**Form filling**: See [FORMS.md](FORMS.md) for complete guide
**API reference**: See [REFERENCE.md](REFERENCE.md) for all methods
**Examples**: See [EXAMPLES.md](EXAMPLES.md) for common patterns
````

Claude 只有在需要時才會載入 FORMS.md、REFERENCE.md 或 EXAMPLES.md。

#### 模式 2：領域分區組織

對於涵蓋多個領域的技能，依領域組織內容以避免載入不相關脈絡。當使用者詢問銷售指標時，Claude 只需要讀銷售相關 schema，而不需要財務或行銷資料。這能降低 token 使用量並保持脈絡聚焦。

```
bigquery-skill/
├── SKILL.md (overview and navigation)
└── reference/
    ├── finance.md (revenue, billing metrics)
    ├── sales.md (opportunities, pipeline)
    ├── product.md (API usage, features)
    └── marketing.md (campaigns, attribution)
```

````markdown SKILL.md theme={null}
# BigQuery Data Analysis

## Available datasets

**Finance**: Revenue, ARR, billing → See [reference/finance.md](reference/finance.md)
**Sales**: Opportunities, pipeline, accounts → See [reference/sales.md](reference/sales.md)
**Product**: API usage, features, adoption → See [reference/product.md](reference/product.md)
**Marketing**: Campaigns, attribution, email → See [reference/marketing.md](reference/marketing.md)

## Quick search

Find specific metrics using grep:

```bash
grep -i "revenue" reference/finance.md
grep -i "pipeline" reference/sales.md
grep -i "api usage" reference/product.md
```
````

#### 模式 3：條件式細節

先顯示基礎內容，連到進階內容：

```markdown  theme={null}
# DOCX Processing

## Creating documents

Use docx-js for new documents. See [DOCX-JS.md](DOCX-JS.md).

## Editing documents

For simple edits, modify the XML directly.

**For tracked changes**: See [REDLINING.md](REDLINING.md)
**For OOXML details**: See [OOXML.md](OOXML.md)
```

只有在使用者需要這些功能時，Claude 才會讀 REDLINING.md 或 OOXML.md。

### 避免過深的巢狀參考

當檔案是從其他被參考的檔案再被參考時，Claude 可能只讀到部分內容。遇到巢狀參考時，Claude 可能使用 `head -100` 之類的指令預覽，而非讀完整檔案，導致資訊不完整。

**讓參考檔只從 SKILL.md 一層連出**。所有參考檔都應該由 SKILL.md 直接連結，以確保 Claude 在需要時讀取完整檔案。

**壞例子：太深**：

```markdown  theme={null}
# SKILL.md
See [advanced.md](advanced.md)...

# advanced.md
See [details.md](details.md)...

# details.md
Here's the actual information...
```

**好例子：一層深**：

```markdown  theme={null}
# SKILL.md

**Basic usage**: [instructions in SKILL.md]
**Advanced features**: See [advanced.md](advanced.md)
**API reference**: See [reference.md](reference.md)
**Examples**: See [examples.md](examples.md)
```

### 以目錄建立較長參考檔的結構

若參考檔超過 100 行，請在頂部加入目錄。即使 Claude 使用部分讀取，也能看到完整資訊範圍。

**範例：**

```markdown  theme={null}
# API Reference

## Contents
- Authentication and setup
- Core methods (create, read, update, delete)
- Advanced features (batch operations, webhooks)
- Error handling patterns
- Code examples

## Authentication and setup
...

## Core methods
...
```

Claude 之後可以讀完整檔案，或直接跳到特定章節。

關於這種檔案系統架構如何支持漸進揭露的更多細節，請見下方進階章節的 [Runtime environment](#runtime-environment)。

## 工作流程與回饋迴圈

### 為複雜任務使用工作流程

把複雜操作拆成清楚的序列步驟。對特別複雜的流程，提供可讓 Claude 複製的檢查清單，並在回應中逐項勾選。

**範例 1：研究彙整流程**（無程式碼的技能）：

````markdown  theme={null}
## Research synthesis workflow

Copy this checklist and track your progress:

```
Research Progress:
- [ ] Step 1: Read all source documents
- [ ] Step 2: Identify key themes
- [ ] Step 3: Cross-reference claims
- [ ] Step 4: Create structured summary
- [ ] Step 5: Verify citations
```

**Step 1: Read all source documents**

Review each document in the `sources/` directory. Note the main arguments and supporting evidence.

**Step 2: Identify key themes**

Look for patterns across sources. What themes appear repeatedly? Where do sources agree or disagree?

**Step 3: Cross-reference claims**

For each major claim, verify it appears in the source material. Note which source supports each point.

**Step 4: Create structured summary**

Organize findings by theme. Include:
- Main claim
- Supporting evidence from sources
- Conflicting viewpoints (if any)

**Step 5: Verify citations**

Check that every claim references the correct source document. If citations are incomplete, return to Step 3.
````

此例展示了工作流程如何用於不需要寫程式碼的分析任務。清單模式適用於任何複雜、多步驟流程。

**範例 2：PDF 表單填寫流程**（有程式碼的技能）：

````markdown  theme={null}
## PDF form filling workflow

Copy this checklist and check off items as you complete them:

```
Task Progress:
- [ ] Step 1: Analyze the form (run analyze_form.py)
- [ ] Step 2: Create field mapping (edit fields.json)
- [ ] Step 3: Validate mapping (run validate_fields.py)
- [ ] Step 4: Fill the form (run fill_form.py)
- [ ] Step 5: Verify output (run verify_output.py)
```

**Step 1: Analyze the form**

Run: `python scripts/analyze_form.py input.pdf`

This extracts form fields and their locations, saving to `fields.json`.

**Step 2: Create field mapping**

Edit `fields.json` to add values for each field.

**Step 3: Validate mapping**

Run: `python scripts/validate_fields.py fields.json`

Fix any validation errors before continuing.

**Step 4: Fill the form**

Run: `python scripts/fill_form.py input.pdf fields.json output.pdf`

**Step 5: Verify output**

Run: `python scripts/verify_output.py output.pdf`

If verification fails, return to Step 2.
````

清楚的步驟可避免 Claude 跳過關鍵驗證。清單可讓 Claude 與你一起追蹤進度。

### 實作回饋迴圈

**常見模式**：執行驗證器 → 修正錯誤 → 重複

這種模式能大幅提升輸出品質。

**範例 1：風格指南合規**（無程式碼的技能）：

```markdown  theme={null}
## Content review process

1. Draft your content following the guidelines in STYLE_GUIDE.md
2. Review against the checklist:
   - Check terminology consistency
   - Verify examples follow the standard format
   - Confirm all required sections are present
3. If issues found:
   - Note each issue with specific section reference
   - Revise the content
   - Review the checklist again
4. Only proceed when all requirements are met
5. Finalize and save the document
```

這展示了用參考文件（而不是腳本）實作驗證迴圈。「驗證器」是 STYLE_GUIDE.md，Claude 透過閱讀與比對進行檢查。

**範例 2：文件編輯流程**（有程式碼的技能）：

```markdown  theme={null}
## Document editing process

1. Make your edits to `word/document.xml`
2. **Validate immediately**: `python ooxml/scripts/validate.py unpacked_dir/`
3. If validation fails:
   - Review the error message carefully
   - Fix the issues in the XML
   - Run validation again
4. **Only proceed when validation passes**
5. Rebuild: `python ooxml/scripts/pack.py unpacked_dir/ output.docx`
6. Test the output document
```

驗證迴圈能早期抓到錯誤。

## 內容指引

### 避免時間敏感資訊

不要加入會過時的資訊：

**壞例子：時間敏感**（會變錯）：

```markdown  theme={null}
If you're doing this before August 2025, use the old API.
After August 2025, use the new API.
```

**好例子**（使用「舊模式」段落）：

```markdown  theme={null}
## Current method

Use the v2 API endpoint: `api.example.com/v2/messages`

## Old patterns

<details>
<summary>Legacy v1 API (deprecated 2025-08)</summary>

The v1 API used: `api.example.com/v1/messages`

This endpoint is no longer supported.
</details>
```

「舊模式」段落提供歷史脈絡，但不會干擾主要內容。

### 使用一致術語

選定一個術語並在整個技能中一致使用：

**好 — 一致：**

* Always "API endpoint"
* Always "field"
* Always "extract"

**壞 — 不一致：**

* 混用 "API endpoint"、"URL"、"API route"、"path"
* 混用 "field"、"box"、"element"、"control"
* 混用 "extract"、"pull"、"get"、"retrieve"

一致性有助於 Claude 理解並遵循指示。

## 常見模式

### 模板模式

提供輸出格式模板。依需求調整嚴格程度。

**對嚴格需求**（如 API 回應或資料格式）：

````markdown  theme={null}
## Report structure

ALWAYS use this exact template structure:

```markdown
# [Analysis Title]

## Executive summary
[One-paragraph overview of key findings]

## Key findings
- Finding 1 with supporting data
- Finding 2 with supporting data
- Finding 3 with supporting data

## Recommendations
1. Specific actionable recommendation
2. Specific actionable recommendation
```
````

**對彈性指引**（可調整時）：

````markdown  theme={null}
## Report structure

Here is a sensible default format, but use your best judgment based on the analysis:

```markdown
# [Analysis Title]

## Executive summary
[Overview]

## Key findings
[Adapt sections based on what you discover]

## Recommendations
[Tailor to the specific context]
```

Adjust sections as needed for the specific analysis type.
````

### 範例模式

當輸出品質取決於範例時，提供與一般 prompting 相同的輸入/輸出對：

````markdown  theme={null}
## Commit message format

Generate commit messages following these examples:

**Example 1:**
Input: Added user authentication with JWT tokens
Output:
```
feat(auth): implement JWT-based authentication

Add login endpoint and token validation middleware
```

**Example 2:**
Input: Fixed bug where dates displayed incorrectly in reports
Output:
```
fix(reports): correct date formatting in timezone conversion

Use UTC timestamps consistently across report generation
```

**Example 3:**
Input: Updated dependencies and refactored error handling
Output:
```
chore: update dependencies and refactor error handling

- Upgrade lodash to 4.17.21
- Standardize error response format across endpoints
```

Follow this style: type(scope): brief description, then detailed explanation.
````

範例能比描述更清楚地讓 Claude 理解期待的風格與細節程度。

### 條件式流程模式

引導 Claude 走決策點：

```markdown  theme={null}
## Document modification workflow

1. Determine the modification type:

   **Creating new content?** → Follow "Creation workflow" below
   **Editing existing content?** → Follow "Editing workflow" below

2. Creation workflow:
   - Use docx-js library
   - Build document from scratch
   - Export to .docx format

3. Editing workflow:
   - Unpack existing document
   - Modify XML directly
   - Validate after each change
   - Repack when complete
```

<Tip>
  如果流程步驟很多且複雜，考慮移到獨立檔案，並指示 Claude 依任務閱讀對應檔案。
</Tip>

## 評估與迭代

### 先建立評估

**在撰寫大量文件之前，先建立評估。**這能確保你的技能解決真實問題，而不是記錄想像中的需求。

**評估驅動開發：**

1. **找出缺口**：在沒有技能下跑 Claude 的代表性任務，記錄具體失敗或缺少的脈絡
2. **建立評估**：設計三個測試這些缺口的情境
3. **建立基線**：衡量沒有技能時的表現
4. **寫最小指引**：只寫足以補上缺口並通過評估的內容
5. **迭代**：執行評估、與基線比較並修正

此方法可確保你解決的是實際問題，而不是預期中永遠不會出現的需求。

**評估結構：**

```json  theme={null}
{
  "skills": ["pdf-processing"],
  "query": "Extract all text from this PDF file and save it to output.txt",
  "files": ["test-files/document.pdf"],
  "expected_behavior": [
    "Successfully reads the PDF file using an appropriate PDF processing library or command-line tool",
    "Extracts text content from all pages in the document without missing any pages",
    "Saves the extracted text to a file named output.txt in a clear, readable format"
  ]
}
```

<Note>
  這個範例展示了資料驅動評估與簡單的測試規範。我們目前沒有內建執行這些評估的方式。使用者可自行建立評估系統。評估是衡量技能效果的真實依據。
</Note>

### 用 Claude 迭代開發技能

最有效的技能開發流程包含 Claude 本身。用一個 Claude 實例（"Claude A"）建立技能，供另一個實例（"Claude B"）使用。Claude A 協助你設計與精煉指示，Claude B 在實際任務中測試。這有效是因為 Claude 模型理解如何撰寫有效的代理指示，也理解代理需要哪些資訊。

**建立新技能：**

1. **在沒有技能下完成任務**：使用 Claude A 以一般 prompting 完成任務。過程中你會自然提供脈絡、偏好與流程知識。注意你反覆提供的資訊。

2. **找出可重用模式**：完成任務後，找出哪些脈絡適合用於未來相似任務。

   **例子**：若你做的是 BigQuery 分析，你可能提供了表名稱、欄位定義、過濾規則（如「永遠排除測試帳戶」）與常見查詢模式。

3. **請 Claude A 建立技能**：「建立一個 Skill，捕捉我們剛才使用的 BigQuery 分析模式，包含表格 schema、命名慣例與排除測試帳戶的規則。」

   <Tip>
     Claude 模型原生理解技能格式與結構。不需要特殊系統提示或「writing skills」技能就能讓 Claude 協助建立技能。只要請 Claude 建立技能，它就會產生結構正確的 SKILL.md（含 frontmatter 與正文）。
   </Tip>

4. **檢查精簡度**：確認 Claude A 沒加多餘解釋。可以說：「刪掉 win rate 的解釋 — Claude 已經知道。」

5. **改善資訊架構**：請 Claude A 更有效地組織內容，例如：「把表格 schema 放到獨立參考檔。我們可能會新增更多表。」

6. **在相似任務中測試**：用 Claude B（載入技能的全新實例）跑相關使用情境，觀察 Claude B 是否找到正確資訊、正確套用規則並完成任務。

7. **依觀察迭代**：若 Claude B 卡住或漏掉內容，回到 Claude A 並提供具體觀察：「Claude 用這個技能時忘了 Q4 的日期過濾。我們要加一段日期過濾模式嗎？」

**迭代既有技能：**

改善技能時同樣遵循這個分工模式：

* **與 Claude A 合作**（協助精煉技能）
* **用 Claude B 測試**（在真實工作中使用）
* **觀察 Claude B 行為**並把洞見帶回 Claude A

1. **在真實流程中使用技能**：給 Claude B（載入技能）真實任務，而非測試情境

2. **觀察 Claude B 行為**：記下它卡住、成功或做出意外選擇的地方

   **觀察例子**：「我請 Claude B 做區域銷售報表時，它寫了查詢，但忘了排除測試帳戶，雖然技能有說這條規則。」

3. **回到 Claude A 改善**：分享目前的 SKILL.md 與你的觀察，並詢問：「Claude B 做區域報表時忘了排除測試帳戶。技能有提，但可能不夠顯眼？」

4. **檢視 Claude A 的建議**：Claude A 可能建議重排結構讓規則更顯眼，改用更強語氣（"MUST filter" 取代 "always filter"），或重寫流程段落。

5. **套用並測試**：用 Claude A 的修正更新技能，再讓 Claude B 用相似請求測試

6. **根據使用重複**：持續這個觀察-修正-測試的循環。每次迭代都基於真實行為而不是假設。

**蒐集團隊回饋：**

1. 與隊友分享技能並觀察使用情況
2. 詢問：技能是否在預期情境被觸發？指示是否清楚？缺了什麼？
3. 納入回饋，修補你自身使用模式的盲點

**為何此方法有效**：Claude A 理解代理需求，你提供領域專長，Claude B 透過真實使用揭露缺口，而迭代式修正能依觀察行為改善技能，而非建立在假設上。

### 觀察 Claude 如何導覽技能

在迭代技能時，注意 Claude 實際如何使用它們。觀察：

* **非預期的探索路徑**：Claude 是否以你未預期的順序讀檔？可能表示結構不如你想像直覺
* **遺漏連結**：Claude 是否沒跟到重要檔案的引用？連結可能需要更明確或更顯眼
* **過度依賴某段內容**：若 Claude 反覆讀同一檔案，可能該內容應該放在主 SKILL.md
* **被忽略的內容**：若 Claude 從不讀某個打包檔，可能是不必要或在主指示中不夠明顯

依觀察而非假設迭代。技能的中繼資料中的 `name` 與 `description` 尤其關鍵。Claude 會用它們判斷是否觸發技能。請確保它們清楚描述技能做什麼，以及何時使用。

## 需避免的反模式

### 避免 Windows 風格路徑

即使在 Windows，也永遠使用正斜線：

* ✓ **好**：`scripts/helper.py`, `reference/guide.md`
* ✗ **避免**：`scripts\helper.py`, `reference\guide.md`

Unix 風格路徑可跨平台使用；Windows 風格會在 Unix 系統出錯。

### 避免提供過多選項

除非必要，不要提供多個作法：

````markdown  theme={null}
**Bad example: Too many choices** (confusing):
"You can use pypdf, or pdfplumber, or PyMuPDF, or pdf2image, or..."

**Good example: Provide a default** (with escape hatch):
"Use pdfplumber for text extraction:
```python
import pdfplumber
```

For scanned PDFs requiring OCR, use pdf2image with pytesseract instead."
````

## 進階：含可執行程式碼的技能

以下章節針對包含可執行腳本的技能。若你的技能只有 markdown 指示，請跳到 [有效技能檢查表](#checklist-for-effective-skills)。

### 直接解決，不要推給 Claude

撰寫技能的腳本時，請處理錯誤狀況，不要把問題丟給 Claude。

**好例子：明確處理錯誤**：

```python  theme={null}
def process_file(path):
    """Process a file, creating it if it doesn't exist."""
    try:
        with open(path) as f:
            return f.read()
    except FileNotFoundError:
        # Create file with default content instead of failing
        print(f"File {path} not found, creating default")
        with open(path, 'w') as f:
            f.write('')
        return ''
    except PermissionError:
        # Provide alternative instead of failing
        print(f"Cannot access {path}, using default")
        return ''
```

**壞例子：丟給 Claude**：

```python  theme={null}
def process_file(path):
    # Just fail and let Claude figure it out
    return open(path).read()
```

設定參數也應該有理由並被文件化，以避免「巫毒常數」（Ousterhout 定律）。如果你不知道正確值，Claude 要怎麼判斷？

**好例子：自我文件化**：

```python  theme={null}
# HTTP requests typically complete within 30 seconds
# Longer timeout accounts for slow connections
REQUEST_TIMEOUT = 30

# Three retries balances reliability vs speed
# Most intermittent failures resolve by the second retry
MAX_RETRIES = 3
```

**壞例子：Magic numbers**：

```python  theme={null}
TIMEOUT = 47  # Why 47?
RETRIES = 5   # Why 5?
```

### 提供工具腳本

即使 Claude 能自己寫腳本，預先提供的腳本仍有優勢：

**工具腳本的好處：**

* 比生成式程式碼更可靠
* 節省 token（不必把程式碼放進脈絡）
* 節省時間（不需生成程式碼）
* 確保跨使用的一致性

<img src="https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-executable-scripts.png?fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=4bbc45f2c2e0bee9f2f0d5da669bad00" alt="Bundling executable scripts alongside instruction files" data-og-width="2048" width="2048" data-og-height="1154" height="1154" data-path="images/agent-skills-executable-scripts.png" data-optimize="true" data-opv="3" srcset="https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-executable-scripts.png?w=280&fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=9a04e6535a8467bfeea492e517de389f 280w, https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-executable-scripts.png?w=560&fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=e49333ad90141af17c0d7651cca7216b 560w, https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-executable-scripts.png?w=840&fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=954265a5df52223d6572b6214168c428 840w, https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-executable-scripts.png?w=1100&fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=2ff7a2d8f2a83ee8af132b29f10150fd 1100w, https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-executable-scripts.png?w=1650&fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=48ab96245e04077f4d15e9170e081cfb 1650w, https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-executable-scripts.png?w=2500&fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=0301a6c8b3ee879497cc5b5483177c90 2500w" />

上圖顯示可執行腳本如何與指示檔共存。指示檔（forms.md）引用腳本，Claude 可以執行它而不必把內容載入脈絡。

**重要區分**：在指示中要清楚說明 Claude 應該：

* **執行腳本**（最常見）："Run `analyze_form.py` to extract fields"
* **讀取作為參考**（較複雜邏輯）："See `analyze_form.py` for the field extraction algorithm"

對多數工具腳本，執行更可靠也更高效。腳本執行的細節請見下方 [Runtime environment](#runtime-environment)。

**範例：**

````markdown  theme={null}
## Utility scripts

**analyze_form.py**: Extract all form fields from PDF

```bash
python scripts/analyze_form.py input.pdf > fields.json
```

Output format:
```json
{
  "field_name": {"type": "text", "x": 100, "y": 200},
  "signature": {"type": "sig", "x": 150, "y": 500}
}
```

**validate_boxes.py**: Check for overlapping bounding boxes

```bash
python scripts/validate_boxes.py fields.json
# Returns: "OK" or lists conflicts
```

**fill_form.py**: Apply field values to PDF

```bash
python scripts/fill_form.py input.pdf fields.json output.pdf
```
````

### 使用視覺分析

當輸入可以渲染為圖片時，讓 Claude 進行視覺分析：

````markdown  theme={null}
## Form layout analysis

1. Convert PDF to images:
   ```bash
   python scripts/pdf_to_images.py form.pdf
   ```

2. Analyze each page image to identify form fields
3. Claude can see field locations and types visually
````

<Note>
  在此例中，你需要撰寫 `pdf_to_images.py` 腳本。
</Note>

Claude 的視覺能力有助於理解版面與結構。

### 建立可驗證的中間產物

當 Claude 執行複雜、開放式任務時，可能會犯錯。「plan-validate-execute」模式可讓 Claude 先建立結構化計畫，再用腳本驗證，最後執行，藉此提早抓錯。

**範例**：假設你要 Claude 依試算表更新 PDF 裡 50 個欄位。若沒有驗證，Claude 可能會引用不存在的欄位、產生衝突值、漏掉必填欄位，或套用錯誤更新。

**解法**：使用上面的流程模式（PDF 表單填寫），但加入中間的 `changes.json` 檔，在套用變更前先驗證。流程變成：分析 → **建立計畫檔** → **驗證計畫** → 執行 → 驗證。

**為何有效：**

* **早期抓錯**：驗證在變更套用前就能發現問題
* **可機器驗證**：腳本提供客觀驗證
* **可逆規劃**：Claude 可在不動原檔的情況下反覆修正計畫
* **清楚除錯**：錯誤訊息指向具體問題

**適用時機**：批次操作、破壞性變更、複雜驗證規則、高風險操作。

**實作提示**：驗證腳本應提供具體錯誤訊息，例如「Field 'signature_date' not found. Available fields: customer_name, order_total, signature_date_signed」，以便 Claude 修正。

### 套件依賴

技能在程式碼執行環境中運作，依平台而異：

* **claude.ai**：可從 npm 與 PyPI 安裝套件，也可從 GitHub 取得
* **Anthropic API**：無網路存取，也無法於執行時安裝套件

請在 SKILL.md 列出需要的套件，並在 [程式碼執行工具文件](/en/docs/agents-and-tools/tool-use/code-execution-tool) 中確認可用性。

### 執行環境

技能在可存取檔案系統、支援 bash 與程式執行的環境中運作。關於這種架構的概念說明，請見概覽中的 [The Skills architecture](/en/docs/agents-and-tools/agent-skills/overview#the-skills-architecture)。

**對撰寫的影響：**

**Claude 如何存取技能：**

1. **中繼資料預載**：啟動時會將所有技能 YAML frontmatter 的 name 與 description 載入系統提示
2. **按需讀檔**：Claude 透過 bash Read 工具在需要時讀取 SKILL.md 與其他檔案
3. **腳本可直接執行**：工具腳本可透過 bash 執行，無需把內容載入脈絡。只有腳本輸出會消耗 token
4. **大型檔案無脈絡成本**：參考檔、資料或文件只有在讀取時才消耗 token

* **檔案路徑很重要**：Claude 以檔案系統導覽技能目錄。使用正斜線（`reference/guide.md`），不要用反斜線
* **檔名要有意義**：使用能表達內容的名稱：`form_validation_rules.md`，不要用 `doc2.md`
* **以可發現性組織**：依領域或功能組織目錄
  * 好：`reference/finance.md`、`reference/sales.md`
  * 壞：`docs/file1.md`、`docs/file2.md`
* **打包完整資源**：包含完整 API 文件、豐富範例、大型資料集；未存取前無脈絡成本
* **偏好腳本處理可決定性操作**：寫 `validate_form.py`，不要要求 Claude 生成驗證程式
* **清楚標示執行意圖**：
  * "Run `analyze_form.py` to extract fields"（執行）
  * "See `analyze_form.py` for the extraction algorithm"（作為參考閱讀）
* **測試檔案存取路徑**：用真實請求驗證 Claude 能否導覽你的目錄結構

**範例：**

```
bigquery-skill/
├── SKILL.md (overview, points to reference files)
└── reference/
    ├── finance.md (revenue metrics)
    ├── sales.md (pipeline data)
    └── product.md (usage analytics)
```

當使用者詢問營收時，Claude 會讀 SKILL.md，看到 `reference/finance.md`，並用 bash 讀該檔案。sales.md 與 product.md 仍保留在檔案系統中，不會消耗任何脈絡 token。這種基於檔案系統的模型使漸進揭露得以成立。Claude 可導航並選擇性載入每個任務所需內容。

完整技術架構細節，請見 Skills 概覽中的 [How Skills work](/en/docs/agents-and-tools/agent-skills/overview#how-skills-work)。

### MCP 工具引用

若你的技能使用 MCP（Model Context Protocol）工具，務必使用完整限定名稱，避免「tool not found」錯誤。

**格式**：`ServerName:tool_name`

**範例**：

```markdown  theme={null}
Use the BigQuery:bigquery_schema tool to retrieve table schemas.
Use the GitHub:create_issue tool to create issues.
```

其中：

* `BigQuery` 與 `GitHub` 是 MCP server 名稱
* `bigquery_schema` 與 `create_issue` 是伺服器內的工具名稱

沒有 server 前綴時，Claude 可能找不到工具，尤其是有多個 MCP server 可用時。

### 避免假設工具已安裝

不要假設套件可用：

````markdown  theme={null}
**Bad example: Assumes installation**:
"Use the pdf library to process the file."

**Good example: Explicit about dependencies**:
"Install required package: `pip install pypdf`

Then use it:
```python
from pypdf import PdfReader
reader = PdfReader("file.pdf")
```"
````

## 技術備註

### YAML frontmatter 要求

SKILL.md frontmatter 只能包含 `name`（最多 64 字元）與 `description`（最多 1024 字元）。完整結構細節見 [Skills overview](/en/docs/agents-and-tools/agent-skills/overview#skill-structure)。

### Token 預算

SKILL.md 內文保持在 500 行內以確保最佳效能。若超過，請使用上述漸進揭露模式拆成多檔案。架構細節見 [Skills overview](/en/docs/agents-and-tools/agent-skills/overview#how-skills-work)。

## 有效技能檢查表

在分享技能前，請確認：

### 核心品質

* [ ] 描述具體且包含關鍵字
* [ ] 描述同時包含「技能做什麼」與「何時使用」
* [ ] SKILL.md 內文少於 500 行
* [ ] 需要時把細節放到獨立檔案
* [ ] 無時間敏感資訊（或放在「舊模式」段落）
* [ ] 全文術語一致
* [ ] 範例具體，不抽象
* [ ] 檔案引用不超過一層
* [ ] 漸進揭露使用得當
* [ ] 工作流程步驟清楚

### 程式碼與腳本

* [ ] 腳本能直接解決問題，而不是丟給 Claude
* [ ] 錯誤處理清楚且有幫助
* [ ] 無「巫毒常數」（所有值都有理由）
* [ ] 指示中列出所需套件並確認可用
* [ ] 腳本有清楚文件
* [ ] 無 Windows 風格路徑（全為正斜線）
* [ ] 關鍵操作有驗證/確認步驟
* [ ] 品質關鍵任務有回饋迴圈

### 測試

* [ ] 至少建立三個評估
* [ ] 以 Haiku、Sonnet、Opus 測試
* [ ] 用真實使用情境測試
* [ ] 納入團隊回饋（若適用）

## 下一步

<CardGroup cols={2}>
  <Card title="Get started with Agent Skills" icon="rocket" href="/en/docs/agents-and-tools/agent-skills/quickstart">
    Create your first Skill
  </Card>

  <Card title="Use Skills in Claude Code" icon="terminal" href="/en/docs/claude-code/skills">
    Create and manage Skills in Claude Code
  </Card>

  <Card title="Use Skills with the API" icon="code" href="/en/api/skills-guide">
    Upload and use Skills programmatically
  </Card>
</CardGroup>
