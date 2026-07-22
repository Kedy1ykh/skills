---
name: codebase-onboarding
description: Use whenever someone needs to deeply understand an unfamiliar codebase fast — onboarding a new engineer, inheriting legacy code, due diligence before a decision, or any broad "what is this / how does this work / how is this architected" question about code the current team didn't write. TRIGGER on "onboard me to this repo," "explain this codebase," "document the architecture," "help a new engineer understand this project," "generate architecture docs," "walk me through this system," or a cold open of a repo needing business, architecture, domain, data, security, or ops context — even without the word "onboarding." Produces a cited Markdown report plus offline-capable HTML (diagrams, no CDN) covering an exec summary, architecture/context/dependency/domain/data-flow diagrams, a DB ERD, a workflow walkthrough, a deployment diagram, ADRs, a risk map, and open questions. Do NOT use for narrow single-function questions — answer those directly.
---

# Codebase Onboarding

You're producing what a sharp new engineer would want on day one: not a tour of file names, but
a reconstruction of *why the system is shaped the way it is*, backed by evidence, honest about
what evidence can't tell you. The output serves two readers at once — someone skimming for the
big picture, and someone who wants to click through to the exact line that justifies a claim.
Both need to be served by the same document.

The core discipline is **top-down before bottom-up, and cited fact before inferred narrative**.
Figure out what the system is *for* and how its pieces relate before you catalog functions, and
mark every claim about intent, users, or business rationale as either sourced or a flagged guess.
A report that quietly presents guesses as facts is worse than useless — it actively misleads the
next person who trusts it.

## When this does and doesn't apply

This skill is for *broad* understanding of a system you (or the person asking) haven't worked
in — not for narrow lookups. A few boundary cases to make the distinction concrete:

- "What does this repo do, and how would I start contributing?" → applies. This is exactly the
  cold-open case.
- "We're inheriting this service from another team, what are we taking on?" → applies, and the
  risk/tech-debt map and unknowns list matter most here.
- "What does `parseInvoice()` do?" → doesn't apply. Answer it directly by reading the function;
  spinning up a full deep-dive for a single-function question wastes the reader's time waiting
  on work they didn't need.
- "Is this codebase secure?" in isolation, with no broader onboarding need → borderline. If they
  want a full audit, `security-review` (if available) is the sharper tool; this skill's security
  section is one piece of a bigger picture, not a substitute for a dedicated security review.
- "Explain this one file to me" → doesn't apply, same reasoning as the single-function case —
  just read and explain the file.

When in doubt, the tell is scope: one function or file, answer directly; the shape of an
unfamiliar system, run the full skill.

## Step 1 — Orient before you fan out

Before spawning any research, get a lightweight map of the territory yourself:

- Read the root README, any `ARCHITECTURE.md`/`PLAN.md`/`CLAUDE.md`/`AGENTS.md`/design docs, and the
  top-level package manifest(s) (`package.json`, `pyproject.toml`, `go.mod`, `Cargo.toml`, ...).
  Design docs are a hypothesis about the system, not ground truth — you'll check them against
  real code in the fan-out.
- If `codebase-memory` MCP tools are available, index the repo and call
  `get_architecture(aspects=["all"])` — it hands you package structure, entry points, routes, and
  clusters in one shot, far faster than guessing from folder names. If it's not available,
  `Glob`/`ls` the top two directory levels and skim for the obvious seams (monorepo or not? one
  service or several? one language or a polyglot mix, e.g. a TS frontend plus a Python backend?).
- Cheaply check provenance and shape: `git log --oneline | wc -l` (one squashed commit vs. real
  history changes how much you can trust "why" questions), and look for `.github/`,
  `Dockerfile`/`Containerfile`, `docker-compose*`, CI config, `.env*`/env-schema files, and test
  directories — their mere presence or absence answers several Section A questions (build/release,
  security, ops) before you've read a line of business logic.
- Form a rough hypothesis of what this thing *is* — a library, a CLI, a web service, a monorepo
  of several services — before deciding how to split the research. You will revise this
  hypothesis as findings come in; that's expected, not a failure.
- Notice if the system has one obviously dominant architectural theme — everything in service of
  how it talks to one expensive external dependency, say, or everything in service of one sandbox
  boundary. If so, you'll lead Section A's executive summary with a handful of theme-specific
  questions before the generic template (see Step 4) — a reader who knows the system's real
  question gets more from that than from a generic frame applied uniformly.

Don't skip this step to save time. The fan-out in Step 2 is only as good as the seams you
identify here — split along the wrong lines and you'll get several reports that all cover the
same file and miss another entirely.

