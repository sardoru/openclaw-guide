# OpenClaw Production Upgrades — Phase 11

## Phase 11: Custom Skills

Skills are instruction files that teach your agent specific capabilities. Create them in a `skills/` directory.

### Structure

```
skills/
├── [skill-name]/
│   └── SKILL.md    # Instructions the agent reads when the skill is invoked
```

### Example: Retro Skill

```markdown
# retro — Engineering Retrospective

## When to Use
Weekly or on-demand to review recent engineering work.

## Process
1. Run `git log --oneline --since="7 days ago"` to get recent commits
2. Analyze: file hotspots, LOC changes, focus score vs priorities
3. Grade focus: % of work aligned with top 3 priorities
4. Save snapshot to memory/retros/YYYY-MM-DD-retro.md

## Output
- Commit count, net LOC, dominant work areas
- Focus score (% aligned with priorities)
- Scatter score (how spread across unrelated areas)
- Recommendations for next week
```

### Registering Skills

Add skills to your OpenClaw config or place them where the agent can discover them. Reference them in AGENTS.md:

```markdown
## Tools
Skills are in the skills/ directory. When you need one, read its SKILL.md.
```

---

## Summary: Build Harness Skill Files

| Skill | Path | Purpose |
|-------|------|---------|
| evaluator | `skills/evaluator/SKILL.md` | Skeptical critic with 4 criteria, browser testing, anti-leniency |
| build-harness | `skills/build-harness/SKILL.md` | 3-agent orchestration: planner → generator → evaluator |
| jonyive | `skills/jonyive/SKILL.md` | Design principles mapped to evaluator criteria |

## Summary: Full Cron Schedule

| Job | Schedule | Model | Purpose |
|-----|----------|-------|---------|
| hourly-memory-summarizer | :55 every hour | Sonnet | Keep context fresh |
| vector-memory-reindex | Every 4h | Sonnet | Rebuild semantic search |
| morning-brief | 7 AM daily | Sonnet | Daily briefing |
| forge-daily-review | 6:30 AM daily | Sonnet | Ops audit |
| daily-roundtable | 9 PM daily | Sonnet | Cross-agent synthesis |
| forge-weekly-synthesis | Sunday 9 PM | Sonnet | Weekly rollup |
| weekly-memory-compound | Sunday 10 PM | Sonnet | Weekly distillation |

## Summary: File Map

```
workspace/
├── AGENTS.md              # Boot sequence
├── SOUL.md                # Agent identity
├── USER.md                # Human profile
├── MEMORY.md              # Long-term memory index
├── CONTEXT_INJECTION.md   # Auto-generated recent context
├── HEARTBEAT.md           # Heartbeat checklist
├── active-tasks.md        # Crash recovery
├── memory/
│   ├── YYYY-MM-DD.md      # Daily logs
│   ├── hourly/            # Hourly summaries
│   ├── weekly/            # Weekly distillations
│   ├── ingested/          # Brain-ingest outputs
│   ├── retros/            # Engineering retrospectives
│   ├── trace-analysis/    # Forge reports + pattern database
│   ├── user-profile.md    # User details
│   ├── infrastructure.md  # Tools & infra notes
│   ├── projects.md        # Active projects
│   └── lessons-learned.md # Hard-won lessons
├── shared-context/
│   ├── priorities.md      # Priority stack
│   ├── agent-outputs/     # Per-agent output dirs
│   ├── feedback/          # Structured feedback
│   ├── roundtable/        # Cross-agent synthesis
│   ├── kpis/              # Metrics
│   └── calendar/          # Events
├── tools/
│   ├── vector-memory/     # FAISS semantic search
│   ├── brain-ingest/      # URL → knowledge pipeline
│   ├── forge/             # Monitoring agent prompts
│   ├── hourly-summarizer.py
│   ├── context-injector.py
│   ├── weekly-compound.py
│   ├── roundtable.py
│   ├── feedback-logger.py
│   └── ensure-daily-log.sh
├── skills/                # Custom skills (SKILL.md each)
│   ├── evaluator/         # GAN-style skeptical evaluator
│   ├── build-harness/     # 3-agent build orchestration
│   └── [other skills]/
├── .harness/              # Build harness working directory (per-project)
│   ├── spec.md            # Planner output
│   ├── design-language.md # Visual direction
│   ├── contracts/         # Sprint contract negotiations
│   ├── evaluations/       # Evaluator scores + feedback
│   └── harness-log.md     # Running log
└── decisions/             # Documented decisions with review dates
```

---

## Cost Expectations

With all crons running on Sonnet and direct chat on Opus:
- **Hourly summarizer:** ~$0.01/run × 24 = ~$0.24/day
- **Vector reindex:** Free (local Python)
- **Roundtable:** ~$0.02/run = ~$0.02/day
- **Forge daily:** ~$0.05/run = ~$0.05/day
- **Morning brief:** ~$0.10/run = ~$0.10/day
- **Weekly crons:** ~$0.15/week
- **Heartbeats (Sonnet, every 30min):** ~$0.005/run × 48 = ~$0.24/day
- **Total background cost: ~$0.65/day or ~$20/month**

Direct conversations (Opus) are additional, based on usage.

---
