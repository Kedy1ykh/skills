---
name: subsystem-deep-dive
description: Use when someone needs an exhaustive, line-level trace of ONE specific subsystem or mechanism in a codebase, not a whole-repo overview. TRIGGER on "trace exactly how the request pipeline works," "deep dive into the caching layer," "I need to understand the router/scheduler/compaction logic in detail," "what does every stage of X actually do," "walk me through the exact call sequence in Y," or any follow-up after a codebase-onboarding pass that names one subsystem to go deeper on. Produces a single dense reference doc for that one subsystem — exact stage/step sequences with file:line citations, a field-by-field state/metadata-flow table, key constants and thresholds, and explicit naming-trap call-outs. Do NOT use for whole-repo onboarding (use codebase-onboarding instead) or for extracting transferable design lessons for a different project (use engineering-pattern-extraction instead).
---

# Subsystem Deep Dive

Depth instead of breadth. The reader already knows roughly what the system is — they need to modify, debug, or extend *one* mechanism correctly, and a generic architecture overview won't get them there. Success here looks different from `codebase-onboarding`: instead of "does a new hire know what to do on day one," it's "can a contributor debugging this exact subsystem at 2am find the exact function and line they need in under 30 seconds."

## Step 1 — Nail the scope before you go deep

Going deep on the wrong subsystem wastes the entire point of this skill, so confirm the boundary before spawning anything:

- If the requester named a specific module/directory/mechanism, use that as the scope directly.
- If the request is vague ("the caching stuff," "how routing works"), do a 2-minute orientation pass yourself — grep for the term, skim the matching directory names, glance at import graphs if `codebase-memory` MCP tools are available — and state back what you think the boundary is before committing real effort. A wrong guess here costs the whole pass; a two-line confirmation costs nothing.
- Decide whether the subsystem has internal sub-parts worth splitting across workers (e.g. "the provider layer" might split into "the protocol/selector" and "the per-vendor adapter quirks") or whether it's cohesive enough for one worker to cover end to end. Most subsystems are the latter — don't manufacture a fan-out where one focused pass would do.

## Step 2 — Dispatch with a much narrower, much deeper brief

Use `subagent_type: "codebase-archaeologist"` if installed (same worker `codebase-onboarding` uses), but the brief is different in kind, not just scope:

```
Slice: <the exact subsystem>, e.g. "the request-routing pipeline in src/engine/".

This is a DEEP trace, not a broad survey. For this one subsystem:
1. Trace the exact ordered sequence of stages/steps/functions a request
   or unit of work passes through, with a file:line citation for each step.
2. If state/config/decisions flow between steps via a shared object (a
   context dict, a metadata bag, a config record), build a field-by-field
   table: field name, who writes it, who reads it, what it's for, cited.
3. Extract every constant, threshold, magic number, or tunable default
   that controls this subsystem's behavior, with its file:line and, if
   findable, why that value (a comment, a test, a config default).
4. Flag any naming trap: the same name reused for two different things
   here, a name that implies one behavior but delivers another.
5. Note where this subsystem's behavior is a deliberate design decision
   vs. where it looks like an accident or unaddressed gap.

Cite file:line for near every sentence — this doc's whole value is that a
reader can jump straight to the code. Return findings as your final
message text, not a file.
```

If you split into sub-parts (Step 1), give each worker a comparably narrow, comparably deep brief for its slice, and be explicit about the shared field/state table so their pieces reassemble cleanly — this is the one place duplication across workers is actively bad, not just tolerable, because the field-flow table needs to be one coherent table, not two overlapping ones.

## Step 3 — Synthesize into one dense reference doc, not the 12-item template

This is a single-purpose technical reference, not a stakeholder report — skip the `codebase-onboarding` Section A/B shape entirely. Structure:

1. **A compact flow diagram** (ASCII or Mermaid, whichever is clearer for this mechanism) showing the ordered sequence at a glance.
2. **The ordered stage/step table** — one row per step, with what it does and its file:line citation.
3. **The field/state-flow table** — if the subsystem passes state between steps, this table is often the single most valuable artifact in the whole doc, since it's rarely written down anywhere else.
4. **Constants and thresholds table** — value, meaning, citation, and "why this value" if you found evidence for it.
5. **Naming traps**, if any were found — don't force this section if the subsystem is cleanly named.
6. **One short "for future editors" paragraph** (optional) — the kind of thing you'd tell a teammate before they touch this code for the first time. This is *not* the full copy/avoid/build-order treatment (that's `engineering-pattern-extraction`'s job) — just a sentence or two of hard-won context, only if you have something real to say.

There's no fixed length target — depth scales to the subsystem's actual complexity. A three-function utility doesn't need six sections; a multi-stage pipeline with a dozen config-driven branches might need all of them and run long. What doesn't scale down is citation density — this doc earns its keep by being more exhaustively cited than a broad onboarding report, not less.

## Step 4 — Self-check

- Could a contributor use this doc alone to find the exact line responsible for a specific behavior, without grepping first?
- Does every step in the ordered sequence have a citation, not just the ones that were easy to find?
- If there's a field/state-flow table, does it account for every field that crosses a step boundary, or only the obvious ones?
- Is anything here duplicating what a full `codebase-onboarding` pass would already say at the architecture level? If so, cut it — this doc should add depth, not repeat breadth.

## Output

Default to one Markdown file (no HTML/diagram-bundling requirement — this is a technical reference read in an editor or terminal, not a stakeholder deliverable) at `docs/<subsystem-name>-deep-dive.md` in the target repo, unless the requester specifies otherwise. Mermaid diagrams are welcome if they clarify a flow, but plain ASCII is fine too and avoids the offline-rendering concerns that matter for `codebase-onboarding`'s HTML output.

## Relationship to codebase-onboarding

Run `codebase-onboarding` first if nobody has mapped the system's seams yet — it tells you what subsystems exist and roughly how they fit together, which is exactly the input Step 1 here needs. This skill is what you reach for *after* that pass names something worth going deeper on, or when a request already arrives pre-scoped to one mechanism.
