# OpenClaw Production Upgrades — Phase 10

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

### Build Harness Skill Files

| Skill | Path | Purpose |
|-------|------|---------|
| evaluator | `skills/evaluator/SKILL.md` | Skeptical critic with 4 criteria, browser testing, anti-leniency |
| build-harness | `skills/build-harness/SKILL.md` | 3-agent orchestration: planner → generator → evaluator |
| jonyive | `skills/jonyive/SKILL.md` | Design principles mapped to evaluator criteria |

---
