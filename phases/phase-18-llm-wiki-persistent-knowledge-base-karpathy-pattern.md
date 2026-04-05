# OpenClaw Production Upgrades — Phase 18

## Phase 18: LLM Wiki — Persistent Knowledge Base (Karpathy Pattern)

Inspired by Andrej Karpathy's "llm-wiki" gist (2026). The core problem: most agents treat knowledge like RAG — re-discover from scratch on every query. Nothing accumulates. Ask a question that requires synthesizing five documents, and the agent pieces fragments together every time.

This phase builds a **persistent, compounding wiki** — structured, interlinked markdown files that the agent maintains. When you add a new source, the agent doesn't just index it. It reads it, extracts key information, and **integrates it into existing wiki pages** — updating entities, revising summaries, flagging contradictions.

### 18.1 Three Layers

| Layer | What It Is | Who Maintains It |
|-------|-----------|------------------|
| **Raw Sources** | `memory/ingested/` — articles, papers, transcripts | Immutable. Agent reads but never modifies. |
| **The Wiki** | `memory/wiki/entities/` + `memory/wiki/concepts/` | Agent owns entirely. Creates, updates, cross-references. |
| **The Schema** | `AGENTS.md` instructions | You and the agent co-evolve. |

### 18.2 Wiki Structure

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

### 18.3 Operations

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

### 18.4 Contradiction Detection

When new information arrives, check if it conflicts with existing wiki pages:

```bash
bash tools/wiki/check-contradictions.sh
```

Flags:
- Price/number claims that differ between pages
- Status claims that conflict (one page says "active", another says "blocked")
- Date claims that are inconsistent
- Stale financial data from older sources

### 18.5 Integration

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

### 18.6 Key Insight

From Karpathy: *"The tedious part of maintaining a knowledge base is not the reading or the thinking — it's the bookkeeping. Updating cross-references, keeping summaries current, noting when new data contradicts old claims. Humans abandon wikis because the maintenance burden grows faster than the value. LLMs don't get bored and can touch 15 files in one pass."*

The human's job: curate sources, direct analysis, ask good questions, think about what it means.
The agent's job: everything else.

### 18.7 Benefits

| Before | After |
|--------|-------|
| Ingested articles sit as standalone files | Each ingest updates entity/concept pages across the wiki |
| No cross-referencing between sources | `[[wikilinks]]` connect related knowledge automatically |
| Contradictions go unnoticed | Contradiction detector flags conflicting claims |
| Stale info stays forever | Lint catches outdated pages, low-source claims |
| Knowledge scatters | Knowledge compounds — richer with every source added |
| Ask the same synthetic question repeatedly | Synthesis exists as persistent wiki pages |
