# Microsoft Foundry - Automated Deployment with Bicep & GitHub Actions

## Overview

- **Bicep** for declarative infrastructure
- **Azure Developer CLI (azd)** for interactive multi-subscription deployments
- **GitHub Actions** with OIDC authentication (no stored secrets)
- **Environment-specific parameters** (dev, staging, production)
- **Simple two-stage pipeline**: automated `validate` on every PR/push, on-demand `deploy-manual` for actual deployments
- **Multi-environment support** (dev → stg → prod) with GitHub Environment approval gates on manual deploys

> **Resource model:** This deploys a **Microsoft Foundry resource** — a `Microsoft.CognitiveServices/accounts` resource (kind `AIServices`) with `allowProjectManagement: true` — with Foundry Projects as child resources (`Microsoft.CognitiveServices/accounts/projects`).

> **Authentication:** Local (API key) authentication is disabled on the Foundry resource (`disableLocalAuth: true`). Access is via **Microsoft Entra ID (AAD)** only, using RBAC role assignments on the resource's SystemAssigned managed identity.

## Repository Structure

```
.
├── .github
│   └── workflows
│       └── deploy-foundry.yml       # CI/CD pipeline (validate, deploy-manual)
├── infra
│   ├── modules
│   │   ├── foundry.bicep            # Microsoft Foundry resource + model deployments + optional account capability host
│   │   ├── project.bicep            # Foundry Project (child resource) + optional Standard Agent Setup connections
│   │   ├── appinsights.bicep        # Application Insights module
│   │   ├── kv.bicep                 # Key Vault module
│   │   ├── storage.bicep            # Storage Account module
│   │   ├── cosmosdb.bicep           # Cosmos DB account (Standard Agent Setup only)
│   │   ├── aisearch.bicep           # Azure AI Search service (Standard Agent Setup only)
│   │   ├── project-capability-host.bicep          # Project-level Agents capability host (Standard Agent Setup only)
│   │   ├── format-workspace-id.bicep               # Formats project internalId into GUID form (Standard Agent Setup only)
│   │   ├── storage-container-role-assignment.bicep # Container-scoped Storage RBAC with ABAC condition (Standard Agent Setup only)
│   │   ├── cosmosdb-container-role-assignment.bicep # Database-scoped Cosmos DB SQL role assignment (Standard Agent Setup only)
│   │   ├── entra-app-registration.bicep # Entra ID App Registration + Service Principal (Apigee gateway integration only)
│   │   ├── types.bicep              # Shared user-defined types (tags, model deployments, projects)
│   │   └── role-assignment.bicep    # RBAC role assignment (Key Vault, Storage, Cognitive Services, Cosmos DB, AI Search)
│   ├── main.bicep                   # Main orchestrator (subscription scope)
│   ├── dev.main.bicepparam          # Dev parameters, minimal (Basic Agent Setup, Foundry-only)
│   ├── dev-standard.main.bicepparam # Dev parameters, full BYO-storage (Standard Agent Setup)
│   ├── stg.main.bicepparam          # Staging parameters (Basic Agent Setup)
│   ├── prod.main.bicepparam         # Production parameters, eastus (Standard Agent Setup example)
│   └── prod-secondary-region.main.bicepparam  # Production parameters, westus2 (multi-region example)
├── docs
│   ├── architecture.md              # Resource model, auth, multi-region, Apigee/Entra ID design
│   ├── deployment-guide.md          # Step-by-step deployment instructions
│   ├── model-lifecycle-demo.md      # Scripted model add/delete demo runbook
│   ├── operational-handoff.md       # Ownership, monitoring, incident response, cost management
│   └── knowledge-transfer.md        # KT session agenda, success-criteria demo mapping, FAQ
├── azure.yaml                        # Azure Developer CLI project config
├── .gitignore
└── README.md
```

## Agent Setup Modes

This template supports two ways of provisioning the backing infrastructure that Microsoft Foundry Agents use for thread/conversation storage, vector search, and file storage. Choose the mode per environment with the `agentSetupType` parameter (`main.bicep`):

