## Purpose

Standards for building well-documented, well-behaved .NET APIs — OpenAPI annotations, error handling, parameter validation, and structured logging. Scalar and Swagger both consume the same OpenAPI spec; the rules here apply to both.

---

## OpenAPI Setup

### .csproj

Enable XML documentation generation so doc comments flow into the OpenAPI spec:

```xml
<PropertyGroup>
  <GenerateDocumentationFile>true</GenerateDocumentationFile>
  <NoWarn>$(NoWarn);1591</NoWarn>  <!-- suppress missing XML doc warnings on non-public members -->
</PropertyGroup>
```

### Program.cs

```csharp
builder.Services.AddOpenApi();          // .NET 9+
// or
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen(options =>
{
    options.IncludeXmlComments(Path.Combine(
        AppContext.BaseDirectory,
        $"{Assembly.GetExecutingAssembly().GetName().Name}.xml"));
});

// Scalar UI
app.MapScalarApiReference();

// Swagger UI (alternative)
app.UseSwaggerUI();
```

---

## Endpoint Documentation

Every endpoint needs at minimum a `summary`. Add `remarks` when the behaviour isn't obvious from the summary alone.

```csharp
/// <summary>
/// Get an order by ID.
/// </summary>
/// <remarks>
/// Returns the full order including line items. Only returns orders belonging to the authenticated user.
/// </remarks>
/// <param name="id">The unique identifier of the order.</param>
/// <response code="200">Order found and returned.</response>
/// <response code="401">Not authenticated.</response>
/// <response code="403">Authenticated but not the order owner.</response>
/// <response code="404">Order not found.</response>
[HttpGet("{id}")]
[ProducesResponseType(typeof(OrderResponse), StatusCodes.Status200OK)]
[ProducesResponseType(typeof(ProblemDetails), StatusCodes.Status401Unauthorized)]
[ProducesResponseType(typeof(ProblemDetails), StatusCodes.Status403Forbidden)]
[ProducesResponseType(typeof(ProblemDetails), StatusCodes.Status404NotFound)]
public async Task<IActionResult> GetOrder(string id, CancellationToken ct) { ... }
```

### Rules

- `summary` is the title shown in Scalar/Swagger — keep it short and specific.
- `remarks` explains non-obvious behaviour, side effects, or important constraints.
- Every `[ProducesResponseType]` must have a matching `<response code="...">` doc comment.
- Document every response code the endpoint can realistically return — not just `200`.
- Use `[Tags("Orders")]` to group endpoints in the UI — one tag per controller/resource.

---

## Model Documentation

Every public property on a request or response model needs a `<summary>`. For non-obvious properties, include valid values, format, or constraints.

```csharp
/// <summary>
/// Represents a customer order.
/// </summary>
public class OrderResponse
{
    /// <summary>
    /// Unique identifier for the order.
    /// </summary>
    public string Id { get; set; }

    /// <summary>
    /// Current status of the order.
    /// </summary>
    /// <example>Pending</example>
    public OrderStatus Status { get; set; }

    /// <summary>
    /// Total order value in AUD, inclusive of GST.
    /// </summary>
    /// <example>149.99</example>
    public decimal Total { get; set; }

    /// <summary>
    /// UTC timestamp when the order was placed.
    /// </summary>
    /// <example>2025-06-15T10:30:00Z</example>
    public DateTime CreatedAt { get; set; }
}
```

Use `<example>` on properties where a sample value makes the format clearer — especially dates, enums, and monetary values.

---

## Parameter Validation

Validate at the API boundary. Services and domain logic should never receive invalid input.

### Data annotations for simple rules

```csharp
public class CreateOrderRequest
{
    /// <summary>Customer ID placing the order.</summary>
    [Required]
    public string CustomerId { get; set; }

    /// <summary>Line items in the order. At least one required.</summary>
    [Required, MinLength(1)]
    public List<OrderItemRequest> Items { get; set; }

    /// <summary>Discount coupon code, if applicable.</summary>
    [MaxLength(50)]
    public string? CouponCode { get; set; }
}
```

### FluentValidation for complex rules

Use FluentValidation when rules involve business logic, cross-property checks, or async lookups.

```csharp
public class CreateOrderRequestValidator : AbstractValidator<CreateOrderRequest>
{
    public CreateOrderRequestValidator()
    {
        RuleFor(x => x.CustomerId).NotEmpty();
        RuleFor(x => x.Items).NotEmpty();
        RuleForEach(x => x.Items).ChildRules(item =>
        {
            item.RuleFor(x => x.Quantity).GreaterThan(0);
            item.RuleFor(x => x.ProductId).NotEmpty();
        });
    }
}
```

### Validation failure response

Validation failures return `400` with a `ProblemDetails` body listing every field error:

```json
{
  "type": "https://tools.ietf.org/html/rfc7807",
  "title": "Validation Failed",
  "status": 400,
  "errors": {
    "CustomerId": ["Customer ID is required."],
    "Items": ["At least one item is required."]
  }
}
```

ASP.NET Core does this automatically when `ModelState` is invalid. For FluentValidation, register it with `AddFluentValidationAutoValidation()`.

