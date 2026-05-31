# 16 — Infrastructure & Deployment

## Purpose

Make the dev environment work today and define what the real implementation needs in stg/prod.

## Current State

### Dev environment (provisioned, working)

- Resource group `rg-dops-dev` in `canadacentral`.
- ACR for app images.
- Container Apps Environment connected to Log Analytics + App Insights.
- Container Apps:
  - `ca-dops-api-dev` (FastAPI) — external ingress, port 8000.
  - `ca-dops-frontend-dev` (React/Nginx) — external ingress, port 80.
  - `ca-dops-worker-dev` — placeholder.
- PostgreSQL Flexible Server with `dops` database.
- Key Vault for secrets.
- Storage account with containers `documents`, `certificates`, `public-assets` — provisioned but unused.
- Service Bus namespace + queues `dops-events`, `dops-automation`, `dops-notifications`, `dops-deadletter` — provisioned but unused.
- App Configuration for non-secret feature flags.

### Pipelines

- `dops-infra` ([`pipelines/azure-pipelines-infra.yml`](../dops-infra/pipelines/azure-pipelines-infra.yml)): validate / what-if on PR; deploy + publish outputs on `main`.
- `dops-api` ([`pipelines/azure-pipelines-api.yml`](../dops-api/pipelines/azure-pipelines-api.yml)): build + push to ACR, update Container App, run Alembic migration job, seed (dev), verify health.
- `dops-frontend` ([`pipelines/azure-pipelines-frontend.yml`](../dops-frontend/pipelines/azure-pipelines-frontend.yml)) (verify): build with `VITE_API_BASE_URL` baked, push to ACR, update Container App, run verify script.

### Documentation already in `dops-infra/docs`

- [`ARCHITECTURE.md`](../dops-infra/docs/ARCHITECTURE.md)
- [`DEV_DEPLOYMENT.md`](../dops-infra/docs/DEV_DEPLOYMENT.md)
- [`END_TO_END_DEV_DEPLOYMENT_RUNBOOK.md`](../dops-infra/docs/END_TO_END_DEV_DEPLOYMENT_RUNBOOK.md)
- [`SECRETS_AND_CONFIG.md`](../dops-infra/docs/SECRETS_AND_CONFIG.md)
- [`APP_PIPELINE_INTEGRATION.md`](../dops-infra/docs/APP_PIPELINE_INTEGRATION.md)

## Gaps For Real Implementation

| # | Gap | Where |
| - | --- | ----- |
| 1 | API & Postgres on public networking | `dops-infra` |
| 2 | No private endpoints / VNet | `dops-infra` |
| 3 | No Front Door / WAF | `dops-infra` |
| 4 | No wildcard cert / custom domains | `dops-infra` |
| 5 | No identity provider (Azure AD B2C) | `dops-infra` |
| 6 | Worker Container App empty | `dops-automation-worker` |
| 7 | No PSP webhook ingress (Stripe) | `dops-infra` + `dops-api` |
| 8 | No virus-scan side-car for documents | `dops-automation-worker` or sidecar |
| 9 | No CDN for public site assets | `dops-infra` |
| 10 | No alerting rules on App Insights | `dops-infra` |
| 11 | No backup/restore policy on Postgres + Storage | `dops-infra` |
| 12 | No DR plan / region pair | `dops-infra` |

## Real Implementation Plan

### A. Environments

Introduce three:

- **dev** — current setup; auto-deployed from `develop` branches.
- **stg** — private networking, real IdP, deployment approvals.
- **prod** — paid SKUs, multi-zone, Front Door + WAF, region pair (canadacentral primary, canadaeast secondary).

### B. Networking

- VNet with subnets for Container Apps environment, PostgreSQL, private endpoints.
- Private endpoints for: PostgreSQL, Storage, Key Vault, App Configuration, Service Bus.
- Container Apps environment in internal-only mode for non-public services; public ingress goes through Front Door.

### C. Edge

- Azure Front Door Standard with WAF.
- Wildcard cert for `*.drivingops.ca`.
- Origin = frontend Container App (admin SPA) and public-site app.
- Path-based routing for `/api/*` to API container app.
- Rate-limiting policies per origin path.

### D. Identity provider

- New Azure AD B2C tenant `drivingopsidentity.onmicrosoft.com`.
- Custom user flows: sign-up/sign-in, password reset, MFA-enforced for admin roles.
- Tenant federation for Google + Microsoft (school owners).
- Secrets stored in Key Vault; consumed by API via managed identity.

### E. Worker Container App

- Build the worker repo skeleton (see [08-automations.md](./08-automations.md)).
- Container App with managed identity, Service Bus consumer scale rule, min replicas 1, max 10.
- App Insights instrumentation key shared.

### F. Stripe webhook ingress

- Front Door route `POST /webhooks/stripe` → API container.
- Stripe webhook secret in Key Vault; signature verification middleware in API.

### G. Document virus scan

- Add ClamAV side-car container in worker app OR an Azure Container Instance triggered by Service Bus.
- Updates `documents.status` from `scanning` to `pending_review` or rejected.

### H. Observability

- App Insights smart detection rules.
- Custom alerts: error rate > 1% over 5 min, p95 latency > 1.5 s, Service Bus dead-letter depth > 0, Postgres CPU > 80% sustained.
- Workbook for tenant-by-tenant request volume.

### I. Backups & DR

- Postgres: PITR retention 7d in dev, 35d in prod; geo-redundant backups in prod.
- Storage: soft-delete + versioning on `documents` and `certificates` containers; ZRS in dev, GZRS in prod.
- Quarterly DR rehearsal documented in `dops-infra/docs`.

### J. CI/CD hardening

- OIDC-based service connection to Azure (replace `sc-dops-dev` SP secret).
- Approval gates for stg + prod deploys.
- SBOM + Trivy scan in the image-build stage.
- Verify scripts gated by required-success status checks.

## Acceptance Criteria

- A single command (`scripts/deploy-stg.ps1`) provisions stg with private networking and approval gates.
- Front Door fronts both apps in prod; certificate auto-renewal verified.
- Stripe webhooks reach the API only via Front Door + WAF.
- Postgres restore drill succeeds within target RTO < 1h, RPO < 5 min.
- Alert rules fire during fault injection and resolve when remediated.

## Dependencies

All app features depend on the infrastructure being right. See per-feature docs for the exact dependency.
