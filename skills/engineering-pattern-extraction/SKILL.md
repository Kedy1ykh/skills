---
name: engineering-pattern-extraction
description: Use when someone wants transferable engineering lessons from a codebase to apply to a DIFFERENT, similar project they're building — not to understand this codebase for its own sake. TRIGGER on "if I were building something like this, what should I copy or avoid," "extract the design patterns from this codebase," "what can we learn from how X built their Y," "give me a build-order playbook based on this system's architecture," "what would you steal from this repo," or similar. Produces a compact patterns-to-copy vs. traps-to-avoid comparison, a recommended build order, and a one-paragraph thesis — written for someone designing a new system, not for someone onboarding onto this one. Do NOT use for understanding this codebase itself (use codebase-onboarding instead) or for an exhaustive trace of one mechanism (use subsystem-deep-dive instead).
---

# Engineering Pattern Extraction

The reader isn't joining this codebase — they're building something else and want to know what to steal and what to dodge. That reframes everything: you're not covering the system, you're mining it for the handful of decisions that are actually load-bearing and actually transferable. A pattern-extraction doc that tries to say something about every corner of the codebase is doing `codebase-onboarding`'s job badly instead of doing its own job well.

## Step 1 — Find the load-bearing decisions, not the whole system

Every codebase worth this treatment has one or two decisions that make it interesting or hard — the thing that, if you got it wrong, the whole system would be worse or wouldn't work at all. Everything else is implementation detail that doesn't generalize. Spend your attention finding those decisions before you spend any of it writing:

- What's the one expensive, risky, or hard-to-get-right thing this system does, and what did it decide to do about it? (A system built around calling an LLM cheaply will have a cost-minimization decision at its center; a system built around running untrusted code will have an isolation decision; a system built around eventual consistency will have a reconciliation decision.)
- What would a naive first attempt at this system have gotten wrong, and what does this codebase do instead? Traps are often easier to spot than copy-worthy patterns, because you can see the scar tissue (a comment, a test, a "we used to do X but" note) — work backward from those.
- If a `codebase-onboarding` or `subsystem-deep-dive` pass already exists for this repo, read it first — it's already done the discovery work of finding what the subsystems are; you're doing a different, narrower synthesis pass on top, not re-discovering the system from scratch.

## Step 2 — Research only the load-bearing parts, deeply

Dispatch `codebase-archaeologist` workers (or reuse existing research from Step 1) scoped tightly to the decisions you identified — not one worker per directory, one worker per *decision*. A single load-bearing decision often spans multiple files/modules (a routing decision touches the classifier, the config, and the call sites that consult it); scope the worker to the decision, and let it follow the decision across file boundaries.

```
Decision to investigate: <e.g. "how this system decides which of ~20
downstream providers to call and keeps that swap-able without code changes">

Find: the exact mechanism, the file:line evidence, what would have gone
wrong with an obvious simpler alternative, any comments/tests/docs that
reveal the reasoning, and any place the decision is NOT honored
consistently (a place where the abstraction leaks or was bypassed).

Cite file:line throughout. Return findings as your final message text.
```

## Step 3 — Synthesize as advice for a builder, not a description of this system

Structure the output around the reader's actual question:

1. **COPY** — a short list (as many as are genuinely load-bearing, not padded to a round number), each with: the pattern in one line, why it matters, and a citation. If a pattern only makes sense combined with another, say so rather than listing them as independent choices.
2. **AVOID** — a short list of traps, each with: what goes wrong if you do the naive thing, what this codebase does instead, and a citation to the evidence (a comment, a test name, a changelog entry) that shows this was a real problem, not a hypothetical one.
3. **Build order** — a numbered sequence for building something similar from scratch, where each step is justified by depending on the one before it (you can't sensibly add a cost-routing layer before you have a working request loop to route through, for instance). This is prescriptive and opinionated — that's the point, don't hedge it into uselessness.
4. **One-paragraph thesis** — if you had to tell someone the single sentence that captures why this architecture works, what would it be? This is the hardest part to write and the most valuable; don't skip to a generic platitude ("good architecture matters") — it should be specific enough that it would be *wrong* advice for a sufficiently different kind of system.

Be honest about scale: if the codebase doesn't actually have much worth extracting (a straightforward CRUD app with no unusual decisions), say that plainly instead of manufacturing lessons to fill out the sections. A short, honest doc beats a padded one that oversells a boring codebase's cleverness.

## Step 4 — Self-check

- Could someone who has never opened this codebase's source make good early decisions on a similar project using only this doc?
- Is every COPY/AVOID item something a *different* system would plausibly also need, or did you list something so specific to this codebase's domain that it wouldn't transfer?
- Does the build order actually reflect dependency order, or is it just the order this codebase happened to grow in?
- Would the thesis sentence be *wrong* advice for a sufficiently different system? If it's true of almost any well-built system, it's too generic — sharpen it.

## Output

One Markdown file, no HTML/diagram-bundling requirement. Default to `docs/<project-name>-build-guidance.md` in the target repo unless told otherwise. Keep it short — this genre earns its value from being distilled, not exhaustive; if it's approaching the length of a full onboarding report, you've drifted into describing the system instead of extracting lessons from it.
