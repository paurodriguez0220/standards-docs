## Purpose

Security fundamentals that apply to every project. These are non-negotiable floors, not optional enhancements.

---

## Secrets

- Never commit secrets to source control. Not even temporarily. Not even in a branch.
- Use a secrets manager (Key Vault, AWS Secrets Manager) in deployed environments. Environment variables for local dev.
- `.env` files are always in `.gitignore`. No exceptions.
- Rotate secrets when anyone with access leaves the team.
- Audit who has access to production secrets. Access should be minimal and intentional.

If a secret is ever accidentally committed: rotate it immediately, then remove it from history. Removing it from history does not make it safe — it was already exposed.

---

## Authentication & Authorisation

- Use established auth libraries — never roll your own crypto or auth flows.
- Prefer short-lived tokens (JWT with expiry + refresh rotation) over long-lived sessions.
- Validate tokens on every request: signature, expiry, issuer, and audience.
- Authorise at the resource level, not just the route level. A user authenticated to hit `/orders` does not automatically have access to every order.
- Validate on the server — client-side checks are UX, not security.
- Use managed identities for service-to-service auth in Azure. No stored credentials, no connection strings with passwords.

### OAuth2 Scope Naming

Scopes describe what the token grants access to — they are not system roles.

- Name scopes after the access level, not the service or a role: `service:manage` / `service:read`, not `service-admin` or `admin`.
- Don't embed the service name as an entity in its own scope. `duende:manage` not `duende-admin` — a server does not need to describe itself as an admin resource.
- Use a consistent verb pattern across all services: `:manage` for write access, `:read` for read-only.
- Keep scopes coarse unless a real consumer needs finer granularity. Don't pre-optimise into per-entity scopes (`users:read`, `clients:read`) until a specific use case demands it — the added complexity rarely pays off.

### Dev-Only Auth Shortcuts

Dev-only auth setups (in-memory signing keys, developer certificates, ephemeral token stores) must never reach production.

- Gate every dev-only auth configuration with an `IsDevelopment()` check.
- Throw a descriptive `InvalidOperationException` in the `else` branch so the app fails at startup rather than running silently insecure.

```csharp
if (builder.Environment.IsDevelopment())
    identityServer.AddDeveloperSigningCredential();
else
    throw new InvalidOperationException(
        "A signing certificate must be configured for non-development environments.");
```

Never rely on a deployment pipeline to "just not deploy dev settings" — make the code itself refuse to start in production without explicit secure configuration.

---

## Input Validation

- Never trust user input. Validate at every system boundary — API, message queue, file upload.
- Parameterise all database queries — never concatenate user input into SQL.
- Encode output to prevent XSS. Treat data from the DB as untrusted when rendering it.
- Validate file uploads: allowed MIME types, maximum size, and scan content where risk warrants it.
- Reject unexpected input early. Don't silently ignore unknown fields.

---

## Transport

- HTTPS everywhere. No plain HTTP in any environment beyond local dev.
- TLS 1.2 minimum. Disable TLS 1.0 and 1.1 explicitly.
- HSTS enabled on all public-facing endpoints.

---

## Security Headers

Every HTTP response from a web-facing app should include:

| Header | Value |
| --- | --- |
| `Strict-Transport-Security` | `max-age=31536000; includeSubDomains` |
| `X-Content-Type-Options` | `nosniff` |
| `X-Frame-Options` | `DENY` |
| `Referrer-Policy` | `strict-origin-when-cross-origin` |
| `Content-Security-Policy` | Tailored per app — no blanket `unsafe-inline` |

---

## CORS

- Explicitly define allowed origins — never use `*` for APIs that handle authenticated requests.
- Allowed methods and headers should be the minimum required.
- Credentials (`withCredentials`) + wildcard origin is not valid and will fail — if you need credentials, pin the origin.

---

## Error Handling

- Never expose stack traces, exception messages, or internal identifiers in API responses.
- Return a generic error message to the client. Log the full detail server-side.
- All error responses use RFC 7807 ProblemDetails — see [OpenAPI, Errors & Logging](openapi.md).

```json
// Good — ProblemDetails, no internals
{ "type": "...", "title": "An unexpected error occurred.", "status": 500 }

// Bad — exposes internals
{ "error": "NullReferenceException at OrderService.cs:47" }
```

---

## Logging

- Never log secrets, passwords, tokens, or PII.
- Log enough to reconstruct what happened — not so much you create a data liability.
- Structured logging only. No free-text log messages that mix data and prose.
- Authentication failures, authorisation denials, and input validation rejections should always be logged.

---

## Dependencies

- Keep dependencies updated. Unpatched packages are the most common attack vector.
- Run vulnerability scans regularly: `npm audit`, `dotnet list package --vulnerable`, or equivalent in CI.
- Remove unused dependencies — every package is an attack surface.
- Pin versions in production. Floating ranges (`^`, `~`) mean an upstream compromise can reach you silently.

---

## Infrastructure & Least Privilege

- Every identity (user, service, app) gets the minimum permissions it needs — nothing more.
- No shared service accounts. Each app has its own managed identity.
- Disable public network access on data resources (storage, databases, caches) where private endpoints exist.
- Production subscriptions are restricted — developers do not have standing write access to production.
- All infrastructure changes go through code (bicep, Terraform) — no manual portal changes.

---

## CI / Supply Chain

- Secret scanning enabled on every repository.
- No secrets in workflow files — use environment secrets or variables.
- Pin GitHub Actions to a specific commit SHA, not a mutable tag.
- Review third-party actions before adopting them. Prefer first-party (`actions/`) or well-maintained actions.

---

## Related

- [API Design](api-design.md)
- [Azure Infrastructure](azure-infra.md)
- [Git Workflow](git-workflow.md)

---
*Maintained by paurodriguez0220 · Last updated: 2026-06-15*
*Standards: https://github.com/paurodriguez0220/standards-docs*
