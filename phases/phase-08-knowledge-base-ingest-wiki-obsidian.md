# OpenClaw Production Upgrades — Phase 8

## Phase 8: Knowledge Base — Ingest, Wiki & Obsidian

This phase builds a complete knowledge pipeline: ingesting external content (URLs, videos, articles) into structured memory, maintaining a persistent wiki that compounds over time, and syncing it all with Obsidian as your visual graph layer. The three components form a knowledge stack — raw sources at the bottom, curated wiki in the middle, Obsidian graph at the top.

### Part A: Brain-Ingest Pipeline

#### 8.1 Brain-Ingest — `tools/brain-ingest/brain-ingest.sh`

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

#### 8.2 Skill File — `skills/brain-ingest/SKILL.md`

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

### Part B: LLM Wiki — Persistent Knowledge Base (Karpathy Pattern)

Inspired by Andrej Karpathy's "llm-wiki" gist (2026). The core problem: most agents treat knowledge like RAG — re-discover from scratch on every query. Nothing accumulates. Ask a question that requires synthesizing five documents, and the agent pieces fragments together every time.

The wiki builds on the brain-ingest pipeline: when you ingest a new source, the agent doesn't just index it. It reads it, extracts key information, and **integrates it into existing wiki pages** — updating entities, revising summaries, flagging contradictions.

#### 8.3 Three Layers

| Layer | What It Is | Who Maintains It |
|-------|-----------|------------------|
| **Raw Sources** | `memory/ingested/` — articles, papers, transcripts | Immutable. Agent reads but never modifies. |
| **The Wiki** | `memory/wiki/entities/` + `memory/wiki/concepts/` | Agent owns entirely. Creates, updates, cross-references. |
| **The Schema** | `AGENTS.md` instructions | You and the agent co-evolve. |

#### 8.4 Wiki Structure

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

#### 8.5 Operations

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

#### 8.6 Contradiction Detection

When new information arrives, check if it conflicts with existing wiki pages:

```bash
bash tools/wiki/check-contradictions.sh
```

Flags:
- Price/number claims that differ between pages
- Status claims that conflict (one page says "active", another says "blocked")
- Date claims that are inconsistent
- Stale financial data from older sources

### Part C: Obsidian Integration

The wiki uses `[[wikilinks]]` throughout, so opening `memory/wiki/` in Obsidian gives you a full graph view of your knowledge base with all cross-references visualized.

#### 8.7 Integration

**AGENTS.md** — Add to boot sequence:
```markdown
### Knowledge Base
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

### 8.8 The Full Knowledge Pipeline

```
External content (URLs, videos, articles)
  → brain-ingest (download, transcribe, extract)
    → memory/ingested/ (raw, immutable source)
      → Wiki update (entities + concepts pages updated/created)
        → Obsidian vault (graph view, wikilinks, frontmatter)
          → Vector index (FAISS semantic search across everything)
```

Each stage adds value:
- **brain-ingest** captures and structures raw content
- **memory/ingested/** preserves the source of truth
- **Wiki** synthesizes across sources, maintains entity pages, detects contradictions
- **Obsidian** provides visual graph exploration and human editing
- **Vector index** enables semantic search across the entire knowledge base

### 8.9 Key Insight

From Karpathy: *"The tedious part of maintaining a knowledge base is not the reading or the thinking — it's the bookkeeping. Updating cross-references, keeping summaries current, noting when new data contradicts old claims. Humans abandon wikis because the maintenance burden grows faster than the value. LLMs don't get bored and can touch 15 files in one pass."*

The human's job: curate sources, direct analysis, ask good questions, think about what it means.
The agent's job: everything else.

### 8.10 Benefits

| Before | After |
|--------|-------|
| URLs saved but never processed | brain-ingest extracts, structures, and indexes automatically |
| Ingested articles sit as standalone files | Each ingest updates entity/concept pages across the wiki |
| No cross-referencing between sources | `[[wikilinks]]` connect related knowledge automatically |
| Contradictions go unnoticed | Contradiction detector flags conflicting claims |
| Stale info stays forever | Lint catches outdated pages, low-source claims |
| Knowledge scatters across tools | Single pipeline: ingest → wiki → Obsidian → vector search |
| Ask the same synthetic question repeatedly | Synthesis exists as persistent wiki pages |

---
