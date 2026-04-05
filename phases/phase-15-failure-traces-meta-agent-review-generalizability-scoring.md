# OpenClaw Production Upgrades — Phase 15

## Phase 15: Failure Traces, Meta-Agent Review & Generalizability Scoring

Phase 13 gave your agent episodic memory — structured logs of what it did and whether it worked. But logging outcomes alone isn't enough. Research from Kevin Gu's AutoAgent project (1.2M impressions, #1 on SpreadsheetBench and TerminalBench) proved that **traces are everything**: when a meta-agent only received success/failure scores without reasoning traces, its improvement rate dropped dramatically.

This phase upgrades your episodic memory from "what happened" to "why it happened, what caused failures, what fixed them, and whether the approach is worth reusing."

### Why This Matters

Without failure traces, your agent repeats the same mistakes. It knows a task failed but not *why*. Without generalizability scoring, it accumulates approaches without knowing which ones transfer to new situations vs. which are one-off hacks. Without a meta-review loop, no one analyzes the patterns.

With this phase:
- Every failure is a documented lesson with root cause and fix
- Every approach is tagged as reusable or task-specific
- A weekly meta-agent reviews all episodes and recommends improvements to your agent's skills and instructions

### 15.1 Failure Trace Logging

Upgrade your episode logger to accept three new fields:

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
- 🔄 = generalizable approach (reusable)
- 📌 = task-specific (don't try to reuse)

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

### 15.2 The Generalizability Check

Inspired by AutoAgent's overfitting problem — their meta-agent got lazy, gaming metrics with rubric-specific hacks instead of genuine improvements. Their fix: force self-reflection.

When logging an episode, the generalizability flag answers one question:

> *"If this exact task disappeared tomorrow, would this approach still be a worthwhile improvement?"*

- **Yes (🔄)** — The approach transfers. Example: "Always create a Vercel serverless function for any POST endpoint" is useful for every future deployment.
- **No (📌)** — The approach is a one-off. Example: "Renamed a specific CSS class to fix a layout bug" only matters for that component.

Add to AGENTS.md:

```markdown
### Generalizability Check
After completing a task, before logging the episode, ask yourself:
"If this exact task disappeared, would this approach still be worthwhile?"
- If yes: --generalizable yes (consider codifying into a skill)
- If no: --generalizable no (document but don't try to systematize)
```

Over time, your episode index becomes a curated library of proven approaches tagged by reusability.

### 15.3 Weekly Meta-Agent Review

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
4. Identifies high-quality approaches (≥0.90) worth codifying
5. Flags low-quality episodes (≤0.60) for review
6. Counts generalizable vs task-specific approaches
7. Generates recommendations

**What it reports:**

```markdown
# Meta-Agent Review — 2026-04-03

**Period:** Last 7 days | **Episodes:** 11 | **Success rate:** 72%

## Recommendations
- ⚠️ Improve trace coverage: 0/3 non-success episodes have failure traces
- 📊 Score generalizability: 11/11 episodes missing --generalizable flag
- 🔴 Review failure patterns below for repeated root causes
- 🔄 3 generalizable approaches found — consider codifying into skills

## Failure Patterns
- **Migrate cron jobs** (failure)
  - Root cause: OpenClaw cron has different semantics

## High Quality Approaches (≥0.90)
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

### 15.4 Integration

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

### 15.5 Benefits Analysis

| Before (Phase 13) | After (Phase 15) | Impact |
|--------------------|-------------------|--------|
| Episodes log success/failure | Episodes log *why* things failed + what fixed them | Failures become reusable knowledge, not just noise |
| All approaches treated equally | Generalizable vs task-specific tagging | Agent knows which patterns to reuse vs ignore |
| No systematic review | Weekly meta-agent analyzes patterns | Continuous improvement loop — agent gets better over time |
| Failure patterns stay in daily logs | Root causes extracted and grouped | Repeated failures get caught and prevented |
| High-quality approaches undiscovered | Meta-review surfaces approaches worth codifying | Best practices naturally emerge into skills |

**The key insight from AutoAgent:** Agents are better at understanding agents than we are. By giving your agent its own failure traces to analyze, it develops what Kevin Gu calls "model empathy" — an implicit understanding of its own limitations and tendencies. The meta-review loop operationalizes this: the agent reads its own reasoning traces, understands failure modes as part of its worldview, and corrects them.

Same-model pairings win. Your meta-agent (reviewing episodes) and your task agent (doing the work) should run on the same model. The meta-agent writes improvements the task agent actually understands because they share the same weights.

---
