# Clarify Action

> **Part of the do-work skill.** Invoked when routing determines the user wants to answer open questions in a UR. Provides interactive Q&A to complete incomplete specifications.

An interactive clarification system that helps users answer open questions generated during request capture. Transforms vague specs into complete, actionable requirements.

## Philosophy

- **Make gaps explicit** — every ambiguity becomes a concrete question
- **Speed through defaults** — AI suggestions let users confirm quickly
- **Transparency over magic** — user sees what AI assumed and can override
- **Progressive refinement** — partial answers are fine, complete later
- **Expansion is natural** — new requirements discovered during clarification are captured

## When to Use

- After `do work add` when questions were generated
- When `do work verify` shows unanswered questions
- When user says "clarify", "answer questions", "complete questions"
- Before `do work run` to ensure specs are complete

## Three-Command Model

| Command | Purpose | Modifies Files? |
|---------|---------|----------------|
| `do work add` | Capture + generate questions | Yes (creates UR) |
| `do work verify` | Report completeness status | No (read-only) |
| `do work clarify` | Answer open questions | Yes (updates UR) |

## Workflow

### Step 1: Find the Target UR

1. **If user specifies a UR** (e.g., "clarify UR-015"): Use that UR directly
2. **If user specifies a REQ** (e.g., "clarify REQ-018"): Read the REQ's `user_request` field to find the UR
3. **If no target specified**: Find the most recent UR folder in `do-work/user-requests/` (highest UR number)

### Step 2: Read the UR and Identify Questions

1. Read `do-work/user-requests/UR-NNN/input.md`
2. Locate the "Open Questions" section
3. Parse each question:
   - Question ID (Q1, Q2, etc.)
   - Priority level (CRITICAL, IMPORTANT, MINOR)
   - Question text
   - Options (if multiple choice)
   - AI-suggested default (marked with `[x]` and `← AI suggested`)
   - Current answer status (answered or not)

**If no Open Questions section exists:**
- Report this is a legacy UR without questions
- Offer to generate questions retroactively (spawn a subagent to analyze the input and add questions)

### Step 3: Present Questions to User

Present ALL unanswered questions at once. Never ask questions one-by-one.

**For tools with structured question UI (e.g., ask_user_input):**

```javascript
{
  "questions": [
    {
      "question": "Q1 [CRITICAL]: Export format?",
      "header": "Q1",
      "multiSelect": false,
      "options": [
        { "label": "CSV only", "description": "Simple, Excel-compatible" },
        { "label": "Both CSV and JSON (Recommended)", "description": "AI suggested: flexibility for different use cases" },
        { "label": "Multiple formats (CSV, JSON, Excel)", "description": "Maximum flexibility" }
      ]
    },
    // ... all remaining questions in ONE call
  ]
}
```

**For text-based tools:**

```
=== CLARIFICATION FOR UR-015 (5 questions) ===

Q1 [CRITICAL]: Export format?
  [a] CSV only
  [b] Both CSV and JSON  ← AI suggested
  [c] Multiple formats (CSV, JSON, Excel)
  [d] Other

Q2 [CRITICAL]: Data scope?
  [a] All table data
  [b] Currently filtered/visible data  ← AI suggested
  [c] User-selected rows only
  [d] Other

Q3 [IMPORTANT]: UI placement?
  [a] Main toolbar
  [b] Table header  ← AI suggested
  [c] Context menu (right-click)
  [d] Other

Q4 [IMPORTANT]: Size constraints?
  [a] No special handling
  [b] Warn if export >10,000 rows  ← AI suggested
  [c] Hard limit at _____ rows
  [d] Other

Q5 [MINOR]: File naming?
  [a] Auto-generate (table-date.csv)  ← AI suggested
  [b] User specifies each time
  [c] Use template
  [d] Other

---
Provide answers (format: 1=b, 2=b, 3=a, 4=b, 5=a)
OR answer each line separately
OR type "keep defaults" to accept all AI suggestions
```

**Batch size control:**
- Default: max 10 questions per interaction
- If more than 10 questions, batch them (show 10, then next 10)
- Each batch still shows multiple questions, never one-by-one

### Step 4: Record Answers

For each answer received:

1. **Update the checkbox in Open Questions section:**

   **If user confirmed AI suggestion:**
   ```markdown
   - [x] Both CSV and JSON  ← Confirmed
   ```

   **If user changed AI suggestion:**
   ```markdown
   - [x] CSV only  ← Changed (was: Both CSV and JSON AI suggested)
   ```

   **If user selected option without AI default:**
   ```markdown
   - [x] CSV only
   ```

