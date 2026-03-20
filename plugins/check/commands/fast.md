---
description: Fast check -- verify correctness and update the plan or report code issues
---

# Check: Fast

Quick verification for correctness, completeness, and assumptions. In plan mode, updates the plan to address findings. In code mode, reports findings on the diff.

**Output convention**: When launching an agent, print its model in brackets after the description, e.g. `Launching verify-agent... [sonnet]`

## Step 1: Detect Input

1. Check your system prompt for a plan file path (e.g., "A plan file already exists at /path/to/plan.md"). If found: **MODE = plan**, read the plan file. Note the plan file path.
2. If no plan path, run `git diff HEAD` (staged + unstaged). If non-empty: **MODE = code**, DIFF_TEXT = output.
3. If git diff HEAD is empty AND you are not on main/master branch, run `git diff main...HEAD` (or `git diff master...HEAD` if main doesn't exist). If non-empty: **MODE = code**, DIFF_TEXT = output.
4. If nothing found: print "Nothing to check. Enter plan mode or make code changes, then re-run." and **STOP**.

## Step 2: Verification

**Scope context**: Produce SCOPE_SUMMARY based on MODE:
- **Code mode**: Run `git diff --stat DIFF_REF` (where DIFF_REF is the same git ref used in Step 1: `HEAD` if item 2 triggered, or `main...HEAD`/`master...HEAD` if item 3 triggered).
- **Plan mode**: Count section headings (H2/H3) and total lines. Do NOT run git diff -- there is no diff in plan mode.

Launch the **verify-agent** agent. Prepend its input with the scope calibration header:

```
Scope context: <SCOPE_SUMMARY>
Calibrate your analysis to the scope of these changes. If the change is
outside your analysis domain, return no findings rather than forcing
low-value observations.
```

Pass it:
- **Plan mode**: the full plan text and the plan file path
- **Code mode**: the DIFF_TEXT

The agent reads source files itself as needed.

If the project has tests (glob for `test_*`, `*_test.*`, `*_spec.*`, `tests/`, `__tests__/`, `spec/` patterns), instruct the agent to emphasize the test quality analysis dimension -- pay particular attention to whether proposed tests are real behavioral tests or trivially self-passing.

The agent will return findings in VFY-NNN format.

## Step 3: Process Output

**Plan mode**:
1. Read the plan file path from Step 1
2. Collect all findings and plan amendments from Step 2
3. Apply ALL amendments (every severity level -- Low through Critical), processing in severity order (Critical first, then High, Medium, Low). If two amendments target the same section with incompatible operations, apply the higher-severity amendment and print a note flagging the conflict.
4. Append a "Verification Notes" section listing all changes made, with finding IDs and severity levels
5. Print a brief summary: count of findings by severity, confirmation that the plan was updated

**Code mode**:
1. Collect all findings from Step 2
2. Print a findings report grouped by severity (Critical first, then High, Medium, Low)
3. For each finding, print the ID, severity, category, description, evidence, and recommendation
4. Print a summary: count of findings by severity
5. Do NOT edit any files
