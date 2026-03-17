---
description: Slow comprehensive analysis combining verification, breakage, test review, simplification, and precedent scanning
---

# Plan Check: Slow Analysis

Run a comprehensive multi-agent analysis with precedent scanning and second-wave validation, then update the plan to address all findings.

**Output convention**: When launching an agent, print its model in brackets after the description, e.g. `Launching plan-verifier... [sonnet]` or `Launching precedent discovery... [haiku]`

## Step 1: First Wave (5 Sonnet agents + Haiku discovery, in parallel)

Find the plan using this priority order:
1. If your system prompt indicates a plan file path (e.g., "A plan file already exists at /path/to/plan.md"), use that path directly.
2. Otherwise, glob `~/.claude/plans/*.md` and read the most recently modified file (use `stat` to compare).
3. If no plan file found on disk, extract plan text from conversation context.

Note the plan file path (or null if from conversation).

Launch **all 6 tasks in parallel**, passing each the full plan text and the plan file path.

1. **plan-verifier** (VFY prefix): Full verification -- correctness, completeness, edge cases, error handling, assumptions. If the project has tests (glob for `test_*`, `*_test.*`, `*_spec.*`, `tests/`, `__tests__/`, `spec/` patterns), emphasize test quality.

2. **breakage-analyst** (BRK prefix): Full breakage analysis -- caller analysis, interface changes, import cascades, shared state, config drift, test breakage, regression test recommendations.

3. **test-reviewer** (TST prefix): Test coverage and quality review -- existing coverage, proposed test quality, missing scenarios, test design smells.

4. **simplification-analyst** (SMP prefix): Over-engineering and reuse -- code reuse opportunities, unnecessary abstractions, pattern conformance, consolidation.

5. **data-analyst** (DAT prefix): If the plan references or implies a database or data store, check whether adequate data analysis (before/after impact, schema inspection, affected row counts, etc.) has already been performed. If gaps exist, run read-only queries to fill them. If the plan has no database involvement, return no findings.

6. **Precedent discovery** (Haiku agent): For each file the plan proposes to create or modify, search the codebase for similar implementations. For each planned change, grep for similar function names, class names, patterns, imports, and approaches. Build a candidate list pairing each planned change with existing codebase locations that solve similar problems. Return the candidate list with file paths and brief descriptions of each existing approach.

## Step 2: Precedent Analysis (Sonnet agent)

After Step 1 completes, launch the **precedent-scanner** agent (PRC prefix). Pass it:
- The full plan text and plan file path
- The precedent candidate list from the Haiku discovery agent in Step 1

The agent evaluates each candidate pair bidirectionally:
- If the existing approach is more idiomatic or simpler, it recommends the plan adopt it
- If the planned approach is better, it recommends refactoring the existing code to match
- It flags style mismatches where planned changes don't fit naturally with surrounding code

The agent returns findings in PRC-NNN format with plan amendments.

## Step 3: Collection + Deduplication

After all 6 analysis agents complete (5 from Step 1 + precedent-scanner from Step 2):

1. Collect all findings from all agents into a single list
2. **Deduplicate**: if two agents flagged the same underlying issue, merge into one finding keeping the highest confidence and richest evidence. Note which agents flagged it. Pay particular attention to overlap between SMP (code reuse) and PRC (missed reuse / divergence) findings.
3. Compile the merged finding list with associated plan amendments

## Step 4: Second Wave (Haiku agents)

For every **Critical** or **High** finding from the deduplicated list, launch a parallel **Haiku** agent (cap at 5 concurrent) that:

1. Receives the finding, the relevant plan section, and the specific source file(s) cited in the finding's Evidence field
2. Re-examines the issue with full file context
3. Returns one of:
   - **Confirmed**: with additional supporting evidence
   - **Downgraded**: with justification and new severity level

After all second-wave agents complete:
- Update findings with confirmed/downgraded status
- **Filter out** any findings with final confidence < 60

## Step 5: Update Plan

1. Read the plan file path from Step 1
2. Collect all confirmed amendments from all agents
3. Apply ALL amendments (every severity level -- Low through Critical), processing in severity order (Critical first, then High, Medium, Low). If two amendments target the same plan section with incompatible operations, apply the higher-severity amendment and print a note flagging the conflict.
4. Address all findings in the plan:
   - Fix correctness issues (from plan-verifier)
   - Add regression tests (from breakage-analyst)
   - Add missing test items (from test-reviewer)
   - Simplify over-engineered steps (from simplification-analyst)
   - Adopt existing patterns or add refactoring steps (from precedent-scanner)
   - Address data concerns or add data verification steps (from data-analyst)
5. Append a "Slow Analysis Notes" section listing all findings with:
   - Finding ID, severity, description
   - Disposition: addressed | merged (with other finding ID) | downgraded (from original severity)
   - What changed in the plan
6. If the plan was from conversation context (no file path), write the amended plan to `~/.claude/plans/amended-plan.md` and report its path
7. Print summary: total findings by severity, how many addressed/merged/downgraded/filtered, confirmation that the plan was updated
