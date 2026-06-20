## Purpose

Testing philosophy and standards that apply across projects regardless of language or framework.

---

## Every Project Must Have Tests

No exceptions. A project without a test project has no regression safety net. Shipping code without tests means every future change is a gamble.

- Create a test project alongside the main project from the start — adding tests later is harder and rarely happens.
- At minimum, cover the happy path and at least one error path for every public-facing behaviour.

---

## Philosophy

- Tests are first-class code. They deserve the same care as production code.
- A test that never fails is not a test — it's a false guarantee.
- Test behaviour, not implementation. Tests should survive refactors.
- Aim for fast feedback. Slow tests don't get run.
- Coverage is a floor, not a goal. 80% with meaningful assertions beats 100% with shallow ones.

---

## Test Types

| Type | What it tests | Speed | When to use |
| --- | --- | --- | --- |
| Unit | A single function or class in isolation | Fast | Business logic, calculations, transformations |
| Integration | Multiple components working together | Medium | DB queries, external service calls, API endpoints |
| E2E | Full flows from the user's perspective | Slow | Critical user journeys against a live environment — see [Playwright](playwright.md) |

Write more unit tests than integration tests. Write more integration tests than E2E tests.

---

## Structure — Arrange, Act, Assert

Every test follows three phases, with a blank line separating each:

```ts
it('returns discounted price when coupon is valid', () => {
  // Arrange
  const cart = new Cart([{ price: 100 }]);
  const coupon = new Coupon('SAVE10', 0.1);

  // Act
  const total = cart.applyDiscount(coupon);

  // Assert
  expect(total).toBe(90);
});
```

One assertion per test where possible. Multiple assertions are fine if they all verify the same behaviour — not if they're testing different scenarios.

---

## Naming

Test names describe: **what** is being tested, **under what condition**, and **what the expected outcome is**.

```
// Good
calculateTotal_whenDiscountApplied_returnsReducedPrice
getUser_whenUserNotFound_throws404

// Bad
test1
testCalculate
itWorks
```

---

## What to Test

- Happy path — the expected use case
- Edge cases — empty input, nulls, zero, boundaries
- Error paths — what happens when things go wrong
- Regression cases — add a test every time a bug is fixed

## What Not to Test

- Framework or library code you didn't write
- Trivial getters/setters with no logic
- Private implementation details — test through the public interface
- Configuration values

---

## Test Doubles

Prefer real dependencies over mocks where practical. Mocks that don't reflect real behaviour give false confidence.

| Double | Use when |
| --- | --- |
| **Fake** (lightweight real impl) | In-memory DB, in-process queue — fast and realistic |
| **Stub** | Controlling indirect inputs (e.g. fixed return values from a date service) |
| **Mock** | Verifying interactions — use sparingly, only when the call itself is the behaviour under test |

Rules:
- Never mock what you don't own. Wrap third-party dependencies and mock the wrapper.
- If a mock setup is longer than the test itself, the test is testing the wrong thing.
- Integration tests should hit real infrastructure where feasible (real DB, real queue) — mocked infra has hidden divergence.

---

## Test Data

- Tests own their data. Don't rely on pre-seeded data that can change.
- Each test sets up what it needs and tears it down after.
- Use builders or factories for complex objects — avoid large literal object construction inline.
- Never use production data in tests.

---

## CI Requirements

- All tests must pass before a PR can merge. No exceptions.
- Tests must be deterministic — flaky tests are fixed or deleted immediately.
- Unit tests must run in under 30 seconds total. Slow unit tests are a design smell.
- Integration and E2E tests run in CI on every PR but may run in a separate stage.

---

## Related

- [Code Style & Conventions](code-style.md)
- [API Design](api-design.md)
- [Web Components & Storybook](web-components.md)

---
*Maintained by paurodriguez0220 · Last updated: 2026-06-18*
*Standards: https://github.com/paurodriguez0220/standards-docs*
