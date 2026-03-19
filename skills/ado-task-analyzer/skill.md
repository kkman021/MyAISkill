---
name: ado-task-analyzer
description: Fetch work items from Azure DevOps query, analyze intent from descriptions, explore codebase to locate issues, then post execution plans or clarification requests as comments. Use this skill when the user mentions Azure DevOps tasks, ADO work items, analyzing ADO bugs, processing DevOps query results, investigating work items, replying to ADO issues, or creating implementation plans from Azure DevOps tasks. Trigger even if the user says "幫我看 ADO task"、"分析 DevOps 的工作項目"、"去 Azure DevOps 抓 task" or similar casual phrasing.
---

# ADO Task Analyzer

Analyzes Azure DevOps work items from a configured query and responds to each with either an execution plan or a clarification request, depending on how well the issue is described and whether the codebase reveals the root cause.

## Prerequisites

Read these environment variables before starting. If any required one is missing, tell the user which ones are needed and stop.

| Variable | Required | Description |
|---|---|---|
| `AzureDevOps_PAT` | Yes | Personal Access Token |
| `AzureDevOps_ORG` | Yes | Organization name (e.g. `mycompany`) |
| `AzureDevOps_PROJECT` | Yes | Project name (e.g. `MyProject`) |
| `AzureDevOps_QUERY_ID` | Only if no specific IDs given | Saved query GUID — not needed when user provides Work Item IDs directly |

Authentication: Base64-encode the string `:<PAT>` and send as `Authorization: Basic <encoded>`.

> **CRITICAL RULE — NO AUTO-IMPLEMENTATION**: After posting the ADO comment (Step 6), this skill's job is complete. Do NOT proceed to implement code changes, create files, or modify anything in the codebase. Implementation is a separate task that requires explicit user instruction.

## Step 0 — Build reusable AUTH header and encoded project name

Before making any API call, run this **once** to set up shell variables for the entire session:

```bash
ADO_AUTH=$(printf ':%s' "$AzureDevOps_PAT" | base64 -w 0)
ADO_ORG="$AzureDevOps_ORG"
ADO_PROJECT_ENCODED=$(printf '%s' "$AzureDevOps_PROJECT" | sed 's/ /%20/g')
ADO_QUERY_ID="$AzureDevOps_QUERY_ID"
```

**Key points:**
- `printf` instead of `echo -n` — portable across bash/zsh/mingw
- `base64 -w 0` — prevents line-wrapping that breaks the Authorization header (mingw/Linux default wraps at 76 chars)
- URL-encode `AzureDevOps_PROJECT` — project names may contain spaces (e.g. `OIDC Provider` → `OIDC%20Provider`); a bare space in a URL causes curl exit code 3
- Short aliases (`ADO_AUTH`, `ADO_ORG`, `ADO_PROJECT_ENCODED`, `ADO_QUERY_ID`) are used in all subsequent commands for readability

All subsequent curl commands must use these short aliases:

```bash
curl -s -H "Authorization: Basic $ADO_AUTH" "https://dev.azure.com/$ADO_ORG/$ADO_PROJECT_ENCODED/_apis/..."
```

## Step 1 — Determine which Work Items to process

If the user provided specific Work Item IDs in their message, use those. Otherwise, fetch all IDs from the configured query:

```bash
curl -s -H "Authorization: Basic $ADO_AUTH" \
  "https://dev.azure.com/$ADO_ORG/$ADO_PROJECT_ENCODED/_apis/wit/wiql/$ADO_QUERY_ID?api-version=7.0"
```

Response contains `workItems[].id`. Collect all IDs.

Then fetch details for each ID (batch up to 200 at a time):

```bash
curl -s -H "Authorization: Basic $ADO_AUTH" \
  "https://dev.azure.com/$ADO_ORG/$ADO_PROJECT_ENCODED/_apis/wit/workitems?ids={ids}&fields=System.Id,System.Title,System.Description,System.WorkItemType,System.State&api-version=7.0"
```

