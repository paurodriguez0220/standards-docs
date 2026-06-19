## Purpose

Coding conventions that apply across projects regardless of language.

---

## Naming

- Names should reveal intent. If a name needs a comment to explain it, rename it.
- Avoid abbreviations unless they're universally understood (`id`, `url`, `api`)
- When a property originates from an external system or API, prefix it with the source name to reveal intent. `PseCompanyId` is unambiguous; `CmpyId` is not. This also prevents collisions when multiple external systems are integrated.
- Boolean names should read as questions: `isActive`, `hasPermission`, `canEdit`
- Functions should be verbs: `getUser`, `calculateTotal`, `sendEmail`
- Classes and types should be nouns: `UserService`, `OrderRepository`

### Repository Naming

Repository and root directory names follow `{purpose}-{apptype}` where `{apptype}` is one of:

| Suffix | App type |
| --- | --- |
| `-api` | ASP.NET Core REST API |
| `-web` | ASP.NET Core web app with UI (Razor Pages, MVC, Blazor) |
| `-auth` | ASP.NET Core Identity / auth server (Duende, Keycloak-based) |
| `-functions` | Azure Functions app |
| `-worker` | Long-running background service (Generic Host, no HTTP) |
| `-console` | CLI or one-shot console tool |
| `-mcp` | Model Context Protocol server |
| `-lib` | Reusable class library |
| `-docs` | Documentation, standards, or reference repository |

If `{purpose}` is multiple words, separate them with underscores: `resume_matcher-console`, `price_sync-worker`.

Examples: `billing-api`, `admin-web`, `soa-functions`, `jarvis-mcp`, `price_sync-worker`, `resume_matcher-console`, `standards-docs`.

---

## Functions & Methods

- One function, one responsibility. If you can't describe a function in one sentence, split it.
- Aim for under ~20 lines. More than that: consider splitting.
- Prefer fewer parameters. More than 3–4 is a smell — consider a config object.
- Avoid side effects where possible. Pure functions are easier to test and reason about.

---

## Files & Modules

- One primary concept per file.
- File name matches the primary class/function/module it contains.
- Group related files in folders by domain, not by type.

---

## General Rules

- No magic strings or numbers — use named constants.
- No commented-out code — that's what git history is for.
- No TODO comments without a linked issue.
- Fail fast and explicitly. Don't silently swallow errors.
- Prefer explicit over implicit — don't make the reader guess.

---

## ORM / Entity Framework (Code-First)

Always use code-first migrations. The codebase is the source of truth for the schema — never modify the database directly.

### Models

- One class per file, named after the entity: `Order.cs`, `UserProfile.cs`
- Use data annotations or Fluent API — pick one per project and stay consistent. Prefer Fluent API for complex mappings; keep models clean.
- Explicit column types on **every** `decimal` property — not just "where precision matters". Convention defaults vary by provider and can silently produce `TEXT` or lose precision on migration. Configure all of them.
- Always configure string lengths — don't rely on `nvarchar(max)` by default.

```csharp
// Fluent API — preferred for anything beyond simple mappings
entity.Property(e => e.Price)
      .HasColumnType("decimal(18,2)")
      .IsRequired();

entity.Property(e => e.Email)
      .HasMaxLength(256)
      .IsRequired();
```

### Identity Table Naming

ASP.NET Core Identity defaults to `AspNetUsers`, `AspNetRoles`, etc. Strip the framework prefix when Identity tables have a dedicated database — the prefix exists to prevent collisions in shared databases and adds noise otherwise.

```csharp
protected override void OnModelCreating(ModelBuilder builder)
{
    base.OnModelCreating(builder);

    builder.Entity<IdentityUser>().ToTable("Users");
    builder.Entity<IdentityRole>().ToTable("Roles");
    builder.Entity<IdentityUserRole<string>>().ToTable("UserRoles");
    builder.Entity<IdentityUserClaim<string>>().ToTable("UserClaims");
    builder.Entity<IdentityUserLogin<string>>().ToTable("UserLogins");
    builder.Entity<IdentityUserToken<string>>().ToTable("UserTokens");
    builder.Entity<IdentityRoleClaim<string>>().ToTable("RoleClaims");
}
```

