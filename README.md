# claude-check

A Claude Code plugin that analyzes implementation plans or code changes using parallel specialized agents.

## Modes

Auto-detected based on context:

- **Plan mode**: system prompt contains a plan file path (plan mode injects this) -- agents analyze the plan and edit it directly to address findings
- **Code mode**: no plan in system prompt, but git diff has changes -- agents analyze the diff and report findings

Detection order:
1. Plan file path in system prompt → plan mode
2. `git diff HEAD` has changes → code mode
3. `git diff main...HEAD` (branch commits) → code mode
4. Nothing found → error, stop

## Installation

Add the marketplace and install:

```bash
claude plugin marketplace add bissli/claude-check \
  && claude plugin install check@claude-check
```

For local development:

```
claude --plugin-dir ./claude-check
```

## Usage

Two subcommands are available:

```
/check:fast       # Fast check: correctness, completeness, assumptions
/check:slow       # Slow analysis: all agents + precedent scanning
```

## How It Works

Every command follows the same pattern:

1. **Detect input** -- auto-detect plan mode or code mode
2. **Agent analysis** (Sonnet) -- specialized agents receive the plan text or diff and read source files directly as needed, returning structured findings
3. **Process output** -- plan mode: apply all amendments to the plan file; code mode: print findings report

### Commands

**`/check:fast`** launches the verify agent for correctness, completeness, edge cases, error handling, assumptions, and test quality. In plan mode, updates the plan. In code mode, prints findings.

**`/check:slow`** is the most thorough analysis. A Haiku pre-classifier first determines scope: trivial changes (targeted fixes, renames, minor additions) get verify-agent, breakage-agent, and precedent discovery only; non-trivial changes get the full suite. The full suite launches 6 Sonnet agents in plan mode (verify-agent, breakage-agent, tests-agent, refactor-hawk-agent, refactor-dove-agent, database-agent) or 5 in code mode (no database-agent), plus Haiku precedent discovery. The precedent candidates feed into a Sonnet precedent-agent that evaluates pattern divergence. Then the refactor-judge-agent receives hawk, dove, and precedent findings to render verdicts on each refactoring proposal. After deduplication, a second wave of Haiku agents re-evaluates Critical and High findings. In plan mode, all confirmed amendments are applied to the plan. In code mode, findings are printed as a report.

## Agents

| Agent                    | Prefix | Color  | Focus                                                                              |
| ------------------------ | ------ | ------ | ---------------------------------------------------------------------------------- |
| **verify-agent**         | VFY    | yellow | Correctness, completeness, edge cases, assumptions, test quality, over-engineering |
| **breakage-agent**       | BRK    | red    | Caller breakage, interface changes, import cascades, test breakage                 |
| **tests-agent**          | TST    | cyan   | Test coverage, proposed test quality, missing scenarios, smells                    |
| **refactor-hawk-agent**  | RHK    | orange | Cross-file architectural refactoring, consolidation, pattern promotion             |
| **refactor-dove-agent**  | RDV    | pink   | Within-file preservation, scope containment, risk assessment                       |
| **refactor-judge-agent** | RFJ    | purple | Arbitrates hawk/dove/precedent, renders style evolution verdicts                   |
| **precedent-agent**      | PRC    | blue   | Codebase precedent, approach divergence, bidirectional improvement                 |
| **database-agent**       | DAT    | green  | Database impact, schema concerns, data integrity (plan mode only)                  |

## Plan Amendment Model

In plan mode, agents are read-only -- they analyze the codebase and return structured findings with plan amendments. The orchestrating command (the main Claude thread) collects amendments and edits the plan file.

Each amendment specifies an operation (`add`, `replace`, `remove`, or `append-section`), a target location in the plan, and the content to apply. Amendments are processed in severity order (Critical first). Conflicts between amendments targeting the same section are resolved by applying the higher-severity amendment.

In code mode, no amendments are produced. Each finding's Recommendation field is the actionable output.

## Scope-Aware Calibration

All agents receive a scope context header (git diff summary or plan size metrics) and are instructed to calibrate analysis depth accordingly. Small changes get proportionally focused analysis; agents return no findings when the change is outside their domain.

`/check:slow` adds a Haiku pre-classifier gate before launching agents. It classifies the change as TRIVIAL or NON-TRIVIAL. Trivial changes (targeted fixes, renames, minor additions) get verify-agent, breakage-agent, and precedent discovery only. Non-trivial changes get the full agent suite. Defaults to non-trivial when uncertain.

## Confidence Threshold

In `/check:slow`, the second wave of Haiku agents re-evaluates Critical and High findings. Findings below 60% confidence are filtered out after the second wave.

## Uninstallation

```bash
claude plugin uninstall check@claude-check \
  && claude plugin marketplace remove claude-check \
  && rm -rf ~/.claude/plugins/cache/claude-check
```

## License

MIT
