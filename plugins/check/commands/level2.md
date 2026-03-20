---
description: verify + simplify + breakage + tests + precedent
---

# Check: Level 2

Multi-agent review combining verification, code quality/reuse/efficiency, breakage analysis, test review, and precedent scanning. In plan mode, updates the plan to address findings. In code mode, reports findings on the diff.

**Output convention**: When launching an agent, print its model in brackets after the description, e.g. `Launching verify-agent... [sonnet]` or `Launching precedent discovery... [haiku]`

## Step 1: Detect Input

1. Check your system prompt for a plan file path (e.g., "A plan file already exists at /path/to/plan.md"). If found: **MODE = plan**, read the plan file. Note the plan file path.
2. If no plan path, run `git diff HEAD` (staged + unstaged). If non-empty: **MODE = code**, DIFF_TEXT = output.
3. If git diff HEAD is empty AND you are not on main/master branch, run `git diff main...HEAD` (or `git diff master...HEAD` if main doesn't exist). If non-empty: **MODE = code**, DIFF_TEXT = output.
4. If nothing found: print "Nothing to check. Enter plan mode or make code changes, then re-run." and **STOP**.

## Step 2: Scope Summary

Produce SCOPE_SUMMARY based on MODE:
- **Code mode**: Run `git diff --stat DIFF_REF` (where DIFF_REF is the same git ref used in Step 1: `HEAD` if item 2 triggered, or `main...HEAD`/`master...HEAD` if item 3 triggered).
- **Plan mode**: Count section headings (H2/H3) and total lines (e.g., "Plan has 8 sections, 142 lines"). Scan for a "Files to Modify" or "Files to Change" section and count bullet entries if present; otherwise omit the file count. Do NOT run git diff -- there is no diff in plan mode.

## Step 3: First Wave (parallel agents)

Launch all 5 agents in parallel, passing each the full plan text and plan file path (plan mode) or the DIFF_TEXT (code mode). **Prepend each agent's input with the scope calibration header:**

```
Scope context: <SCOPE_SUMMARY>
Calibrate your analysis to the scope of these changes. If the change is
outside your analysis domain, return no findings rather than forcing
low-value observations.
```

Launch 5 tasks in parallel:

1. **verify-agent** (VFY prefix): Full verification -- correctness, completeness, edge cases, error handling, assumptions. If the project has tests (glob for `test_*`, `*_test.*`, `*_spec.*`, `tests/`, `__tests__/`, `spec/` patterns), emphasize test quality.

2. **simplify-agent** (SMP prefix): Code reuse, quality, and efficiency review. Replaces the hawk/dove/judge triad and efficiency-agent with a single lightweight pass.

3. **breakage-agent** (BRK prefix): Full breakage analysis -- caller analysis, interface changes, import cascades, shared state, config drift, test breakage, regression test recommendations.

4. **tests-agent** (TST prefix): Test coverage and quality review -- existing coverage, proposed test quality, missing scenarios, test design smells.

5. **Precedent discovery** (Haiku agent): For each file in the changes, search the codebase for similar implementations. For each change, grep for similar function names, class names, patterns, imports, and approaches. Build a candidate list pairing each change with existing codebase locations that solve similar problems. Return the candidate list with file paths and brief descriptions of each existing approach.

## Step 4: Precedent Analysis (Sonnet agent)

**Skip conditions**: If Haiku precedent discovery found no candidates, skip this step.

After Step 3 completes, launch the **precedent-agent** agent (PRC prefix). Prepend its input with the scope calibration header (SCOPE_SUMMARY + calibration instruction). Pass it:
- Plan mode: the full plan text and plan file path
- Code mode: the DIFF_TEXT
- The precedent candidate list from the Haiku discovery agent in Step 3

The agent evaluates each candidate pair bidirectionally:
- If the existing approach is more idiomatic or simpler, it recommends the changes adopt it
- If the new approach is better, it recommends refactoring the existing code to match
- It flags style mismatches where changes don't fit naturally with surrounding code

The agent returns findings in PRC-NNN format.

## Step 5: Collection + Deduplication

After all analysis agents complete (from Steps 3 and 4):

1. Collect all findings from all agents into a single list (VFY, SMP, BRK, TST, PRC -- only those that ran)
2. **Deduplicate**: When two agents flag the same underlying issue, merge into one finding keeping the highest confidence and richest evidence, noting which agents flagged it.
3. Compile the merged list with associated plan amendments (plan mode) or recommendations (code mode)

## Step 6: Process Output

**Plan mode**:
1. Read the plan file path from Step 1
2. Collect all deduplicated findings and plan amendments from Step 5
3. Apply ALL amendments (every severity level -- Low through Critical), processing in severity order (Critical first, then High, Medium, Low). If two amendments target the same plan section with incompatible operations, apply the higher-severity amendment and print a note flagging the conflict.
4. Address all findings in the plan (only for agents that ran):
   - Fix correctness issues (from verify-agent)
   - Address reuse, quality, and efficiency concerns (from simplify-agent)
   - Add regression tests (from breakage-agent)
   - Add missing test items (from tests-agent)
   - Adopt existing patterns or add refactoring steps (from precedent-agent)
5. Append a "Level 2 Analysis Notes" section listing all findings with:
   - Finding ID, severity, description
   - Disposition: addressed | merged (with other finding ID)
6. Print summary: total findings by severity, how many addressed/merged, confirmation that the plan was updated

**Code mode**:
1. Collect all findings from all agents
2. Print a findings report grouped by agent prefix (VFY, SMP, BRK, TST, PRC -- only those that ran), then by severity within each group
3. For each finding, print the ID, severity, category, description, evidence, and recommendation
4. Print a summary: total findings by severity across all agents
5. Do NOT edit any files
