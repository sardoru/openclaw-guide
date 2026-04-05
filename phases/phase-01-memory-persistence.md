# OpenClaw Production Upgrades — Phase 1

## Phase 1: Memory & Persistence

Vanilla OpenClaw has no memory between sessions. Each conversation starts from zero. These upgrades fix that.

### 1.1 Workspace Structure

Create the core directory structure in your OpenClaw workspace (the directory set in your agent config):

```bash
mkdir -p memory/hourly memory/weekly memory/ingested memory/retros memory/trace-analysis
mkdir -p shared-context/{agent-outputs/general,feedback,kpis,calendar,content-calendar,roundtable}
mkdir -p tools/{vector-memory,forge,brain-ingest}
mkdir -p decisions
```

### 1.2 AGENTS.md — Session Boot Sequence

Create `AGENTS.md` in your workspace root. This is the first file your agent reads every session. It tells the agent *how to wake up*:

```markdown
# AGENTS.md

## Every Session

Before doing anything else:

1. Read `SOUL.md` — this is who you are
2. Read `USER.md` — this is who you're helping
3. Read `memory/YYYY-MM-DD.md` (today + yesterday) for recent context
4. Read `CONTEXT_INJECTION.md` if it exists — compiled context from last compaction
5. Read `shared-context/priorities.md` — current priority stack
6. Read `active-tasks.md` — resume any in-flight work from crashed/restarted sessions

Don't ask permission. Just do it.

## Memory

You wake up fresh each session. These files are your continuity:

- **Daily notes:** `memory/YYYY-MM-DD.md` — raw logs of what happened
- **Long-term:** `MEMORY.md` — your curated memories

Capture what matters. Decisions, context, things to remember.

### Write It Down — No "Mental Notes"!

Memory is limited. If you want to remember something, WRITE IT TO A FILE.
"Mental notes" don't survive session restarts. Files do.

## Safety

- Don't exfiltrate private data
- `trash` > `rm` (recoverable beats gone)
- When in doubt, ask
```

### 1.3 SOUL.md — Agent Identity

```markdown
# SOUL.md

Be genuinely helpful, not performatively helpful. Skip "Great question!" — just help.
Have opinions. Be resourceful before asking. Earn trust through competence.
Remember you're a guest — you have access to someone's life. Treat it with respect.

Concise when needed, thorough when it matters.
```

### 1.4 USER.md — Human Profile

```markdown
# USER.md

- **Name:** [Your name]
- **Timezone:** [e.g., America/Chicago]
- **Notes:** [Preferences, projects, communication style]
```

### 1.5 MEMORY.md — Long-Term Memory Index

Create `MEMORY.md` as an index pointing to subfiles. Don't dump everything in one file — keep it structured:

```markdown
# Long-Term Memory — Index

This file is an **index**. Details live in subfiles under `memory/`.
Load only what you need for the current task.

## Subfiles

| File | Contents |
|------|----------|
| `memory/user-profile.md` | Bio, contacts, preferences |
| `memory/infrastructure.md` | Tools, APIs, crons, cost optimization |
| `memory/projects.md` | Active projects |
| `memory/lessons-learned.md` | Hard-won lessons, gotchas |
| `memory/YYYY-MM-DD.md` | Daily logs (raw session notes) |

## Quick Reference

- **User:** [Name]
- **Timezone:** [TZ]
- **Style:** [Communication preferences]
```

### 1.6 Daily Memory Logs

Your agent should write to `memory/YYYY-MM-DD.md` during every session. Add this instruction to AGENTS.md:

> At the end of every significant interaction, append notes to today's `memory/YYYY-MM-DD.md`. Include: what was discussed, decisions made, tasks completed, and anything to remember.

### 1.7 Crash Recovery — active-tasks.md

Create `active-tasks.md` in workspace root:

```markdown
# Active Tasks

Track in-flight work here. If a session crashes, the next session reads this to resume.

## Format
- **Task:** [description]
- **Status:** [in-progress | blocked | waiting]
- **Context:** [what was happening when interrupted]
- **Next step:** [what to do to resume]
```

Add to AGENTS.md: "On startup, read `active-tasks.md`. If there's in-flight work, resume it."

### 1.8 Auto-Stub for Missing Daily Logs

Some days you won't talk to your agent. This script creates stub logs so there are no gaps:

**`tools/ensure-daily-log.sh`:**

```bash
#!/bin/bash
# Ensure a daily memory log exists. Run nightly from heartbeat or cron.

DATE="${1:-$(date +%Y-%m-%d)}"
WORKSPACE="/path/to/your/workspace"  # ← Change this
LOG_FILE="$WORKSPACE/memory/${DATE}.md"

if [ -f "$LOG_FILE" ]; then
    echo "✓ Daily log already exists: $LOG_FILE"
    exit 0
fi

echo "⚠ No daily log for $DATE — creating stub..."
{
    echo "# ${DATE} (auto-generated stub)"
    echo ""
    echo "_No main-session activity recorded._"
    echo ""
    echo "## Notes"
    echo "- (no activity recorded)"
} > "$LOG_FILE"
echo "✓ Created stub: $LOG_FILE"
```

```bash
chmod +x tools/ensure-daily-log.sh
```

---
