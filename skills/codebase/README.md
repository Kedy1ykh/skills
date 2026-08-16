# Codebase

Understanding codebases you didn't write, and writing code that fits them — onboarding, deep-dive traces, transferable-pattern extraction, docs-drift audits, and in-repo precedent lookups.

## Model-invoked

Model- or user-reachable (rich trigger phrasing so the model can reach for them).

- **[codebase-onboarding](./codebase-onboarding/SKILL.md)** — Cold-open an unfamiliar repo and reconstruct the whole picture: business context, architecture, domain model, risks, unknowns.
- **[subsystem-deep-dive](./subsystem-deep-dive/SKILL.md)** — Exhaustive, line-level trace of one mechanism, for someone about to modify or debug it.
- **[engineering-pattern-extraction](./engineering-pattern-extraction/SKILL.md)** — Mine a codebase for transferable lessons (copy/avoid, build order, thesis) to apply to a *different* project.
- **[docs-drift-audit](./docs-drift-audit/SKILL.md)** — Check whether an existing architecture-doc corpus is still accurate against recent commits.
- **[pattern-finder](./pattern-finder/SKILL.md)** — Find existing precedent in *this* repo to model new code after, right before writing it.

See the [top-level README](../../README.md) for the fuller rationale behind each skill and the four supporting agents (`codebase-archaeologist`, `docs-drift-auditor`, `codebase-locator`, `codebase-pattern-finder`) they dispatch to.