For each work item, extract: `id`, `title`, `description`, `workItemType`, `state`.

> **Note**: Do not include `Microsoft.VSTS.Common.ReproSteps` in the fields list — this field may not exist in all projects and will cause a 400 error. If additional bug-specific fields are needed, query them in a separate request only after confirming the field exists.

## Step 2 — Analyze intent per Work Item

For each work item, decide: **can you understand what the user wants to achieve or what problem they're reporting?**

Intent is clear when the description answers:
- What is happening (or what is expected to happen)?
- In what context or scenario?
- What outcome is desired (for tasks) or what is broken (for bugs)?

Intent is **not** clear when descriptions are: blank, a single word, a vague phrase like "有問題" or "fix this", an internal code/ticket reference with no explanation, or a sentence that references something without explaining it.

Use judgment. A short but precise description ("登入頁面按下送出後沒有跳轉到首頁") is clear. A long but meandering description that never specifies the actual problem is not.

**If intent is unclear → go to Step 5a (Clarification).**
**If intent is clear → go to Step 3.**

## Step 3 — Explore the codebase

Working directory is the current project root. Use Grep, Glob, and Read to find code relevant to the issue.

Search strategy:
1. Extract key terms from the title and description: feature names, function names, error messages, UI element names, API paths, class or method names mentioned.
2. Search for those terms in source files.
3. If you find candidates, read the surrounding code to confirm relevance.
4. Look for likely failure points: validation logic, API calls, event handlers, state transitions, permission checks.

After exploration, decide: **can you identify the specific location or mechanism of the problem?**

- "Identify" means you found the code that is likely responsible — a specific file, method, or logic path — and you can reason about what is wrong or what needs to change.
- If you found relevant code but cannot determine the root cause, that is a partial finding. Report what you found without pretending it's conclusive.
- If you found nothing relevant, that is also a valid finding. Don't fabricate confidence.

**If you can identify the problem → go to Step 4 (Execution Plan).**
**If you cannot → go to Step 5b (Investigation Report).**

## Step 4 — Create execution plan and post comment

Use the Plan agent to design a concrete implementation plan. The plan should include:
- Root cause explanation (what the code is doing wrong and why)
- Files and locations that need to change
- Step-by-step changes, each scoped and actionable
- Any risks or considerations (e.g., migration needed, affects other callers)

Post the result as a comment on the work item (see Step 6). Format in Markdown.

**Comment template — Execution Plan:**

```markdown
## 🔍 分析結果

**類型**: {workItemType} | **狀態**: {state}

### 問題重述

{Re-state the problem from the analyst's perspective in 2–4 sentences. This is not a copy-paste of the original description — it is your interpretation: what is broken, under what condition, what the user observes, and what the expected behavior should be. This section helps confirm that the issue was understood correctly.}

### 問題定位

{explain the root cause and where it lives in the code — specific files, methods, and logic paths}

### 執行計畫

{paste plan agent output here}

### 影響範圍

{list files/components affected}

---
*由 Claude Code 自動分析 · {date}*
```

## Step 5a — Clarification comment (intent unclear)

The goal here is to help the reporter understand *how* to write a useful description, not just ask them to "add more detail". Point to what's missing specifically.

**Comment template — Clarification Request:**

```markdown
## ℹ️ 需要補充資訊

感謝您建立此工作項目。目前的描述不足以讓我們識別問題或開始實作。

### 目前缺少的資訊

{choose whichever apply, remove the rest}

- **重現步驟**: 如何操作才能觸發這個問題？
- **預期行為**: 你期望發生什麼？
- **實際行為**: 實際上發生了什麼？（如有錯誤訊息請附上）
- **影響範圍**: 哪個功能、頁面、或 API endpoint？
- **目標描述**: 這個 Task 希望達成什麼結果？（針對非 Bug 類型）

### 描述範例

**Bug 類型**:
> 在 [頁面/功能] 執行 [操作] 時，預期會 [預期結果]，但實際上 [實際結果]。錯誤訊息為：`[錯誤訊息]`

**Task 類型**:
> 需要在 [模組/功能] 新增/修改 [具體內容]，目的是 [商業目標或技術需求]。完成條件：[驗收標準]

---
*由 Claude Code 自動分析 · {date}*
```

