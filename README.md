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

**`/check:slow`** is the most thorough analysis. In plan mode, it launches 5 Sonnet agents in parallel (verify-agent, breakage-agent, tests-agent, simplify-agent, database-agent) alongside a Haiku precedent discovery pass. In code mode, it launches 4 Sonnet agents (no database-agent) plus Haiku discovery. The precedent candidates then feed into a Sonnet precedent-agent that evaluates whether changes diverge from existing codebase patterns. After deduplication, a second wave of Haiku agents re-evaluates Critical and High findings. In plan mode, all confirmed amendments are applied to the plan. In code mode, findings are printed as a report.

## Agents

| Agent               | Prefix | Color  | Focus                                                              |
| ------------------- | ------ | ------ | ------------------------------------------------------------------ |
| **verify-agent**    | VFY    | yellow | Correctness, completeness, edge cases, assumptions, test quality   |
| **breakage-agent**  | BRK    | red    | Caller breakage, interface changes, import cascades, test breakage |
| **tests-agent**     | TST    | cyan   | Test coverage, proposed test quality, missing scenarios, smells    |
| **simplify-agent**  | SMP    | purple | Code reuse, over-engineering, pattern conformance, consolidation   |
| **precedent-agent** | PRC    | blue   | Codebase precedent, approach divergence, bidirectional improvement |
| **database-agent**  | DAT    | green  | Database impact, schema concerns, data integrity (plan mode only)  |

## Plan Amendment Model

In plan mode, agents are read-only -- they analyze the codebase and return structured findings with plan amendments. The orchestrating command (the main Claude thread) collects amendments and edits the plan file.

Each amendment specifies an operation (`add`, `replace`, `remove`, or `append-section`), a target location in the plan, and the content to apply. Amendments are processed in severity order (Critical first). Conflicts between amendments targeting the same section are resolved by applying the higher-severity amendment.

In code mode, no amendments are produced. Each finding's Recommendation field is the actionable output.

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
