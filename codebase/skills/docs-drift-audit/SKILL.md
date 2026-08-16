---
name: docs-drift-audit
description: Use when a codebase already has architecture/design docs and someone wants to know which parts are now stale relative to current code. TRIGGER on "are these docs still accurate," "audit our architecture notes against recent commits," "what's changed since we wrote this doc," "docs freshness check," "review our design docs for drift," or opening an existing docs/ or architecture-notes/ folder and asking if it can still be trusted. Produces a per-doc-file staleness table (unchanged / needs-update / high-priority) with exact commit citations and old-vs-new behavior for every flagged claim. REQUIRES an existing doc corpus to audit against — do NOT use to generate documentation from scratch (use codebase-onboarding instead) and do NOT use for a single subsystem with no prior docs (use subsystem-deep-dive instead).
---

# Docs Drift Audit

The question here isn't "what does this codebase do" — it's "is what we already wrote about it still true." That's a diff-and-verify operation, not a generate-fresh one, and it needs a different loop: find the baseline, find what changed since, check only the claims that could plausibly be affected, and report per-file so a reader knows exactly which docs to still trust.

## Step 1 — Establish the baseline

You need a starting point to diff from. Look for, in order:

- A stated version/date/commit in the docs themselves ("docs written against v0.1.0" or similar) — this is the strongest signal, use it directly.
- The last commit that touched the docs directory itself (`git log -1 --format=%H -- <docs-dir>`) — a reasonable fallback if the docs don't self-report a baseline.
- If neither exists, ask rather than guess — auditing against the wrong baseline produces a report that's confidently wrong about what's new.

Once you have a baseline, get the commit range: `git log --oneline <baseline>..HEAD | wc -l` gives you a sense of how much has happened, which tells you whether this will be a quick pass or a real one.

## Step 2 — Scope the check to what could plausibly have moved

Don't re-verify every sentence in every doc uniformly — that's the whole system's onboarding effort repeated, not an audit. Instead:

- Extract every file:line citation each doc makes (a doc worth auditing already anchors its claims this way, which is what makes this kind of audit cheap at all).
- For each cited path, `git log --oneline <baseline>..HEAD -- <path>` — if empty, every claim citing that path can be marked "unchanged" without further reading.
- For paths with commits in range, that's where real reading time goes.
- Claims with no citation to anchor on are a separate, smaller bucket — flag them as "not independently checkable" rather than skipping them silently; that's useful signal about which parts of the doc corpus are harder to trust in general.

## Step 3 — Dispatch one worker per doc file (or small group)

Use `subagent_type: "docs-drift-auditor"` if installed — it's built for exactly this loop (anchor to citations, check git history, distinguish changed-but-still-true from changed-and-now-wrong). Group only very small/related doc files together; most docs get their own worker so findings don't blur across files.

```
Doc file: <path to the doc>
Baseline: <commit hash / tag / date>

Extract this doc's checkable claims (prioritize file:line-cited ones),
check git history for each cited path since baseline, and for anything
that changed, read the actual diff/current code to determine if the
SPECIFIC claim is now wrong — not just "the file changed." Classify the
doc's overall urgency (high/medium/low/none) and return per-claim
findings as your final message text, not a file.
```

Run all workers in parallel, one message, same as any other fan-out.

## Step 4 — Synthesize

Structure mirrors what makes this kind of report actually useful — a summary table first, detail second, and an explicit "still fine" section (which is not filler; it's the trust signal that lets a reader stop worrying about the rest of the corpus):

```
# Summary
| Doc | Status | Urgency |
|-----|--------|---------|
(one row per doc file: unchanged / minor update / needs update, low/medium/high)

# Findings, one section per doc that needs a change
## <doc file> — <urgency>
### <short title for the drift>
**Commit:** <hash> — "<commit message>"
<what the doc says, what actually changed, what to edit — specific enough
that someone could paste the commit hash into `git show` and verify it
themselves>

# What Is Unchanged
(the docs/sections verified stable — this section existing at all is
what makes the "needs update" list trustworthy rather than paranoid)
```

Urgency is a judgment call, not a formula: something's high-priority when a reader acting on the stale claim would do the wrong thing or hit a bug the fix already addressed; medium when the gap is real but the doc is still directionally useful; low/none when nothing checkable actually moved. Don't flag trivial renames or refactors that didn't change behavior — that's noise that trains readers to ignore the report.

## Step 5 — Self-check

- For every "needs update" item, is there a commit hash a reader could actually paste into `git show` to verify it themselves — not just your paraphrase of what changed?
- Did you distinguish "this claim was always wrong" (a documentation error) from "this claim used to be right and drifted" (staleness)? They need different fixes and shouldn't be conflated.
- Is the "What Is Unchanged" section actually populated, or did you skip it because everything felt urgent? A corpus where nothing is stable is possible but rare — if that's genuinely the case, say so; if it's not, the unchanged list is doing real work for the reader.
- Did you avoid re-reading files with zero commits in range, or did you burn budget re-verifying things git already told you hadn't moved?

## Output

One Markdown file (`docs/DOCS_UPDATE_ASSESSMENT.md` or similar, next to the audited corpus, unless told otherwise). No HTML/diagram requirement — this is a working audit report, read once and acted on, not a stakeholder deliverable.

## Relationship to codebase-onboarding and subsystem-deep-dive

This skill assumes a doc corpus already exists — it never generates architecture understanding from nothing. If there's no existing doc set, that's a `codebase-onboarding` (or `subsystem-deep-dive`, for one mechanism) job, not this one. A natural cadence is: onboarding/deep-dive produces docs once, this skill keeps them honest as the codebase moves.
