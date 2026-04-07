# OpenClaw Production Upgrades — Phase 7

## Phase 7: Monitoring, Failure Analysis & Meta-Review

This phase combines three related capabilities into a unified monitoring and continuous improvement system: Forge (the meta-agent that watches your system), failure trace logging (structured root-cause analysis), and weekly meta-review (pattern detection and skill improvement recommendations).

The core insight: monitoring without structured failure analysis is just alerting. Failure analysis without a review loop is just logging. All three together create a system that gets measurably better over time.

### Part A: Forge — System Monitoring

Forge is a meta-agent that monitors the entire system. It catches failures, diagnoses root causes, and applies fixes automatically.

#### 7.1 Forge Daily Review

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

#### 7.2 Forge Weekly Synthesis

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

#### 7.3 Pattern Database

Create `memory/trace-analysis/pattern-database.md`:

```markdown
# Failure Pattern Database

Maintained by Forge. Each pattern includes occurrence count and resolution status.

## Known Patterns

(Forge will populate this automatically as it discovers patterns)
```

### Part B: Failure Trace Logging

Logging outcomes alone isn't enough. Research from Kevin Gu's AutoAgent project (1.2M impressions, #1 on SpreadsheetBench and TerminalBench) proved that **traces are everything**: when a meta-agent only received success/failure scores without reasoning traces, its improvement rate dropped dramatically.

This upgrades your episodic memory (see Phase 12) from "what happened" to "why it happened, what caused failures, what fixed them, and whether the approach is worth reusing."

#### 7.4 Enhanced Episode Fields

Upgrade your episode logger to accept failure-specific fields:

```bash
bash tools/episodic-memory/episode-logger.sh \
  --task "Deploy app to Vercel" \
  --approach "Used vercel CLI with --prod flag" \
  --outcome "success" \
  --duration "15m" \
  --quality "0.85" \
  --tags "vercel,deploy" \
  --failure-trace "POST /api/mail/ingest returned HTML instead of JSON" \
  --root-cause "Vite configureServer middleware only runs in dev. Production needs serverless functions." \
  --fix-applied "Created api/mail/ingest.ts as Vercel serverless function" \
  --generalizable "yes" \
  --notes "Any Vite dev-only API needs a parallel serverless function for production"
```

**New fields:**

| Field | Purpose |
|-------|--------|
| `--failure-trace` | What went wrong — the error message, unexpected behavior, or symptom |
| `--root-cause` | Why it happened — the underlying cause, not just the symptom |
| `--fix-applied` | What fixed it — the specific change that resolved the issue |
| `--generalizable` | `yes` or `no` — is this approach reusable across similar tasks? |

These fields are stored in the episode JSON and appended to the human-readable `index.md` with badges:
- reusable = generalizable approach
- pinned = task-specific (don't try to reuse)

**Implementation:** Add the new flags to your episode logger's argument parser. Store them as nullable fields in the JSON — `null` when not provided, so existing episodes remain compatible:

```json
{
  "id": "ep-20260403-064816-fix-mail-scan-ocr",
  "task": "Fix mail scan OCR on Vercel production",
  "outcome": "success",
  "failure_trace": "POST returned HTML instead of JSON",
  "root_cause": "Vite middleware only runs in dev mode",
  "fix_applied": "Created Vercel serverless function",
  "generalizable": true
}
```

#### 7.5 The Generalizability Check

Inspired by AutoAgent's overfitting problem — their meta-agent got lazy, gaming metrics with rubric-specific hacks instead of genuine improvements. Their fix: force self-reflection.

When logging an episode, the generalizability flag answers one question:

> *"If this exact task disappeared tomorrow, would this approach still be a worthwhile improvement?"*

- **Yes (reusable)** — The approach transfers. Example: "Always create a Vercel serverless function for any POST endpoint" is useful for every future deployment.
- **No (pinned)** — The approach is a one-off. Example: "Renamed a specific CSS class to fix a layout bug" only matters for that component.

Add to AGENTS.md:

```markdown
### Generalizability Check
After completing a task, before logging the episode, ask yourself:
"If this exact task disappeared, would this approach still be worthwhile?"
- If yes: --generalizable yes (consider codifying into a skill)
- If no: --generalizable no (document but don't try to systematize)
```

Over time, your episode index becomes a curated library of proven approaches tagged by reusability.

### Part C: Weekly Meta-Agent Review

The meta-review closes the loop: Forge monitors the system, failure traces capture what went wrong, and the meta-review analyzes patterns across all episodes to generate actionable recommendations.

#### 7.6 Meta-Review Script

Create a meta-review script that analyzes recent episodes and generates actionable recommendations:

**`tools/episodic-memory/meta-review.sh`:**

```bash
#!/usr/bin/env bash
# Usage: bash tools/episodic-memory/meta-review.sh --days 7 --output stdout
```

The script:
1. Collects all episode JSON files from the last N days
2. Calculates success rate, failure count, trace coverage
3. Groups failure patterns by root cause
4. Identifies high-quality approaches (>=0.90) worth codifying
5. Flags low-quality episodes (<=0.60) for review
6. Counts generalizable vs task-specific approaches
7. Generates recommendations

**What it reports:**

```markdown
# Meta-Agent Review — 2026-04-03

**Period:** Last 7 days | **Episodes:** 11 | **Success rate:** 72%

## Recommendations
- Improve trace coverage: 0/3 non-success episodes have failure traces
- Score generalizability: 11/11 episodes missing --generalizable flag
- Review failure patterns below for repeated root causes
- 3 generalizable approaches found — consider codifying into skills

## Failure Patterns
- **Migrate cron jobs** (failure)
  - Root cause: OpenClaw cron has different semantics

## High Quality Approaches (>=0.90)
- Build Puck Tracker real-time scoring (0.92) — SSE pattern
- Exchange Building STR revenue tracking (0.91) — Supabase views

## Generalizable Approaches Worth Codifying
- Vercel serverless function pattern for Vite POST endpoints
- Multi-agent parallel builds for complex feature sets
```

**Recommendations the meta-agent generates:**

| Signal | Recommendation |
|--------|---------------|
| Low trace coverage | "All failures should include --failure-trace and --root-cause" |
| Repeated root causes | "Add to lessons-learned.md or create preventive checks" |
| 3+ generalizable approaches | "Consider codifying into reusable skill files" |
| Success rate < 70% | "Review failure patterns, add guardrails to AGENTS.md" |
| Missing generalizability flags | "Tag approaches as reusable or task-specific" |

#### 7.7 Integration

All three parts feed into each other:

```
Forge Daily Review (6:30 AM)
  │
  ├─ Discovers failures from last 24h
  ├─ Categorizes by type
  ├─ Updates pattern database
  └─ Writes report to memory/trace-analysis/

Failure Trace Logging (continuous, during work)
  │
  ├─ Captures root cause + fix for every non-success
  ├─ Tags generalizable vs task-specific
  └─ Feeds into episodic memory (Phase 12)

Meta-Agent Review (Sundays)
  │
  ├─ Reads all episodes from past 7 days
  ├─ Groups failure patterns by root cause
  ├─ Identifies approaches worth codifying into skills
  └─ Generates actionable recommendations
```

**HEARTBEAT.md** — Add weekly meta-review:

```markdown
## Meta-Agent Episode Review
- Weekly (Sundays), run: `bash tools/episodic-memory/meta-review.sh --days 7 --output stdout`
- If recommendations found, review and act on them
- If success rate < 70%, flag for user review
- Track in memory/heartbeat-state.json under "lastMetaReview"
```

**AGENTS.md** — Update episode logging instructions:

```markdown
### Episodic Memory — Log What Worked (and What Didn't)
After completing significant tasks, log an episode with:
- --task, --approach, --outcome, --quality (always)
- --failure-trace, --root-cause, --fix-applied (on any non-success)
- --generalizable yes/no (always — ask: "would this approach matter if this task disappeared?")
```

**Cron job** (optional) — Automate the weekly review:

```bash
openclaw cron add \
  --name "meta-review" \
  --schedule "0 10 * * 0" \
  --task "Run bash tools/episodic-memory/meta-review.sh --days 7 --output stdout. Summarize findings. If there are actionable recommendations, implement the top 1-2. Update lessons-learned.md with any new failure patterns."
```

### 7.8 Benefits Analysis

| Before | After | Impact |
|--------|-------|--------|
| No system-wide monitoring | Forge daily review catches failures across all crons/agents | Issues discovered within 24h, not when something visibly breaks |
| Episodes log success/failure only | Failure traces capture root cause + fix | Failures become reusable knowledge |
| All approaches treated equally | Generalizable vs task-specific tagging | Agent knows which patterns to reuse vs ignore |
| No systematic review | Weekly meta-agent analyzes patterns | Continuous improvement loop |
| Failure patterns stay scattered in daily logs | Root causes extracted, grouped, and tracked | Repeated failures get caught and prevented |
| High-quality approaches go undiscovered | Meta-review surfaces approaches worth codifying | Best practices naturally emerge into skills |

**The key insight from AutoAgent:** Agents are better at understanding agents than we are. By giving your agent its own failure traces to analyze, it develops what Kevin Gu calls "model empathy" — an implicit understanding of its own limitations and tendencies. Same-model pairings win: your meta-agent (reviewing episodes) and your task agent (doing the work) should run on the same model.

---
