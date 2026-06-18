## Purpose

A guide for recognising when a design pattern solves a real problem — and when it doesn't.

Patterns are vocabulary, not rules. Naming a pattern in a code review or design discussion is useful. Reaching for a pattern before you have the problem is not.

---

## Before You Reach for a Pattern

Apply YAGNI and the three-strikes rule first:

- **Do you have the problem now?** If not, don't solve it yet.
- **Have you seen the duplication or coupling three times?** Then extract. Not before.
- **Can a simpler structure — a function, a module boundary, a clear name — solve it?** Prefer that.

A class diagram that impresses no one is still better than one that confuses everyone.

---

## Creational

### Factory / Factory Method

Use when object construction logic is complex, varies by type, or needs to be deferred.

```ts
// instead of callers deciding which concrete type to build:
const notifier = NotifierFactory.create(config.channel); // "email" | "sms" | "slack"
```

Don't use it to avoid a constructor. An `new EmailNotifier(config)` is fine if there's only one type.

### Builder

Use when constructing an object with many optional parts, especially when construction order matters.

```ts
const query = new QueryBuilder()
  .from("orders")
  .where("status", "pending")
  .orderBy("createdAt", "desc")
  .limit(50)
  .build();
```

Don't use it for objects with 2–3 fields. That's a config object or plain constructor.

### Singleton

Use only for genuinely shared, stateless infrastructure — a logger, a connection pool, a config registry.

Rules:
- Never store mutable application state in a singleton.
- Inject it as a dependency — don't reach for it globally.
- Prefer dependency injection containers over hand-rolled singletons.

---

## Structural

### Repository

Use to abstract data access behind a domain-focused interface. Callers work with domain objects — the repository owns the persistence logic.

```ts
interface OrderRepository {
  findById(id: string): Promise<Order>;
  findPending(): Promise<Order[]>;
  save(order: Order): Promise<void>;
}
```

The repository hides the ORM, query builder, or HTTP call. The domain layer never imports infrastructure.

### Adapter

Use to wrap an incompatible interface so the rest of the system doesn't have to care about it.

Good fit for: third-party SDKs, legacy systems, external APIs. Localises the ugly.

### Decorator

Use to add behaviour to an object without modifying it — logging, caching, validation layered around a core service.

Prefer over inheritance. Composition is more flexible and easier to test.

---

## Behavioural

### Strategy

Use when you have a family of interchangeable algorithms or behaviours and want to select one at runtime.

```ts
interface PricingStrategy {
  calculate(order: Order): number;
}
// StandardPricing, DiscountPricing, SubscriberPricing all implement the same interface
```

Replaces large `switch` or `if/else` blocks that would otherwise grow with each new case.

### Observer / Event-Driven

Use when one thing needs to notify others without knowing what they are.

Good fit for: domain events, UI state changes, cross-module side effects.

Rules:
- Keep event payloads immutable.
- Name events in past tense: `OrderPlaced`, `UserDeactivated`.
- Don't chain events that trigger events — it creates invisible, hard-to-debug flows.

### Command

Use to encapsulate an operation as an object — useful for queuing, undo/redo, or audit logging.

```ts
interface Command {
  execute(): Promise<void>;
}
```

Good fit for workflows, background jobs, and anywhere you need a record of what ran.

---

## Anti-Patterns

### Pattern-hunting
Identifying a GoF pattern and then shaping the code to fit it. The pattern should emerge from the problem, not the other way around.

### Premature abstraction
Creating interfaces and abstractions before there are multiple implementations. One implementation = no interface needed yet.

### God object
A class that knows too much and does too much. Split by responsibility. If you can't describe the class in one sentence, it's already two classes.

### Anemic domain model
Domain objects that are just bags of data with no behaviour. Business logic ends up scattered across services. Move behaviour back to the domain.

### Cargo-cult architecture
Layered architecture, hexagonal ports, CQRS — applied because the pattern is prestigious, not because the problem requires it. Match the architecture to the scale and complexity of the problem.

---

## Related

- [Code Style & Conventions](code-style.md)
- [API Design](api-design.md)
- [Testing](testing.md)

---
*Maintained by paurodriguez0220 · Last updated: 2026-06-15*
*Standards: https://github.com/paurodriguez0220/standards-docs*