**If the repo is huge** (a large monolith, a monorepo with dozens of packages), don't try to read
it uniformly. Use whatever structural signal is available — `codebase-memory`'s fan-in/fan-out
hotspots and cluster detection, or failing that, commit frequency (`git log --stat` on recent
history) and directory size — to find where the real logic concentrates, and weight your reading
time there. A config file with 40 near-identical entries needs one representative read, not 40.

## Reference — what each research category is actually asking

Keep this checklist handy when writing dispatch prompts (Step 3) and when synthesizing (Step 4),
so nobody has to re-derive these from scratch mid-task. Not every category applies to every
slice — that's expected, not a gap.

| Category | Core questions |
|---|---|
| Business context | What problem does this solve? Who are the users? Who pays for it (often unanswerable from code — say so)? What's the critical workflow? What counts as success vs. failure for a transaction? Why was it built, why does it still exist, why would a customer need it? |
| Architecture | Monolith, modular monolith, microservices, event-driven, streaming, batch, or serverless — which, and what's the evidence? What services/APIs/databases/queues/caches/external systems exist? |
| Domain models | What are the core business entities? How do they relate? Who owns (writes) each one vs. merely reads it? |
| Data models | What tables/collections/schemas/views/stored procedures exist, and who owns each? What's the actual source of truth? What's replicated, what's cached, what could be fully reconstructed if lost? |
| Runtime topology | What's the real request path (e.g. client → load balancer → service → database)? What's the deployment/network topology — cloud resources, containers, Kubernetes, scaling model? |
| Dependency graph | What are the internal module/service dependencies? What external dependencies exist (cloud providers, SaaS APIs, auth providers, LLM providers, SMTP, LDAP, etc.)? What breaks, and how, if each disappears? What can't be tested locally? |
| Build & release | What's the build/release pipeline (git → CI → tests → package → deploy), if any? How does code actually reach production? How is rollback performed? How are hotfixes done? |
| Security model | How are authentication, authorization, secrets management, encryption, and audit logging handled — or explicitly, which of these don't exist at all? How are permissions enforced? What happens if an auth dependency fails? What's the implied threat model? |
| Operational model | What monitoring/logging/alerting/incident-response exists? How would someone know the system is broken? What would actually page an engineer at 3AM — and if nothing would, say that plainly. |

## Step 2 — Decide how many workers, and what each one owns

There's no fixed worker count — pick a split that matches the system's actual seams, not an
arbitrary target:

- A small, single-purpose repo (one service, one language) often needs only **one or two**
  research passes covering everything.
- A monorepo with clearly separated apps/services usually warrants **one worker per major
  slice**, because a person fluent in one layer's code is the one who'll actually find the
  interesting details in it. In a past run, a repo with a React frontend, a thin TS/Hono proxy
  layer with an unused ORM scaffold, and a Python FastAPI service holding the real business logic
  split cleanly into: (1) the Python business-logic layer, (2) the TS server/API/DB scaffolding,
  (3) the frontend, (4) tests/security/ops/build considered together. That's an example of a
  split that matched real seams — not a template to force onto every repo.
- Split by **slice of the system**, not by question category. A single worker covering "the
  Python backend" should answer business/architecture/domain/data/runtime/dependency/build/
  security/ops questions *for that backend*, rather than one worker answering "security" for the
  whole repo in isolation — cross-cutting concerns like auth or data ownership only make sense in
  the context of the code that implements them.
- Make sure the union of all workers' assignments covers every slice of the system at least once.
  It's fine — expected, even — for two workers to both touch a shared boundary (e.g. the API
  contract between frontend and backend) from their own side.
- If the first round of findings surfaces a slice that's clearly under-covered or unexpectedly
  complex (a "thin proxy" that turns out to hide the real business logic, say), run a second,
  narrower fan-out into just that slice rather than treating the first pass as final.

**Two worked examples, to show the range this can take:**

