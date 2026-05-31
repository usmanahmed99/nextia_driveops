# 01 — Cross-Repo Architecture

## Purpose

Describe how the four DrivingOps repos fit together, what each one owns, and what the real implementation must enforce at the seams.

## Current Topology (Dev)

```
[ User (web) ]
      │
      ▼
[ ca-dops-frontend-dev ]   Azure Container App, React SPA (Vite build), Nginx
      │
      │ HTTPS, Authorization: Bearer <jwt>, X-Tenant-ID: <id>
      ▼
[ ca-dops-api-dev ]        Azure Container App, FastAPI on Uvicorn
      │
      ├── Azure Database for PostgreSQL Flexible Server  (primary store)
      ├── Azure Key Vault           (secrets)
      ├── Azure Storage             (containers: documents, certificates, public-assets — not yet used)
      ├── Service Bus               (queues: dops-events, dops-automation, dops-notifications, dops-deadletter — not yet used)
      └── App Insights / Log Analytics  (telemetry)

[ Service Bus queues ]
      ▼
[ ca-dops-worker-dev ]     Azure Container App placeholder. Repo is empty.
```

The Bicep that produces this is in [`dops-infra/infra/main.bicep`](../dops-infra/infra/main.bicep) with environment parameters in [`dops-infra/infra/parameters/dev.bicepparam`](../dops-infra/infra/parameters/dev.bicepparam). The pipeline lives at [`dops-infra/pipelines/azure-pipelines-infra.yml`](../dops-infra/pipelines/azure-pipelines-infra.yml).

## Responsibilities

### `dops-frontend`

- All UI. React 18, TypeScript, Vite, Tailwind, Radix, TanStack Query, react-hook-form, Zod, react-i18next, recharts.
- Two data modes selected by `VITE_DATA_MODE`:
  - `mock` — in-memory adapter, no backend needed.
  - `api` — fetch adapter against `VITE_API_BASE_URL` (= `dops-api`).
- Per-feature contracts in `src/lib/contracts/*` declare the wire DTOs.
- DTO mapping (snake_case ↔ camelCase, money, dates, enums) lives in `src/lib/api/mappers/*`. **Mismatches must be fixed in mappers, not in pages**.
- RBAC is UX-only: `PermissionGate`, `ProtectedRoute`, `canAccessRoute`. **The backend is the source of truth**.

### `dops-api`

- All business logic, persistence, authorization, multi-tenant scoping.
- Stack: FastAPI, Pydantic v2, SQLAlchemy 2.x, Alembic, PostgreSQL (SQLite for local-only convenience).
- Layers: `endpoints → services → repositories → models`. Pydantic schemas live alongside endpoints in `schemas/`.
- Authorization: `core/rbac.py` issues `require_permission(...)` dependencies on each route.
- Tenant scoping: `core/tenant_context.py` reads `X-Tenant-ID` or falls back to session-active tenant. All tenant-scoped tables include `tenant_id`.
- Outbound side-effects (email/WhatsApp/PSP/blob) **must** flow through service interfaces, not directly from endpoints, so they can be swapped per environment.

### `dops-automation-worker`

- Empty today. Will own the Service Bus consumer that executes automation flows asynchronously (WhatsApp messages, email, scheduled reminders, escalations).
- Communicates with `dops-api` over HTTP for state updates and reads/writes its own audit rows.

### `dops-infra`

- Sole owner of Azure resources for dev/stg/prod environments.
- Produces a JSON output artifact (`dops-infra-dev-outputs`) consumed by `dops-api` and `dops-frontend` pipelines.
- Provides Container Apps Jobs (`api-migration-job`, `api-seed-job`) so app pipelines do not bake migrations into the runtime image.

## Cross-Cutting Concerns

### Authentication

Today: demo login returns a placeholder JWT and a session in `localStorage`. See [03-authentication.md](./03-authentication.md).

Real impl: replace `dops-api/app/core/security.py` + `services/auth_service.py` with a real OIDC integration (Azure AD B2C or equivalent). The frontend session shape (`Session` in `src/lib/contracts/auth.ts`) stays stable.

### Tenant Resolution

Real impl rules:

1. `X-Tenant-ID` header from the frontend is the source of intent.
2. The API must verify the authenticated user has a membership in that tenant. Reject 403 otherwise.
3. Every tenant-scoped repository call must include `tenant_id` in the `WHERE` clause. Add a SQLAlchemy event listener that fails fast on tenant-scoped queries with no `tenant_id` filter.

### Idempotency & Retries

All side-effect-producing endpoints (sending a WhatsApp message, recording a payment, generating a certificate, generating a schedule) must accept an idempotency key header (`Idempotency-Key`) and persist the (tenant, key) → result mapping with a 24h TTL.

### Observability

- Structured logs (JSON) with `tenant_id`, `user_id`, `request_id`, `route`, `latency_ms`, `outcome`.
- Application Insights traces from FastAPI middleware + frontend telemetry SDK.
- Service Bus messages must include `correlation_id` and `tenant_id` headers.

### Configuration

- Secrets in Key Vault, accessed by managed identity from each Container App.
- Non-secret feature flags in App Configuration.
- App-level configuration lives in `dops-api/app/core/settings.py`.

### CI/CD

- `dops-infra` deploys infra on merge to `main`, publishes outputs.
- `dops-api` builds image → pushes to ACR → triggers migration job → triggers seed job (dev only) → updates Container App → runs `scripts/verify-deployed-api.sh`.
- `dops-frontend` builds image with `VITE_API_BASE_URL` baked in → pushes to ACR → updates Container App → runs `scripts/verify-deployed-frontend.sh`.

## Gaps For Real Implementation

| Gap | Owner repo | Doc reference |
| --- | ---------- | ------------- |
| Real OIDC identity provider | api + frontend | [03-authentication.md](./03-authentication.md) |
| Backend tenant-scoping safety net (query interceptor) | api | [02-multitenancy-rbac.md](./02-multitenancy-rbac.md) |
| Async worker for automations | worker + infra | [08-automations.md](./08-automations.md) |
| Real WhatsApp + email transport | worker | [08-automations.md](./08-automations.md) |
| Real blob storage for documents and certificates | api | [10-certificates.md](./10-certificates.md), [11-documents.md](./11-documents.md) |
| Real payment processor (Stripe Canada or similar) | api | [09-invoices-payments.md](./09-invoices-payments.md) |
| Scheduling engine (real generator, conflict resolver) | api | [06-scheduling.md](./06-scheduling.md), [07-school-schedule-templates.md](./07-school-schedule-templates.md) |
| Public site rendering at the school subdomain | frontend + infra | [12-public-site.md](./12-public-site.md) |
| Idempotency + structured audit log enforcement | api | this doc |

## Acceptance Criteria For "Real Architecture"

- Frontend → API requests carry a verified JWT (no demo bypass paths in dev/stg/prod).
- Every tenant-scoped DB query is automatically filtered or rejected.
- Side-effect endpoints honour `Idempotency-Key`.
- Worker consumes Service Bus messages, retries with backoff, dead-letters after N attempts, and writes an audit row per attempt.
- All three Container Apps expose `/health` (or equivalent) and pipelines fail when verify scripts fail.
- App Insights captures end-to-end traces from frontend XHR through API to worker.