Keep the prefix only when Identity tables share a database with other application tables.

---

### NuGet Feed Isolation

On machines enrolled in a corporate environment, global NuGet config often includes private ADO or Artifactory feeds. These break `dotnet restore` for personal projects that don't have credentials for those feeds.

Add a `NuGet.config` at the solution root for every personal or non-corporate project:

```xml
<?xml version="1.0" encoding="utf-8"?>
<configuration>
  <packageSources>
    <clear />
    <add key="nuget.org" value="https://api.nuget.org/v3/index.json" />
  </packageSources>
</configuration>
```

`<clear />` removes all inherited sources. Without it, restores silently attempt private feeds and fail with 401 errors that look like network problems.

---

### Migrations

- One migration per logical change. Don't batch unrelated schema changes into one migration.
- Migration names describe the change: `AddOrderStatusColumn`, `CreateUserProfileTable`.
- Never edit an applied migration. Add a new one.
- Review the generated migration before applying — EF doesn't always infer intent correctly.
- Migrations run automatically on deploy in non-production. Production migrations are deliberate and reviewed.

### Relationships

- Always define both sides of a relationship explicitly.
- Use navigation properties — don't load related data with raw IDs when the ORM can do it cleanly.
- Be explicit about cascade delete behaviour. Don't rely on convention defaults for anything that could cause data loss.

### Querying

- Never call `.ToList()` or `.ToArray()` before filtering — filter first, materialise last.
- No lazy loading — use explicit `.Include()` for related data. Lazy loading hides N+1 problems.
- Keep queries close to the repository. Business logic doesn't build LINQ expressions.

```csharp
// Good — filter in the query
var pending = await _db.Orders
    .Where(o => o.Status == OrderStatus.Pending)
    .Include(o => o.Items)
    .ToListAsync();

// Bad — loads everything then filters in memory
var pending = (await _db.Orders.ToListAsync())
    .Where(o => o.Status == OrderStatus.Pending);
```

---

## C# / .NET

### Naming

| Element | Convention | Example |
| --- | --- | --- |
| Class, method, property | PascalCase | `OrderService`, `GetById` |
| Private field | `_camelCase` | `_orderRepository` |
| Interface | `I` prefix + PascalCase | `IOrderRepository` |
| Async method | `Async` suffix | `GetUserAsync` |
| Constant | PascalCase | `MaxRetryCount` |

### Async / Await

- Never `.Result` or `.Wait()` — blocks the thread and causes deadlocks in ASP.NET.
- Async all the way down — don't mix sync and async in a call chain.
- Always accept and forward `CancellationToken` in async public methods.
- `ConfigureAwait(false)` in library code. Not required in application code (controllers, services).

```csharp
// Good
public async Task<Order> GetOrderAsync(int id, CancellationToken ct)
    => await _db.Orders.FindAsync([id], ct);

// Bad — deadlock risk
public Order GetOrder(int id)
    => _db.Orders.FindAsync(id).Result;
```

### Null Handling

- Enable nullable reference types in every project: `<Nullable>enable</Nullable>`.
- Use `?.`, `??`, and `??=` — avoid explicit null checks where the operators are cleaner.
- Use pattern matching for null checks in branching logic: `if (user is null)`.

### Dependency Injection

- Constructor injection only — no service locator (`IServiceProvider` injected into business logic is a smell).
- Register by interface, not concrete type.
- Lifetime rules: `Scoped` for DbContext and per-request services, `Transient` for lightweight stateless services, `Singleton` only for truly stateless and thread-safe services.

### Types

- Use `record` for immutable DTOs, value objects, and command/query objects.
- Use `class` for entities with identity and mutable state.
- `var` when the type is obvious from the right-hand side. Explicit type when it isn't.

### Exception Handling

