# Generate and Validate Azure Infrastructure

PortfolioPilot is a stock-portfolio manager with live news, portfolio-aware AI chat (Claude Agent
SDK), research recommendations, watchlists and alerts. It is built step by step across 36 milestones
in this Udemy course. This folder is the state of the project **after milestone 34**.

Milestone 34 describes the whole Azure environment the app will run on **as code**, checks that code
without spending money, and prepares the application to sign in to Azure services without stored
passwords. **Nothing was deployed to Azure in this milestone.**

## Contents

- [Key terms](#key-terms)
- [1. Purpose](#1-purpose)
- [2. Steps performed](#2-steps-performed)
- [3. Results achieved](#3-results-achieved)
- [4. How to run and verify](#4-how-to-run-and-verify)
- [5. Limitations and unfinished work](#5-limitations-and-unfinished-work)
- [Where to read more](#where-to-read-more)

## Key terms

| Term | Meaning |
| --- | --- |
| **Infrastructure as code (IaC)** | Describing cloud resources in files you can review, version and re-run, instead of clicking in the portal. |
| **Bicep** | Microsoft's IaC language for Azure. A `.bicep` file compiles to an ARM template (JSON) that Azure Resource Manager deploys. |
| **ARM / API version** | Azure Resource Manager is the Azure control plane. Every resource type has dated API versions (for example `2025-08-01`) that fix which properties exist. |
| **`what-if`** | A read-only Azure command that shows what a deployment *would* create or change, without doing it. |
| **AKS** | Azure Kubernetes Service, which runs the app's containers. |
| **ACR** | Azure Container Registry, which stores the container images built in milestone 33. |
| **Azure Managed Redis** | Microsoft's managed Redis service (resource type `Microsoft.Cache/redisEnterprise`). |
| **Microsoft Entra ID** | Azure's identity service (formerly Azure AD). It issues **access tokens** that prove who a caller is. |
| **Managed identity / workload identity** | An Azure identity with no password. *Workload identity* lets a Kubernetes pod exchange its own service-account token for an Entra token, so the app needs no stored secret to reach Azure services. |
| **Private endpoint** | A private IP address inside your virtual network for an Azure service, so traffic never uses the public internet. |
| **Private DNS zone** | Makes the service's normal host name resolve to that private IP inside the network. |
| **NAT gateway** | Gives all outbound traffic from the cluster one fixed public IP address. |
| **Least privilege** | Every identity gets only the permissions it needs, on the smallest possible scope. |
| **SCRAM verifier** | The salted hash PostgreSQL stores instead of a password. Sending the verifier means the plain password never reaches the server. |

## 1. Purpose

### What the prompt asked for

The prompt for this milestone asked the coding agent to:

- Write **parameterized Bicep** for AKS, ACR, PostgreSQL Flexible Server, Azure Managed Redis, Key
  Vault, Blob Storage (for Claude SDK session files) and monitoring.
- **Verify every Azure fact** (resource API versions, region and size availability, authentication
  options and prices) against official sources, and never invent resource schemas.
- Use **workload identity** and **least-privilege role assignments**.
- Keep database, cache and storage traffic **private**, with a documented network design that
  includes DNS and the outbound internet access the app needs.
- Define how **Prisma** (the database library) and the **Redis client** get and refresh their
  credentials, without claiming identity support that has not been tested.
- Provide small **development parameters**, tags, naming rules, backup settings and a
  **cost-estimate worksheet**, with **no secrets** in parameter files.
- Explain the **manual choices** a person must make: subscription, region, resource group, public
  DNS name, the sign-in (OIDC) app, provider secrets and budget.
- **Compile and statically validate** the Bicep. Run Azure validation or `what-if` only if already
  signed in, and **create nothing**.
- Write exact **provisioning and cleanup commands**, including warnings about data loss, and
  update the project state.

### Why it matters

Clicking resources together in the Azure portal is hard to review, repeat or tear down. Bicep makes
the environment reviewable, and validating it before deploying catches mistakes for free. The most
common cloud security problems are leaked passwords and over-broad permissions, so this milestone
replaces stored database and cache passwords with short-lived Entra tokens where that path can be
tested. It also keeps every data service off the public internet.

### What you will learn

- How to structure Bicep into modules with parameters, outputs and naming rules.
- How to **check** which API versions and properties really exist instead of guessing.
- How private endpoints, private DNS and a NAT gateway fit together.
- How token-based sign-in works for PostgreSQL and Redis, and why the two refresh differently.
- How to test credential rotation against real servers.
- How to estimate cloud cost from official price meters.
- How to write a safe provisioning and cleanup runbook.

## 2. Steps performed

These are the steps the coding agent actually took, in order.

### Step 1 — Read the project contract and current state

The agent read `AGENTS.md` (project rules) and `docs/project-state.md`. Milestone 33 had produced
container images and a release workflow; nothing was deployed. Dependencies were restored:

```bash
npm ci --ignore-scripts --prefer-offline
```

### Step 2 — Check the Azure tools and sign-in

```bash
az version          # Azure CLI 2.80.0
az bicep version    # Bicep CLI 0.41.2
az account show
```

The CLI showed an account, but every Azure Resource Manager call failed with **AADSTS700082**: the
sign-in token had expired after 90 days of inactivity. The prompt allows Azure validation only if
*already* signed in, so the agent did not sign in on the user's behalf. Azure validation was marked
as skipped and the work continued with official documentation.

### Step 3 — Verify facts from official sources

- **Microsoft Learn** confirmed:
  - Azure Managed Redis Entra sign-in: the user name is the identity's *object ID*, and the client
    must send `AUTH` with a fresh token before the old one expires.
  - The PostgreSQL Entra token flow.
  - Private DNS zone names.
  - Built-in role IDs, the role definitions that grant permissions. One remembered ID turned out to
    belong to a different role.
  - PostgreSQL connection limits: the smallest size, B1ms, allows only 35 connections, so B2s was
    chosen.
- The **Bicep compiler itself** was probed to see which API versions it knows. Newer AKS versions
  exist, but this Bicep version cannot type-check them, so the newest version it *can* check
  (`2025-10-01`) was pinned.
- The public **Azure Retail Prices API** supplied the unit prices for the cost worksheet.
- The **installed packages** (`node_modules`) were read to see how `pg` and `redis` handle
  credentials.

### Step 4 — Add token-based sign-in to the app

New file `packages/db/src/azure-credentials.ts`, wired into the existing database and Redis client
factory in `packages/db/src/index.ts`:

- `DATABASE_AUTH=azure-workload-identity`: Prisma's PostgreSQL pool asks for a **fresh Entra token
  for every new connection**. PostgreSQL checks the token only when a connection signs in.
- `REDIS_AUTH=azure-workload-identity`: the Redis client gets a new token **five minutes (plus
  random jitter) before expiry** and re-authenticates its live connection. Azure Managed Redis
  disconnects clients whose token expires.
- The default (`url`) is unchanged, so local development and Docker Compose work exactly as before.
- New settings are validated in `packages/config/src/server.ts`.

A trap found here: the `pg` library replaces a password *function* with an empty string when a
connection string is also given. The fix passes host, port, user and database as separate fields.

### Step 5 — Fix a least-privilege problem in the configuration

In production, every worker process had to receive the user sign-in secrets (`AUTH_SECRET` and the
Entra client secret), even though workers never sign users in. Now, when `WORKER_ROLE` is set,
production requires only `DATABASE_URL` plus a shared telemetry key.

### Step 6 — Test the credential paths

- Unit tests: `packages/db/test/azure-credentials.test.ts`.
- An integration test against real PostgreSQL 17.6 and Redis 7.4.5:
  `packages/db/test/credential-rotation.integration.test.ts`. It rotates the password underneath
  live clients and checks that:
  - new PostgreSQL connections use the new credential, and a stale one is rejected;
  - Redis re-sends `AUTH` on the *same* connection, and reconnects with the newest credential after
    the old one is revoked.

### Step 7 — Write the Bicep

```text
infra/azure/
├── main.bicep                 # subscription scope: creates the resource group, calls workload.bicep
├── bicepconfig.json           # strict linter rules (secrets, unused params, API versions…)
├── parameters/dev.bicepparam  # small dev sizes, tags, REPLACE_ placeholders, no secrets
├── modules/
│   ├── workload.bicep         # wires every module together; identities and role assignments
│   ├── network.bicep          # VNet, subnets, NSG, NAT gateway + static IP, private DNS zones
│   ├── private-endpoint.bicep # one private endpoint + DNS record
│   ├── monitoring.bicep       # Log Analytics, Application Insights, Container insights rule
│   ├── aks.bicep              # cluster, kubelet AcrPull, audit logs
│   ├── registry.bicep         # ACR
│   ├── postgres.bicep         # PostgreSQL 17 + Entra admin + private endpoint
│   ├── redis.bicep            # Azure Managed Redis + Entra users + private endpoint
│   ├── keyvault.bicep         # Key Vault (RBAC, firewall) + private endpoint
│   ├── storage.bicep          # Blob container for session artifacts + lifecycle rule
│   ├── federation.bicep       # federated credentials (Kubernetes + GitHub)
│   └── budget.bicep           # monthly cost alert
├── sql/                       # PostgreSQL role bootstrap (3 steps)
└── scripts/scram-verifier.mjs # hashes a password client-side for PostgreSQL
```

Key decisions:

- **One managed identity per workload** (API, ingestion, outbox, agent, migration, telemetry),
  each with only its own permissions.
- **Data services have public access disabled**; they are reached through private endpoints.
- **Redis uses the `EnterpriseCluster` policy**, which works with the app's existing non-cluster
  Redis client. The agent checked that every Redis command the app sends touches a single key.
- **The template creates no secret values.** Its only secret input, the PostgreSQL admin password,
  is read from an environment variable at deployment time and never stored.

### Step 8 — Design and test the database roles

PostgreSQL gets three kinds of roles:

- a **migration role** that owns the database;
- **runtime roles** (one per app identity) that can only read and write data;
- a never-stored **local admin** used once for bootstrap.

`scripts/verify-postgres-roles.mjs` checks this model on a throwaway PostgreSQL container, using a
non-superuser administrator just like Azure's.

### Step 9 — Build the validator

`scripts/validate-infra.mjs` runs as `npm run validate:infra`. It:

- compiles and lints the Bicep;
- builds the parameter file;
- checks name rules;
- scans for secrets;
- enforces policy rules (no preview API versions, only verified role IDs, private data services,
  no secret outputs).

`--self-test` plants 10 problems and confirms each one is detected. `--azure` adds the read-only
Azure checks.

### Step 10 — Write the documentation

- `docs/azure/network.md` — network design, DNS, outbound destinations.
- `docs/azure/credentials.md` — identities, roles, credential refresh and rotation, tested vs untested.
- `docs/azure/cost-estimate.md` — cost worksheet with assumptions.
- `docs/azure/manual-choices.md` — every decision a person must make.
- `docs/azure/provisioning.md` — exact provisioning and cleanup commands with data-retention warnings.
- `docs/decisions/0026-azure-infrastructure-and-credentials.md` — the architecture decision record.
- `docs/lessons/34-azure-infrastructure.md` — teaching notes.
- `docs/project-state.md`, `docs/verification-report.md` and `docs/versions.md` were updated.

### Step 11 — Run all regression tests

Typecheck, lint, unit, integration, browser and container tests were re-run (results below).

### Step 12 — Follow-up after the operator signed in (next day)

Once the Azure CLI was signed in again, the read-only preflight checks from provisioning step 2 were
run:

- All required resource providers were registered.
- PostgreSQL `Standard_B2s` and Azure Managed Redis were offered in `eastus2`.
- The **Dsv5** VM family had **no vCPU quota** in that subscription, so a deployment with
  `Standard_D2s_v5` nodes would have failed. The dev node size was changed to `Standard_D2s_v6`,
  which had quota, and the cost worksheet was updated.

`what-if` was **not** run, because the manual choices (above all an Entra operator group) are still
placeholders.

### Files at a glance

| Created | Modified |
| --- | --- |
| `infra/azure/**` (Bicep, parameters, SQL, scram helper) | `packages/db/src/index.ts` |
| `packages/db/src/azure-credentials.ts` | `packages/config/src/server.ts`, `packages/config/test/config.test.ts` |
| `packages/db/test/azure-credentials.test.ts`, `credential-rotation.integration.test.ts` | `scripts/test-integration.mjs` (new suite) |
| `scripts/validate-infra.mjs`, `scripts/verify-postgres-roles.mjs` | `package.json` (two scripts) |
| `docs/azure/*.md`, ADR 0026, lesson 34, this README | `docs/project-state.md`, `docs/verification-report.md`, `docs/versions.md`, `docs/decisions/README.md` |

No npm dependency was added or changed.

## 3. Results achieved

### The Azure environment described by the Bicep (not deployed)

| Service | Dev setting | Security |
| --- | --- | --- |
| AKS | 2–3 × `Standard_D2s_v6` Azure Linux nodes, Free tier | Entra sign-in only (no local admin accounts), workload identity, outbound through NAT |
| Container Registry | Basic | No admin user; nodes pull with the AcrPull role |
| PostgreSQL Flexible Server 17 | Burstable B2s, 32 GB, 7-day backups | Private endpoint only; Entra sign-in for the app |
| Azure Managed Redis | Balanced B0, no high availability | Private endpoint only, TLS, access keys disabled, Entra users |
| Key Vault | Standard, 7-day soft delete | RBAC; firewall allows only operator IPs; private endpoint |
| Blob Storage | Hot LRS, container `session-artifacts` | No shared keys, no public access, private endpoint |
| Monitoring | Log Analytics (30 days, 1 GB/day cap), Application Insights | Audit logs from AKS, PostgreSQL, Key Vault and Blob |

### How the application behaves now

- **Locally, nothing changes.** `DATABASE_AUTH` and `REDIS_AUTH` default to `url`.
- **In Azure (milestone 35 will configure it):**
  - Each pod signs in to PostgreSQL and Redis with its own short-lived Entra token.
  - New database connections fetch a current token.
  - Live Redis connections re-authenticate about 5–6 minutes before their token expires.
- **Production workers** start without the user sign-in secrets.

### Example output (observed)

`npm run validate:infra -- --self-test --azure`, with the expired sign-in:

```text
PASS bicep build main.bicep (zero diagnostics)
PASS bicep lint (zero diagnostics)
PASS bicep build-params infra\azure\parameters\dev.bicepparam
     52 resource declarations, 30 type/API pairs
PASS template policy (APIs, roles, private data services, no secret outputs)
PASS naming constraints for dev parameters
PASS every required parameter supplied
PASS no secrets in infra files, docs or built parameters
     manual choices still open in infra\azure\parameters\dev.bicepparam: REPLACE_owner_alias, ...
PASS self-test detects preview API
...
SKIP Azure validate/what-if — skipped: Azure CLI is not signed in with a usable token (az login)

validate:infra 17 passed, 0 failed, 1 skipped
```

`npm run verify:postgres-roles`:

```text
PASS step 2 refuses a plaintext password
PASS step 2 (roles + database) as non-superuser admin
PASS Prisma migrate deploy as pp_migrator (SCRAM verifier login)
PASS runtime role seeds and reads application data (DML + sequences) — 2 users
PASS runtime role cannot create tables — ... permission denied for schema public
PASS runtime role cannot read the migration ledger — ... permission denied for table _prisma_migrations
PASS old migrator password rejected after rotation
...
verify:postgres-roles 14/14 passed
```

### All check results (observed 2026-10-03, Windows 11, Docker Desktop)

| Command | Result |
| --- | --- |
| `npm run typecheck`, `npm run lint`, `npm run check:browser-boundary` | passed |
| `npm run test:unit` | **369 passed**, 0 failed, 4 skipped (gated live tests) |
| `npm run test:integration` | **129 passed** across 17 suites (includes the new credential-rotation suite, 2/2) |
| `npm run test:browser` | **30 passed** |
| `npm run verify:containers` | **8/8 passed** (images rebuilt with the changed code) |
| `npm run validate:infra -- --self-test --azure` | **17 passed**, 1 skipped (Azure validation) |
| `npm run verify:postgres-roles` | **14/14** |

### Estimated cost (from the worksheet, not a quote)

About **USD 334 per month** (≈ USD 11 per day) for the dev sizes in `eastus2` at retail
pay-as-you-go prices from 2026-10-03/04. The upper bound is about USD 472 if the cluster scales to
three nodes and the log cap is reached. Claude API usage is billed separately by Anthropic. See
`docs/azure/cost-estimate.md`.

## 4. How to run and verify

### Prerequisites

- Node.js **24.21.0** and npm **11.19.0**.
- **Docker Desktop**, for the database and Redis tests.
- **Azure CLI** 2.80 or later with Bicep (`az bicep install`). These checks need **no Azure
  account**.
- Install dependencies once:

```bash
npm ci
```

### A. Validate the infrastructure code (free, no Azure account)

```bash
npm run validate:infra -- --self-test
```

**Expected:** every line `PASS`, plus a list of the `REPLACE_` choices still open. The compiled ARM
template is written to the temporary folder printed at the end.

### B. Prove the database role model (Docker)

```bash
npm run verify:postgres-roles
```

**Expected:** `verify:postgres-roles 14/14 passed`. The throwaway container is removed afterwards.

### C. Prove credential rotation against real PostgreSQL and Redis (Docker)

```bash
npm run test:integration -- credential-rotation
```

**Expected:** `packages/db/test/credential-rotation.integration.test.ts  2  0  0`.

### D. Run the regular test suites

```bash
npm run test:unit
npm run test:integration
npm run test:browser      # needs Google Chrome
```

### E. Optional: read-only check against your own Azure subscription

This creates nothing. You need your own subscription and permission to deploy (Owner, or
Contributor plus User Access Administrator).

1. Sign in and run the preflight checks in `docs/azure/provisioning.md` (steps 1–2). Check that
   your subscription has vCPU quota for the node size's VM family; change `aksNodeVmSize` if it
   doesn't.
2. Replace every `REPLACE_` value in `infra/azure/parameters/dev.bicepparam`
   (see `docs/azure/manual-choices.md`).
3. Run:

```bash
az login --tenant <YOUR_TENANT_ID>
az account set --subscription <YOUR_SUBSCRIPTION_ID>
export PP_PG_ADMIN_PASSWORD="$(openssl rand -base64 48 | tr -d '\n/+=' | cut -c1-40)"
npm run validate:infra -- --azure
```

**Expected:** `az deployment sub validate` and `what-if` lines `PASS`, with the what-if output
saved to a file.

> **Deploying for real** (`docs/azure/provisioning.md` step 5) creates resources that cost roughly
> USD 11 per day. Read the cleanup section, including its data-loss warnings, **before** you
> deploy, and delete the resource group when you are done.

## 5. Limitations and unfinished work

- **Azure validation and `what-if` were not run.**
  - The first attempt was skipped because the Azure CLI sign-in had expired.
  - After the operator signed in, only the read-only preflight ran. `what-if` still needs the
    manual choices, above all an Entra operator group, which does not exist yet.
- **Nothing was deployed.** As a result, these have **not been tested against real Azure**:
  - Azure accepting the Entra tokens: PostgreSQL Entra roles, Redis Entra users, Key Vault access
    through the CSI driver, and Blob access.
  - The bootstrap step that creates PostgreSQL Entra roles (`sql/01-entra-principals.sql`).

  The code paths *are* tested against real PostgreSQL and Redis with rotating passwords. The docs
  describe a password/key fallback (`url` mode) until the Azure smoke tests pass.
- **Some settings can only fail at deployment time**, and static checks cannot catch them:
  diagnostic log category names, whether the AKS identity needs network permissions beyond its
  subnet (the runbook includes the fix), and the Container insights data-collection settings.
- **Cost uncertainties:**
  - The price list shows a new AKS Free-tier charge (USD 0.05/hour from 2026-10-01) that the pricing
    page does not mention; it could add USD 36.50/month.
  - The price effect of Redis high availability is not shown.
  - The private-endpoint price for `eastus2` was inferred from other regions.
- **Two permission steps are commands, not Bicep:** per-secret Key Vault access and the GitHub
  deploy identity's namespace access. They must run after the secrets and namespace exist.
- **The container registry stays reachable from the internet** (sign-in still required). Making it
  private needs the Premium tier.
- **Left for milestone 35:**
  - Kubernetes service accounts, secret mounts, ConfigMaps with the Entra-mode settings, the
    migration Job, network policies, and the public HTTPS ingress with its DNS name.
  - The telemetry collector's Entra export path is unverified, so Application Insights still accepts
    its connection string.
- **Bicep 0.41.2 limits API versions.** Upgrading Bicep would allow newer API versions such as AKS
  `2026-06-01`; that is a deliberate future change.
- **Older lessons contain local Windows paths** (for the Node.js and Prisma install folders) that
  should be made generic before the repository is published.

## Where to read more

- `docs/azure/` — network design, credentials, cost worksheet, manual choices, provisioning and
  cleanup.
- `docs/decisions/0026-azure-infrastructure-and-credentials.md` — why things are built this way.
- `docs/lessons/34-azure-infrastructure.md` — teaching notes and common mistakes.
- `docs/project-state.md` — milestone status and full check results.
