## Purpose

Standards for project documentation — what every project needs, how to structure it, and what belongs where.

---

## Every Project Needs a README

No exceptions. A project without a README is incomplete.

### Required sections

```markdown
# Project Name

One sentence: what this project is and who uses it.

## Getting Started
Prerequisites, how to clone, how to run locally.

## Commands
| Command | What it does |
| --- | --- |
| `dotnet run` | Start the API |
| `dotnet test` | Run all tests |
| `dotnet ef database update` | Apply pending migrations |

## Architecture
Key moving parts — main layers, external dependencies, notable patterns.
Keep it to a paragraph or a simple diagram. Not a novel.
Explicitly name the design patterns used (Options pattern, Strategy, Factory, Orchestrator, etc.).

## Dependencies
All external services, NuGet packages, and integrations the app needs at runtime.
| Dependency | Purpose |
| --- | --- |
| _external service or package_ | _what the app uses it for_ |

List only runtime dependencies — not build tools or test libraries.

## Configuration
Environment variables and secrets required to run — names, not values.

## Links
- Production URL
- Staging/test URL
- Related repos
- Standards: https://github.com/paurodriguez0220/standards-docs
```

### Rules

- Write the README for someone joining the project cold — they should be able to run it locally in under 10 minutes.
- Commands must be copy-pasteable and current. A README with broken setup steps is worse than none.
- Architecture must explicitly name design patterns — "Orchestrator pattern", "Strategy + Factory", not just "there are services".
- Dependencies section must list every runtime external service or significant NuGet package. If a dependency requires setup (API keys, accounts), link to or summarize the setup steps.
- No corporate fluff. No "this project leverages cutting-edge technology to deliver business value."
- Update the README when the setup changes. It's part of the work, not an afterthought.

---

## CLAUDE.md

Every project also needs a `CLAUDE.md` at the repo root for AI agent context. See [Working with AI Agents](ai-workflow.md) for the full template and rules.

---

## Docs Folder

Create a `docs/` folder when the project has content that doesn't belong in the README — design decisions, runbooks, integration guides.

```
docs/
├── architecture.md       ← system overview, diagrams
├── decisions/            ← ADRs (see below)
│   ├── 001-use-ef-core.md
│   └── 002-jwt-auth.md
└── runbooks/             ← operational procedures
    └── rotate-secrets.md
```

Don't create `docs/` just to have it. One well-written README is better than a docs folder with three stale files.

---

## Architecture Decision Records (ADRs)

Write an ADR when you make a significant decision that future contributors will wonder about — why this library, why this pattern, why not the obvious alternative.

### Template

```markdown
# ADR-{number}: {Short title}

**Date:** YYYY-MM-DD
**Status:** Accepted | Superseded by ADR-{n}

## Context
What problem were we solving? What constraints existed?

## Decision
What did we decide to do?

## Consequences
What does this make easier? What does it make harder?
```

### When to write one

- Choosing a framework, library, or external service
- Deviating from the standards in this repo (and why)
- Significant architectural changes — new layer, new pattern, new integration
- Decisions you expect to be questioned in a future code review

### When not to write one

- Implementation details that are obvious from the code
- Decisions that can be reversed cheaply
- Anything already covered by the standards repo

---

## What Not to Document Here

| Belongs in | Not in docs |
| --- | --- |
| Code (names, types, structure) | "What does this function do?" |
| Git history / commit messages | "Why was this changed?" |
| This standards repo | Language conventions, patterns, workflow rules |
| Tickets / issues | Requirements, acceptance criteria |

Don't duplicate information across multiple places. Pick where something lives and link to it everywhere else.

---

## Signature

Every project README and doc file ends with a signature block:

```markdown
---
*Maintained by paurodriguez0220 · Last updated: YYYY-MM-DD*
*Standards: https://github.com/paurodriguez0220/standards*
```

Update the date when the document changes. The signature is a signal that the document is actively owned, not abandoned.

---

## Related

- [Working with AI Agents](ai-workflow.md)
- [Git Workflow](git-workflow.md)

---
*Maintained by paurodriguez0220 · Last updated: 2026-06-18*
*Standards: https://github.com/paurodriguez0220/standards-docs*
