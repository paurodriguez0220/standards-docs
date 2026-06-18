## Purpose

How I work effectively with AI coding agents (Claude Code, Copilot, etc.).

---

## Mindset

- AI does the heavy lifting. You set the standard.
- Treat AI output as a first draft from a capable junior — review it like you own it.
- The AI doesn't know what it doesn't know. Provide context; don't assume it will ask.
- You are responsible for every line that merges. "The AI wrote it" is not a defence.

---

## Prompting

- Give context upfront: what you're building, the constraint, the desired outcome.
- Specify the *why*, not just the *what* — agents make better decisions with motivation.
- Break large tasks into scoped steps. One clear goal per prompt.
- Reference specific files, functions, or line numbers when relevant.
- When a result is wrong, correct it with the reason — not just "try again". The agent will repeat the same mistake without understanding why it was wrong.

---

## Review

- Always read the diff before accepting a change.
- Run tests. AI-generated code can be confidently wrong.
- Check for: security issues, hardcoded values, missing error handling, off-by-one logic.
- Don't merge anything you don't understand well enough to debug.
- Be especially sceptical of generated tests — they tend to test what the implementation does, not what it should do.

---

## What AI is Good For

- Boilerplate and scaffolding
- Writing and expanding tests
- Documentation
- Refactoring to a pattern you've already defined
- Debugging with a full error + context paste
- Exploring unfamiliar APIs or frameworks
- First drafts of standards, ADRs, and PR descriptions

## What AI is Not Good For (Without Supervision)

- Architectural decisions — it will satisfy the prompt, not the system
- Security-sensitive code — verify every line
- Large untested rewrites
- Anything you wouldn't be able to explain to a colleague

---

## Agent Scope

Keep tasks focused. Wide-open prompts produce wide-ranging changes that are hard to review.

- One goal per session where possible.
- If an agent touches files outside what you expected, question whether it went off-track.
- Prefer targeted instructions ("update the `GetOrderAsync` method in `OrderService.cs` to...") over broad ones ("refactor the order service").
- Don't let agents make infrastructure or deployment changes without explicit instruction.

---

## Permissions & Trust

Claude Code operates with the permissions you grant it. Be intentional.

- Allow read-only operations freely — file reads, searches, and git log are low risk.
- Gate write operations you care about — direct commits to main, pushing, running migrations.
- Never allow agent access to production credentials or environments.
- Review `.claude/settings.json` and `settings.local.json` — they define what the agent can do without prompting you.

---

## CLAUDE.md

Every project should have a `CLAUDE.md` at the repo root. It is the agent's operating manual — read at the start of every session.

### What to include

```markdown
## Project
One paragraph: what the project is, what it does, who uses it.

## Architecture
Key moving parts — main projects, layers, external dependencies.

## Structure
Where things live — src, tests, infra, docs.

## Commands
How to build, test, run, and migrate locally.

## Conventions
What to follow — link to this standards repo or inline critical rules.

## Never Do
Explicit prohibitions — things the agent must not do regardless of instruction.
```

### Rules

- Keep it updated. A stale `CLAUDE.md` is worse than none — it gives the agent false confidence.
- Be explicit in "Never Do". Vague guidance gets ignored under pressure from a persuasive prompt.
- Link to this standards repo from every project's `CLAUDE.md` so conventions are inherited, not duplicated.

---

## Commits

- Review every AI-generated commit before it goes anywhere near a shared branch.
- The commit message should explain the *why* — if the agent writes "update OrderService.cs", rewrite it.
- Don't let agents amend or force-push. New commits only.

---

## MCP Servers

MCP (Model Context Protocol) servers extend what an agent can do — connecting it to external tools, APIs, and data sources.

- Only install MCP servers from trusted sources.
- Review what tools an MCP server exposes before enabling it. Each tool is a capability you're granting.
- Treat MCP tool calls the same as code changes — they can have side effects.

### Building .NET MCP Servers

The standard stack for a .NET MCP stdio server:

```xml
<PackageReference Include="ModelContextProtocol" Version="1.4.0" />
<PackageReference Include="Microsoft.Extensions.Hosting" Version="9.0.x" />
```

**Program.cs pattern:**

```csharp
var builder = Host.CreateApplicationBuilder(args);

// All logs must go to stderr — stdout is reserved for JSON-RPC messages.
// Writing logs to stdout corrupts the MCP pipe and breaks tool calls silently.
builder.Logging.AddConsole(options =>
{
    options.LogToStandardErrorThreshold = LogLevel.Trace;
});

builder.Services.AddMcpServer()
    .WithStdioServerTransport()
    .WithTools<MyToolClass>();

await builder.Build().RunAsync();
```

**Tool registration:**

```csharp
[McpServerToolType]
public sealed class MyToolClass
{
    [McpServerTool, Description("What this tool does — shown to the model.")]
    public async Task<string> DoThing(
        [Description("Parameter description.")] string input,
        CancellationToken ct = default)
    {
        // return JSON string
    }
}
```

**SQLite DB path** — resolve relative to the executable, not the working directory:

```csharp
var dbPath = Path.Combine(AppContext.BaseDirectory, "myapp.db");
builder.Services.AddDbContext<MyDbContext>(o => o.UseSqlite($"Data Source={dbPath}"));
```

**Always add `NuGet.config`** with `<clear />` when building on a corporate machine — see [Code Style](code-style.md#nuget-feed-isolation).

### Registering with Claude Desktop

Claude Desktop (Microsoft Store install) config location:

```
%LOCALAPPDATA%\Packages\Claude_pzs8sxrjxfjjc\LocalCache\Roaming\Claude\claude_desktop_config.json
```

Claude Desktop does not inherit the terminal PATH. **Always publish as a self-contained single-file exe** — pointing at a DLL with `dotnet <path>.dll` will fail silently with "unknown skill" errors:

```bash
dotnet publish -c Release -r win-x64 --self-contained true -p:PublishSingleFile=true -o publish
```

Config entry:

```json
{
  "mcpServers": {
    "my-server": {
      "command": "C:\\full\\path\\to\\publish\\MyServer.exe"
    }
  }
}
```

After editing the config, fully quit Claude Desktop (system tray → Quit) and reopen — it does not hot-reload.

### Registering with Claude Code

```bash
claude mcp add <server-name> <command> [args...]
# example:
claude mcp add stocks-mcp dotnet "C:\path\to\Stocks.Mcp.dll"
```

Claude Code inherits the terminal PATH so `dotnet <dll>` works here. Restart the session after adding.

---

## Related

- [Git Workflow](git-workflow.md)
- [Security Practices](security.md)
- [Code Style & Conventions](code-style.md)

---
*Maintained by paurodriguez0220 · Last updated: 2026-06-17*
*Standards: https://github.com/paurodriguez0220/standards-docs*
