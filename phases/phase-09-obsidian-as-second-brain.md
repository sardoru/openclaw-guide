# OpenClaw Production Upgrades — Phase 9

## Phase 9: Obsidian as Second Brain

### 9.1 Brain-Ingest Pipeline — `tools/brain-ingest/brain-ingest.sh`

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

### 9.2 Skill File — `skills/brain-ingest/SKILL.md`

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

---
