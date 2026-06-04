# Performance Optimization

## Model Selection Strategy

**Haiku 4.5** (90% of Sonnet capability, 3x cost savings):
- Lightweight agents with frequent invocation
- Pair programming and code generation
- Worker agents in multi-agent systems

**Sonnet 4.6** (Best coding model):
- Main development work
- Orchestrating multi-agent workflows
- Complex coding tasks

**Opus 4.5** (Deepest reasoning):
- Complex architectural decisions
- Maximum reasoning requirements
- Research and analysis tasks

## Context Window Management

Avoid last 20% of context window for:
- Large-scale refactoring
- Feature implementation spanning multiple files
- Debugging complex interactions

Lower context sensitivity tasks:
- Single-file edits
- Independent utility creation
- Documentation updates
- Simple bug fixes

## Extended Thinking + Plan Mode

Extended thinking is enabled by default, reserving up to 31,999 tokens for internal reasoning.

Control extended thinking via:
- **Toggle**: Option+T (macOS) / Alt+T (Windows/Linux)
- **Config**: Set `alwaysThinkingEnabled` in `~/.claude/settings.json`
- **Budget cap**: `export MAX_THINKING_TOKENS=10000`
- **Verbose mode**: Ctrl+O to see thinking output

For complex tasks requiring deep reasoning:
1. Ensure extended thinking is enabled (on by default)
2. Enable **Plan Mode** for structured approach
3. Use multiple critique rounds for thorough analysis
4. Use split role sub-agents for diverse perspectives

## Build Troubleshooting

If build fails:
1. Use **build-error-resolver** agent
2. Analyze error messages
3. Fix incrementally
4. Verify after each fix

## Reasoning Effort Calibration (proactive flagging)

Regardless of the session's current effort / extended-thinking setting, proactively flag
when a turn's reasoning demand and the configured effort are mismatched. The user should
not have to detect this alone — meet them halfway.

**Heuristic:** effort ≈ *blast-radius-of-being-wrong* × *ambiguity-of-the-answer*. Flag the
mismatch in one line; let the user decide whether to change the setting.

- **Raise a flag to bump UP** before high-stakes turns: decisions that propagate across many
  files (domain models, architecture, security boundaries, API contracts), genuine tradeoffs
  with no obvious default, synthesis across many inputs, novel/unfamiliar territory, or
  non-localized debugging. Example: *"This is an architecture-defining turn — consider
  raising effort to high before I proceed."*
- **Note overkill (drop DOWN)** when a sustained high setting is being spent on low-blast
  mechanical execution (file edits, scaffolding, formatting, git, docs) — say it can safely
  drop to medium.
- **Escalate reactively** when an answer on a decision that matters reads thin: options
  without a recommendation, a missed stated constraint, or plausible-but-unverified output.
  Re-running a turn with more effort is cheap because the conversation context is reused.

Orchestration note: when the deep reasoning is delegated to subagents (parallel research,
adversarial verification), the main loop can run leaner — flag this so effort is spent where
it actually changes the outcome.
