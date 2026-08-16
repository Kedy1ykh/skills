# codebase-skills

A small Claude Code plugin for understanding codebases you didn't write — fast, honestly, and with citations. Five skills, four supporting agents, one shared discipline: **cite the fact, flag the guess, and scale the effort to the system instead of to a checklist.**

> **License note:** this repo currently ships without a `LICENSE` file, so it's all-rights-reserved by default. Add a license before relying on others being able to reuse it.

---

## 1. Summary and Quickstart

| Skill | Use it when | Produces |
|---|---|---|
| [`codebase-onboarding`](skills/codebase/codebase-onboarding/SKILL.md) | You're cold-opening an unfamiliar repo and need the whole picture — business context, architecture, domain model, risks, unknowns. | A cited Markdown report + a self-contained offline HTML version (diagrams included, no CDN dependency). |
| [`subsystem-deep-dive`](skills/codebase/subsystem-deep-dive/SKILL.md) | You already know roughly what the system is and need an exhaustive, line-level trace of *one* mechanism to safely modify or debug it. | One dense reference doc: exact stage sequence, a field-level state-flow table, constants/thresholds, naming traps. |
| [`engineering-pattern-extraction`](skills/codebase/engineering-pattern-extraction/SKILL.md) | You're building something *different but similar* and want to know what to copy and what to avoid — not to understand this codebase for its own sake. | A COPY/AVOID comparison, a recommended build order, and a one-paragraph thesis. |
| [`docs-drift-audit`](skills/codebase/docs-drift-audit/SKILL.md) | A codebase already has architecture docs and you want to know which parts are now stale relative to recent commits. | A per-doc staleness table (unchanged / needs-update / high-priority) with exact commit citations. |
| [`pattern-finder`](skills/codebase/pattern-finder/SKILL.md) | You're about to write new code in a codebase you already work in and want to follow existing convention instead of inventing a new approach. | A short in-chat answer: the real precedent(s) found, file:line citations, and which one to model your code after. |

All five live under [`skills/codebase/`](skills/codebase/README.md) — a category grouping, mirroring [`skills/productivity/`](skills/productivity/README.md) for the general-purpose, non-code-specific skills kept alongside them in this same tree.

Four supporting agents do the actual file-reading legwork so the skills stay focused on orchestration and synthesis:

- [`codebase-archaeologist`](agents/codebase-archaeologist.md) — reconstructs one slice of a system from evidence alone; the fan-out worker behind `codebase-onboarding`, `subsystem-deep-dive`, and `engineering-pattern-extraction`.
- [`docs-drift-auditor`](agents/docs-drift-auditor.md) — checks one documentation file's claims against git history since a baseline; the fan-out worker behind `docs-drift-audit`.
- [`codebase-locator`](agents/codebase-locator.md) — fast, read-only "where does X live" lookups, without deep analysis; a cheap first call other skills can reach for instead of spending an archaeologist's budget on a pure location question.
- [`codebase-pattern-finder`](agents/codebase-pattern-finder.md) — surveys several existing implementations of the same shape and recommends which to copy; the worker behind `pattern-finder`.

You don't invoke the agents directly in normal use — the skills dispatch them. They're listed separately here because Claude Code loads and can select agents independently of the skill that "owns" them.

### Quickstart — once installed, just ask naturally

You don't need slash commands or special syntax. Claude Code picks the right skill from what you ask for:

```
"Onboard me to this repo, I'm new here."
"Explain this codebase — what does it actually do end to end?"
"Trace exactly how the request pipeline works, file by file."
"If I were building something like this myself, what should I copy or avoid?"
"Are the docs in docs/architecture-notes still accurate? We've had ~100 commits since they were written."
"Is there an existing pattern for retrying a failed outbound call here? I'm about to add another one."
```

Each skill scales its own effort — `codebase-onboarding` reads a 200-line CLI tool itself instead of spinning up four parallel agents on it, and fans out one worker per real architectural seam on a large monorepo instead of forcing a fixed count either way.

---

## 2. How to install as a Claude Code plugin

### Option A — install from GitHub as a plugin (recommended once pushed)

From inside Claude Code:

```
/plugin marketplace add Kedy1ykh/skills
/plugin install codebase-skills@codebase-skills
```

The first command registers this repo as a plugin source (it works because the repo ships a `.claude-plugin/marketplace.json` at its root, self-referencing its own plugin — the same pattern used by other single-plugin Claude Code repos). The second command installs the plugin itself. No restart needed; the skills and agents become available in your next turn.

Confirm it worked:

```
/plugin
```

should list `codebase-skills` as installed and enabled, and the five skills above should appear when Claude Code lists available skills at the start of a session.

### Option B — local development / no GitHub required

If you just want to try it from a local clone without publishing anywhere:

```bash
git clone https://github.com/Kedy1ykh/skills.git
claude --plugin-dir /path/to/skills
```

This loads the plugin directly from disk for that session — useful for testing changes before pushing, or if you'd rather not add a marketplace at all.

### Option C — copy the pieces manually (no plugin machinery)

If you don't want the plugin system involved at all, the skills and agents are just files — copy them straight into Claude Code's personal (cross-project) directories:

```bash
for s in codebase-onboarding subsystem-deep-dive engineering-pattern-extraction docs-drift-audit pattern-finder; do
  mkdir -p ~/.claude/skills/"$s"
  cp -r skills/codebase/"$s"/. ~/.claude/skills/"$s"/
done
cp agents/*.md ~/.claude/agents/
```

This makes them available in every project on the machine, same as Option A/B, just without the plugin manifest/versioning layer.

---

## 3. Why these skills exist

