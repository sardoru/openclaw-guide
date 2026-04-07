# OpenClaw Production Upgrades — Phase 5

## Phase 5: Memory Compaction Pipeline (Hourly + Weekly)

Raw memory files grow fast. Without compaction, your agent drowns in context. This phase builds a two-stage pipeline: hourly summarization captures what happened in near-real-time, and weekly compounding distills those summaries into durable knowledge. Together they keep your agent's context fresh without losing signal.

### Stage 1: Hourly Summarizer + Context Injection

Two scripts work together: the summarizer captures what happened each hour, and the context injector compiles recent context into a single file that survives session compaction.

#### 5.1 Hourly Summarizer — `tools/hourly-summarizer.py`

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

#### 5.2 Context Injector — `tools/context-injector.py`

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

### Stage 2: Weekly Memory Compound

Runs Sunday night. Distills the week's daily logs, hourly summaries, and feedback into a concise weekly summary — the second compression stage in the pipeline.

#### 5.3 Weekly Compound — `tools/weekly-compound.py`

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

### 5.4 The Full Pipeline

The two stages form a natural compression pipeline:

```
Session transcripts (raw, unbounded)
  → Hourly summaries (every :55, last 20 messages → ~1 page)
    → Context injection (last 4 hourly summaries → CONTEXT_INJECTION.md)
      → Weekly compound (Sunday 10 PM → memory/weekly/YYYY-WXX.md)
        → MEMORY.md (significant learnings promoted to long-term memory)
```

Each stage compresses further:
- **Hourly:** ~20 messages → 1 page of bullets
- **Context injection:** Last 4 hours → single file for boot
- **Weekly:** 7 days of logs + hourly + feedback → 1-2 page summary
- **MEMORY.md:** Significant learnings only → permanent index

This means your agent always has fresh context (hourly), recent history (context injection), and durable knowledge (weekly + MEMORY.md) — without the context window filling up with raw transcripts.

---
