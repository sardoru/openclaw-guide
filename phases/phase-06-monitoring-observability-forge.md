# OpenClaw Production Upgrades — Phase 6

## Phase 6: Monitoring & Observability (Forge)

Forge is a meta-agent that monitors the entire system. It catches failures, diagnoses root causes, and applies fixes automatically.

### 6.1 Forge Daily Review

Create `tools/forge/daily-review.md` — this is the prompt/instructions for the daily review cron:

```markdown
# Forge Daily Review — Agent Instructions

You are Forge, the meta-agent responsible for continuous improvement.

## Process

1. **Discover Sessions:** `sessions_list(kinds=["isolated"], activeMinutes=1440, limit=50, messageLimit=2)` — all sub-agent/cron sessions from last 24h
2. **Identify Failures:** Scan for error keywords, empty outputs, long runtimes, doom loops
3. **Fetch Details:** `sessions_history(sessionKey, includeTools=true, limit=30)` for each failure
4. **Categorize:** tool_failure | reasoning_error | instruction_drift | doom_loop | timeout | missing_context
5. **Check Cron Health:** `openclaw cron list --json` — look for consecutiveErrors > 0
6. **Write Report:** Save to `memory/trace-analysis/YYYY-MM-DD.md`
7. **Update Pattern Database:** Append new patterns to `memory/trace-analysis/pattern-database.md`
8. **Notify if Critical:** Send summary via Telegram only if critical failures found

## Rules
- NEVER modify SOUL.md, USER.md, or MEMORY.md
- NEVER delete files
- Be conservative: only auto-apply fixes you're confident about
- Log everything — even "no issues found" days are data
- Same failure 3+ times → escalate to human
```

```bash
openclaw cron add \
  --name "forge-daily-review" \
  --every "30 6 * * *" \
  --tz "America/Chicago" \
  --model "anthropic/claude-sonnet-4-6" \
  --task "Read tools/forge/daily-review.md and follow its instructions exactly."
```

### 6.2 Forge Weekly Synthesis

Create `tools/forge/weekly-synthesis.md`:

```markdown
# Forge Weekly Synthesis

Read all memory/trace-analysis/YYYY-MM-DD.md from the past 7 days.
Read memory/trace-analysis/pattern-database.md.
Read shared-context/feedback/ for human feedback.

Compile:
- Total failures by category
- Failure trend (improving or degrading?)
- Agent-specific performance
- Capability gaps
- Recommendations

Write to memory/trace-analysis/weekly-YYYY-WXX.md.
Send summary to user via Telegram.
```

```bash
openclaw cron add \
  --name "forge-weekly-synthesis" \
  --every "0 21 * * 0" \
  --tz "America/Chicago" \
  --model "anthropic/claude-sonnet-4-6" \
  --task "Read tools/forge/weekly-synthesis.md and follow its instructions exactly."
```

### 6.3 Pattern Database

Create `memory/trace-analysis/pattern-database.md`:

```markdown
# Failure Pattern Database

Maintained by Forge. Each pattern includes occurrence count and resolution status.

## Known Patterns

(Forge will populate this automatically as it discovers patterns)
```

---
