---
description: Deep analysis combining verification, breakage, test review, and simplification
---

# Plan Check: Deep Analysis

Run a comprehensive 4-agent analysis with second-wave validation, then update the plan to address all findings.

<!-- Shared step: keep in sync across all plan-check commands -->
## Step 1: Context Gathering (Haiku agent)

Launch a **Haiku** agent to gather project context. The agent should:

1. Find the plan:
   - Glob `~/.claude/plans/*.md` and read the most recently modified file (use `stat` to compare)
   - If no plan file found on disk, extract plan text from conversation context
2. From the plan, list all source files it references (paths mentioned in the plan text)

Return: the full plan text, **the plan file path** (or null if from conversation context), a one-paragraph plan summary, and the list of referenced source files.

<!-- Shared step: keep in sync across all plan-check commands -->
## Step 2: Deep Scope Analysis (Haiku agent)

Launch a **Haiku** agent with the plan text and summary from Step 1. This is the most thorough scope scan. The agent should:

1. List every file the plan proposes to **create**, **modify**, or **delete** (the file manifest)
2. For each file that already exists, read its current contents
3. For each modified file, grep the codebase for callers and importers (search for the filename, exported function names, class names). Build a caller map.
4. Detect whether the project has tests: glob for `test_*`, `*_test.*`, `*_spec.*`, `tests/`, `__tests__/`, `spec/` patterns. Read ALL test files in the project (not just those covering modified code).
5. Identify existing patterns and conventions: scan existing source files for naming, structure, and architectural patterns.

Return: the file manifest, contents of ALL existing files referenced or related, caller map for every modified file, ALL test file contents, `has_tests` boolean, and detected patterns/conventions.

## Step 3: First Wave (4 Sonnet agents in parallel)

Launch **all 4 agents in parallel**. Pass each agent the outputs from Steps 1 and 2 as context.

1. **plan-verifier** (VFY prefix): Full verification -- correctness, completeness, edge cases, error handling, assumptions. If `has_tests`, emphasize test quality.

2. **breakage-analyst** (BRK prefix): Full breakage analysis -- caller analysis, interface changes, import cascades, shared state, config drift, test breakage, regression test recommendations.

3. **test-reviewer** (TST prefix): Test coverage and quality review -- existing coverage, proposed test quality, missing scenarios, test design smells.

4. **simplification-analyst** (SMP prefix): Over-engineering and reuse -- code reuse opportunities, unnecessary abstractions, pattern conformance, consolidation.

All agents return findings in their PREFIX-NNN format with plan amendments.

## Step 4: Collection + Deduplication

After all 4 first-wave agents complete:

1. Collect all findings from all agents into a single list
2. **Deduplicate**: if two agents flagged the same underlying issue, merge into one finding keeping the highest confidence and richest evidence. Note which agents flagged it.
3. Compile the merged finding list with associated plan amendments

## Step 5: Second Wave (Haiku agents)

For every **Critical** or **High** finding from the deduplicated list, launch a parallel **Haiku** agent (cap at 5 concurrent) that:

1. Receives the finding, the relevant plan section, and the specific source file(s) cited in the finding's Evidence field
2. Re-examines the issue with full file context
3. Returns one of:
   - **Confirmed**: with additional supporting evidence
   - **Downgraded**: with justification and new severity level

After all second-wave agents complete:
- Update findings with confirmed/downgraded status
- **Filter out** any findings with final confidence < 60

## Step 6: Update Plan

1. Read the plan file path from Step 1
2. Collect all confirmed amendments from all agents
3. Apply ALL amendments (every severity level -- Low through Critical), processing in severity order (Critical first, then High, Medium, Low). If two amendments target the same plan section with incompatible operations, apply the higher-severity amendment and print a note flagging the conflict.
4. Address all findings in the plan:
   - Fix correctness issues (from plan-verifier)
   - Add regression tests (from breakage-analyst)
   - Add missing test items (from test-reviewer)
   - Simplify over-engineered steps (from simplification-analyst)
5. Append a "Deep Analysis Notes" section listing all findings with:
   - Finding ID, severity, description
   - Disposition: addressed | merged (with other finding ID) | downgraded (from original severity)
   - What changed in the plan
6. If the plan was from conversation context (no file path), write the amended plan to `~/.claude/plans/amended-plan.md` and report its path
7. Print summary: total findings by severity, how many addressed/merged/downgraded/filtered, confirmation that the plan was updated
