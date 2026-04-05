# OpenClaw Production Upgrades — Phase 10

## Phase 10: Heartbeat System

Heartbeats are periodic check-ins (default every 30 min). Instead of just replying "HEARTBEAT_OK", make them productive.

### 10.1 HEARTBEAT.md

```markdown
# HEARTBEAT.md

## Vector Memory Maintenance
- If memory files were updated since last check, run vector memory reindex
- Track in memory/heartbeat-state.json under "lastVectorReindex"

## Daily Memory Log Stub
- After 10pm, check if memory/YYYY-MM-DD.md exists for today
- If not, run: bash tools/ensure-daily-log.sh
- Also backfill yesterday if missing

## Decision Review Check
- Weekly, scan decisions/*.md for any past review_date
- If a decision's review_date has passed, flag it for user review

## Daily Summary
- Once per day, between 9pm-midnight, write today's summary to:
  shared-context/agent-outputs/general/YYYY-MM-DD-daily-summary.md
```

### 10.2 heartbeat-state.json

Track what's been checked to avoid redundant work:

```json
{
  "lastChecks": {},
  "lastVectorReindex": 0,
  "lastDailySummary": 0,
  "lastDailyLogCheck": 0,
  "lastDecisionReview": 0
}
```

---
