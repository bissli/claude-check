# claude-plan-check

A Claude Code plugin that validates implementation plans using parallel specialized agents and **edits the plan directly** to address all findings.

## Installation

Add the marketplace and install:

```
/plugin marketplace add bissli/claude-plan-check
/plugin install plan-check@claude-plan-check
```

For local development:

```
claude --plugin-dir ./claude-plan-check
```

## Usage

Two subcommands are available:

```
/plan-check:fast       # Fast check: correctness, completeness, assumptions
/plan-check:slow       # Slow analysis: all 5 agents + precedent scanning
```

Reads the most recent plan from `~/.claude/plans/`. If no plan file is found, falls back to extracting plan text from conversation context. When the plan comes from conversation context, the amended plan is written to `~/.claude/plans/amended-plan.md`.

## How It Works

Every command follows the same pattern:

1. **Find the plan** -- the parent session locates the most recent plan file (or extracts from conversation)
2. **Agent analysis** (Sonnet) -- specialized agents receive the plan text and read source files directly as needed, returning findings with plan amendments
3. **Update plan** -- the orchestrating command applies all amendments directly to the plan file

The updated plan IS the deliverable. No standalone report is produced.

### Commands

**`/plan-check:fast`** launches the plan-verifier agent for correctness, completeness, edge cases, error handling, assumptions, and test quality.

**`/plan-check:slow`** is the most thorough analysis. It launches 4 Sonnet agents in parallel (plan-verifier, breakage-analyst, test-reviewer, simplification-analyst) alongside a Haiku precedent discovery pass. The precedent candidates then feed into a Sonnet precedent-scanner that evaluates whether planned changes diverge from existing codebase patterns -- recommending the plan adopt existing approaches or refactor existing code to match better planned approaches. After deduplication, a second wave of Haiku agents re-evaluates Critical and High findings. All confirmed amendments are applied to the plan.

## Agents

| Agent                      | Prefix | Color   | Focus                                                              |
| -------------------------- | ------ | ------- | ------------------------------------------------------------------ |
| **plan-verifier**          | VFY    | yellow  | Correctness, completeness, edge cases, assumptions, test quality   |
| **breakage-analyst**       | BRK    | red     | Caller breakage, interface changes, import cascades, test breakage |
| **test-reviewer**          | TST    | cyan    | Test coverage, proposed test quality, missing scenarios, smells    |
| **simplification-analyst** | SMP    | magenta | Code reuse, over-engineering, pattern conformance, consolidation   |
| **precedent-scanner**      | PRC    | blue    | Codebase precedent, approach divergence, bidirectional improvement |

## Plan Amendment Model

Agents are read-only -- they analyze the codebase and return structured findings with plan amendments. The orchestrating command (the main Claude thread) collects amendments and edits the plan file.

Each amendment specifies an operation (`add`, `replace`, `remove`, or `append-section`), a target location in the plan, and the content to apply. Amendments are processed in severity order (Critical first). Conflicts between amendments targeting the same section are resolved by applying the higher-severity amendment.

## Confidence Threshold

In `/plan-check:slow`, the second wave of Haiku agents re-evaluates Critical and High findings. Findings below 60% confidence are filtered out after the second wave.

## Migration from v1

| v1 Command           | v2 Equivalent      |
| -------------------- | ------------------ |
| `/plan-check:all`    | `/plan-check:slow` |
| `/plan-check:review` | `/plan-check:slow` |

Key differences in v2:
- Commands edit the plan file directly instead of producing standalone reports
- All severity levels are addressed (Low through Critical), not just reported
- New agents: plan-verifier (VFY), breakage-analyst (BRK), test-reviewer (TST), simplification-analyst (SMP)
- Removed agents: gap-analyst (GAP), code-impact-analyst (IMP), feasibility-risk-analyst (RSK)

## Uninstallation

To fully remove the plugin:

1. Disable the plugin in Claude Code:
   ```
   /plugin uninstall plan-check@claude-plan-check
   ```

2. Remove the marketplace registration:
   ```
   /plugin marketplace remove claude-plan-check
   ```

3. Delete the cached plugin files:
   ```bash
   rm -rf ~/.claude/plugins/cache/claude-plan-check
   ```

4. Verify removal -- open `~/.claude/settings.json` and confirm:
   - `"plan-check@claude-plan-check"` is gone from `enabledPlugins`
   - `"claude-plan-check"` is gone from `extraKnownMarketplaces`

## License

MIT
