# OpenClaw Production Upgrades — Phase 16

## Phase 16: Canonical Task List, Daily Prep & Email Sweep

Inspired by Ryan Carson's clawchief system (4,800+ bookmarks) — the insight that transformed his OpenClaw from a reactive chatbot into a chief of staff: *"I didn't get the world's best assistant by asking better questions. I got it by giving it a better operating system."*

This phase adds three things: a single source of truth for tasks, a cron that preps your day before you wake up, and an email triage system.

### 16.1 Canonical Task List (`tasks/current.md`)

Stop scattering tasks across daily logs, Obsidian, chat history, and your head. One file. Four sections.

```markdown
# Tasks — Single Source of Truth

## 🔴 Today
- [ ] Review Exchange Building lease renewal
- [ ] Test AIO calendar sync with Google Calendar
- [ ] Reply to Murray Wells re: contractor walkthrough

## 📅 This Week
- [ ] VerdictAI: Define product scope with Arslan
- [ ] Set up GOG for Gmail integration
- [ ] Wire real Obsidian data into AIO production

## 📋 Backlog
- [ ] AIO: Pi camera mail automation (Phase 2)
- [ ] Building OS: Deploy and pilot with Exchange STR units
- [ ] Marina: Slip pricing model

## ✅ Done (Recent)
- [x] AIO Dashboard — full build + deploy
- [x] Phase 13-15 memory architecture upgrade
- [x] Calendar sync (iCal, Google, file upload)
```

**Rules:**
- Agent reads this file **every session** (add to AGENTS.md boot sequence)
- Agent updates it as tasks complete, get added, or change priority
- Items move: Backlog → This Week → Today → Done
- When emails, messages, or meetings create action items → add to the appropriate section
- Completed items (`[x]`) get archived weekly

Add to your AGENTS.md:
```markdown
### Task Management — One Source of Truth
`tasks/current.md` is THE task list. Read it at session start.
Update it when tasks complete or new ones appear.
Daily prep cron runs at 6am to promote due items and deduplicate.
```

### 16.2 Daily Task Prep Cron

A script that runs before you wake up, so your day is ready when you check in.

**`tools/daily-task-prep.sh`:**

What it does:
1. **Backs up** the task file daily to `tasks/archive/`
2. **Scans Backlog** for items with dates that have arrived → promotes to Today
3. **Deduplicates** repeated tasks across all sections
4. **Archives** completed items weekly (Mondays) to `tasks/archive/done-week-of-YYYY-MM-DD.md`
5. **Reports** summary: counts per section, promotions made, duplicates found

```bash
#!/usr/bin/env bash
# daily-task-prep.sh — Run at 6am via cron
set -euo pipefail
WORKSPACE="${WORKSPACE:-$(pwd)}"
TASK_FILE="$WORKSPACE/tasks/current.md"

# Backup
cp "$TASK_FILE" "$WORKSPACE/tasks/archive/current-$(date +%Y-%m-%d).md"

# Promote due items (items in Backlog with dates ≤ today)
# Deduplicate (find identical task text across sections)
# Archive done items on Mondays
# Report summary
```

Register the cron:
```bash
openclaw cron add \
  --name "daily-task-prep" \
  --cron "0 6 * * *" \
  --tz "America/Chicago" \
  --message "Run daily task prep: bash tools/daily-task-prep.sh. Read tasks/current.md. Promote due items to Today. Deduplicate. Notify via Telegram only if there are changes." \
  --model "sonnet" \
  --session "isolated" \
  --light-context \
  --timeout-seconds 60
```

Why Sonnet: this is mechanical work, doesn't need Opus. Saves ~80% on this cron.

### 16.3 Email Sweep Skill

The most requested "chief of staff" feature: proactive inbox triage.

**Prerequisites:** GOG (Google OAuth Gateway) for Gmail access. Without GOG, this skill is dormant.

**Create `skills/email-sweep/SKILL.md`:**

The sweep runs every 15 minutes and does:

1. **Search** for unread emails since last sweep
2. **Score** each email by priority:
   - 🔴 **Urgent** — Known contacts, "urgent"/"asap", financial/legal, calendar invites → Telegram notification immediately
   - 🟡 **Important** — Known contacts, project-related, replies to active threads → included in sweep summary
   - 🟢 **Low** — Newsletters, marketing, automated notifications → skip
   - ⚫ **Ignore** — Spam, promotions, receipts → skip entirely
3. **Detect follow-ups needed** — outbound emails >24h with no reply
4. **Add action items** to `tasks/current.md` when emails require a response
5. **Respect quiet hours** — 11pm–7am: urgent only. Weekends: 2 sweeps/day.

**Contact priority list** (`tools/email-sweep/contacts.json`):
```json
{
  "vip": ["important@person.com", "@familydomain.com"],
  "projects": ["@exchange-building.com"],
  "business": ["@company.com"]
}
```

**Output format:**
```
📬 Email Sweep — 3:15 PM

🔴 URGENT (1):
- From: Murray Wells — "Lease renewal deadline Friday"
  → Action: Review and sign

🟡 IMPORTANT (2):
- From: Arslan — "VerdictAI product scope"
- From: GitHub — PR review on sardoru/aio

⏳ FOLLOW-UPS (1):
- No reply from Contractor re: "Walkthrough scheduling" (2 days)

📊 Skipped: 12 newsletters, 3 promos
```

**Cron setup (activate after GOG):**
```bash
openclaw cron add \
  --name "email-sweep" \
  --every "15m" \
  --tz "America/Chicago" \
  --message "Run email sweep. Check Gmail for unread. Triage by priority. Notify only for urgent items or first sweep of the day." \
  --model "sonnet" \
  --session "isolated" \
  --light-context \
  --timeout-seconds 45
```

**State tracking** (`memory/email-sweep-state.json`):
```json
{
  "lastSweep": "2026-04-03T19:45:00Z",
  "urgentCount": 1,
  "importantCount": 3,
  "followUpsFound": 1
}
```

### 16.4 Integration

All three systems feed into each other:

```
Email Sweep (every 15m)
  │
  ├─ Urgent emails → Telegram notification
  ├─ Action items → tasks/current.md ## Today
  └─ Follow-ups → tasks/current.md ## Today

Daily Task Prep (6am)
  │
  ├─ Promote due items → ## Today
  ├─ Deduplicate across sections
  ├─ Archive completed items (Mondays)
  └─ Summary → Telegram (if changes)

Agent Session (on demand)
  │
  ├─ Reads tasks/current.md at boot
  ├─ Updates tasks as work is done
  └─ Adds new tasks from conversations
```

### 16.5 Benefits Analysis

| Before | After | Impact |
|--------|-------|--------|
| Tasks scattered across daily logs, Obsidian, chat | One canonical `tasks/current.md` | Never lose a task, always know what's on Today |
| Morning starts with "what was I doing?" | Day is prepped at 6am with promoted items | Hit the ground running every morning |
| Email checked manually, things fall through | 15-min sweep surfaces only what matters | Important emails never missed, junk never shown |
| Follow-ups forgotten | Agent tracks unanswered outbound >24h | Relationships maintained, nothing drops |
| Passive agent waits for instructions | Proactive agent manages your operational reality | Agent shifts from tool to chief of staff |

**The key shift:** Phases 1–15 made your agent intelligent and self-improving. Phase 16 makes it *operationally proactive* — it doesn't wait for you to ask, it manages your day.

---