**The problem:** asking an AI assistant to "explain this codebase" tends to produce one of two failure modes — a shallow, generic summary that could describe almost any project, or an exhaustive dump that buries the one thing a new engineer actually needed to know. Neither is honest about what it actually verified versus guessed, and neither knows when to stop.

**`codebase-onboarding`** came first, out of doing this for real on an unfamiliar multi-language monorepo: index the repo, fan out parallel research across business context, architecture, domain model, data model, security, and operations, then synthesize into a fixed set of deliverables — but with two non-negotiable disciplines baked in. First, **every factual claim needs a `file:line` citation, and every claim that can't be verified from code (who pays for this, who the real users are) gets flagged, not guessed at confidently.** A report that quietly presents guesses as facts is worse than no report. Second, **the HTML deliverable has to render offline** — an early version pointed its diagram library at a public CDN, and it silently failed behind a corporate proxy with an untrusted root CA, rendering as raw `flowchart TB` text with no error. Every diagram-bearing report this plugin produces now bundles its rendering library locally instead of trusting a CDN to be reachable.

That skill was then run through a real with-skill-vs-baseline benchmark (spawn a fresh agent with the skill, spawn a fresh agent without it, grade both against the same assertions) on two structurally different repos — a large monorepo and a tiny single-script tool. The result was clarifying: **a capable model already does most of the investigative depth unaided.** What the skill actually adds is disciplined output completeness — the second file format, the real rendered diagrams, the explicit "not applicable" markers instead of silent gaps, the honest unknowns section. That's a useful, measurable thing for a skill to guarantee, even when the underlying research quality converges.

**The other three skills exist because of a comparison, not a hunch.** Running `codebase-onboarding`'s output next to a different, hand-written architecture-notes corpus for another codebase surfaced real gaps — but gaps that turned out to belong to *different audiences*, not to a missing twelfth section:

- An exhaustive, line-by-line trace of one mechanism is a different document for a different reader (someone about to modify that exact code) than a whole-repo onboarding report (someone deciding where to start). Forcing that depth into every subsystem of a broad onboarding pass would make it unreadable; leaving it out entirely leaves a real need unmet. That split became **`subsystem-deep-dive`**.
- Extracting "what should I copy or avoid if I build something similar" is a question for someone building a *different* system, not someone joining *this* one — a fundamentally different synthesis question even when the underlying research overlaps. That became **`engineering-pattern-extraction`**.
- Checking whether existing docs are still true against recent commits is a diff-and-verify operation, not a generate-fresh one — it needs a baseline, a commit range, and a worker built to distinguish "this file changed" from "this specific claim is now wrong." That became **`docs-drift-audit`**, backed by its own **`docs-drift-auditor`** agent because that verification loop is genuinely different from reconstructing something from scratch.

**`pattern-finder`, `codebase-locator`, and `codebase-pattern-finder` came from comparing this plugin against a different, narrower family of research subagents** — the kind that just locate or explain, with no orchestration layer on top. That comparison went both ways: this plugin's synthesis and deliverable discipline is the thing that family doesn't have, but it also doesn't have any answer at all for "I'm about to write code in this repo, show me how we already do this" — every one of the original four skills is about *understanding* a system, none of them is about *matching* it while writing new code. That's a different question from all four originals: narrower in scope than onboarding or a deep-dive, and (unlike `engineering-pattern-extraction`) answered for someone who's staying in this repo, not leaving it. **`pattern-finder`** exists for exactly that moment, backed by **`codebase-pattern-finder`**, a worker that surveys several existing implementations of the same shape instead of explaining just one — and explicitly surfaces it when two precedents disagree, rather than picking one silently. Splitting **`codebase-locator`** out of `codebase-archaeologist` at the same time closed a smaller, related gap: a pure "where does X live" question was previously either answered by spinning up a full analysis worker (expensive for what it is) or by the orchestrating skill doing it inline with no consistent methodology.

Each skill's frontmatter says explicitly what the *other* skills are for, so a router picking between them has a real shot at getting it right instead of defaulting to whichever loads first.

---

## Repository layout

```
codebase-skills/
├── .claude-plugin/
│   ├── plugin.json         # plugin manifest
│   └── marketplace.json    # self-referencing marketplace (enables Option A install)
├── skills/
│   ├── codebase/                            # category: understanding & fitting into codebases
│   │   ├── README.md
│   │   ├── codebase-onboarding/
│   │   │   ├── SKILL.md
│   │   │   └── evals/evals.json
│   │   ├── subsystem-deep-dive/
│   │   │   ├── SKILL.md
│   │   │   └── evals/evals.json
│   │   ├── engineering-pattern-extraction/
│   │   │   ├── SKILL.md
│   │   │   └── evals/evals.json
│   │   ├── docs-drift-audit/
│   │   │   ├── SKILL.md
│   │   │   └── evals/evals.json
│   │   └── pattern-finder/
│   │       ├── SKILL.md
│   │       └── evals/evals.json
│   └── productivity/                        # category: general workflow, not code-specific
│       ├── README.md
│       └── ... (grill-me, handoff, teach, writing-for-agents, ...)
├── agents/
│   ├── codebase-archaeologist.md
│   ├── docs-drift-auditor.md
│   ├── codebase-locator.md
│   └── codebase-pattern-finder.md
└── README.md
```

## Contributing / feedback

This is a personal skills repo, evolved from real use rather than designed up front. If a skill mis-triggers, produces a report that quietly guesses instead of flagging, or you hit a case none of the five cover well, open an issue with the prompt that caused it — that's exactly the kind of feedback that shaped the naming-traps and domain-adaptive-framing additions already baked into `codebase-onboarding`.
