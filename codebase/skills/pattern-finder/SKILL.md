---
name: pattern-finder
description: Use when someone is about to write new code in a codebase and wants to follow existing convention rather than invent a new approach — "how do we normally do X here," "show me an example of Y elsewhere in this repo," "before I add this feature, what's the established pattern for it," "is there already a way we do this." TRIGGER on "is there an existing pattern for," "how does this codebase usually handle," "find me an example to model this after," "what's the convention here for," or any moment right before implementing a feature in a codebase someone already works in. Produces a short, in-chat list of real in-repo precedents with file:line citations, what differs between them, and a recommendation for which to follow. Do NOT use this for mining a codebase for lessons to apply to a DIFFERENT project (use engineering-pattern-extraction instead) and do NOT use it for whole-repo or single-mechanism understanding (use codebase-onboarding or subsystem-deep-dive instead) — this skill answers "what should my new code look like," nothing broader.
---

# Pattern Finder

The reader is about to write code, not study the system. That reframes the job: don't explain how the codebase works in general, find the *specific* existing precedent for the *specific* thing being built, and give a recommendation the reader can act on in the next few minutes — not a doc to read later.

This is the smallest-scoped skill in this set on purpose. If the question turns out to be broader than "what should my new code look like," that's a sign to hand off rather than stretch this skill to cover it — see Relationship section below.

## Step 1 — Narrow the shape before searching anything

"How do we do validation" is too broad to search well. Narrow it with the reader if needed, or narrow it yourself and state the narrowing back to them: "validation on a single form field on submit" is a different search than "validation across a multi-step wizard." A search scoped to the wrong shape wastes the one dispatch this skill is supposed to need.

If the reader already named the exact mechanism ("how do we paginate list endpoints"), skip straight to Step 2.

## Step 2 — Dispatch

Most requests need exactly **one** `codebase-pattern-finder` agent call — this skill's entire value is being fast, so don't manufacture a multi-agent fan-out for a single pattern lookup. Dispatch more than one only when the reader is asking about genuinely separate shapes at once (e.g. "how do we do both retries and idempotency keys for outbound calls" — two different mechanisms, two dispatches, run in parallel in one message).

```
Pattern to investigate: <the narrowed shape from Step 1, as concrete as possible>

Find 2-4 real, working examples of this shape already in the codebase.
For each: file:line, what it actually does, and anything relevant to
picking a template (stale per a comment/test, handles an edge case the
others miss, etc). If examples disagree on approach, say so explicitly.
If nothing similar exists, say that plainly rather than inventing an
idealized example. Recommend which one to model new code after, with
reasoning. Return findings as your final message text.
```

If `codebase-pattern-finder` isn't installed in the current environment, fall back to `general-purpose` or `fork` and paste the operating principles inline (concrete shape, real examples only, surface disagreement, no fabricated best-practice code).

## Step 3 — Hand back an answer, not a report

Structure the response so the reader can act immediately:

```
**Convention here:** <one line — what this codebase does for this shape>

**Best example to follow:** `file:line` — <why this one>

**Other examples:** `file:line`, `file:line` — <one line each, only if
genuinely useful as alternates or if they disagree with the recommended one>

**Heads up:** <any inconsistency across examples, or "nothing like this
exists yet" if that's the honest answer — don't skip this line>
```

Default to answering inline as chat text — this is a quick, in-the-moment lookup, not a stakeholder deliverable. Only write a file if the reader is asking about several patterns at once and wants something to reference later; if so, a short Markdown file at `docs/<topic>-patterns.md` is fine, but ask first rather than assuming.

## Step 4 — Self-check

- Did you actually narrow the pattern to something searchable, or did you search on the reader's original broad phrasing and hope?
- Is every example a real, confirmed-existing piece of code, not something that sounds plausible but was never checked?
- If the codebase has no precedent at all, did you say that honestly instead of stretching a loosely-related file into a "pattern"?
- Would the reader know, from your answer alone, which file to open and roughly what to copy?

## Relationship to sibling skills and agents

- **codebase-locator** (agent) — if the real ask turns out to be "where does X live," not "what should my new code look like," that's a plain location lookup, not a pattern comparison. Hand back a location instead of padding out a comparison from one hit.
- **engineering-pattern-extraction** — asks a different question for a different audience: "what should I copy or avoid if I'm building a *different*, similar system," synthesized into COPY/AVOID lists and a build order. This skill never leaves the current repo and never produces that kind of document; if the reader isn't currently working in this codebase, redirect there instead.
- **codebase-onboarding** / **subsystem-deep-dive** — if the reader doesn't yet know the system well enough to say what they're building, or the question is really "how does this whole area work" rather than "show me a template," one of those two is the right tool; run this skill after, once they know what they're about to add.
