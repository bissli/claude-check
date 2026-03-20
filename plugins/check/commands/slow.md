---
description: Comprehensive analysis combining verification, breakage, test review, refactoring judgment (hawk/dove/judge triad), and precedent scanning
---

# Check: Slow Analysis

Run a comprehensive multi-agent analysis with precedent scanning and second-wave validation. In plan mode, updates the plan to address findings. In code mode, reports findings on the diff.

**Output convention**: When launching an agent, print its model in brackets after the description, e.g. `Launching verify-agent... [sonnet]` or `Launching precedent discovery... [haiku]`

## Step 1: Detect Input

1. Check your system prompt for a plan file path (e.g., "A plan file already exists at /path/to/plan.md"). If found: **MODE = plan**, read the plan file. Note the plan file path.
2. If no plan path, run `git diff HEAD` (staged + unstaged). If non-empty: **MODE = code**, DIFF_TEXT = output.
3. If git diff HEAD is empty AND you are not on main/master branch, run `git diff main...HEAD` (or `git diff master...HEAD` if main doesn't exist). If non-empty: **MODE = code**, DIFF_TEXT = output.
4. If nothing found: print "Nothing to check. Enter plan mode or make code changes, then re-run." and **STOP**.

## Step 2: First Wave (parallel agents)

Launch agents in parallel, passing each the full plan text and plan file path (plan mode) or the DIFF_TEXT (code mode).

**Plan mode** -- launch all 7 tasks in parallel:

1. **verify-agent** (VFY prefix): Full verification -- correctness, completeness, edge cases, error handling, assumptions. If the project has tests (glob for `test_*`, `*_test.*`, `*_spec.*`, `tests/`, `__tests__/`, `spec/` patterns), emphasize test quality.

2. **breakage-agent** (BRK prefix): Full breakage analysis -- caller analysis, interface changes, import cascades, shared state, config drift, test breakage, regression test recommendations.

3. **tests-agent** (TST prefix): Test coverage and quality review -- existing coverage, proposed test quality, missing scenarios, test design smells.

4. **refactor-hawk-agent** (RHK prefix): Cross-file architectural analysis -- reuse opportunities, consolidation, interface unification, pattern promotion.

5. **refactor-dove-agent** (RDV prefix): Within-file preservation analysis -- scope containment, risk assessment, stability defense, local improvements only.

6. **database-agent** (DAT prefix): If the plan references or implies a database or data store, check whether adequate data analysis (before/after impact, schema inspection, affected row counts, etc.) has already been performed. If gaps exist, run read-only queries to fill them. If the plan has no database involvement, return no findings.

7. **Precedent discovery** (Haiku agent): For each file in the changes, search the codebase for similar implementations. For each change, grep for similar function names, class names, patterns, imports, and approaches. Build a candidate list pairing each change with existing codebase locations that solve similar problems. Return the candidate list with file paths and brief descriptions of each existing approach.

**Code mode** -- launch 6 tasks in parallel (no database-agent):

1. **verify-agent** (VFY prefix): Same as above.
2. **breakage-agent** (BRK prefix): Same as above.
3. **tests-agent** (TST prefix): Same as above.
4. **refactor-hawk-agent** (RHK prefix): Same as above.
5. **refactor-dove-agent** (RDV prefix): Same as above.
6. **Precedent discovery** (Haiku agent): Same as above, but using files touched in the diff.

## Step 3: Precedent Analysis (Sonnet agent)

After Step 2 completes, launch the **precedent-agent** agent (PRC prefix). Pass it:
- Plan mode: the full plan text and plan file path
- Code mode: the DIFF_TEXT
- The precedent candidate list from the Haiku discovery agent in Step 2

The agent evaluates each candidate pair bidirectionally:
- If the existing approach is more idiomatic or simpler, it recommends the changes adopt it
- If the new approach is better, it recommends refactoring the existing code to match
- It flags style mismatches where changes don't fit naturally with surrounding code

The agent returns findings in PRC-NNN format.

## Step 4: Refactoring Judgment (Sonnet agent)

After BOTH Step 2 (hawk + dove) AND Step 3 (precedent) complete, launch the **refactor-judge-agent** (RFJ prefix). Pass it a single concatenated input block:

**Plan mode:**
```
=== Hawk findings (RHK) ===
<all RHK-NNN findings from Step 2>

=== Dove findings (RDV) ===
<all RDV-NNN findings from Step 2>

=== Precedent findings (PRC) ===
<all PRC-NNN findings from Step 3>

=== Plan ===
<full plan text and plan file path>
```

**Code mode:** Same structure but with DIFF_TEXT instead of plan text.

The judge evaluates hawk, dove, and precedent findings on engineering merits and renders verdicts (adopt-hawk, adopt-hawk-scoped, preserve-and-align, preserve) per finding. It returns findings in RFJ-NNN format with Verdict, Reasoning, and Style note fields.

**Skip logic**: If both hawk and dove produce zero findings, skip the judge entirely. If only one side produces findings, still run the judge -- it evaluates findings against precedent even without counterarguments.

**Truncation mitigation**: If combined input exceeds reasonable size, summarize hawk/dove findings to key claims (ID, severity, one-line description, file paths) rather than passing full text. The judge has tool access and can read referenced files directly.

## Step 5: Collection + Deduplication

After all analysis agents complete (from Steps 2, 3, and 4):

1. Collect all findings from all agents into a single list (VFY, BRK, TST, RHK, RDV, DAT, PRC, RFJ)
2. **Deduplicate**:
   - PRC findings that the judge explicitly addressed in an RFJ verdict are marked "subsumed by RFJ-NNN" -- not merged as duplicates, since the verdict context must be preserved
   - RHK and RDV findings on the same file/section that reach opposite conclusions are NOT deduplicated -- they are opposing arguments the judge already adjudicated; keep both in the internal dedup list for audit purposes (they are suppressed from final output per Step 7), noting the RFJ verdict that resolved them
   - For all other overlaps (two agents flagging the same underlying issue), merge into one finding keeping the highest confidence and richest evidence, noting which agents flagged it. When an RFJ finding participates in a merge, the merged finding must retain the Verdict, Reasoning, and Style note fields from the RFJ source.
   - Pay particular attention to overlap between RFJ (refactoring judgment) and PRC (missed reuse / divergence) findings not already subsumed
3. Compile the merged list with associated plan amendments (plan mode) or recommendations (code mode)

## Step 6: Second Wave (Haiku agents)

For every **Critical** or **High** finding from the deduplicated list, launch a parallel **Haiku** agent (cap at 5 concurrent) that:

1. Receives the finding, the relevant section (plan section or diff context), and supporting context (see mode-specific notes below)
2. Re-examines the issue with full context
3. Returns one of:
   - **Confirmed**: with additional supporting evidence
   - **Downgraded**: with justification and new severity level

**Plan mode**: Two differences from code mode:
1. The Haiku agent receives this scoped instruction: "This finding is about a PLAN for future changes. Evaluate whether the plan adequately addresses this concern. Do NOT check the current codebase for the existence of files or agents that the plan says will be CREATED -- those don't exist yet. Existing codebase files that the plan references for context ARE fair game to check."
2. Instead of passing the specific source file(s) cited in the finding's Evidence field (which may reference files that don't exist yet), pass the relevant plan section(s) as context.

