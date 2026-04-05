# OpenClaw Production Upgrades — Phase 7

## Phase 7: Cross-Agent Synthesis (Roundtable)

The roundtable script reads all agent outputs from the last 24 hours, extracts entities (people, companies, tickers, countries), detects cross-agent themes, and writes a daily synthesis.

### 7.1 Roundtable — `tools/roundtable.py`

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
