# OpenClaw Production Upgrades — Phase 8

## Phase 8: Weekly Memory Compound

Runs Sunday night. Distills the week's daily logs into a concise weekly summary.

### 8.1 `tools/weekly-compound.py`

Core logic:
- Reads all `memory/YYYY-MM-DD.md` files from the past 7 days
- Reads `shared-context/feedback/` for the week
- Reads hourly summaries
- Writes a distilled `memory/weekly/YYYY-WXX.md`
- Optionally updates MEMORY.md with significant learnings

```bash
openclaw cron add \
  --name "weekly-memory-compound" \
  --every "0 22 * * 0" \
  --tz "America/Chicago" \
  --model "anthropic/claude-sonnet-4-6" \
  --task "Run: python3 /path/to/workspace/tools/weekly-compound.py. Then review the output and update MEMORY.md if there are significant learnings worth keeping long-term."
```

---