**Code mode**: Pass the specific source file(s) cited in the finding's Evidence field. The Haiku agent re-examines the issue with full file context.

**For RFJ findings** (either mode): The Haiku agent receives the finding including its Verdict field. A downgrade changes the severity but does NOT change the Verdict.

After all second-wave agents complete:
- Update findings with confirmed/downgraded status
- **Filter out** any findings with final confidence < 60

## Step 7: Process Output

**Plan mode**:
1. Read the plan file path from Step 1
2. Collect all confirmed amendments from all agents
3. Apply ALL amendments (every severity level -- Low through Critical), processing in severity order (Critical first, then High, Medium, Low). If two amendments target the same plan section with incompatible operations, apply the higher-severity amendment and print a note flagging the conflict.
4. Address all findings in the plan:
   - Fix correctness issues (from verify-agent)
   - Add regression tests (from breakage-agent)
   - Add missing test items (from tests-agent)
   - Apply refactoring recommendations (from refactor-judge-agent)
   - Adopt existing patterns or add refactoring steps (from precedent-agent)
   - Address data concerns or add data verification steps (from database-agent)
5. Append a "Slow Analysis Notes" section listing all findings with:
   - Finding ID, severity, description
   - Disposition: addressed | merged (with other finding ID) | downgraded (from original severity)
   - What changed in the plan
6. Print summary: total findings by severity, how many addressed/merged/downgraded/filtered, confirmation that the plan was updated

RHK and RDV findings do NOT appear in the final plan mode output -- they are intermediate inputs consumed by the judge. Only RFJ findings are surfaced for refactoring concerns.

**Code mode**:
1. Collect all confirmed findings from all agents
2. Print a findings report grouped by agent prefix (VFY, BRK, TST, RFJ, PRC), then by severity within each group. RHK and RDV findings are NOT listed separately -- they are intermediate inputs to the judge.
3. For each finding, print the ID, severity, category, description, evidence, and recommendation. For RFJ findings, also print the Verdict, Reasoning, and Style note fields.
4. Print a summary: total findings by severity across all agents, how many confirmed/downgraded/filtered
5. Do NOT edit any files
