# OpenClaw Production Upgrades — Phase 3

## Phase 3: Shared Brain (Multi-Agent Coordination)

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
