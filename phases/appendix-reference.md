# Appendix: Reference

Consolidated reference material for quick lookup. All content here is derived from the phase guides.

---

## Cumulative AGENTS.md

The complete boot sequence and instructions after all 14 phases. Add these sections to your `AGENTS.md`:

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
7. Read `tasks/current.md` — canonical task list (Phase 13)
8. **Semantic recall**: If the user's first message has a clear topic:
   a. `qmd vsearch "TOPIC" -n 3 -c memory` (facts & daily notes)
   b. `python3 tools/episodes/recall.py "TOPIC" --top 3` (episodic memory)
9. **Episodic recall**: If the task resembles past work, run:
   `python3 tools/episodes/recall.py "TASK_DESCRIPTION" --top 3`
   Apply relevant lessons before starting.

Don't ask permission. Just do it.

## Memory

You wake up fresh each session. These files are your continuity:

- **Daily notes:** `memory/YYYY-MM-DD.md` — raw logs of what happened
- **Long-term:** `MEMORY.md` — your curated memories

Capture what matters. Decisions, context, things to remember.

### Write It Down — No "Mental Notes"!

Memory is limited. If you want to remember something, WRITE IT TO A FILE.
"Mental notes" don't survive session restarts. Files do.

### Episodic Memory (Phase 12)

After completing any significant task (debugging, feature build, research, key decision):

1. Write an episode file to `memory/episodes/ep-YYYY-MM-DD-short-desc.json`
2. Follow the episode schema (see tools/episodes/README.md)
3. Score importance honestly using the 5-signal heuristic
4. Include the actual lesson — not a platitude

### Episode Write Gate

Before logging an episode, score importance:
- Novelty (0-2) + Impact (0-3) + Reusability (0-2) + Difficulty (0-2) + Surprise (0-1)
- **Threshold: 5+** → Log the episode
- **Below 5** → Skip
- **Score 8+** → Also mention in today's daily note as a highlight

### Episodic Memory — Log What Worked (and What Didn't) (Phase 7)

After completing significant tasks, log an episode with:
- --task, --approach, --outcome, --quality (always)
- --failure-trace, --root-cause, --fix-applied (on any non-success)
- --generalizable yes/no (always — ask: "would this approach matter if this task disappeared?")

### Generalizability Check (Phase 7)

