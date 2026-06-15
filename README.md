# Standards

How I build, ship, and run software. Living documents — update them when experience proves a standard wrong, or when something is missing.

---

## Engineering Principles

### Clarity over cleverness.
Code is read ten times more than it's written. Write for the reader. Meaningful names, small functions, obvious intent. If the next person — or future you — has to decode it, it's not done.

### Commit small. Push daily.
Code on your machine is risk. Small, focused commits pushed daily protect your work and keep every diff reviewable. Atomic commits tell a story — make them.

### Codify everything.
If you've done it manually twice, write the script. Build for repeatability. Tribal knowledge dies when you leave the room. Code and documentation survive.

### Surgical over sweeping.
The smallest change that solves the problem. Fewer moving parts means fewer failure modes. Rewrites are a last resort.

### Don't reinvent — but understand what you use.
Reach for proven tools and patterns for the plumbing. Know enough to debug them when they fail. Spend creative energy where it creates real value.

### Design for change.
Loose coupling. Clear boundaries. Meaningful names. Build code so the next person — or the next agent — can change it without fear.

### DRY, but not prematurely.
Extract a pattern after you've seen it three times — not before. The wrong abstraction is worse than duplication.

### YAGNI.
Don't build for hypothetical requirements. Speculative architecture is tech debt in disguise. Build for today, design for tomorrow, don't code for next year.

### Own your work.
If you build it, you own it. Red CI is your problem. Broken features are your problem. Quality is your job — not QA's.

### Everything is traceable.
If it's not in source control, it doesn't exist. No manual deployments, no config in someone's head, no one-off fixes run locally and forgotten.

### Embrace AI. Raise the bar.
AI does the heavy lifting. You set the standard. Use AI to do what you'd otherwise be too time-poor to do well — more test coverage, better documentation, cleaner code. Don't use it to ship faster shortcuts.

---

## What's Here

### Code

| Document | Status |
| --- | --- |
| [Code Style & Conventions](code-style.md) | 📝 TODO |
| [Design Patterns](design-patterns.md) | ✅ First draft |
| [API Design](api-design.md) | ✅ First draft |
| [Testing](testing.md) | ✅ First draft |

### Architecture

> Architecture standards are platform-scoped. They apply when a project uses that platform — skip them when it doesn't.

| Document | Status |
| --- | --- |
| [Azure Infrastructure](azure-infra.md) | ✅ First draft |

### Delivery

| Document | Status |
| --- | --- |
| [Git Workflow](git-workflow.md) | ✅ First draft |

### Security

| Document | Status |
| --- | --- |
| [Security Practices](security.md) | ✅ First draft |

### AI & Tooling

| Document | Status |
| --- | --- |
| [Working with AI Agents](ai-workflow.md) | 📝 TODO |

---

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md).