### Constrained string parameters

When a string parameter accepts a fixed set of values, use `[AllowedValues]` with named constants — never accept an open string and silently default to a fallback.

```csharp
// constants
public static class GrantTypeNames
{
    public const string ClientCredentials = "client_credentials";
    public const string AuthorizationCode  = "authorization_code";
}

// request model
[AllowedValues(GrantTypeNames.ClientCredentials, GrantTypeNames.AuthorizationCode)]
[Description("Grant type: 'client_credentials' or 'authorization_code'.")]
public string? GrantType { get; set; }
```

A parameter that silently falls through to a default on invalid input hides bugs and misleads callers. `[AllowedValues]` produces a `400` validation error with a clear message instead.

### Rules

- Never trust client input. Validate every request body, query parameter, and route value.
- Return all validation errors at once — not just the first one. Force clients to fix everything in one round trip.
- Route and query parameters that are IDs should be validated for format, not just presence.
- Don't validate in services — that's the controller/middleware boundary's job.

---

## Error Handling

Use `ProblemDetails` (RFC 7807) for all error responses. It is built into ASP.NET Core.

### Global exception handler

Register a global handler so unhandled exceptions never leak stack traces:

```csharp
// Program.cs
app.UseExceptionHandler(errorApp =>
{
    errorApp.Run(async context =>
    {
        var problem = context.RequestServices
            .GetRequiredService<IProblemDetailsService>();

        context.Response.StatusCode = StatusCodes.Status500InternalServerError;
        await problem.WriteAsync(new ProblemDetailsContext
        {
            HttpContext = context,
            ProblemDetails =
            {
                Title = "An unexpected error occurred.",
                Detail = null   // never expose internals
            }
        });
    });
});
```

### Domain exception mapping

Map known domain exceptions to HTTP status codes in one place — not scattered across controllers:

```csharp
public class DomainExceptionHandler : IExceptionHandler
{
    public async ValueTask<bool> TryHandleAsync(
        HttpContext context, Exception exception, CancellationToken ct)
    {
        var (status, title) = exception switch
        {
            NotFoundException    => (404, "Resource not found."),
            ForbiddenException   => (403, "Access denied."),
            ConflictException    => (409, "Conflict."),
            ValidationException  => (422, "Validation failed."),
            _                    => (0, null)
        };

        if (status == 0) return false;    // let the global handler take it

        context.Response.StatusCode = status;
        await context.Response.WriteAsJsonAsync(new ProblemDetails
        {
            Status = status,
            Title = title
        }, ct);

        return true;
    }
}
```

### Rules

- Every error response is `ProblemDetails`. No custom error shapes.
- `500` responses never include exception messages, stack traces, or internal identifiers.
- Log the full exception server-side before returning the sanitised response.
- Domain exceptions are caught and mapped — they never reach the global handler as unhandled.

---

## Logging

Use Serilog with structured logging. No `string.Format`-style log messages.

### Setup

```csharp
// Program.cs
builder.Host.UseSerilog((ctx, config) =>
{
    config
        .ReadFrom.Configuration(ctx.Configuration)
        .Enrich.FromLogContext()
        .Enrich.WithCorrelationId()
        .WriteTo.Console(new JsonFormatter());
});
```

### Request logging

Log every inbound request and its outcome with `UseSerilogRequestLogging()`:

```csharp
app.UseSerilogRequestLogging(options =>
{
    options.EnrichDiagnosticContext = (diagnostics, context) =>
    {
        diagnostics.Set("UserId", context.User?.FindFirst("sub")?.Value);
        diagnostics.Set("ClientIp", context.Connection.RemoteIpAddress);
    };
});
```

### Structured log messages

```csharp
// Good — structured, properties are queryable
_logger.LogInformation("Order {OrderId} placed by {CustomerId}", order.Id, order.CustomerId);

// Bad — interpolated string, properties are lost
_logger.LogInformation($"Order {order.Id} placed by {order.CustomerId}");
```

### Log levels

| Level | Use for |
| --- | --- |
| `Debug` | Detailed diagnostics — dev only, never in prod by default |
| `Information` | Normal business events: order placed, user logged in |
| `Warning` | Unexpected but recoverable: retry triggered, deprecated endpoint hit |
| `Error` | Failures that need attention: unhandled exception, external service down |
| `Critical` | System-wide failure: database unreachable, app cannot start |

### What to log

- Inbound requests (handled by `UseSerilogRequestLogging`)
- Business events: resource created, status changed, key actions taken
- All unhandled exceptions (full exception with stack trace)
- Authentication failures and authorisation denials
- External service calls that fail or are slow

### What never to log

- Passwords, tokens, API keys, or any credential
- PII — names, emails, phone numbers, addresses (unless a specific compliance need is documented)
- Full request/response bodies by default — they often contain sensitive data
- High-frequency noise that adds no diagnostic value

---

## Related

- [API Design](api-design.md)
- [Security Practices](security.md)
- [Testing](testing.md)

---
*Maintained by paurodriguez0220 · Last updated: 2026-06-15*
*Standards: https://github.com/paurodriguez0220/standards-docs*
