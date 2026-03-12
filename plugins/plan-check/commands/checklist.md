---
description: Add or improve an implementation checklist in the plan
---

# Plan Check: Checklist

Evaluate the plan's implementation checklist (or create one) and update the plan directly.

<!-- Shared step: keep in sync across all plan-check commands -->
## Step 1: Context Gathering (Haiku agent)

Launch a **Haiku** agent to gather project context. The agent should:

1. Find the plan:
   - Glob `~/.claude/plans/*.md` and read the most recently modified file (use `stat` to compare)
   - If no plan file found on disk, extract plan text from conversation context
2. From the plan, list all source files it references (paths mentioned in the plan text)

Return: the full plan text, **the plan file path** (or null if from conversation context), a one-paragraph plan summary, and the list of referenced source files.

<!-- Shared step: keep in sync across all plan-check commands -->
## Step 2: Scope Analysis (Haiku agent)

Launch a **Haiku** agent with the plan text and summary from Step 1. The agent should:

1. List every file the plan proposes to **create**, **modify**, or **delete** (the file manifest)
2. For each file that already exists, read its current contents
3. Search the codebase for similar existing features or patterns (grep for related function names, class names, or module names mentioned in the plan)
4. Identify conventions used in those similar files (naming, structure, imports)

Return: the file manifest (create/modify/delete), contents of existing files, related files found, and detected conventions.

## Step 3: Checklist Generation (Sonnet agent)

Launch the **checklist-architect** agent. Pass it the outputs from Steps 1 and 2 as context.

The agent will return findings in CHK-NNN format with plan amendments containing the complete checklist.

## Step 4: Update Plan

1. Read the plan file path from Step 1
2. Collect all findings and plan amendments from Step 3
3. Apply ALL amendments (every severity level -- Low through Critical), processing in severity order (Critical first, then High, Medium, Low). If two amendments target the same section with incompatible operations, apply the higher-severity amendment and print a note flagging the conflict. In practice, the checklist-architect typically emits a single `replace` or `append-section` amendment for the full checklist -- if the plan has an existing checklist section, replace it; if not, append the new checklist section.
4. Append a "Checklist Notes" section listing CHK findings with IDs, severity, and what was changed
5. If the plan was from conversation context (no file path), write the amended plan to `~/.claude/plans/amended-plan.md` and report its path
6. Print a brief summary: count of findings by severity, confirmation that the plan was updated
