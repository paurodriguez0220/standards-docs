## Purpose

How I provision and manage Azure infrastructure — naming conventions, bicep structure, and environment configuration.

---

## Naming Convention

Follow [Microsoft Cloud Adoption Framework](https://learn.microsoft.com/en-us/azure/cloud-adoption-framework/ready/azure-best-practices/resource-abbreviations) abbreviations.

Pattern: `{type}-{app}[-{purpose}]-{env}-{region}`

| Resource Type | Abbr | Example |
| --- | --- | --- |
| Resource Group | `rg` | `rg-api-dev-ase` |
| App Service Plan | `asp` | `asp-api-dev-ase` |
| App Service | `app` | `app-api-dev-ase` |
| Function App | `func` | `func-api-dev-ase` |
| Key Vault | `kv` | `kv-api-dev-ase` |
| Storage Account | `st` | `stApiDevAse` (no hyphens — see below) |
| SQL Server | `sql` | `sql-api-dev-ase` |
| Redis Cache | `redis` | `redis-api-dev-ase` |
| Cosmos DB | `cosmos` | `cosmos-api-dev-ase` |
| App Insights | `appi` | `appi-api-dev-ase` |
| Log Analytics | `log` | `log-api-dev-ase` |

**Storage accounts** cannot contain hyphens — concatenate: `st{app}[{purpose}]{env}{region}`.

**App code** is the universal key. It maps a source folder to a resource group to a workflow. Pick it once, use it everywhere. No exceptions.

### Purpose Discriminator

When an app needs more than one instance of the same resource type, add a short `{purpose}` segment.

```
{type}-{app}-{purpose}-{env}-{region}
```

Rules:
- Omit `{purpose}` for the default/primary instance — add it only to peers.
- 2–6 lowercase alphanumeric characters. Short — storage account names cap at 24 characters.
- One code, one meaning across the entire codebase.

### Environments

| Environment | Abbr |
| --- | --- |
| Development | `dev` |
| Test | `test` |
| Production | `prod` |
| Disaster Recovery | `dr` |

### Regions

Region is **hardcoded in bicep** per app — not a parameter.

---

## Bicep Structure

### Module per resource

`main.bicep` is a **thin orchestrator** — it declares parameters, calls modules, and wires outputs. Target: ~150 lines. If it's growing past 200, resource definitions are in the orchestrator. Extract them.

Every Azure resource gets its own module. RBAC assignments get their own module — don't bury role assignments inside the resource module.

```
infra/{app}/
├── main.bicep                   ← thin orchestrator
├── modules/
│   ├── appServicePlan.bicep
│   ├── webApp.bicep
│   ├── functionApp.bicep
│   ├── keyVault.bicep
│   ├── keyVault.rbac.bicep      ← RBAC separate from the resource
│   ├── storageAccount.bicep
│   ├── storageAccount.rbac.bicep
│   ├── appInsights.bicep
│   └── logAnalytics.bicep
└── parameters/
    ├── dev.bicepparam
    ├── test.bicepparam
    ├── prod.bicepparam
    └── dr.bicepparam
```

Use what the app needs — not every module is required. The pattern is always: one module per resource, RBAC separate, thin orchestrator.

**Why module-per-resource:**
- Each module is independently reviewable in a PR
- `main.bicep` reads like a table of contents — visible at a glance what an app provisions
- AI agents can work on one module without touching others

**Why separate RBAC modules:**
- Role assignments change independently from the resource itself
- Adding a new consumer means editing the RBAC module, not the resource module
- Access grants are greppable across the repo

### Parameters — one file per environment

Each environment gets its own `.bicepparam` file. No environment detection logic inside bicep — the param file is the only thing that changes between environments.

```bicep
// parameters/dev.bicepparam
using '../main.bicep'

param appName = 'api'
param env = 'dev'
param location = 'australiasoutheast'
param sku = 'B1'
```

For secrets, use `readEnvironmentVariable()` with an empty string default:

```bicep
// parameters/dev.bicepparam
param dbPassword = readEnvironmentVariable('DB_PASSWORD', '')
```

The empty default is load-bearing — it means a local run without the env var skips the Key Vault write rather than writing a blank value. Never use a non-empty default.

In bicep, guard the write conditionally:

```bicep
resource dbSecret 'Microsoft.KeyVault/vaults/secrets@2023-07-01' = if (!empty(dbPassword)) {
  parent: keyVault
  name: 'DbPassword'
  properties: { value: dbPassword }
}
```

### Rules

- Bicep is the single source of truth. If it's not in bicep, it doesn't exist.
- All app settings defined in bicep — nothing added manually after deployment.
- Managed identity + RBAC for service-to-service auth. No connection strings with passwords.
- Key Vault references for runtime secrets: `@Microsoft.KeyVault(SecretUri=...)`.
- `@secure()` for any deployment-time secret parameter.
- No hardcoded secret values. No manual post-deployment steps.
- No staging slots — rollback means redeploying the previous release tag.
- `az cli` is read-only (inspect/query only). Bicep is the write tool.

---

## Related

- [Git Workflow](git-workflow.md)
- [Security Practices](security.md)

---
*Maintained by paurodriguez0220 · Last updated: 2026-06-15*
*Standards: https://github.com/paurodriguez0220/standards-docs*