2. **Add detailed answer to Answers section:**

   ```markdown
   ## Answers

   **Q1 - Export format:**
   CSV only for MVP. JSON support can be added later if needed.
   User decided: Start simple, validate use case first.

   **Q2 - Data scope:**
   Currently filtered/visible data. AI suggestion confirmed.
   Reasoning: "Export what I see" matches user mental model.
   ```

3. **Update frontmatter:**

   ```yaml
   ---
   questions_total: 5
   questions_answered: 5
   ai_suggested: 5
   human_confirmed: 3
   human_changed: 2
   status: clarified
   clarified: 2025-02-06T14:30:00Z
   ---
   ```

### Step 5: Detect and Handle Expansions

**Watch for expansion signals in answers:**
- Keywords: "also", "and", "plus", "additionally", "oh and"
- New features mentioned in context of answering
- Scope beyond original request

**When expansion detected:**

1. **Capture the expansion:**
   ```markdown
   ## Additional Requirements
   <!-- Added during clarify 2025-02-06 14:30 -->
   Add import functionality to upload CSVs.
   ```

2. **Generate new questions for expansion:**
   - Q6 [CRITICAL]: Import formats?
   - Q7 [CRITICAL]: Duplicate handling?
   - Q8 [IMPORTANT]: Import UI placement?

3. **Update frontmatter:**
   ```yaml
   questions_total: 8  # was 5
   has_expansion: true
   ```

4. **Continue clarification with new questions**

**Expansion modes (controlled via user preference):**

- **append** (default): Keep expanded requirements in same UR
- **split**: Ask user if they want a new UR for the expansion
- **ask**: Prompt user each time expansion is detected

### Step 6: Generate Specification Summary

After all questions are answered (or all critical ones):

1. **Auto-generate a Specification Summary section** that consolidates:
   - Original user input
   - All answers to questions
   - Any expanded requirements

2. **Add as final section:**

   ```markdown
   ## Specification Summary

   <!-- Auto-generated from Original Input + Answers Q1-Q5 -->

   Add export functionality to data table supporting CSV format.

   **Format Selection:**
   CSV only for MVP. Default format, no user selection needed.

   **Data Scope:**
   Export only currently filtered and visible table data. Respect active search, column filters, and hidden columns.

   **UI Placement:**
   Context menu (right-click on table). Export is a secondary action.

   **Performance:**
   Warn if export >10,000 rows. Show loading indicator. No hard limit.

   **File Naming:**
   Auto-generate: {table-name}-{YYYY-MM-DD}.csv
   Example: users-2025-02-06.csv
   ```

3. **Update status:**
   ```yaml
   status: clarified
   clarified: 2025-02-06T14:30:00Z
   ```

### Step 7: Report Completion

```
✅ Clarification complete for UR-015

Questions reviewed: 5/5
  - AI suggestions confirmed: 3
  - User changed: 2

Updated: do-work/user-requests/UR-015/input.md

Next steps:
  - Run `do work verify UR-015` to check completeness
  - Run `do work run` to start building
```

## Command Reference

```bash
# Basic usage
do work clarify              # Latest UR
do work clarify UR-015       # Specific UR

# Multiple URs
do work clarify UR-015 UR-017 UR-020
do work clarify --batch      # All pending URs with unanswered questions

# Question filtering
do work clarify --critical-only    # Answer only CRITICAL questions
do work clarify --skip-minor       # Skip MINOR priority questions

# AI defaults handling
do work clarify --keep-ai-defaults   # Accept all AI suggestions without review
do work clarify --review-ai          # Review each AI suggestion (default)
do work clarify --clear-ai           # Clear AI defaults, start fresh

# Expansion handling
do work clarify --expand=append    # Append expansions to same UR (default)
do work clarify --expand=split     # Create new UR for expansions
do work clarify --expand=ask       # Ask user per expansion
```

## UR File Structure After Clarify