| Mode | `agentSetupType` | Description | Extra resources deployed |
|------|-------------------|--------------|---------------------------|
| **Basic** (default) | `'Basic'` | Microsoft manages the Agents backing infrastructure automatically. Simplest option — no Key Vault/Storage/Cosmos DB/AI Search deployed by this template. | Foundry resource + Projects + Application Insights only |
| **Standard** | `'Standard'` | You own and deploy the Key Vault (secrets), Storage (file storage), Cosmos DB (thread storage), and AI Search (vector store) resources used by Agents, wired up via AAD-only connections and Agents capability hosts (account + per-project). Gives full control over these resources (networking, backup, cost, compliance). | Key Vault, Storage account, Cosmos DB account, AI Search service, per-project capability host, container/database-scoped RBAC |

To use **Standard** mode for an environment:

1. Set `param agentSetupType = 'Standard'` in that environment's `.bicepparam` file.
2. Provide globally-unique `kvName`, `storageName`, `cosmosDBName`, and `aiSearchName` param values.
3. Ensure `deployRoleAssignments = true` (Standard mode's capability hosts and container-scoped RBAC require the deploying identity to grant roles).

See `infra/dev-standard.main.bicepparam` and `infra/prod.main.bicepparam` for working Standard Agent Setup examples, and `infra/stg.main.bicepparam` for the commented-out guidance on switching from Basic to Standard. `infra/dev.main.bicepparam` is deliberately minimal (Basic mode, no backing-resource parameters) so DEV can be deployed either way without editing parameters — pick `DEV` or `DEV-STANDARD` in the workflow.

## Multi-Region Deployment

Every resource name and the `location`/`resourceGroupName` parameters are supplied per `.bicepparam` file, so deploying to additional Azure regions requires no Bicep changes — just a new parameter file:

- `infra/prod.main.bicepparam` — production, `eastus`
- `infra/prod-secondary-region.main.bicepparam` — production, `westus2` (working example, same Standard Agent Setup + model set, separate resource group and globally-unique names)

The `deploy-manual` GitHub Actions job also accepts an optional `region` input to override the target region at deploy time without a new parameter file. Traffic steering across regions (failover/load balancing) is not managed by this framework — see `docs/architecture.md` §6 for the recommended pattern (gateway/Front Door/Traffic Manager). To add another region, follow `docs/deployment-guide.md` §6.

## Entra ID Authentication via Apigee (API Gateway) Integration

For consumers that call Microsoft Foundry through an enterprise API gateway (e.g., **Apigee**) instead of directly, this framework can optionally provision the Azure-side of an Entra ID OAuth2 client-credentials trust:

| Parameter | Default | Purpose |
|---|---|---|
| `deployApigeeIntegration` | `false` | When `true`, creates an Entra ID App Registration + Service Principal (`modules/entra-app-registration.bicep`) and grants it the **Cognitive Services OpenAI User** role on the Foundry resource. |
| `apigeeGatewayAppDisplayName` | `'${namePrefix}-apigee-gateway'` | Display name for the App Registration. |

Outputs `apigeeGatewayAppId` (client ID) and `apigeeGatewayServicePrincipalId` are produced when enabled. **No client secret is created by Bicep** — generate one out-of-band (`az ad app credential reset --id <appId>`) and configure it in Apigee's OAuth2 policy. Enabling this parameter requires the deploying identity to hold an Entra ID **Application Administrator** or **Cloud Application Administrator** directory role (in addition to Azure RBAC). See `docs/architecture.md` §4.1 and `docs/deployment-guide.md` §7 for the full flow and setup steps.

## Documentation

Detailed engagement documentation lives under `docs/`:

| Document | Contents |
|---|---|
| [`docs/architecture.md`](docs/architecture.md) | Resource model, Agent Setup modes, RBAC/auth design, Entra ID/Apigee flow, multi-region topology, reusability design |
| [`docs/deployment-guide.md`](docs/deployment-guide.md) | Step-by-step deployment instructions (CLI, azd, GitHub Actions), model onboarding, region onboarding, Apigee setup, troubleshooting |
| [`docs/operational-handoff.md`](docs/operational-handoff.md) | Ownership model, monitoring/alerting, scaling, incident response, cost management, support escalation |
| [`docs/knowledge-transfer.md`](docs/knowledge-transfer.md) | KT session agenda, success-criteria-to-demo mapping, deliverables checklist, FAQ |
| [`docs/model-lifecycle-demo.md`](docs/model-lifecycle-demo.md) | Runbook for demoing model add/delete: scripted steps, talking points, expected output, recovery |

### Success Criteria & Final Deliverables

| Success Criterion | Satisfied by |
|---|---|
| Deploy Azure AI Foundry resources using Bicep templates | `infra/main.bicep` + `infra/modules/*.bicep` |
| Deploy supported models through GitHub Actions | `.github/workflows/deploy-foundry.yml` + `foundryModelDeployments` parameter |
| Onboard new models through configuration-driven processes | `.bicepparam` parameter changes only — no module code changes |
| Support deployments across multiple Azure regions | `infra/prod-secondary-region.main.bicepparam` + `region` workflow input |
| Utilize Entra ID-based authentication through Apigee integration | `deployApigeeIntegration` parameter + `modules/entra-app-registration.bicep` |
| Reuse the deployment framework for future Azure AI Foundry initiatives | Modular, environment-parameterized design (see `docs/architecture.md` §7) |

| Final Deliverable | Location |
|---|---|
| Azure AI Foundry Bicep deployment framework | `infra/` |
| Model deployment automation framework | `foundryModelDeployments` parameter + `modules/foundry.bicep` |
| GitHub Actions CI/CD workflows | `.github/workflows/deploy-foundry.yml` |
| Parameter and configuration templates | `infra/*.bicepparam` |
| Architecture documentation | `docs/architecture.md` |
| Deployment guide | `docs/deployment-guide.md` |
| Operational handoff documentation | `docs/operational-handoff.md` |
| Solution demonstration and KT session materials | `docs/knowledge-transfer.md` |

## Components Per Environment

| Component | Purpose | Security |
|-----------|---------|----------|
| **Microsoft Foundry resource** | Cognitive Services account (`kind: AIServices`, `allowProjectManagement: true`) hosting model deployments (GPT-4o, GPT-5.5/5.4/5.3, GPT-5 mini/nano, embeddings) | SystemAssigned managed identity, AAD-only auth (`disableLocalAuth: true`) |
| **Foundry Projects** | Child resources of the Foundry resource (2-3 per env) for access/data isolation | SystemAssigned managed identity |
| **Application Insights** | Monitoring & diagnostics | Performance tracking, logging |
| **Key Vault** *(Standard Agent Setup only)* | Secret/key management used by Foundry agents/tools | RBAC, soft-delete, purge protection |
| **Storage Account** *(Standard Agent Setup only)* | Agent file storage | No public blob access, HTTPS only, TLS 1.2+ |
| **Cosmos DB** *(Standard Agent Setup only)* | Agent thread/conversation storage | AAD-only (`disableLocalAuth: true`), database-scoped SQL role assignment |
| **Azure AI Search** *(Standard Agent Setup only)* | Agent vector store | AAD-only (`disableLocalAuth: true`), SystemAssigned managed identity |

## Prerequisites

- **Azure CLI** ([Install](https://learn.microsoft.com/cli/azure/install-azure-cli)) with Bicep CLI 0.20+ (bundled)
- **Azure Developer CLI (azd)** ([Install](https://learn.microsoft.com/azure/developer/azure-developer-cli/install-azd)) — optional, for local interactive deploys
- One or more Azure subscriptions with **Contributor** + **User Access Administrator** access
- A GitHub repository with **OIDC federation** configured for Azure (see below)

## Before You Deploy

Resource names for the Microsoft Foundry resource must be **globally unique** across Azure. Update the placeholder names (`devmfdfoundry001`, etc.) in `infra/dev.main.bicepparam`, `infra/stg.main.bicepparam`, and `infra/prod.main.bicepparam` before deploying — for example, append your initials or a random suffix.

If an environment uses **Standard Agent Setup** (`agentSetupType = 'Standard'`), also set globally-unique `kvName`, `storageName`, `cosmosDBName`, and `aiSearchName` values for that environment. These four params are optional and ignored — and the corresponding resources are **not deployed** — when `agentSetupType` is `'Basic'` (the default).

## Quick Start — Azure CLI

```powershell
az login
az account set --subscription "<your-subscription-id>"

az deployment sub validate `
  --location eastus `
  --template-file infra/main.bicep `
  --parameters infra/dev.main.bicepparam

az deployment sub create `
  --location eastus `
  --name foundry-dev-deployment `
  --template-file infra/main.bicep `
  --parameters infra/dev.main.bicepparam
```

Repeat with `stg.main.bicepparam` / `prod.main.bicepparam` for the other environments.

## Quick Start — Azure Developer CLI (azd)

```powershell
winget install microsoft.azd
az login

azd env new dev
azd up
```

`azd` will prompt for the target subscription, resource group, and location for the selected environment (`dev`, `stg`, or `prod`).

## GitHub Actions Workflow

`.github/workflows/deploy-foundry.yml` implements:

1. **validate** — `az deployment sub validate` for DEV and DEV-STANDARD on pull requests and pushes to `main` (`fail-fast: false` so both are checked even if one fails). This repository targets the DEV subscription only; see the comment on the `validate` job in `deploy-foundry.yml` for how to add STG/PROD once their federated credentials exist
2. **deploy-manual** — on-demand deployment to a chosen environment (`DEV`/`DEV-STANDARD`/`STG`/`PROD`/`PROD-SECONDARY-REGION`) via `workflow_dispatch`, with an optional `region` input to override the target Azure region without editing the `.bicepparam` file, and an optional `pruneOrphanedModels` input to delete model deployments that are no longer in the parameter file

This is a deliberately simple pipeline: `validate` gives fast automatic feedback, and all real
deployments go through the explicit `deploy-manual` trigger — there is no automatic push-to-deploy
chain and no What-If stage.

### Model Lifecycle

Deployments run in ARM **Incremental** mode, so adding a model to `foundryModelDeployments` creates
it, but *removing* one leaves the live deployment in place — it keeps consuming regional TPM quota
indefinitely. After every successful deploy, the **Reconcile model deployments** step diffs the
`.bicepparam` against `az cognitiveservices account deployment list` and reports any orphans as
warnings plus a job-summary block. Re-run the workflow with **`pruneOrphanedModels`** checked to
actually delete them — deletion is opt-in only, so a routine deploy can never drop a model by
accident. Note that deleting a deployment is immediately breaking for callers of that deployment
name. See `docs/deployment-guide.md` §5.1 for the recommended retirement sequence.

The Bicep CLI is installed/upgraded via `az bicep install` / `az bicep upgrade` (no manual binary downloads), and a `concurrency` group prevents overlapping runs of this workflow on the same branch from racing each other.

### OIDC Setup (no stored secrets)

For each environment (DEV, STG, PROD, and optionally PROD-SECONDARY-REGION), create a GitHub Environment in repo settings and configure:

1. An Entra ID App Registration (or reuse one) with a **federated credential** scoped to this repo/environment
2. Environment secrets: `AZURE_CLIENT_ID`, `AZURE_TENANT_ID`, `AZURE_SUBSCRIPTION_ID`
3. Role assignment: **Contributor** + **User Access Administrator** on the target subscription (or resource group scope, if `deployRoleAssignments` is set to `false`)
4. Optional environment variable `AZURE_LOCATION` (defaults to `eastus`)

Example `az` commands to create the federated credential:

```powershell
az ad app create --display-name "microsoft-foundry-deployment-automation-<env>"
$appId = az ad app list --display-name "microsoft-foundry-deployment-automation-<env>" --query "[0].appId" -o tsv
az ad sp create --id $appId

az ad app federated-credential create --id $appId --parameters '{
  "name": "github-<env>",
  "issuer": "https://token.actions.githubusercontent.com",
  "subject": "repo:<owner>/<repo>:environment:<ENV_NAME>",
  "audiences": ["api://AzureADTokenExchange"]
}'
```

### Environment Protection

Use GitHub Environments (`DEV`, `STG`, `PROD`) with required reviewers on `STG`/`PROD` to gate promotion between environments.

## Troubleshooting

- **Name already taken**: Microsoft Foundry resource name (and Key Vault/Storage/Cosmos DB/AI Search names, if using Standard Agent Setup) must be globally unique — pick a new suffix.
- **Role assignment failures**: `deployRoleAssignments` requires the deploying identity to have **Owner** or **User Access Administrator** on the resource group/subscription. Set it to `false` and assign roles manually if you lack that permission.
- **Soft-deleted Key Vault name conflict** *(Standard Agent Setup only)*: purge the soft-deleted vault (`az keyvault purge --name <name>`) or choose a new name.
- **`az deployment sub validate` shows unexpected changes**: confirm the same `deployRoleAssignments` value and existing resource properties (e.g., Key Vault purge protection) match what's already deployed.
- **`AADSTS700213: No matching federated identity record found for presented assertion subject 'repo:<owner>/<repo>:environment:<ENV_NAME>'`**: the federated credential's `subject` doesn't match the GitHub Environment actually being used for the run. Common causes:
  - The federated credential was created for a different environment name/casing than the one configured in `deploy-foundry.yml`'s `environment:` block for that job (GitHub Environment names are case-sensitive in the OIDC subject).
  - A new `workflow_dispatch` environment option (e.g. `DEV-STANDARD`) was added without either creating a matching GitHub Environment + federated credential for it, or mapping it to an existing environment's credential in the workflow (see how `DEV-STANDARD` reuses `DEV`'s credential in `deploy-foundry.yml`).
  - Update or add the federated credential subject to match exactly:
    ```powershell
    az ad app federated-credential create --id <appId> --parameters '{
      "name": "github-<env>",
      "issuer": "https://token.actions.githubusercontent.com",
      "subject": "repo:<owner>/<repo>:environment:<ENV_NAME>",
      "audiences": ["api://AzureADTokenExchange"]
    }'
    ```
    (use `az ad app federated-credential update` instead if a credential with that `name` already exists but has the wrong `subject`).
- **`AuthorizationFailed` on `Microsoft.Resources/deployments/validate/action` or `/write/action`**: the service principal's role assignment (Contributor/User Access Administrator) hasn't propagated yet, is scoped to the wrong resource group/subscription, or was never created. Re-run `az role assignment list --assignee <appId> --all` to confirm the scope and role, and allow a few minutes for propagation after creating a new assignment.
- **`FlagMustBeSetForRestore` (Cognitive Services/Foundry account)**: the account was soft-deleted by a prior failed/removed deployment and Azure is blocking recreation of the same name. List soft-deleted accounts with `az cognitiveservices account list-deleted`, then purge the specific one: `az cognitiveservices account purge --location <region> --resource-group <rg> --name <name>`.
- **Cosmos DB stuck in `Failed` provisioning state, retries fail with `BadRequest`**: a previous Cosmos DB creation attempt failed partway and left the account in a bad state that blocks recreation. Check with `az cosmosdb show --name <name> --resource-group <rg> --query provisioningState`, then delete it before retrying: `az cosmosdb delete --name <name> --resource-group <rg> --yes`.
- **Cosmos DB `ServiceUnavailable` — "currently experiencing high demand ... for the zonal redundant (Availability Zones) accounts"** *(Standard Agent Setup only)*: this is a genuine, non-transient Azure-side regional capacity shortage for new Cosmos DB accounts — it can appear even when `isZoneRedundant` is already `false` in the template (the message text is generic). Workarounds: retry later, or set the `cosmosDBLocation` parameter to a different region with available capacity (Cosmos DB connects to the Foundry project cross-region without issue — see `dev-standard.main.bicepparam` for an example using `eastus2`).
- **`DeploymentModelNotSupported` for a model deployment** (e.g. `The model 'Format:OpenAI,Name:<model>,Version:<version>' of account deployment is not supported`): the requested model/version isn't available for the Foundry account's region/SKU. List what's actually available before setting `foundryModelDeployments`:
  ```powershell
  az cognitiveservices account list-models --name <foundryName> --resource-group <rg> --query "[].{model:name, version:version, sku:skus[0].name}" -o table
  ```
- **Bicep warning `BCP037: The property "<x>" is not allowed on objects of type "<Y>"`**: usually means a property from a similar-but-different resource type was copy-pasted (e.g. an account-level capability host property applied to a project-level capability host, which has a different schema). Always check the actual resource type schema for the specific API version rather than assuming symmetry between similarly-named types.
- **GitHub Actions "Node.js 20 is deprecated" warning**: upgrade third-party actions to their latest major version (e.g. `actions/checkout@v7`, `azure/login@v3`), which declare `node24`. Some actions (e.g. `azure/arm-deploy@v2`) may not have a newer release yet — GitHub Actions runs them on Node 24 anyway via a compatibility shim, so the warning is cosmetic until a new release is published upstream.

## Next Steps

- Add private networking (disable public network access, add Private Endpoints) for production.
- For **Basic Agent Setup** environments, wire up Foundry Agents via the Azure AI Foundry portal/SDK (no additional Bicep needed — Microsoft manages the backing Cosmos DB/AI Search/Storage).
- For **Standard Agent Setup** environments, the Cosmos DB/AI Search/Storage connections and capability hosts are already provisioned by this template — create Agents via the portal/SDK and they'll use the deployed resources.
- Add a `main.json` (ARM) build artifact to the pipeline if you need to distribute a compiled template.
