---
description: Analyze what existing code might break and update the plan with mitigations
---

# Plan Check: Gap (Breakage Analysis)

Analyze how planned changes will affect existing code and update the plan with mitigations.

<!-- Shared step: keep in sync across all plan-check commands -->
## Step 1: Context Gathering (Haiku agent)

Launch a **Haiku** agent to gather project context. The agent should:

1. Find the plan:
   - Glob `~/.claude/plans/*.md` and read the most recently modified file (use `stat` to compare)
   - If no plan file found on disk, extract plan text from conversation context
2. From the plan, list all source files it references (paths mentioned in the plan text)

Return: the full plan text, **the plan file path** (or null if from conversation context), a one-paragraph plan summary, and the list of referenced source files.

<!-- Shared step: keep in sync across all plan-check commands -->
## Step 2: Codebase Mapping (Haiku agent)

Launch a **Haiku** agent with the plan text and summary from Step 1. The agent should:

1. List every file the plan proposes to **create**, **modify**, or **delete** (the file manifest)
2. For each file that already exists, read its current contents
3. For each modified file, grep the codebase for callers and importers (search for the filename, exported function names, class names). Build a caller map.
4. Detect whether the project has tests: glob for `test_*`, `*_test.*`, `*_spec.*`, `tests/`, `__tests__/`, `spec/` patterns. Read existing test files that cover modified code.

Return: the file manifest, contents of existing files, caller map, test file contents, and `has_tests` boolean.

## Step 3: Breakage Analysis (Sonnet agent)

Launch the **breakage-analyst** agent. Pass it the outputs from Steps 1 and 2 as context.

The agent will return findings in BRK-NNN format with plan amendments, including regression test recommendations.

## Step 4: Update Plan

1. Read the plan file path from Step 1
2. Collect all findings and plan amendments from Step 3
3. Apply ALL amendments (every severity level -- Low through Critical), processing in severity order (Critical first, then High, Medium, Low). If two amendments target the same section with incompatible operations, apply the higher-severity amendment and print a note flagging the conflict.
4. Add regression test items if the breakage-analyst recommended them
5. Append a "Breakage Analysis Notes" section listing all changes made, with finding IDs and severity levels
6. If the plan was from conversation context (no file path), write the amended plan to `~/.claude/plans/amended-plan.md` and report its path
7. Print a brief summary: count of findings by severity, confirmation that the plan was updated