*Small repo* — a single-language CLI tool with no network service, one package manifest, a few
hundred lines across a handful of files. Correct call: skip the fan-out entirely, read it
yourself end to end, and produce a proportionally short report — probably no service-dependency
diagram (there's only one service), no deployment diagram beyond "runs as a local binary," but
still an honest domain model and unknowns list. Spending four parallel agents on this wastes
effort and will produce four overlapping, thin reports instead of one solid one.

*Large repo* — a monorepo with a React frontend, a thin API-gateway layer, a backend service
holding the real domain logic, and a separate data-processing pipeline. Correct call: one worker
per slice (four, here), each answering the categories relevant to it — the gateway worker mostly
covers architecture/dependency/security (what does it forward, what auth does it enforce, what
happens if the backend is down), while the backend worker carries most of the domain-model and
data-model weight. The synthesizer (you) then owns stitching their findings into the twelve-item
Section A, resolving any place two workers described the same boundary slightly differently by
going back to the code yourself rather than picking one worker's version arbitrarily.

## Step 3 — Dispatch research in parallel

Spawn one `Agent` call per slice, **all in a single message** so they run concurrently —
sequential fan-out defeats the point. Use `subagent_type: "codebase-archaeologist"` if that agent
is installed in the current environment; if not, fall back to `"general-purpose"` or `"fork"` and
paste the same operating principles into the prompt inline (evidence-over-narrative, cite
`file:line`, flag inferred claims, don't summarize your own findings away).

A dispatch prompt for one worker should look roughly like this — fill in the brackets, keep the
discipline instructions verbatim:

```
Task #<n> — <slice name>, e.g. "Python backend / apps/server/python_scripts".

Scope: read and report on <specific files/directories>. Out of scope: <other
slices, owned by other workers — name them so you don't duplicate effort>.

Answer whichever of these apply to your slice: business context (what problem
does it solve, who calls into it, success/failure definition), architecture
(service/library/batch/UI, what it talks to), domain model (entities, fields,
relationships, ownership), data model (what's persisted, where, source of
truth vs. cache vs. replica), runtime topology (process, host/port,
lifecycle), dependency graph (internal + external, what breaks if X
disappears, what can't be tested locally), build/release (how it's built,
tested, deployed), security (auth/authz/secrets/encryption — or explicitly
"none"), operational model (logging/monitoring/alerting, or "none").

Rules: every factual claim needs a file:line citation. Anything about intent,
users, or business rationale that you can't cite, mark with a leading ⚠️ and
say what you're inferring from. If a question has zero evidence, write "not
found" rather than skip it. Also flag any naming trap or footgun you notice —
the same identifier reused for two different things, a name that implies one
behavior but delivers another — call it out explicitly, don't bury it. Return
full findings as your final message text — do not summarize them away, and do
not write any files.
```

Read each worker's returned findings before synthesizing — don't let a background-completion
notification substitute for actually reading what came back.

If you're interrupted or run low on context mid-task, don't restart the fan-out from scratch —
the worker findings you already have are the expensive part to reproduce. Save what's come back
so far (even informally, in your own notes) and resume by synthesizing what you have plus
dispatching only the slices still missing, rather than re-running workers whose findings you
already hold.

## Step 4 — Synthesize into two deliverables, in this order

Assemble one document with two sections. **Put the synthesized deliverables first**, and the
supporting raw research after — a reader should be able to stop after Section A and have the
real picture; Section B is there for people who want to verify or dig deeper. Use this skeleton
(adapt or drop entries per the scaling note at the end of this skill — see below):

```
# Section A — Synthesized Deliverables
1. Executive business summary
2. Architecture diagram
3. System context diagram
4. Service dependency diagram
5. Domain model diagram
6. Data flow diagram
7. Database ERD (or: actual system of record, if there's no real DB)
8. Critical workflow walkthrough
9. Deployment diagram
10. ADR reconstruction
11. Risk & technical-debt map
12. Unknowns and questions for the original developers

# Section B — Raw Findings
- Business context / Architecture context / Domain models / Data models /
  Runtime topology / Dependency graph / Build & release / Security model /
  Operational model — one subsection each, per-worker findings, file:line cited
- Top-Down map: business goal → capabilities → services → modules → functions
- Bottom-Up map: entry points → request handlers → business logic →
  persistence → infrastructure
```

Notes on the items that most often go wrong:

- **Executive summary** (#1): state what the system is, who it's for (or an honest "unknown"),
  and why it likely exists, in a few dense paragraphs — this is the single section a busy reader
  will actually finish, so don't bury the real answer under hedging. If Step 1 surfaced one
  dominant architectural theme, open with 3-5 tailored questions specific to that theme (e.g., for
  a system built around one expensive external call: how does it use that dependency, what does
  it lean on it for, what's the cost-control strategy, what runs locally instead) before the
  generic "what/who/why" framing.
- **Architecture diagram** (#2): label the style (monolith / modular monolith / microservices /
  event-driven / batch / serverless) and justify the label from evidence — "it has multiple
  `apps/` directories" is not, by itself, evidence of microservices if they're not independently
  deployable.
- **Database ERD** (#7): if there's a scaffolded-but-unused ORM/DB — this happens more often than
  you'd expect, e.g. a starter-kit database dependency nothing ever queries — say so explicitly
  and diagram whatever the *actual* system of record is instead (flat files, an external store,
  another service's API). Don't produce a technically-accurate-but-misleading ERD of a schema
  nothing touches.
- **Deployment diagram** (#9): draw the runtime topology as it exists today, not an aspirational
  one. If there's no real deployment (a local-only tool, a prototype never shipped), say that
  plainly rather than inventing infrastructure to fill the diagram.
- **ADR reconstruction** (#10): for each decision, state the decision, the evidence, the likely
  rationale (⚠️ if inferred), and whether it's still honored consistently or has drifted from
  later code.
- **Risk & tech-debt map** (#11): every row must trace to something a worker actually found.
  Don't pad with generic advice ("add more tests" on every row) — a risk map that applies to any
  codebase in existence isn't telling anyone anything.
- **Unknowns list** (#12): should never be empty. If a worker's findings didn't surface open
  questions, that's a sign the research wasn't skeptical enough — go back and ask "what would I
  need to know that this code cannot tell me?"

Use Mermaid for every diagram (`flowchart`, `erDiagram`, `sequenceDiagram` as appropriate) so
both the Markdown and HTML render them natively. If a diagram type genuinely doesn't apply (e.g.
no database exists at all, or the system has no deployment target), say so explicitly in that
section instead of silently omitting it — a missing section reads as an oversight; a stated "N/A
because X" reads as a judgment call.

Throughout both sections: mark inferred/unverifiable claims with ⚠️ inline, right where the claim
is made — not in a disclaimer buried at the top that the reader forgets by section 5.

## Step 5 — Produce both file formats; keep the HTML network-independent

Write the Markdown report with Mermaid code blocks (renders natively on GitHub/GitLab and in most
editors). Then produce a self-contained HTML version with the same content, styled for
readability, with Mermaid diagrams actually rendering.

**Do not point the HTML's Mermaid loader at a CDN `<script src>`.** CDN fetches silently fail
behind corporate TLS-inspecting proxies (confirmed firsthand: a proxy with an untrusted root CA
blocks the connection, and the diagrams render as raw `flowchart TB` text instead of a diagram,
with no obvious error) — this makes the deliverable look broken for a large fraction of corporate
readers. Instead:

- Check whether a local `mermaid.min.js` already exists somewhere reusable in the target repo or
  a previous report's output directory, and reuse it.
- Otherwise fetch it once (e.g.
  `curl -sL https://cdn.jsdelivr.net/npm/mermaid@10/dist/mermaid.min.js -o mermaid.min.js`) and
  save it as a sibling file next to the HTML report, referencing it with a relative
  `<script src="./mermaid.min.js">`.
- **Verify the download before trusting it** — check the first bytes look like real JS (e.g.
  starts with `(function(` or another UMD wrapper), not an HTML error/login page from a captive
  proxy. A corrupted or truncated file fails exactly the same way a CDN block does, silently.
- If there's genuinely no network access available at all, ship the Markdown as the primary
  deliverable and tell the user plainly that the HTML's diagrams won't render offline without a
  manually-supplied `mermaid.min.js` — don't ship a document that looks complete but silently
  fails for the reader.

Default output location is a `docs/` directory at the target repo's root, as two files (e.g.
`<PROJECT>_ARCHITECTURE_REPORT.md` and `.html`, plus the bundled `mermaid.min.js`). Only ask the
user for a different location if the repo layout makes `docs/` ambiguous or already used for
something else. If a prior run's report already exists at that path, overwrite it rather than
versioning filenames (`_v2`, `_final`) — the repo's own git history is where prior versions
belong, and a stale duplicate report sitting alongside the current one is a good way for a future
reader to trust the wrong one.

## Also flag: naming traps and footguns

Beyond the nine research categories, actively watch for places where the codebase's own naming
will mislead a newcomer — the same identifier reused for two different things (two classes both
called `Context`, say), a name that implies one behavior but delivers another, a config file
whose name doesn't match what it actually controls. These aren't risks and they aren't decisions,
so they don't belong in the Risk Map or the ADR table — they're reading-comprehension hazards
specific to this codebase, and they're cheap for a research pass to catch precisely because a
worker unfamiliar with the system runs straight into them. When you find one, name both things
explicitly (both classes, both files, both meanings) in Section B under whichever category it
most naturally sits in — usually Architecture or Domain Models — rather than inventing a new
category for it. Don't manufacture one if the codebase is straightforwardly named; this is a
"call it out when it's real" check, not a mandatory quota.

## Common failure modes to avoid

These are the specific ways this kind of report goes wrong — watch for them in your own draft,
not just in theory:

- **Trusting a design doc over the code that implements it.** `PLAN.md`-style documents describe
  intent at some point in the past; always verify against current code, and treat any gap you
  find as a headline finding, not a footnote.
- **Treating an imported dependency as evidence it's used.** A database client instantiated at
  import time, or an ORM schema file present in the repo, does not mean anything ever calls it —
  grep for actual call sites before drawing an ERD or a dependency edge.
- **Forcing the full twelve-item template onto a trivial project**, or conversely,
  under-covering a genuinely complex one because the first orientation pass undersold its scope.
- **Letting "no authentication" or "no tests" register as automatically low-severity** without
  checking the actual exposure (a localhost-only dev tool and an internet-facing service with no
  auth are not the same risk, even though the code looks identical).
- **Writing the unknowns list as an afterthought.** It's often the most useful section for a real
  new hire — questions like "who pays for this" or "is this actually deployed anywhere" save them
  from confidently building on a wrong assumption.
- **Shipping an HTML report that depends on a CDN** — see Step 5. This is the single most common
  way this kind of deliverable silently fails for a real reader.
- **Confusing "no evidence of X" with "X is definitely false."** Say what you checked and what
  you found (or didn't); don't overclaim the absence of something you didn't have time to fully
  rule out.
- **Diagramming the aspirational architecture instead of the real one.** If two components are
  wired together in config but nothing at runtime ever exercises the path, that's worth a note in
  the risk map, not a clean arrow in the architecture diagram as if it were load-bearing.
- **Letting citation density substitute for correctness.** A claim with a `file:line` next to it
  still needs to actually say what that line says — dropping a citation on a paraphrase that
  drifted from the source is worse than no citation, because it looks verified when it isn't.

## Relationship to the codebase-archaeologist agent

This skill is the orchestrator; `codebase-archaeologist` is the worker it fans out to. That split
exists so the research methodology (evidence discipline, what questions to ask, how to report
back) lives in one place and stays consistent across every worker, while this file stays focused
on the parts that are inherently sequential and judgment-heavy — deciding the split, synthesizing
across workers, and producing the final artifacts. If you're editing this skill and find yourself
duplicating instructions that already live in the agent's system prompt, put them in the agent
instead; if you find the agent needs to know something specific to how *this* skill uses it, put
that in the dispatch prompt template in Step 3 rather than growing the agent's own instructions to
cover every possible caller.

## Step 6 — Self-check before calling it done

Ask yourself these, don't just tick boxes:

- Does every Section A item either appear or carry an explicit "not applicable because X"?
- Is there any sentence in Section A stating a business fact (users, payer, motive, deployment
  target) without either a citation or a ⚠️ marking it as inferred?
- Is the unknowns list actually non-empty?
- Open the HTML mentally and check: does the script tag point at a local file, and did you verify
  that file's contents?
- Did the fan-out in Step 2 actually cover the whole system, or is there a slice (a directory, a
  service, a language) that no worker touched?
- Would a new engineer reading only Section A know what to do on their first day, and would they
  know exactly where to look if they doubted a claim?
- If any worker flagged a naming trap or footgun, did it make it into Section B rather than get
  lost in synthesis?

## What a good run looks like

A run has succeeded when a reader who has never seen the repo can, from Section A alone: name the
system's purpose (or know honestly that it can't be determined), point at the one or two
workflows that matter most, know what would break the system and how they'd find out, and know
what to ask the original team if they could. A run has *not* succeeded just because it produced
twelve headings and some Mermaid diagrams — a technically complete report that's wrong about the
architecture style, or that presents an inferred business motive as settled fact, has failed even
though it looks done. If you're unsure whether a draft clears this bar, reread Section A cold, as
if you were the new engineer, before shipping it.

## Scale the effort to the system, not to this checklist

A 200-line CLI script doesn't need twelve diagrams and four parallel research agents — one
focused pass and a proportionally shorter report is the right call, and forcing the full template
onto it produces padding, not insight. A sprawling polyglot monorepo might need more than four
workers, or a second round of fan-out once the first pass reveals a slice that needs its own deep
dive. Use judgment on depth and worker count; the only non-negotiable parts are the discipline
(cite or flag, top-down before bottom-up, honest unknowns) and the network-independence of the
HTML output.