After completing a task, before logging the episode, ask yourself:
"If this exact task disappeared, would this approach still be worthwhile?"
- If yes: --generalizable yes (consider codifying into a skill)
- If no: --generalizable no (document but don't try to systematize)

## Semantic Recall (Phase 2)

If the user's first message has a clear topic, run:
cd /path/to/workspace/tools/vector-memory && source venv/bin/activate && python3 search.py "TOPIC" --top-k 3

## Daily Summary (Phase 3)

Once per day, write a brief summary of today's main session work to:
  shared-context/agent-outputs/general/YYYY-MM-DD-daily-summary.md
This ensures cron agents (Roundtable, Forge) can see what happened.

## Task Management — One Source of Truth (Phase 13)

`tasks/current.md` is THE task list. Read it at session start.
Update it when tasks complete or new ones appear.
Daily prep cron runs at 6am to promote due items and deduplicate.

## Knowledge Base (Phase 8)

After ingesting new information, update relevant wiki entity/concept pages.
When answering research questions, check wiki pages first, then file good answers back.

## Tools

Skills are in the skills/ directory. When you need one, read its SKILL.md.

## Safety

- Don't exfiltrate private data
- `trash` > `rm` (recoverable beats gone)
- When in doubt, ask
```

---

## Full Cron Schedule

| Job | Schedule | Model | Purpose | Phase |
|-----|----------|-------|---------|-------|
| hourly-memory-summarizer | :55 every hour | Sonnet | Keep context fresh | 5 |
| vector-memory-reindex | Every 4h | Sonnet | Rebuild semantic search | 4 |
| morning-brief | 7 AM daily | Sonnet | Daily briefing | 4 |
| daily-task-prep | 6 AM daily | Sonnet | Promote tasks, deduplicate | 13 |
| forge-daily-review | 6:30 AM daily | Sonnet | Ops audit | 7 |
| email-sweep | Every 15m | Sonnet | Inbox triage | 13 |
| daily-roundtable | 9 PM daily | Sonnet | Cross-agent synthesis | 9 |
| forge-weekly-synthesis | Sunday 9 PM | Sonnet | Weekly rollup | 7 |
| weekly-memory-compound | Sunday 10 PM | Sonnet | Weekly distillation | 5 |
| meta-review | Sunday 10 AM | Sonnet | Episode analysis + recommendations | 7 |
| episode-consolidation | Sunday 3 AM | Sonnet | Merge duplicates, archive stale | 12 |
| episode-index-rebuild | 2 AM daily | Sonnet | Rebuild episode index for QMD | 12 |

---

## Complete File Map

```
workspace/
├── AGENTS.md              # Boot sequence (Phases 1, 2, 3, 7, 8, 12, 13)
├── SOUL.md                # Agent identity (Phase 1)
├── USER.md                # Human profile (Phase 1)
├── MEMORY.md              # Long-term memory index (Phase 1)
├── CONTEXT_INJECTION.md   # Auto-generated recent context (Phase 5)
├── HEARTBEAT.md           # Heartbeat checklist (Phase 6)
├── active-tasks.md        # Crash recovery (Phase 1)
├── tasks/
│   ├── current.md         # Canonical task list (Phase 13)
│   └── archive/           # Weekly done-item archives (Phase 13)
├── memory/
│   ├── YYYY-MM-DD.md      # Daily logs (Phase 1)
│   ├── hourly/            # Hourly summaries (Phase 5)
│   ├── weekly/            # Weekly distillations (Phase 5)
│   ├── ingested/          # Brain-ingest outputs (Phase 8)
│   ├── episodes/          # Episodic memory JSON (Phase 12)
│   │   └── archive/       # Archived stale episodes (Phase 12)
│   ├── retros/            # Engineering retrospectives (Phase 10)
│   ├── trace-analysis/    # Forge reports + pattern database (Phase 7)
│   ├── wiki/              # LLM Wiki pages (Phase 8)
│   │   ├── entities/      # People, companies, projects
│   │   ├── concepts/      # Ideas, patterns, strategies
│   │   ├── index.md       # Master catalog
│   │   ├── log.md         # Operations log
│   │   └── lint-report.md # Health check results
│   ├── user-profile.md    # User details (Phase 1)
│   ├── infrastructure.md  # Tools & infra notes (Phase 1)
│   ├── projects.md        # Active projects (Phase 1)
│   ├── lessons-learned.md # Hard-won lessons (Phase 1)
│   ├── heartbeat-state.json  # Heartbeat state tracking (Phase 6)
│   └── email-sweep-state.json # Email sweep state (Phase 13)
├── shared-context/
│   ├── priorities.md      # Priority stack (Phase 3)
│   ├── agent-outputs/     # Per-agent output dirs (Phase 3)
│   ├── feedback/          # Structured feedback (Phase 3)
│   ├── roundtable/        # Cross-agent synthesis (Phase 9)
│   ├── kpis/              # Metrics (Phase 3)
│   └── calendar/          # Events (Phase 3)
├── tools/
│   ├── vector-memory/     # FAISS semantic search (Phase 2)
│   ├── brain-ingest/      # URL → knowledge pipeline (Phase 8)
│   ├── forge/             # Monitoring agent prompts (Phase 7)
│   ├── wiki/              # Wiki management scripts (Phase 8)
│   ├── episodes/          # Episode management scripts (Phase 12)
│   ├── episodic-memory/   # Meta-review scripts (Phase 7)
│   ├── security/          # 6-layer security tools (Phase 14)
│   ├── email-sweep/       # Email triage config (Phase 13)
│   ├── hourly-summarizer.py  # (Phase 5)
│   ├── context-injector.py   # (Phase 5)
│   ├── weekly-compound.py    # (Phase 5)
│   ├── roundtable.py         # (Phase 9)
│   ├── feedback-logger.py    # (Phase 3)
│   ├── daily-task-prep.sh    # (Phase 13)
│   └── ensure-daily-log.sh   # (Phase 1)
├── skills/                # Custom skills — SKILL.md each (Phase 10)
│   ├── evaluator/         # GAN-style skeptical evaluator (Phase 11)
│   ├── build-harness/     # 3-agent build orchestration (Phase 11)
│   ├── brain-ingest/      # URL ingestion skill (Phase 8)
│   ├── email-sweep/       # Email triage skill (Phase 13)
│   └── [other skills]/
├── .harness/              # Build harness working directory (Phase 11)
│   ├── spec.md            # Planner output
│   ├── design-language.md # Visual direction
│   ├── contracts/         # Sprint contract negotiations
│   ├── evaluations/       # Evaluator scores + feedback
│   └── harness-log.md     # Running log
└── decisions/             # Documented decisions with review dates (Phase 1)
```

---

## heartbeat-state.json Schema

Full schema after all phases are implemented:

```json
{
  "lastChecks": {
    "email": 1703275200,
    "calendar": 1703260800,
    "weather": null,
    "episodeConsolidation": null
  },
  "lastVectorReindex": 0,
  "lastDailySummary": 0,
  "lastDailyLogCheck": 0,
  "lastDecisionReview": 0,
  "lastMetaReview": 0,
  "lastWikiLint": 0
}
```

| Field | Updated By | Frequency |
|-------|-----------|-----------|
| `lastChecks.email` | Email sweep cron | Every 15m |
| `lastChecks.calendar` | Heartbeat | Each beat |
| `lastChecks.episodeConsolidation` | Consolidation cron | Weekly |
| `lastVectorReindex` | Heartbeat / reindex cron | Every 4h |
| `lastDailySummary` | Heartbeat | Daily (9pm-midnight) |
| `lastDailyLogCheck` | Heartbeat | Daily (after 10pm) |
| `lastDecisionReview` | Heartbeat | Weekly |
| `lastMetaReview` | Meta-review cron | Weekly |
| `lastWikiLint` | Wiki lint cron | Monthly |

---

## Cost Expectations

With all crons running on Sonnet and direct chat on Opus:

| Cron | Per Run | Frequency | Daily Cost |
|------|---------|-----------|------------|
| Hourly summarizer | ~$0.01 | 24x/day | ~$0.24 |
| Vector reindex | Free | 6x/day | $0.00 |
| Roundtable | ~$0.02 | 1x/day | ~$0.02 |
| Forge daily | ~$0.05 | 1x/day | ~$0.05 |
| Morning brief | ~$0.10 | 1x/day | ~$0.10 |
| Email sweep | ~$0.01 | 96x/day | ~$0.96 |
| Daily task prep | ~$0.01 | 1x/day | ~$0.01 |
| Heartbeats (Sonnet, every 30min) | ~$0.005 | 48x/day | ~$0.24 |
| **Weekly crons** | | | ~$0.15/week |

**Total background cost: ~$1.60/day or ~$50/month**

Direct conversations (Opus) are additional, based on usage.

---