- Catch specific exceptions — not bare `catch (Exception)`.
- Never swallow exceptions silently (`catch { }` is always wrong).
- Custom exception types for domain errors: `OrderNotFoundException`, `InsufficientStockException`.
- Let unhandled exceptions bubble to the global handler — don't catch-and-rethrow without adding context.

---

## Azure Functions (.NET Isolated)

### Keeping Diagnostic Tools In-Repo Without Including Them in the Build

Diagnostic or investigative code (one-off scripts, database query helpers, font analysis tools) is worth keeping in the repository for future reference, but must not be compiled into the Functions worker.

Exclude subfolders from the main `.csproj` using `<Compile Remove>`:

```xml
<ItemGroup>
  <Compile Remove="helpers/**" />
</ItemGroup>
```

A single wildcard covers every helper inside the folder — no need to update the exclusion when a new helper is added.

The excluded folders can contain their own `.csproj` (making them runnable standalone projects) or just raw `.cs` files — either way the Functions SDK ignores them.

**Rule:** every excluded subfolder must be its own self-contained project (with a separate `.csproj` and `<TargetFramework>`) so it can be run independently with `dotnet run` from within that folder. Do not leave dangling `.cs` files that can only be read, not built.

### Companion Diagnostic CLI Pattern

When a project has a SQLite database (or any other local data store that can be queried directly), add a companion diagnostic project under `helpers/`:

```
MyApp/
├── MyApp.csproj            ← main build
├── helpers/                ← excluded from main build via <Compile Remove="helpers/**" />
│   ├── MyAppDiag/
│   │   ├── MyAppDiag.csproj
│   │   └── Program.cs      ← CLI verbs: list, show, search, count
│   └── BpiDiag/            ← one-off investigative tool, kept for reference
│       ├── BpiDiag.csproj
│       └── Program.cs
```

The Diag project references only what it needs (e.g. `Microsoft.Data.Sqlite`) and connects to the same database file used by the running app. Standard verbs:

| Verb | What it does |
|---|---|
| `statements` | List all records with summary counts |
| `transactions <id>` | Show all child records for a given parent |
| `search <keyword>` | Full-text filter across description fields |
| `counts` | Count child records grouped by parent |

This is faster and safer than loading assemblies into PowerShell or writing ad-hoc SQL in a GUI tool.

---

## TypeScript

### Strict Mode

Always enable strict mode in `tsconfig.json`. No exceptions.

```json
{ "compilerOptions": { "strict": true } }
```

### Naming

| Element | Convention | Example |
| --- | --- | --- |
| Type, interface, class, enum | PascalCase | `OrderStatus`, `UserProfile` |
| Variable, function, method | camelCase | `getOrderById` |
| Interface | No `I` prefix (unlike C#) | `OrderRepository` not `IOrderRepository` |
| Enum values | PascalCase | `OrderStatus.Pending` |
| File | kebab-case | `order-service.ts` |

### Types

- No `any`. If the type is unknown, use `unknown` and narrow it before use.
- No non-null assertion (`!`) unless you can prove at the call site it cannot be null.
- Use `interface` for object shapes. Use `type` for unions, intersections, and aliases.
- Explicit return types on exported and public functions.

```ts
// Good
interface Order { id: string; status: OrderStatus; }
type OrderStatus = 'pending' | 'fulfilled' | 'cancelled';

function getOrder(id: string): Promise<Order> { ... }

// Bad
const getOrder = async (id: any) => { ... }
```

### Async / Await

- Always `async/await` over raw `.then()/.catch()` chains.
- Never let a Promise float unhandled — always `await` or explicitly handle the rejection.

### Imports & Exports

- Named exports over default exports — easier to refactor and grep.
- No barrel files (`index.ts` that re-exports everything) — they create circular dependency risks and slow down build tools.

### Null / Undefined

- Prefer `undefined` over `null` for optional values — it's what TypeScript's optional fields produce.
- Use optional chaining `?.` and nullish coalescing `??` consistently.

---

## Related

- [API Design](api-design.md)
- [Testing](testing.md)

---
*Maintained by paurodriguez0220 · Last updated: 2026-06-19*
*Standards: https://github.com/paurodriguez0220/standards-docs*