## Step 5b — Investigation Report (code not located)

Report what you explored honestly. This gives the team context even if the issue isn't resolved.

**Comment template — Investigation Report:**

```markdown
## 🔍 調查報告

**類型**: {workItemType} | **狀態**: {state}

### 調查摘要

{summarize what you understood from the description}

### 探索結果

{describe what you searched for and what you found}

**搜尋關鍵字**: `{terms searched}`
**相關檔案**:
- `{file path}` — {why it seemed relevant, what it contains}
- ...

### 無法定位的原因

{explain why you couldn't pinpoint the problem: feature not found in codebase, description is too abstract to map to code, issue may be in external system, etc.}

### 建議下一步

{one concrete action the team can take to move forward — e.g., add logging, reproduce locally, clarify which module is affected}

---
*由 Claude Code 自動分析 · {date}*
```

## Step 6 — Post comment to Azure DevOps

Post a comment to each processed work item. Use **Markdown format** via the `format=markdown` query parameter and API version `7.1-preview.4`. Write the comment templates directly in Markdown — the API handles rendering to HTML server-side.

Always write the JSON body to a temp file first to avoid shell escaping issues:

```bash
cat > /tmp/ado_comment.json << 'JSONEOF'
{
  "text": "## Heading\n\n**bold** and `code`\n\n- item 1\n- item 2"
}
JSONEOF

curl -s -X POST \
  -H "Authorization: Basic $ADO_AUTH" \
  -H "Content-Type: application/json" \
  "https://dev.azure.com/$ADO_ORG/$ADO_PROJECT_ENCODED/_apis/wit/workitems/{id}/comments?format=markdown&api-version=7.1-preview.4" \
  -d @/tmp/ado_comment.json
```

**Key points:**
- `format=markdown` is a **query parameter**, NOT a request body field
- API version must be `7.1-preview.4` or later (earlier versions ignore the format parameter)
- The `text` field contains raw Markdown — no HTML conversion needed
- **JSON escaping**: The `text` value is a JSON string. Use `\n` for newlines within the Markdown content. Escape `"` as `\"`

**ADO Markdown rendering pitfall — HTML-like tags in code blocks:**
ADO's Markdown renderer does NOT treat fenced code blocks as opaque text. Any content that looks like an HTML tag (e.g. `<el-form>`, `<div>`, `<template>`) inside triple-backtick code blocks will be interpreted as real HTML and **silently stripped**, resulting in empty code blocks in the rendered comment.

**Workaround**: When code snippets contain angle brackets (`<`, `>`), always escape them as HTML entities (`&lt;`, `&gt;`) in the comment text. Also escape `&` as `&amp;` when it appears in attribute values (e.g. `&&` → `&amp;&amp;`). This applies to ALL code blocks in the comment, including inline code and fenced blocks. Example:

```
Wrong (will be stripped):
\`\`\`html
<el-form ref="ruleFormRef">
\`\`\`

Correct (renders properly):
\`\`\`
&lt;el-form ref="ruleFormRef"&gt;
\`\`\`
```

Verify success by checking that the response contains `"id":` (a numeric comment ID) and `"format": "markdown"`. If the response contains `"message":`, it indicates an error — log it and continue to the next item.

> **STOP after this step.** Do not implement any code changes. The skill's responsibility ends when the comment is posted.

## Step 7 — Summary

After processing all work items, print a summary table:

| ID | Title | Outcome |
|---|---|---|
| #{id} | {title} | Execution Plan / Investigation Report / Clarification Requested |

List any errors (e.g., failed to post comment) separately.
