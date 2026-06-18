## Purpose

Conventions for GitHub Actions workflows — naming, structure, secrets, environment variables, and triggers.

---

## Workflow File Naming

```
{type}-{app}.yml
```

| Type | Purpose | Example |
| --- | --- | --- |
| `cicd` | Build, test, deploy | `cicd-api.yml` |
| `e2e` | Playwright E2E tests | `e2e-api.yml` |
| `infra` | Infrastructure provisioning | `infra-api.yml` |
| `cron` | Scheduled jobs | `cron-data-sync.yml` |

One workflow per concern. Don't combine deploy and infra in the same file.

---

## Workflow Structure

Every workflow follows the same skeleton:

```yaml
name: "{AppName} {Type}"           # human-readable, shown in GitHub UI

on:
  ...                               # triggers

permissions:
  contents: read                    # least privilege — grant only what's needed
  id-token: write                   # only if using OIDC

env:
  DOTNET_VERSION: '9.0.x'          # shared across jobs — define once at top level

jobs:
  job-name:                         # kebab-case
    name: "Human readable name"     # shown in GitHub UI
    runs-on: ubuntu-latest
    environment: ${{ inputs.environment }}
    steps:
      - name: "Verb phrase"         # every step has a name, starts with a capital verb
        ...
```

### Rules

- Workflow `name` is displayed in the GitHub UI — make it human-readable.
- Every step has a `name`. Unnamed steps are unreadable in logs.
- Step names start with a capital verb: `Install dependencies`, `Run tests`, `Deploy to Azure`.
- Jobs use kebab-case: `build`, `run-tests`, `deploy-infra`.

---

## Triggers

| Trigger | Use for |
| --- | --- |
| `push` to `main` | Automatic deploy to dev/test |
| `workflow_dispatch` | Manual deploys, infra runs, E2E on demand |
| `workflow_call` | Reusable workflows called from other workflows |
| `schedule` | Cron jobs |
| `pull_request` | Unit/integration tests on PRs |

**Infra workflows are manual only** (`workflow_dispatch`). Infrastructure changes are intentional, never automatic.

**E2E workflows are manual or post-deploy** (`workflow_dispatch` + `workflow_call`). Never triggered on `push` or `pull_request`.

---

## Secrets vs Variables

| Type | Use for | Access in workflow |
| --- | --- | --- |
| **Secret** (`secrets.`) | Passwords, tokens, private keys — anything sensitive | `${{ secrets.NAME }}` |
| **Variable** (`vars.`) | Non-sensitive config — URLs, client IDs, feature flags | `${{ vars.NAME }}` |

Never store non-sensitive values as secrets — it makes them invisible and harder to audit. OIDC identifiers (`CLIENT_ID`, `TENANT_ID`, `SUBSCRIPTION_ID`) are variables, not secrets.

---

## Naming Conventions

### Secrets

```
SCREAMING_SNAKE_CASE
{APP}_{SECRET_NAME}        ← app-scoped secrets
{SECRET_NAME}              ← repo-wide or shared secrets
```

| Example | Scope |
| --- | --- |
| `DB_PASSWORD` | Repo-wide |
| `EFS_SFTP_PASSWORD` | App-scoped (`efs`) |
| `E2E_TEST_USER_PASSWORD` | E2E test user |

### Variables

Same pattern as secrets:

| Example | Value |
| --- | --- |
| `AZURE_CLIENT_ID` | Repo-wide OIDC client ID |
| `AZURE_TENANT_ID` | Repo-wide OIDC tenant ID |
| `AZURE_SUBSCRIPTION_ID` | Repo-wide subscription ID |
| `E2E_BASE_URL` | Base URL for Playwright tests |
| `EFS_AZURE_CLIENT_ID` | App-scoped override (different subscription) |

---

## Environments

Use GitHub Environments for all deployments. They provide:
- Protection rules (required reviewers for prod)
- Environment-scoped secrets and variables
- Deployment history

Every deployable environment maps to a GitHub Environment: `dev`, `test`, `prod`, `dr`.

```yaml
jobs:
  deploy:
    environment: ${{ inputs.environment }}   # always scoped to an environment
```

Never access production secrets outside a `prod` environment — scoping is the guard.

---

## Passing Secrets to Steps

Secrets are passed as environment variables into steps — never interpolated directly into run commands.

```yaml
# Good — secret passed as env var
- name: Deploy
  run: ./deploy.sh
  env:
    DB_PASSWORD: ${{ secrets.DB_PASSWORD }}

# Bad — secret interpolated into the command (visible in logs)
- name: Deploy
  run: ./deploy.sh --password ${{ secrets.DB_PASSWORD }}
```

---

## Pre-flight Guards

Any step that requires a secret should validate it exists before doing real work.

```yaml
- name: Validate required secrets
  run: |
    if [ -z "$DB_PASSWORD" ]; then
      echo "::error::DB_PASSWORD not set in GitHub Environment — aborting"
      exit 1
    fi
  env:
    DB_PASSWORD: ${{ secrets.DB_PASSWORD }}
```

Silent skips (`exit 0` on missing secret) are not acceptable — fail loudly.

---

## Pinning Actions

Pin third-party actions to a full commit SHA, not a mutable tag.

```yaml
# Good — pinned to SHA
- uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683  # v4.2.2

# Bad — mutable tag, can be changed upstream
- uses: actions/checkout@v4
```

First-party actions (`actions/`, `azure/`) may use version tags if the org has verified them. Third-party actions always pin to SHA.

---

## Reusable Workflows

Extract a reusable workflow (`workflow_call`) when the same job sequence appears in more than one workflow file.

```yaml
# .github/workflows/e2e-api.yml
on:
  workflow_call:
    inputs:
      environment:
        required: true
        type: string
```

Call it with `uses: ./.github/workflows/e2e-api.yml` and pass `secrets: inherit` so the calling workflow's environment secrets are available.

---

## Related

- [Git Workflow](git-workflow.md)
- [Azure Infrastructure](azure-infra.md)
- [Playwright](playwright.md)
- [Security Practices](security.md)

---
*Maintained by paurodriguez0220 · Last updated: 2026-06-15*
*Standards: https://github.com/paurodriguez0220/standards-docs*
