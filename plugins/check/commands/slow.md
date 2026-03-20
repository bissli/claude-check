---
description: Comprehensive analysis combining verification, breakage, test review, simplification, and precedent scanning
---

# Check: Slow Analysis

Run a comprehensive multi-agent analysis with precedent scanning and second-wave validation. In plan mode, updates the plan to address findings. In code mode, reports findings on the diff.

**Output convention**: When launching an agent, print its model in brackets after the description, e.g. `Launching verifier... [sonnet]` or `Launching precedent discovery... [haiku]`

## Step 1: Detect Input

1. Check your system prompt for a plan file path (e.g., "A plan file already exists at /path/to/plan.md"). If found: **MODE = plan**, read the plan file. Note the plan file path.
2. If no plan path, run `git diff HEAD` (staged + unstaged). If non-empty: **MODE = code**, DIFF_TEXT = output.
3. If git diff HEAD is empty AND you are not on main/master branch, run `git diff main...HEAD` (or `git diff master...HEAD` if main doesn't exist). If non-empty: **MODE = code**, DIFF_TEXT = output.
4. If nothing found: print "Nothing to check. Enter plan mode or make code changes, then re-run." and **STOP**.

## Step 2: First Wave (parallel agents)

Launch agents in parallel, passing each the full plan text and plan file path (plan mode) or the DIFF_TEXT (code mode).

**Plan mode** -- launch all 7 tasks in parallel:

1. **verifier** (VFY prefix): Full verification -- correctness, completeness, edge cases, error handling, assumptions. If the project has tests (glob for `test_*`, `*_test.*`, `*_spec.*`, `tests/`, `__tests__/`, `spec/` patterns), emphasize test quality.

2. **breakage-analyst** (BRK prefix): Full breakage analysis -- caller analysis, interface changes, import cascades, shared state, config drift, test breakage, regression test recommendations.

3. **test-reviewer** (TST prefix): Test coverage and quality review -- existing coverage, proposed test quality, missing scenarios, test design smells.

4. **simplification-analyst** (SMP prefix): Over-engineering and reuse -- code reuse opportunities, unnecessary abstractions, pattern conformance, consolidation.

5. **data-analyst** (DAT prefix): If the plan references or implies a database or data store, check whether adequate data analysis (before/after impact, schema inspection, affected row counts, etc.) has already been performed. If gaps exist, run read-only queries to fill them. If the plan has no database involvement, return no findings.

6. **Precedent discovery** (Haiku agent): For each file in the changes, search the codebase for similar implementations. For each change, grep for similar function names, class names, patterns, imports, and approaches. Build a candidate list pairing each change with existing codebase locations that solve similar problems. Return the candidate list with file paths and brief descriptions of each existing approach.

**Code mode** -- launch 5 tasks in parallel (no data-analyst):

1. **verifier** (VFY prefix): Same as above.
2. **breakage-analyst** (BRK prefix): Same as above.
3. **test-reviewer** (TST prefix): Same as above.
4. **simplification-analyst** (SMP prefix): Same as above.
5. **Precedent discovery** (Haiku agent): Same as above, but using files touched in the diff.

## Step 3: Precedent Analysis (Sonnet agent)

After Step 2 completes, launch the **precedent-scanner** agent (PRC prefix). Pass it:
- Plan mode: the full plan text and plan file path
- Code mode: the DIFF_TEXT
- The precedent candidate list from the Haiku discovery agent in Step 2

The agent evaluates each candidate pair bidirectionally:
- If the existing approach is more idiomatic or simpler, it recommends the changes adopt it
- If the new approach is better, it recommends refactoring the existing code to match
- It flags style mismatches where changes don't fit naturally with surrounding code

The agent returns findings in PRC-NNN format.

## Step 4: Collection + Deduplication

After all analysis agents complete (from Steps 2 + 3):

1. Collect all findings from all agents into a single list
2. **Deduplicate**: if two agents flagged the same underlying issue, merge into one finding keeping the highest confidence and richest evidence. Note which agents flagged it. Pay particular attention to overlap between SMP (code reuse) and PRC (missed reuse / divergence) findings.
3. Compile the merged finding list with associated plan amendments (plan mode) or recommendations (code mode)

## Step 5: Second Wave (Haiku agents)

For every **Critical** or **High** finding from the deduplicated list, launch a parallel **Haiku** agent (cap at 5 concurrent) that:

1. Receives the finding, the relevant section (plan section or diff context), and the specific source file(s) cited in the finding's Evidence field
2. Re-examines the issue with full file context
3. Returns one of:
   - **Confirmed**: with additional supporting evidence
   - **Downgraded**: with justification and new severity level

After all second-wave agents complete:
- Update findings with confirmed/downgraded status
- **Filter out** any findings with final confidence < 60

## Step 6: Process Output

**Plan mode**:
1. Read the plan file path from Step 1
2. Collect all confirmed amendments from all agents
3. Apply ALL amendments (every severity level -- Low through Critical), processing in severity order (Critical first, then High, Medium, Low). If two amendments target the same plan section with incompatible operations, apply the higher-severity amendment and print a note flagging the conflict.
4. Address all findings in the plan:
   - Fix correctness issues (from verifier)
   - Add regression tests (from breakage-analyst)
   - Add missing test items (from test-reviewer)
   - Simplify over-engineered steps (from simplification-analyst)
   - Adopt existing patterns or add refactoring steps (from precedent-scanner)
   - Address data concerns or add data verification steps (from data-analyst)
5. Append a "Slow Analysis Notes" section listing all findings with:
   - Finding ID, severity, description
   - Disposition: addressed | merged (with other finding ID) | downgraded (from original severity)
   - What changed in the plan
6. Print summary: total findings by severity, how many addressed/merged/downgraded/filtered, confirmation that the plan was updated

**Code mode**:
1. Collect all confirmed findings from all agents
2. Print a findings report grouped by agent prefix (VFY, BRK, TST, SMP, PRC), then by severity within each group
3. For each finding, print the ID, severity, category, description, evidence, and recommendation
4. Print a summary: total findings by severity across all agents, how many confirmed/downgraded/filtered
5. Do NOT edit any files
