## Purpose

Every project ships with Playwright for E2E testing. Playwright validates behaviour against a **live deployed environment** — it is not a pre-merge gate. Unit and integration tests block merges; Playwright confirms the deployment worked.

---

## When Playwright Runs

| Trigger | When | Target |
| --- | --- | --- |
| Post-deployment | Automatically after a successful deploy | The just-deployed environment |
| Manual (on demand) | Developer triggered during active development | Dev or test environment |

Playwright **never runs in the unit test pipeline** and **never blocks a merge**. It runs after code is already deployed.

---

## Project Structure

Playwright lives in `tests/e2e/` within the repo, versioned with the application it tests.

```
tests/
├── unit/                   ← runs in CI, blocks merge
└── e2e/                    ← runs against live env, never blocks merge
    ├── pages/              ← Page Object Models
    │   ├── login.page.ts
    │   └── orders.page.ts
    ├── tests/              ← spec files
    │   ├── auth.spec.ts
    │   └── orders.spec.ts
    ├── fixtures/           ← shared setup, custom fixtures
    │   └── index.ts
    ├── playwright.config.ts
    └── .env.example        ← required env vars, no values
```

---

## Workflows

Two separate workflow files per app:

```
.github/workflows/
├── cicd-{app}.yml          ← build, test (unit/integration), deploy
└── e2e-{app}.yml           ← Playwright only, post-deploy + manual
```

### e2e-{app}.yml

```yaml
name: "{AppName} E2E Tests"

on:
  workflow_dispatch:
    inputs:
      environment:
        description: 'Target environment'
        required: true
        type: choice
        options: [dev, test, prod]
  workflow_call:
    inputs:
      environment:
        required: true
        type: string

jobs:
  e2e:
    runs-on: ubuntu-latest
    environment: ${{ inputs.environment }}
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: 20

      - name: Install dependencies
        run: npm ci
        working-directory: tests/e2e

      - name: Install Playwright browsers
        run: npx playwright install --with-deps chromium
        working-directory: tests/e2e

      - name: Run Playwright tests
        run: npx playwright test
        working-directory: tests/e2e
        env:
          BASE_URL: ${{ vars.E2E_BASE_URL }}
          TEST_USER_EMAIL: ${{ secrets.E2E_TEST_USER_EMAIL }}
          TEST_USER_PASSWORD: ${{ secrets.E2E_TEST_USER_PASSWORD }}

      - name: Upload test report
        if: failure()
        uses: actions/upload-artifact@v4
        with:
          name: playwright-report-${{ inputs.environment }}
          path: tests/e2e/playwright-report/
          retention-days: 7
```

### Calling e2e from cicd after deploy

```yaml
# cicd-{app}.yml — after the deploy job completes
  e2e:
    needs: deploy
    uses: ./.github/workflows/e2e-{app}.yml
    with:
      environment: ${{ github.event.inputs.environment }}
    secrets: inherit
```

---

## Configuration

```ts
// tests/e2e/playwright.config.ts
import { defineConfig, devices } from '@playwright/test';

export default defineConfig({
  testDir: './tests',
  fullyParallel: true,
  forbidOnly: !!process.env.CI,
  retries: process.env.CI ? 2 : 0,
  reporter: process.env.CI ? 'github' : 'list',
  use: {
    baseURL: process.env.BASE_URL,
    trace: 'on-first-retry',
    screenshot: 'only-on-failure',
  },
  projects: [
    { name: 'chromium', use: { ...devices['Desktop Chrome'] } },
  ],
});
```

Rules:
- `baseURL` always from `process.env.BASE_URL` — never hardcoded.
- `forbidOnly` in CI — a committed `.only` breaks the build.
- `retries: 2` in CI only — retries locally hide flaky tests.
- Traces and screenshots on failure, always.

---

## Page Object Model

Tests never contain raw selectors or direct Playwright calls — that logic lives in page objects.

```ts
// tests/e2e/pages/login.page.ts
import { Page, Locator } from '@playwright/test';

export class LoginPage {
  private readonly emailInput: Locator;
  private readonly passwordInput: Locator;
  private readonly submitButton: Locator;

  constructor(private page: Page) {
    this.emailInput = page.getByLabel('Email');
    this.passwordInput = page.getByLabel('Password');
    this.submitButton = page.getByRole('button', { name: 'Sign in' });
  }

  async goto() {
    await this.page.goto('/login');
  }

  async login(email: string, password: string) {
    await this.emailInput.fill(email);
    await this.passwordInput.fill(password);
    await this.submitButton.click();
  }
}
```

---

## Selectors

| Priority | Selector | Example |
| --- | --- | --- |
| 1 | `getByRole` | `getByRole('button', { name: 'Submit' })` |
| 2 | `getByLabel` | `getByLabel('Email address')` |
| 3 | `getByText` | `getByText('Order confirmed')` |
| 4 | `getByTestId` | `getByTestId('order-summary')` |
| 5 | CSS / XPath | Last resort only |

Add `data-testid` to elements with no semantic role or label. Never use generated class names or DOM structure.

---

## What to Test

Critical user journeys only — the flows that, if broken, mean the deployment failed.

Good candidates:
- Authentication (login, logout)
- Core create / read / update flows
- Submission or checkout flows
- Permission boundaries

Not a good fit:
- Unit-level logic
- Every form validation message
- Anything already covered by integration tests

---

## Test Isolation

- Each test creates its own state — no relying on data from a previous test.
- Tests run in any order and in parallel without affecting each other.
- Never commit `test.only` — `forbidOnly: true` in CI enforces this.
- Use fixtures for repeated setup (authenticated session, seeded record).

---

## Assertions

Use Playwright's built-in `expect` — it auto-retries. Never use `waitForTimeout` as a substitute.

```ts
// Good
await expect(page.getByRole('heading', { name: 'Dashboard' })).toBeVisible();
await expect(page).toHaveURL('/dashboard');

// Bad
await page.waitForTimeout(2000);
```

---

## Related

- [Testing](testing.md)
- [GitHub Actions](github-actions.md)

---
*Maintained by paurodriguez0220 · Last updated: 2026-06-15*
*Standards: https://github.com/paurodriguez0220/standards-docs*
