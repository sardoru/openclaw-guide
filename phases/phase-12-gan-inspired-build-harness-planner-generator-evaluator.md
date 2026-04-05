# OpenClaw Production Upgrades — Phase 12

## Phase 12: GAN-Inspired Build Harness (Planner → Generator → Evaluator)

Single-agent builds have a ceiling. The agent builds something, evaluates its own work, praises it, and ships broken code. This phase adds a multi-agent architecture inspired by [Anthropic's harness research](https://www.anthropic.com/engineering/harness-design-long-running-apps) that separates building from judging — the same insight behind Generative Adversarial Networks.

**The core problem:** LLMs are terrible at evaluating their own work. They find bugs, then talk themselves into deciding they're not a big deal. They approve stubbed features. They call mediocre design "clean and modern." Separating the builder from the critic is the single biggest quality lever.

### 12.1 Architecture

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

### 12.2 The Evaluator — Calibrated Skepticism

The evaluator scores against 4 weighted criteria:

| Criterion | Weight | Min to Pass | What It Checks |
|-----------|--------|-------------|----------------|
| Design Quality | 30% | 5/10 | Cohesive palette, typography hierarchy, spacing rhythm, distinct mood |
| Originality | 25% | 5/10 | Custom decisions vs AI slop patterns |
| Craft | 20% | 6/10 | Font scale, spacing base unit, WCAG contrast, hover/focus states |
| Functionality | 25% | 7/10 | Every button works (not just logs), forms submit, data persists |

**Overall pass:** ALL criteria meet minimum AND weighted average ≥ 6.0.

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
- The 3-column feature grid (icon-in-circle + bold title + 2-line description × 3)
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

### 12.3 The Planner — $0.50 That Saves Hours

The planner takes a short prompt and produces a full product spec. This is the highest-ROI component — 5 minutes and $0.50 of planning prevents hours of building the wrong thing.

Planner instructions:
- **Be ambitious about scope** — feature-rich, not minimal
- **Stay high-level** — describe WHAT, not HOW (technical errors in the spec cascade into the build)
- **Create a design language** — mood, color direction, typography, layout philosophy
- **Include anti-slop guidance** — explicitly ban the blacklisted patterns
- **Weave in AI features** where they'd genuinely add value

Output: `.harness/spec.md` (product spec) + `.harness/design-language.md` (visual direction)

**Critical:** Present the spec to the human for review before building. This is the checkpoint.

### 12.4 Sprint Contracts

Before each chunk of work, the generator and evaluator negotiate what "done" looks like:

1. **Generator proposes** `.harness/contracts/sprint-01-proposal.md`:
   - Features to implement
   - Definition of done (specific, testable behaviors)
   - Verification method (how to test each criterion)

2. **Evaluator reviews** and either approves or requests changes

3. **Both agree** before any code is written

This prevents the generator from building the wrong thing or stubbing features. The contract is what the evaluator scores against — not vibes, not the spec's vague user stories.

### 12.5 Sprint vs Continuous Mode

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

### 12.6 File-Based Communication

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

### 12.7 Implementation Options

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

### 12.8 The Numbers

From Anthropic's benchmarks:

| Approach | Duration | Cost | Quality |
|----------|----------|------|---------|
| Solo (no harness) | 20 min | ~$9 | Core features often broken |
| Full harness (Opus 4.5) | 6 hr | ~$200 | Working, polished, feature-rich |
| Simplified harness (Opus 4.6) | 4 hr | ~$125 | Same quality, less overhead |

The harness is 10-20x more expensive. The output is categorically different — not incrementally better, but the difference between "core feature doesn't work" and "everything works, and it looks good."

### 12.9 The Key Principle

> "Every component in a harness encodes an assumption about what the model can't do on its own. Those assumptions are worth stress testing, both because they may be incorrect, and because they can quickly go stale as models improve."

Periodically strip components and test. What was needed for Sonnet 4.5 (sprint decomposition, context resets) became unnecessary on Opus 4.6. The interesting work is finding the next novel combination, not cargo-culting last month's scaffolding.

### Skill Files

Create these skills in your `skills/` directory:

**`skills/evaluator/SKILL.md`** — Full evaluator instructions with scoring rubric, anti-leniency rules, AI slop blacklist, few-shot calibration examples, and browser testing process.

**`skills/build-harness/SKILL.md`** — Orchestration pattern documentation covering all three phases, sprint contract templates, mode selection heuristics, and implementation options.

---
