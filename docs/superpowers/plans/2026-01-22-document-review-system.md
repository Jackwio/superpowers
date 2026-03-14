# 文件審查系統實作計畫

> **For agentic workers:** REQUIRED: Use superpowers:subagent-driven-development (if subagents available) or superpowers:executing-plans to implement this plan.

**目標：**在 brainstorming 與 writing-plans 技能中加入規格與計畫文件的審查迴圈。

**架構：**在各技能目錄建立審查者提示範本。修改技能檔案，在文件建立後加入審查迴圈。使用 Task 工具搭配 general-purpose 子代理派發審查者。

**技術棧：**Markdown 技能檔案、透過 Task 工具派發子代理

**規格：**docs/superpowers/specs/2026-01-22-document-review-system-design.md

---

## 區塊 1：規格文件審查者

此區塊將規格文件審查者加入 brainstorming 技能。

### 任務 1：建立規格文件審查者提示範本

**檔案：**
- 新增：`skills/brainstorming/spec-document-reviewer-prompt.md`

- [ ] **步驟 1：**建立審查者提示範本檔案

```markdown
# Spec Document Reviewer Prompt Template

Use this template when dispatching a spec document reviewer subagent.

**Purpose:** Verify the spec is complete, consistent, and ready for implementation planning.

**Dispatch after:** Spec document is written to docs/superpowers/specs/

```
Task tool (general-purpose):
  description: "Review spec document"
  prompt: |
    You are a spec document reviewer. Verify this spec is complete and ready for planning.

    **Spec to review:** [SPEC_FILE_PATH]

    ## What to Check

    | Category | What to Look For |
    |----------|------------------|
    | Completeness | TODOs, placeholders, "TBD", incomplete sections |
    | Coverage | Missing error handling, edge cases, integration points |
    | Consistency | Internal contradictions, conflicting requirements |
    | Clarity | Ambiguous requirements |
    | YAGNI | Unrequested features, over-engineering |

    ## CRITICAL

    Look especially hard for:
    - Any TODO markers or placeholder text
    - Sections saying "to be defined later" or "will spec when X is done"
    - Sections noticeably less detailed than others

    ## Output Format

    ## Spec Review

    **Status:** ✅ Approved | ❌ Issues Found

    **Issues (if any):**
    - [Section X]: [specific issue] - [why it matters]

    **Recommendations (advisory):**
    - [suggestions that don't block approval]
```

**Reviewer returns:** Status, Issues (if any), Recommendations
```

- [ ] **步驟 2：**驗證檔案建立正確

執行：`cat skills/brainstorming/spec-document-reviewer-prompt.md | head -20`
預期：顯示標題與目的區段

- [ ] **步驟 3：**提交

```bash
git add skills/brainstorming/spec-document-reviewer-prompt.md
git commit -m "feat: add spec document reviewer prompt template"
```

---

### 任務 2：在 Brainstorming 技能加入審查迴圈

**檔案：**
- 修改：`skills/brainstorming/SKILL.md`

- [ ] **步驟 1：**閱讀目前的 brainstorming 技能

執行：`cat skills/brainstorming/SKILL.md`

- [ ] **步驟 2：**在「After the Design」後加入審查迴圈

找到「After the Design」段落，於文件區塊之後、實作之前新增「Spec Review Loop」：

```markdown
**Spec Review Loop:**
After writing the spec document:
1. Dispatch spec-document-reviewer subagent (see spec-document-reviewer-prompt.md)
2. If ❌ Issues Found:
   - Fix the issues in the spec document
   - Re-dispatch reviewer
   - Repeat until ✅ Approved
3. If ✅ Approved: proceed to implementation setup

**Review loop guidance:**
- Same agent that wrote the spec fixes it (preserves context)
- If loop exceeds 5 iterations, surface to human for guidance
- Reviewers are advisory - explain disagreements if you believe feedback is incorrect
```

- [ ] **步驟 3：**驗證變更

執行：`grep -A 15 "Spec Review Loop" skills/brainstorming/SKILL.md`
預期：顯示新的審查迴圈區段

- [ ] **步驟 4：**提交

```bash
git add skills/brainstorming/SKILL.md
git commit -m "feat: add spec review loop to brainstorming skill"
```

---

## 區塊 2：計畫文件審查者

此區塊將計畫文件審查者加入 writing-plans 技能。

### 任務 3：建立計畫文件審查者提示範本

**檔案：**
- 新增：`skills/writing-plans/plan-document-reviewer-prompt.md`

- [ ] **步驟 1：**建立審查者提示範本檔案

```markdown
# Plan Document Reviewer Prompt Template

Use this template when dispatching a plan document reviewer subagent.

**Purpose:** Verify the plan chunk is complete, matches the spec, and has proper task decomposition.

**Dispatch after:** Each plan chunk is written

```
Task tool (general-purpose):
  description: "Review plan chunk N"
  prompt: |
    You are a plan document reviewer. Verify this plan chunk is complete and ready for implementation.

    **Plan chunk to review:** [PLAN_FILE_PATH] - Chunk N only
    **Spec for reference:** [SPEC_FILE_PATH]

    ## What to Check

    | Category | What to Look For |
    |----------|------------------|
    | Completeness | TODOs, placeholders, incomplete tasks, missing steps |
    | Spec Alignment | Chunk covers relevant spec requirements, no scope creep |
    | Task Decomposition | Tasks atomic, clear boundaries, steps actionable |
    | Task Syntax | Checkbox syntax (`- [ ]`) on tasks and steps |
    | Chunk Size | Each chunk under 1000 lines |

    ## CRITICAL

    Look especially hard for:
    - Any TODO markers or placeholder text
    - Steps that say "similar to X" without actual content
    - Incomplete task definitions
    - Missing verification steps or expected outputs

    ## Output Format

    ## Plan Review - Chunk N

    **Status:** ✅ Approved | ❌ Issues Found

    **Issues (if any):**
    - [Task X, Step Y]: [specific issue] - [why it matters]

    **Recommendations (advisory):**
    - [suggestions that don't block approval]
```

**Reviewer returns:** Status, Issues (if any), Recommendations
```

- [ ] **步驟 2：**驗證檔案建立

執行：`cat skills/writing-plans/plan-document-reviewer-prompt.md | head -20`
預期：顯示標題與目的區段

- [ ] **步驟 3：**提交

```bash
git add skills/writing-plans/plan-document-reviewer-prompt.md
git commit -m "feat: add plan document reviewer prompt template"
```

---

### 任務 4：在 Writing-Plans 技能加入審查迴圈

**檔案：**
- 修改：`skills/writing-plans/SKILL.md`

- [ ] **步驟 1：**閱讀目前技能檔案

執行：`cat skills/writing-plans/SKILL.md`

- [ ] **步驟 2：**加入逐區塊審查段落

在「Execution Handoff」之前加入：

```markdown
## Plan Review Loop

After completing each chunk of the plan:

1. Dispatch plan-document-reviewer subagent for the current chunk
   - Provide: chunk content, path to spec document
2. If ❌ Issues Found:
   - Fix the issues in the chunk
   - Re-dispatch reviewer for that chunk
   - Repeat until ✅ Approved
3. If ✅ Approved: proceed to next chunk (or execution handoff if last chunk)

**Chunk boundaries:** Use `## Chunk N: <name>` headings to delimit chunks. Each chunk should be ≤1000 lines and logically self-contained.
```

- [ ] **步驟 3：**更新任務語法範例為勾選框

將 Task Structure 區段改為勾選框語法：

```markdown
### Task N: [Component Name]

- [ ] **Step 1:** Write the failing test
  - File: `tests/path/test.py`
  ...
```

- [ ] **步驟 4：**驗證已加入審查迴圈段落

執行：`grep -A 15 "Plan Review Loop" skills/writing-plans/SKILL.md`
預期：顯示新的審查迴圈段落

- [ ] **步驟 5：**驗證任務語法範例已更新

執行：`grep -A 5 "Task N:" skills/writing-plans/SKILL.md`
預期：顯示勾選框語法 `### Task N:`

- [ ] **步驟 6：**提交

```bash
git add skills/writing-plans/SKILL.md
git commit -m "feat: add plan review loop and checkbox syntax to writing-plans skill"
```

---

## 區塊 3：更新計畫文件標頭

此區塊更新計畫文件標頭範本，以提及新的勾選框語法需求。

### 任務 5：更新 Writing-Plans 技能中的計畫標頭範本

**檔案：**
- 修改：`skills/writing-plans/SKILL.md`

- [ ] **步驟 1：**讀取目前的計畫標頭範本

執行：`grep -A 20 "Plan Document Header" skills/writing-plans/SKILL.md`

- [ ] **步驟 2：**更新標頭範本提及勾選框語法

計畫標頭需註明任務與步驟使用勾選框語法。更新標頭註解：

```markdown
> **For agentic workers:** REQUIRED: Use superpowers:subagent-driven-development (if subagents available) or superpowers:executing-plans to implement this plan. Tasks and steps use checkbox (`- [ ]`) syntax for tracking.
```

- [ ] **步驟 3：**驗證變更

執行：`grep -A 5 "For agentic workers:" skills/writing-plans/SKILL.md`
預期：顯示已更新的標頭，包含勾選框語法說明

- [ ] **步驟 4：**提交

```bash
git add skills/writing-plans/SKILL.md
git commit -m "docs: update plan header to reference checkbox syntax"
```
