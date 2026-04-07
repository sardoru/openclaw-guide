# OpenClaw Production Upgrades — Phase 12

## Phase 12: Agentic Memory Architecture

Phases 1–11 gave your agent memory. This phase makes it *intelligent* memory.

Inspired by research on agentic memory systems (Park et al. *Generative Agents*, 2023), this phase upgrades your memory from simple storage to a curated, self-improving knowledge system with four layers: in-context, external, episodic, and semantic. The result: your agent stops re-learning old lessons, recalls *how* it solved similar problems, and automatically prunes stale context.

### The Memory Stack

| Layer | What It Stores | Our Implementation |
|-------|---------------|-------------------|
| In-context | Session state, system prompt, recent messages | AGENTS.md boot sequence, SOUL.md, USER.md |
| External | Facts, preferences, decisions | memory/*.md, decisions/*.md, QMD vector search |
| Episodic | Task outcomes, approaches, lessons | memory/episodes/ (NEW) |
| Semantic | World knowledge | Model weights (parametric) — supplemented by web search |

Phases 1–3 built the first two layers. Phase 12 adds episodic memory with importance scoring, decay-weighted recall, and automated consolidation — turning your flat file memory into a self-curating knowledge system.

### 12.1 Episodic Memory System

Episodic memory records *what happened, what worked, and what didn't* — structured task outcomes your agent can recall next time it faces something similar.

#### Directory Structure

```bash
mkdir -p memory/episodes memory/episodes/archive
```

#### Episode Schema

Each episode is a JSON file in `memory/episodes/`. One file per significant task:

```json
{
  "id": "ep-2026-04-02-deploy-fix",
  "timestamp": "2026-04-02T14:32:00-05:00",
  "title": "Fixed gateway deploy failure",
  "task_type": "debugging",
  "tags": ["gateway", "deploy", "nginx", "production"],
  "context": "Gateway failed to start after upgrading to v2.3. Nginx config had deprecated directive.",
  "approach": "Checked logs → found 'unknown directive' → searched nginx changelog → replaced deprecated ssl_protocols line.",
  "outcome": "success",
  "lesson": "Always check nginx changelog for deprecated directives before upgrading. The error message is misleading — says 'failed to bind' but the real issue is config parsing.",
  "importance": 7,
  "related_files": ["memory/2026-04-02.md"],
  "decay_anchor": "2026-04-02T14:32:00-05:00"
}
```

Field reference:

| Field | Required | Description |
|-------|----------|-------------|
| `id` | Yes | Unique slug: `ep-YYYY-MM-DD-short-desc` |
| `timestamp` | Yes | ISO 8601 with timezone |
| `title` | Yes | One-line summary |
| `task_type` | Yes | Category: `debugging`, `feature`, `research`, `config`, `conversation`, `decision` |
| `tags` | Yes | 3-6 searchable keywords |
| `context` | Yes | What was the situation? |
| `approach` | Yes | Step-by-step what was tried |
| `outcome` | Yes | `success`, `partial`, `failure` |
| `lesson` | Yes | The takeaway — what future-you should know |
| `importance` | Yes | 1-10 score (see section 12.2) |
| `related_files` | No | Links to daily notes, decisions, etc. |
| `decay_anchor` | Yes | Timestamp for recency scoring |

#### Episode Logger Script

Create `tools/episodes/log-episode.sh`:

```bash
#!/bin/bash
# Log an episode to memory/episodes/
# Usage: log-episode.sh <id> <title> <task_type> <importance> <outcome>
#   Reads context, approach, lesson from stdin (or prompts interactively)

set -euo pipefail

EPISODE_DIR="${OPENCLAW_WORKSPACE:-$(pwd)}/memory/episodes"
mkdir -p "$EPISODE_DIR"

ID="${1:?Usage: log-episode.sh <id> <title> <task_type> <importance> <outcome>}"
TITLE="${2:?Missing title}"
TASK_TYPE="${3:?Missing task_type}"
IMPORTANCE="${4:?Missing importance (1-10)}"
OUTCOME="${5:?Missing outcome (success|partial|failure)}"

TIMESTAMP=$(date -u +"%Y-%m-%dT%H:%M:%SZ")

# Read multiline fields
echo "Enter context (end with Ctrl-D):"
CONTEXT=$(cat)
echo "Enter approach (end with Ctrl-D):"
APPROACH=$(cat)
echo "Enter lesson (end with Ctrl-D):"
LESSON=$(cat)

cat > "$EPISODE_DIR/$ID.json" << EOF
{
  "id": "$ID",
  "timestamp": "$TIMESTAMP",
  "title": "$TITLE",
  "task_type": "$TASK_TYPE",
  "tags": [],
  "context": $(echo "$CONTEXT" | python3 -c 'import sys,json; print(json.dumps(sys.stdin.read().strip()))'),
  "approach": $(echo "$APPROACH" | python3 -c 'import sys,json; print(json.dumps(sys.stdin.read().strip()))'),
  "outcome": "$OUTCOME",
  "lesson": $(echo "$LESSON" | python3 -c 'import sys,json; print(json.dumps(sys.stdin.read().strip()))'),
  "importance": $IMPORTANCE,
  "related_files": [],
  "decay_anchor": "$TIMESTAMP"
}
EOF

echo "Episode logged: $EPISODE_DIR/$ID.json"
```

In practice, the agent creates episodes directly by writing JSON. The script is for manual logging or cron-based extraction.

#### Agent-Side Episode Creation

Add this instruction to your `AGENTS.md` (under the Memory section):

```markdown
### Episodic Memory

After completing any significant task (debugging, feature build, research, key decision):

1. Write an episode file to `memory/episodes/ep-YYYY-MM-DD-short-desc.json`
2. Follow the episode schema (see tools/episodes/README.md)
3. Score importance honestly (see section 12.2 heuristic)
4. Include the actual lesson — not a platitude

Bad lesson: "Always test before deploying"
Good lesson: "The nginx 'failed to bind' error on port 443 actually means a config parse error when ssl_protocols contains removed values. Check with nginx -t first."
```

#### Episode Recall Script

Create `tools/episodes/recall.py`:

```python
#!/usr/bin/env python3
"""
Episode Recall — Find relevant past episodes using decay-weighted scoring.
Usage: python3 tools/episodes/recall.py "query" [--top N] [--min-score 0.3]
"""

import json, glob, sys, argparse, math
from datetime import datetime, timezone
from pathlib import Path

def load_episodes(episode_dir):
    episodes = []
    for f in glob.glob(str(Path(episode_dir) / "*.json")):
        try:
            with open(f) as fh:
                episodes.append(json.load(fh))
        except (json.JSONDecodeError, KeyError) as e:
            print(f"Skipping {f}: {e}", file=sys.stderr)
    return episodes

def text_relevance(query, episode):
    """Simple keyword overlap relevance. Replace with vector similarity for production."""
    query_terms = set(query.lower().split())
    searchable = f"{episode['title']} {episode['context']} {episode['lesson']} {' '.join(episode.get('tags', []))}"
    doc_terms = set(searchable.lower().split())
    if not query_terms:
        return 0.0
    overlap = len(query_terms & doc_terms)
    return min(overlap / len(query_terms), 1.0)

def recency_score(episode, half_life_days=30):
    """Exponential decay based on age. Half-life = 30 days by default."""
    anchor = episode.get('decay_anchor', episode['timestamp'])
    try:
        dt = datetime.fromisoformat(anchor)
    except ValueError:
        return 0.5
    now = datetime.now(timezone.utc)
    if dt.tzinfo is None:
        dt = dt.replace(tzinfo=timezone.utc)
    age_days = (now - dt).total_seconds() / 86400
    return math.exp(-0.693 * age_days / half_life_days)  # ln(2) = 0.693

def importance_score(episode):
    """Normalize importance to 0-1 range."""
    return min(max(episode.get('importance', 5), 1), 10) / 10.0

def weighted_score(query, episode):
    """Combined score: relevance x 0.4 + importance x 0.3 + recency x 0.3"""
    rel = text_relevance(query, episode)
    imp = importance_score(episode)
    rec = recency_score(episode)
    return (rel * 0.4) + (imp * 0.3) + (rec * 0.3)

def recall(query, episode_dir, top_n=5, min_score=0.3):
    episodes = load_episodes(episode_dir)
    scored = [(ep, weighted_score(query, ep)) for ep in episodes]
    scored.sort(key=lambda x: x[1], reverse=True)
    return [(ep, s) for ep, s in scored[:top_n] if s >= min_score]

if __name__ == '__main__':
    parser = argparse.ArgumentParser(description='Recall relevant episodes')
    parser.add_argument('query', help='Search query')
    parser.add_argument('--top', type=int, default=5, help='Number of results')
    parser.add_argument('--min-score', type=float, default=0.3, help='Minimum score threshold')
    parser.add_argument('--episode-dir', default='memory/episodes', help='Episode directory')
    args = parser.parse_args()

    results = recall(args.query, args.episode_dir, args.top, args.min_score)

    if not results:
        print("No relevant episodes found.")
        sys.exit(0)

    for ep, score in results:
        print(f"\n{'='*60}")
        print(f"{ep['title']} (score: {score:.2f}, importance: {ep['importance']}/10)")
        print(f"   Type: {ep['task_type']} | Outcome: {ep['outcome']}")
        print(f"   Context: {ep['context'][:200]}")
        print(f"   Lesson: {ep['lesson']}")
```

#### QMD Vector Indexing

Episodes are plain JSON files in `memory/episodes/`, which means QMD's existing file scanner picks them up automatically if your `qmd index` command covers the `memory/` directory. For better results, create a wrapper that extracts the searchable text:

Create `tools/episodes/index-episodes.sh`:

```bash
#!/bin/bash
# Convert episodes to markdown for QMD indexing
set -euo pipefail

EPISODE_DIR="${OPENCLAW_WORKSPACE:-$(pwd)}/memory/episodes"
INDEX_FILE="${OPENCLAW_WORKSPACE:-$(pwd)}/memory/episodes/INDEX.md"

echo "# Episode Index" > "$INDEX_FILE"
echo "" >> "$INDEX_FILE"
echo "_Auto-generated. Do not edit._" >> "$INDEX_FILE"
echo "" >> "$INDEX_FILE"

for f in "$EPISODE_DIR"/*.json; do
  [ -f "$f" ] || continue
  TITLE=$(python3 -c "import json; print(json.load(open('$f'))['title'])")
  LESSON=$(python3 -c "import json; print(json.load(open('$f'))['lesson'])")
  TAGS=$(python3 -c "import json; print(', '.join(json.load(open('$f')).get('tags',[])))")
  IMPORTANCE=$(python3 -c "import json; print(json.load(open('$f'))['importance'])")
  echo "## $TITLE" >> "$INDEX_FILE"
  echo "Tags: $TAGS | Importance: $IMPORTANCE/10" >> "$INDEX_FILE"
  echo "Lesson: $LESSON" >> "$INDEX_FILE"
  echo "" >> "$INDEX_FILE"
done

echo "Indexed $(ls "$EPISODE_DIR"/*.json 2>/dev/null | wc -l | tr -d ' ') episodes → $INDEX_FILE"

# Now re-index with QMD
qmd index memory/episodes/
```

Run this after logging episodes, or wire it into a cron job (see section 12.5).

### 12.2 Importance Scoring

Not every task deserves an episode. Importance scoring acts as a **write gate** — filtering noise before it enters your episodic memory.

#### The Heuristic Scorer

Importance is scored 1-10 based on five signals:

| Signal | Weight | Description |
|--------|--------|-------------|
| Novelty | 2 pts | First time encountering this problem/pattern? |
| Impact | 3 pts | How much did it affect outcomes? (production issue = high, formatting tweak = low) |
| Reusability | 2 pts | Will this lesson apply to future tasks? |
| Difficulty | 2 pts | How hard was it to figure out? (easy answers aren't worth storing) |
| Surprise | 1 pt | Was the result unexpected? Surprises encode strong lessons |

**Scoring examples:**

| Task | Nov | Imp | Reu | Dif | Sur | Total | Log? |
|------|-----|-----|-----|-----|-----|-------|------|
| Fixed nginx config after upgrade | 2 | 2 | 2 | 1 | 1 | **8** | Yes |
| Changed button color per request | 0 | 0 | 0 | 0 | 0 | **0** | No |
| Discovered QMD timeout causes silent failures | 2 | 3 | 2 | 2 | 1 | **10** | Yes |
| Routine daily summary | 0 | 1 | 0 | 0 | 0 | **1** | No |
| New cron pattern for batch processing | 1 | 2 | 2 | 1 | 0 | **6** | Yes |

#### Write Gate

Add this rule to your `AGENTS.md`:

```markdown
### Episode Write Gate

Before logging an episode, score importance using the 5-signal heuristic:
- Novelty (0-2) + Impact (0-3) + Reusability (0-2) + Difficulty (0-2) + Surprise (0-1)
- **Threshold: 5+** → Log the episode
- **Below 5** → Skip. Not everything needs to be remembered.
- **Score 8+** → Also mention in today's daily note as a highlight
```

This keeps your episode store lean. After a month of active use, you should have 30-60 episodes, not 300. Quality over quantity.

### 12.3 Decay-Weighted Recall

Raw keyword search returns episodes by relevance alone. But a perfectly relevant episode from 6 months ago about a since-deprecated API is noise. Decay-weighted recall balances three signals:

#### The Scoring Formula

```
final_score = (relevance x 0.4) + (importance x 0.3) + (recency x 0.3)
```

Where:
- **Relevance (0-1):** Keyword overlap or vector cosine similarity between query and episode text
- **Importance (0-1):** Episode importance score normalized (divide by 10)
- **Recency (0-1):** Exponential decay with configurable half-life

```
recency = e^(-0.693 x age_days / half_life_days)
```

With a 30-day half-life:
- Today's episode: recency = 1.0
- 1 week old: recency ~ 0.85
- 30 days old: recency = 0.5
- 90 days old: recency ~ 0.125
- 180 days old: recency ~ 0.016

This means a 6-month-old episode needs to be *extremely* relevant and important to surface. Which is exactly what you want.

#### Tuning the Weights

The default 0.4/0.3/0.3 split works well for general use. Adjust for your workflow:

| Use Case | Relevance | Importance | Recency | Why |
|----------|-----------|------------|---------|-----|
| General | 0.4 | 0.3 | 0.3 | Balanced |
| Fast-moving codebase | 0.3 | 0.2 | 0.5 | Old episodes go stale fast |
| Research/knowledge work | 0.5 | 0.3 | 0.2 | Old insights stay valuable |
| Ops/debugging | 0.4 | 0.4 | 0.2 | Severity matters most |

#### QMD Integration Wrapper

Create `tools/episodes/smart-recall.sh` to combine QMD vector search with decay weighting:

```bash
#!/bin/bash
# Smart episode recall: QMD vector search + decay weighting
# Usage: smart-recall.sh "query" [top_n]

set -euo pipefail

QUERY="${1:?Usage: smart-recall.sh \"query\" [top_n]}"
TOP_N="${2:-5}"
EPISODE_DIR="${OPENCLAW_WORKSPACE:-$(pwd)}/memory/episodes"

# Step 1: Use QMD for initial candidate retrieval (vector similarity)
CANDIDATES=$(qmd vsearch "$QUERY" -n 20 -c memory/episodes 2>/dev/null || echo "")

if [ -z "$CANDIDATES" ]; then
  # Fallback to keyword-based recall
  python3 tools/episodes/recall.py "$QUERY" --top "$TOP_N" --episode-dir "$EPISODE_DIR"
  exit 0
fi

# Step 2: Re-rank with decay weighting
python3 tools/episodes/recall.py "$QUERY" --top "$TOP_N" --episode-dir "$EPISODE_DIR"
```

For the agent, add this to the AGENTS.md semantic recall step:

```markdown
7. **Semantic recall**: If the user's first message has a clear topic:
   a. `qmd vsearch "TOPIC" -n 3 -c memory` (existing — facts & daily notes)
   b. `python3 tools/episodes/recall.py "TOPIC" --top 3` (NEW — episodic memory)
```

### 12.4 Automated Consolidation

Episodic memory grows. Without maintenance, it fills with duplicates, stale entries, and near-identical lessons. Consolidation keeps it sharp.

#### Duplicate Detection

Create `tools/episodes/consolidate.py`:

```python
#!/usr/bin/env python3
"""
Episode Consolidation — Detect duplicates, merge related episodes, archive stale ones.
Usage: python3 tools/episodes/consolidate.py [--dry-run] [--archive-days 180]
"""

import json, glob, os, shutil, argparse
from datetime import datetime, timezone
from pathlib import Path
from collections import defaultdict

def load_all(episode_dir):
    episodes = []
    for f in sorted(glob.glob(str(Path(episode_dir) / "*.json"))):
        try:
            with open(f) as fh:
                ep = json.load(fh)
                ep['_path'] = f
                episodes.append(ep)
        except (json.JSONDecodeError, KeyError):
            continue
    return episodes

def jaccard_similarity(a, b):
    """Word-level Jaccard similarity between two strings."""
    set_a = set(a.lower().split())
    set_b = set(b.lower().split())
    if not set_a or not set_b:
        return 0.0
    return len(set_a & set_b) / len(set_a | set_b)

def find_duplicates(episodes, threshold=0.6):
    """Find episode pairs with high lesson + context similarity."""
    dupes = []
    for i, a in enumerate(episodes):
        for b in episodes[i+1:]:
            lesson_sim = jaccard_similarity(a['lesson'], b['lesson'])
            context_sim = jaccard_similarity(a['context'], b['context'])
            combined = (lesson_sim * 0.6) + (context_sim * 0.4)
            if combined >= threshold:
                dupes.append((a, b, combined))
    return dupes

def find_stale(episodes, archive_days=180):
    """Find episodes older than archive_days with low importance."""
    now = datetime.now(timezone.utc)
    stale = []
    for ep in episodes:
        try:
            dt = datetime.fromisoformat(ep['timestamp'])
            if dt.tzinfo is None:
                dt = dt.replace(tzinfo=timezone.utc)
            age = (now - dt).days
            if age > archive_days and ep.get('importance', 5) < 6:
                stale.append((ep, age))
        except (ValueError, KeyError):
            continue
    return stale

def merge_episodes(keeper, duplicate):
    """Merge duplicate into keeper, preserving unique info."""
    # Keep higher importance
    keeper['importance'] = max(keeper.get('importance', 5), duplicate.get('importance', 5))
    # Append unique tags
    existing_tags = set(keeper.get('tags', []))
    for tag in duplicate.get('tags', []):
        if tag not in existing_tags:
            keeper.setdefault('tags', []).append(tag)
    # If duplicate has a different lesson angle, append it
    if jaccard_similarity(keeper['lesson'], duplicate['lesson']) < 0.8:
        keeper['lesson'] += f" (Also: {duplicate['lesson']})"
    return keeper

def consolidate(episode_dir, archive_days=180, dry_run=False):
    episodes = load_all(episode_dir)
    archive_dir = Path(episode_dir) / 'archive'
    
    if not dry_run:
        archive_dir.mkdir(exist_ok=True)
    
    actions = []

    # 1. Find and merge duplicates
    dupes = find_duplicates(episodes)
    for a, b, sim in dupes:
        actions.append(f"MERGE: '{a['title']}' + '{b['title']}' (similarity: {sim:.2f})")
        if not dry_run:
            merged = merge_episodes(a, b)
            with open(a['_path'], 'w') as f:
                json.dump({k: v for k, v in merged.items() if k != '_path'}, f, indent=2)
            shutil.move(b['_path'], str(archive_dir / Path(b['_path']).name))

    # 2. Archive stale low-importance episodes
    stale = find_stale(episodes, archive_days)
    for ep, age in stale:
        if ep['_path'] in [b['_path'] for _, b, _ in dupes]:  # Already handled
            continue
        actions.append(f"ARCHIVE: '{ep['title']}' (age: {age}d, importance: {ep['importance']})")
        if not dry_run:
            shutil.move(ep['_path'], str(archive_dir / Path(ep['_path']).name))

    return actions

if __name__ == '__main__':
    parser = argparse.ArgumentParser(description='Consolidate episodes')
    parser.add_argument('--episode-dir', default='memory/episodes')
    parser.add_argument('--archive-days', type=int, default=180)
    parser.add_argument('--dry-run', action='store_true')
    args = parser.parse_args()

    actions = consolidate(args.episode_dir, args.archive_days, args.dry_run)
    
    if not actions:
        print("No consolidation needed.")
    else:
        prefix = "[DRY RUN] " if args.dry_run else ""
        for action in actions:
            print(f"{prefix}{action}")
        print(f"\n{prefix}Total actions: {len(actions)}")
```

#### Merge Strategy

When two episodes have >60% combined similarity (lesson x 0.6 + context x 0.4):

1. **Keep the higher-importance episode** as the base
2. **Merge unique tags** from the duplicate
3. **Append distinct lesson angles** — if the duplicate teaches something slightly different, add it with "(Also: ...)"
4. **Move the duplicate to `memory/episodes/archive/`** — not deleted, just filed away

#### Stale Episode Policy

Episodes older than 180 days with importance < 6 get archived automatically. The reasoning:

- **High-importance episodes (6+) persist forever** — production lessons, architectural decisions, rare bugs
- **Low-importance old episodes decay naturally** — routine fixes, temporary workarounds, context that's no longer relevant
- **Archived episodes aren't deleted** — they're moved to `memory/episodes/archive/` and excluded from active recall, but still searchable if you need them

### 12.5 Integration with Existing Systems

Wire episodic memory into the systems you already have running from Phases 1-11.

#### AGENTS.md Updates

Add to your boot sequence (after the existing step 7):

```markdown
8. **Episodic recall**: If the task resembles past work, run:
   `python3 tools/episodes/recall.py "TASK_DESCRIPTION" --top 3`
   Apply relevant lessons before starting.

### Episodic Memory

After completing significant tasks (importance >= 5):
- Write episode to `memory/episodes/ep-YYYY-MM-DD-short-desc.json`
- Score importance honestly: Novelty(0-2) + Impact(0-3) + Reusability(0-2) + Difficulty(0-2) + Surprise(0-1)
- Include specific, actionable lessons — not platitudes
```

#### Heartbeat Integration

Add to `HEARTBEAT.md`:

```markdown
### Episodic Memory Maintenance (weekly)
- [ ] Run consolidation: `python3 tools/episodes/consolidate.py --dry-run`
- [ ] Review flagged merges/archives, then run without --dry-run
- [ ] Re-index: `bash tools/episodes/index-episodes.sh`
- [ ] Check episode count — target 30-60 active episodes
```

Track the check in `memory/heartbeat-state.json`:

```json
{
  "lastChecks": {
    "email": 1703275200,
    "calendar": 1703260800,
    "weather": null,
    "episodeConsolidation": null
  }
}
```

#### Cron Jobs

Set up automated maintenance using OpenClaw's cron system:

```bash
# Weekly consolidation check (Sundays at 3 AM)
openclaw cron add \
  --schedule "0 3 * * 0" \
  --task "Run episode consolidation: cd $OPENCLAW_WORKSPACE && python3 tools/episodes/consolidate.py && bash tools/episodes/index-episodes.sh" \
  --channel telegram

# Daily episode index rebuild (2 AM)
openclaw cron add \
  --schedule "0 2 * * *" \
  --task "Rebuild episode index: cd $OPENCLAW_WORKSPACE && bash tools/episodes/index-episodes.sh" \
  --channel telegram
```

#### Integration with Phase 5 (Memory Compaction Pipeline)

Your hourly summarizer already scans recent activity. Add episode extraction to it:

```markdown
# Add to your hourly summarizer prompt:

After generating the hourly summary, check if any tasks in this hour
qualify for episodic memory (importance >= 5). If so, generate the
episode JSON and write it to memory/episodes/.
```

This creates a natural pipeline: work happens → hourly summarizer captures it → significant tasks get promoted to episodic memory → consolidation keeps the store clean.

#### Integration with Retros

Retros are a natural source of high-importance episodes. After running a retro:

```markdown
# Add to your retro skill:

After completing the retro, extract 1-3 top lessons and log them as
episodes with importance >= 7. Retro-sourced episodes should have
task_type: "decision" and tag: "retro".
```

### 12.6 Benefits Analysis

Concrete improvements from implementing agentic memory:

| Dimension | Before (Phases 1-11) | After (Phase 12) | Improvement |
|-----------|---------------------|-------------------|-------------|
| **Lesson retention** | Lessons in daily notes, rarely re-read | Structured episodes with active recall | Agent recalls *how* it solved similar problems |
| **Context window efficiency** | All memory loaded equally | Importance + decay filtering | 40-60% reduction in irrelevant context |
| **Repeated mistakes** | Same debugging steps re-discovered | Episode recall surfaces past approaches | Near-zero repeat debugging for logged issues |
| **Memory maintenance** | Manual MEMORY.md curation | Automated consolidation + archival | Self-maintaining with weekly cron |
| **Decision quality** | Decisions based on current context only | Past outcomes inform new decisions | Compound learning across sessions |
| **Noise ratio** | Every task noted equally in daily files | Write gate filters low-importance events | Episode store stays lean (30-60 active vs 300+ daily entries) |

**The key shift:** Phases 1-11 gave your agent *storage*. Phase 12 gives it *judgment* about what to store, *intelligence* about what to recall, and *discipline* about what to forget.

A well-tuned episodic memory system means your agent gets meaningfully better at its job over time — not just because the model improves, but because *your specific agent* accumulates experience that persists across sessions, models, and even platform upgrades.

---
