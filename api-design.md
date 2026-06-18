## Purpose

REST API design standards that apply across projects.

---

## URL Design

- Use nouns, not verbs: `/users` not `/getUsers`
- Plural for collections: `/users`, `/orders`
- Nested resources for ownership: `/users/{id}/orders`
- Lowercase, hyphen-separated: `/user-profiles` not `/userProfiles`
- Keep nesting shallow — max two levels deep. Deeper than that, flatten it.

```
// Good
GET /users/{id}/orders

// Too deep — flatten it
GET /users/{id}/orders/{orderId}/items/{itemId}
→ GET /order-items/{itemId}
```

---

## HTTP Methods

| Method | Use |
| --- | --- |
| `GET` | Read — never mutate state |
| `POST` | Create a new resource |
| `PUT` | Replace a resource entirely |
| `PATCH` | Partial update |
| `DELETE` | Remove a resource |

`GET` requests must be safe and idempotent. `PUT` and `DELETE` must be idempotent.

---

## Status Codes

Use the right code — don't return `200` for everything.

| Code | When to use |
| --- | --- |
| `200 OK` | Success with body |
| `201 Created` | Resource created — include `Location` header pointing to the new resource |
| `204 No Content` | Success, no body (DELETE, some PATCHes) |
| `400 Bad Request` | Client sent invalid data — missing fields, wrong types |
| `401 Unauthorized` | Not authenticated |
| `403 Forbidden` | Authenticated but not allowed |
| `404 Not Found` | Resource doesn't exist |
| `409 Conflict` | State conflict — duplicate, optimistic concurrency mismatch |
| `422 Unprocessable Entity` | Valid syntax, invalid semantics — failed business rules |
| `429 Too Many Requests` | Rate limit exceeded — include `Retry-After` header |
| `500 Internal Server Error` | Unhandled server failure |

Never return `200` with an error in the body. The status code is the contract.

---

## Request & Response

- JSON everywhere. `Content-Type: application/json`.
- camelCase for JSON field names.
- Never expose internal database IDs as the only identifier — use UUIDs for public-facing resources.
- Dates in ISO 8601 UTC: `2025-06-15T10:30:00Z`.
- Empty collections return `[]`, not `null` or `404`.

### Error shape

All error responses use RFC 7807 **ProblemDetails**. This is the standard — no custom error shapes.

```json
{
  "type": "https://tools.ietf.org/html/rfc7807",
  "title": "Validation Failed",
  "status": 400,
  "errors": {
    "email": ["Email is required."]
  }
}
```

ASP.NET Core produces this automatically via `Results.ValidationProblem()` and `IProblemDetailsService`. See [OpenAPI, Errors & Logging](openapi.md) for the full error handling implementation.

Never expose stack traces, internal exception messages, or database errors in the response body.

---

## Pagination

Use cursor-based pagination for large or frequently updated collections. Use offset pagination only for small, stable datasets.

### Cursor-based (preferred)

```
GET /orders?limit=20&cursor=eyJpZCI6MTAwfQ

{
  "data": [...],
  "pagination": {
    "nextCursor": "eyJpZCI6MTIwfQ",
    "hasMore": true
  }
}
```

### Offset-based

```
GET /orders?page=2&pageSize=20

{
  "data": [...],
  "pagination": {
    "page": 2,
    "pageSize": 20,
    "totalCount": 150,
    "totalPages": 8
  }
}
```

Rules:
- Always cap `pageSize` / `limit` server-side. Don't let clients request unbounded results.
- Default page size should be documented and consistent across endpoints.
- Never return paginated data without a `pagination` envelope — callers need to know there's more.

---

## Versioning

Version in the URL path: `/api/v1/users`

- Introduce a new version only for breaking changes.
- Additive changes (new fields, new endpoints) are not breaking — no new version needed.
- Support the previous version for a defined migration window. Communicate the deprecation timeline.
- Document what changed between versions.

Breaking changes (require a new version):
- Removing or renaming a field
- Changing a field's type
- Changing status code semantics
- Removing an endpoint

---

## Authentication & Authorisation

- Use bearer tokens: `Authorization: Bearer {token}`.
- JWT for stateless APIs. Validate signature, expiry, and audience on every request.
- Return `401` for missing or invalid token, `403` for valid token but insufficient permissions.
- Never put sensitive data in the JWT payload — it's encoded, not encrypted.
- Short token expiry + refresh token rotation for user-facing APIs.

---

## Rate Limiting

All public-facing APIs must implement rate limiting.

- Return `429 Too Many Requests` when the limit is exceeded.
- Include response headers so clients can self-throttle:

```
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 14
X-RateLimit-Reset: 1718445600
Retry-After: 30
```

---

## General Rules

- Design for the consumer — the API exists to serve callers, not to mirror the database schema.
- Be consistent. Same patterns, same naming, same error shape across every endpoint in the project.
- Never break a published contract without a versioning strategy.
- Document every endpoint — at minimum: URL, method, request shape, response shape, error codes.

---

## Related

- [OpenAPI, Errors & Logging](openapi.md)
- [Code Style & Conventions](code-style.md)
- [Testing](testing.md)
- [Security Practices](security.md)

---
*Maintained by paurodriguez0220 · Last updated: 2026-06-15*
*Standards: https://github.com/paurodriguez0220/standards-docs*
