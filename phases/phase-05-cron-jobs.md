# OpenClaw Production Upgrades — Phase 5

## Phase 5: Cron Jobs

This is where the system comes alive. Cron jobs are OpenClaw's native scheduled tasks — they run as isolated sessions on a schedule.

### 5.1 Core Crons

Set these up via `openclaw cron add`:

```bash
# Hourly memory summarizer (keeps context fresh)
openclaw cron add \
  --name "hourly-memory-summarizer" \
  --every "55 * * * *" \
  --model "anthropic/claude-sonnet-4-6" \
  --task "Run: python3 /path/to/workspace/tools/hourly-summarizer.py && python3 /path/to/workspace/tools/context-injector.py"

# Vector memory reindex (every 4 hours)
openclaw cron add \
  --name "vector-memory-reindex" \
  --every "0 */4 * * *" \
  --model "anthropic/claude-sonnet-4-6" \
  --task "Run: cd /path/to/workspace/tools/vector-memory && source venv/bin/activate && python3 ingest.py"

# Daily roundtable (9 PM, cross-agent synthesis)
openclaw cron add \
  --name "daily-roundtable" \
  --every "0 21 * * *" \
  --tz "America/Chicago" \
  --model "anthropic/claude-sonnet-4-6" \
  --task "Run: python3 /path/to/workspace/tools/roundtable.py"

# Weekly memory compound (Sunday 10 PM)
openclaw cron add \
  --name "weekly-memory-compound" \
  --every "0 22 * * 0" \
  --tz "America/Chicago" \
  --model "anthropic/claude-sonnet-4-6" \
  --task "Run: python3 /path/to/workspace/tools/weekly-compound.py"
```

### 5.2 Morning Brief (optional but powerful)

A daily briefing cron that researches and delivers a personalized morning update:

```bash
openclaw cron add \
  --name "morning-brief" \
  --every "0 7 * * *" \
  --tz "America/Chicago" \
  --model "anthropic/claude-sonnet-4-6" \
  --task "Research and deliver a morning brief covering: [your topics — crypto, markets, geopolitics, AI news, etc.]. Search the web for the latest. Send the brief to the user via Telegram."
```

### 5.3 Cost Optimization for Crons

**Critical:** Route all background crons to a cheaper model. Direct conversations use your best model (e.g., Opus), but crons should use Sonnet:

- Set `payload.model` to `anthropic/claude-sonnet-4-6` on every cron
- Set `heartbeat.model` to `anthropic/claude-sonnet-4-6` in config
- Enable prompt caching: `cacheControlTtl: "1h"` in config
- **Estimated savings: 40-60% on background costs**

---
