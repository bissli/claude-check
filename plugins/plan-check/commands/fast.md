---
description: Fast plan check -- verify correctness and update the plan to address issues
---

# Plan Check: Fast

Quick verification of an implementation plan for correctness, completeness, and assumptions, then update the plan to address all findings.

**Output convention**: When launching an agent, print its model in brackets after the description, e.g. `Launching plan-verifier... [sonnet]`

## Step 1: Verification

Find the plan using this priority order:
1. If your system prompt indicates a plan file path (e.g., "A plan file already exists at /path/to/plan.md"), use that path directly.
2. Otherwise, glob `~/.claude/plans/*.md` and read the most recently modified file (use `stat` to compare).
3. If no plan file found on disk, extract plan text from conversation context.

Note the plan file path (or null if from conversation).

Launch the **plan-verifier** agent, passing it the full plan text and the plan file path. The agent reads source files itself as needed.

If the project has tests (glob for `test_*`, `*_test.*`, `*_spec.*`, `tests/`, `__tests__/`, `spec/` patterns), instruct the agent to emphasize the test quality analysis dimension -- pay particular attention to whether proposed tests are real behavioral tests or trivially self-passing.

The agent will return findings in VFY-NNN format with plan amendments.

## Step 2: Update Plan

1. Read the plan file path from Step 1
2. Collect all findings and plan amendments from Step 1
3. Apply ALL amendments (every severity level -- Low through Critical), processing in severity order (Critical first, then High, Medium, Low). If two amendments target the same section with incompatible operations, apply the higher-severity amendment and print a note flagging the conflict.
4. Append a "Verification Notes" section listing all changes made, with finding IDs and severity levels
5. If the plan was from conversation context (no file path), write the amended plan to `~/.claude/plans/amended-plan.md` and report its path
6. Print a brief summary: count of findings by severity, confirmation that the plan was updated
