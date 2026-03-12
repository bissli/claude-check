---
description: Verify plan correctness and update the plan to address all issues
---

# Plan Check: Verify

Verify an implementation plan for correctness, completeness, and assumptions, then update the plan to address all findings.

<!-- Shared step: keep in sync across all plan-check commands -->
## Step 1: Context Gathering (Haiku agent)

Launch a **Haiku** agent to gather project context. The agent should:

1. Find the plan:
   - Glob `~/.claude/plans/*.md` and read the most recently modified file (use `stat` to compare)
   - If no plan file found on disk, extract plan text from conversation context
2. From the plan, list all source files it references (paths mentioned in the plan text)

Return: the full plan text, **the plan file path** (or null if from conversation context), a one-paragraph plan summary, and the list of referenced source files.

<!-- Shared step: keep in sync across all plan-check commands -->
## Step 2: Scope Scan (Haiku agent)

Launch a **Haiku** agent with the plan text and summary from Step 1. The agent should:

1. List every file the plan proposes to **create**, **modify**, or **delete** (the file manifest)
2. For each file that already exists, read its current contents
3. Detect whether the project has tests: glob for `test_*`, `*_test.*`, `*_spec.*`, `tests/`, `__tests__/`, `spec/` patterns. Return `has_tests` boolean and test file paths.

Return: the file manifest (create/modify/delete), contents of existing files, `has_tests` boolean, and test file paths.

## Step 3: Verification (Sonnet agent)

Launch the **plan-verifier** agent. Pass it the outputs from Steps 1 and 2 as context.

If `has_tests` is true, emphasize the test quality analysis dimension -- instruct the agent to pay particular attention to whether proposed tests are real behavioral tests or trivially self-passing.

The agent will return findings in VFY-NNN format with plan amendments.

## Step 4: Update Plan

1. Read the plan file path from Step 1
2. Collect all findings and plan amendments from Step 3
3. Apply ALL amendments (every severity level -- Low through Critical), processing in severity order (Critical first, then High, Medium, Low). If two amendments target the same section with incompatible operations, apply the higher-severity amendment and print a note flagging the conflict.
4. Append a "Verification Notes" section listing all changes made, with finding IDs and severity levels
5. If the plan was from conversation context (no file path), write the amended plan to `~/.claude/plans/amended-plan.md` and report its path
6. Print a brief summary: count of findings by severity, confirmation that the plan was updated