```markdown
---
id: UR-015
created: 2025-02-06T10:30:00Z
status: clarified
questions_total: 5
questions_answered: 5
ai_suggested: 5
human_confirmed: 3
human_changed: 2
clarified: 2025-02-06T14:30:00Z
requests: [REQ-030]
word_count: 3
---

# Add export functionality

## Summary

User requested export functionality. Clarified to CSV-only export of filtered data via context menu.

## Extracted Requests

| ID | Title | Summary |
|----|-------|---------|
| REQ-030 | Export to CSV | Export filtered table data to CSV file |

## Original User Input

add export functionality

## Open Questions

**Q1 [CRITICAL]:** Export format?
- [x] CSV only  ← Changed (was: Both CSV and JSON AI suggested)
- [ ] JSON only
- [ ] Both CSV and JSON

**Q2 [CRITICAL]:** Data scope?
- [ ] All table data
- [x] Currently filtered/visible data  ← Confirmed
- [ ] User-selected rows

**Q3 [IMPORTANT]:** UI placement?
- [ ] Main toolbar
- [ ] Table header
- [x] Context menu (right-click)  ← Changed (was: Table header AI suggested)

**Q4 [IMPORTANT]:** Size constraints?
- [ ] No special handling
- [x] Warn if export >10,000 rows  ← Confirmed
- [ ] Hard limit

**Q5 [MINOR]:** File naming?
- [x] Auto-generate (table-2025-02-06.csv)  ← Confirmed
- [ ] User specifies

## Answers

**Q1 - Export format:**
CSV only for MVP. JSON support can be added later if needed.
User decided: Start simple, validate use case first.

**Q2 - Data scope:**
Currently filtered/visible data. AI suggestion confirmed.
Reasoning: "Export what I see" matches user mental model.

**Q3 - UI placement:**
Context menu (right-click on table). Changed from AI suggestion.
User reasoning: Export is secondary action, doesn't need toolbar space.

**Q4 - Size constraints:**
Warn if >10k rows. AI suggestion confirmed.
Reasoning: Good balance between safety and flexibility.

**Q5 - File naming:**
Auto-generate with date. AI suggestion confirmed.
Reasoning: Prevents overwrites, includes context automatically.

## Specification Summary

<!-- Auto-generated 2025-02-06 14:30 -->

Add export functionality to data table supporting CSV format.

**Format Selection:**
CSV only for MVP. Default format, no user selection needed.

**Data Scope:**
Export only currently filtered and visible table data. Respect active search, column filters, and hidden columns.

**UI Placement:**
Context menu (right-click on table). Export is a secondary action.

**Performance:**
Warn if export >10,000 rows. Show loading indicator. No hard limit.

**File Naming:**
Auto-generate: {table-name}-{YYYY-MM-DD}.csv
Example: users-2025-02-06.csv

---
*Captured: 2025-02-06T10:30:00Z*
*Clarified: 2025-02-06T14:30:00Z*
```

## Legacy UR Handling

For URs created before the Open Questions system:

1. **Detect legacy format:**
   - No "Open Questions" section in input.md
   - No `questions_total` in frontmatter

2. **Offer retroactive question generation:**
   ```
   This UR was created before the Open Questions system.
   Would you like me to analyze the input and generate questions?

   [a] Yes, generate questions
   [b] No, proceed without questions
   ```

3. **If yes:**
   - Analyze the original input
   - Generate questions with AI defaults
   - Add Open Questions section
   - Update frontmatter
   - Continue with normal clarify flow

## Expansion Example

```bash
do work clarify UR-015
```

```
[Shows widget with Q1-Q5]

User answers Q3: "Table header toolbar.
Also add import for uploading CSVs."

⚠️ EXPANSION DETECTED

New requirement: "Add import for uploading CSVs"
Generating questions Q6-Q8...

[Shows updated widget with Q1-Q8]

User completes all 8 questions.

✅ 8/8 answered (5 original + 3 expansion)
✅ Specification Summary generated
Updated UR-015/input.md
```

## What NOT To Do

- **Don't ask questions one-by-one** — always batch questions together
- **Don't lose AI suggestions** — preserve them even when user changes
- **Don't modify Original User Input** — it's the source of truth, keep verbatim
- **Don't skip Specification Summary** — it's needed for REQ extraction
- **Don't ignore expansions** — capture new requirements when they emerge
- **Don't require all questions answered** — partial clarification is valid
- **Don't block on MINOR questions** — critical questions matter most

## Tool Compatibility

This action works with any AI coding assistant that can:
- Read/write markdown files
- Present questions to users (structured UI preferred, text fallback works)

**Adaptation notes:**

| Assistant | Question Presentation | Answer Collection |
|-----------|----------------------|------------------|
| Claude Code | AskUserQuestion tool | Structured responses |
| Cursor | Chat with options | Text parsing |
| Aider | CLI prompt | stdin input |
| Copilot | PR comment checklist | Checkbox parsing |
| Windsurf | Interactive chat | Text/structured |

Use your platform's native question UI when available. Fall back to text-based presentation when not.
