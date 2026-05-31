# 20 — First Production Slice: Status & Next Steps

This document tracks the "First Production Slice" implementation against the plan you authored. It supersedes the planning text in [17-build-roadmap.md](./17-build-roadmap.md) for *that slice only*.

## Slice Scope

Phase 0 + the learner portion of Phase 1:

1. Azure AD B2C JWT-verifying auth + login-config endpoint.
2. Tenant + membership + invitation + plan + subscription + onboarding-task tables and endpoints.
3. Audit log, domain-event outbox, idempotency store.
4. Normalised learner module: paged list with filters, workspace endpoint, holds, timeline, withdraw, lifecycle state machine.

Out of scope here (deferred): real WhatsApp/email, Stripe payments, scheduling engine, school-schedule templates, public-site rendering, portals re-wiring, document storage, certificate PDFs.

## What Is Already In Place (verified in code)

### Backend — landed

- [`dops-api/alembic/versions/0002_saas_foundation_and_learners.py`](../dops-api/alembic/versions/0002_saas_foundation_and_learners.py): users B2C columns, tenant_memberships status/branch_ids/invited_at/accepted_at, invitations table, plans, tenant_subscriptions, tenant_onboarding_tasks, audit_logs before/after/request/ip/user_agent + index, domain_events, idempotency_keys, learner normalised columns + indexes, learner_holds, learner_timeline_events.
- [`dops-api/app/models/__init__.py`](../dops-api/app/models/__init__.py): all new SQLA models exported.
- [`dops-api/app/core/security.py`](../dops-api/app/core/security.py): `decode_b2c_token` with JWKS lookup, audience + issuer verification.
- [`dops-api/app/core/config.py`](../dops-api/app/core/config.py): `azure_b2c_*`, `frontend_auth_redirect_uri`, `frontend_auth_logout_redirect_uri` settings.
- [`dops-api/app/api/deps.py`](../dops-api/app/api/deps.py): bearer → demo token OR B2C token; falls back to demo only when valid.
- [`dops-api/app/services/auth_service.py`](../dops-api/app/services/auth_service.py): `session_from_claims` upserts users by `(issuer, subject)` then `email`, sets `last_login_at`; `demo_login` gated to `local|dev|test`.
- [`dops-api/app/api/v1/endpoints/auth.py`](../dops-api/app/api/v1/endpoints/auth.py): `GET /auth/login-config` + `/demo-login` env gate + existing `/session`, `/logout`, `/refresh`.
- [`dops-api/app/core/tenant_context.py`](../dops-api/app/core/tenant_context.py): membership + tenant-status checks, `tenant_scope` and `bypass_tenant_scope` context managers, `BLOCKED_TENANT_STATUSES`.
- [`dops-api/app/services/operations.py`](../dops-api/app/services/operations.py): `AuditLogger`, `EventOutbox`, `IdempotencyStore`.
- [`dops-api/app/services/tenant_service.py`](../dops-api/app/services/tenant_service.py): tenant CRUD, branches, memberships with invitation token + email lookup, invitations accept, subscription CRUD; `create_tenant` and `update_status` audit + emit.
- [`dops-api/app/api/v1/endpoints/onboarding.py`](../dops-api/app/api/v1/endpoints/onboarding.py): `register-tenant`, `accept-invite`.
- [`dops-api/app/api/v1/endpoints/platform.py`](../dops-api/app/api/v1/endpoints/platform.py): tenants list/create/status + subscription get/update.
- [`dops-api/app/api/v1/endpoints/tenants.py`](../dops-api/app/api/v1/endpoints/tenants.py): tenant profile, branches CRUD, memberships CRUD.

### Frontend — landed

- [`dops-frontend/src/lib/contracts/auth.ts`](../dops-frontend/src/lib/contracts/auth.ts): `AuthProviderType` includes `azure_ad_b2c`. (Wiring is *not* in place yet.)

## What Is Still Open

### Backend gaps

1. **LearnerService still uses old JSON `profile`/`enrollment`/`progress` fields**, ignoring the normalised columns added by migration 0002. New writes never populate `first_name`, `last_name`, `email`, `phone`, etc. New reads can't filter by `branch_id`, `program_id`, `preferred_language`, or `current_phase`.
2. **No paged list endpoint**. `GET /learners` returns a plain array; no cursor, no filters beyond `status` and `search`.
3. **No `GET /learners/{id}/workspace`**. Frontend orchestrates many requests today.
4. **No `POST /learners/{id}/holds`** / **resolve** / **list**.
5. **No `POST /learners/{id}/withdraw`**.
6. **Learner mutations write nothing to audit_logs, learner_timeline_events, or domain_events.**
7. **No idempotency middleware/decorator** wired to learner POSTs (or anywhere).
8. **Lifecycle state machine not enforced** — `UpdateLearnerRequest.status` accepts any enum value, with no allowed-transition table.
9. **Seed leaves the new normalised columns NULL** for the 8 demo learners.
10. **Tests cover only the old happy paths**.

### Frontend gaps

1. **No MSAL / `@azure/msal-browser` integration**. Login page only does demo-role login.
2. **No call to `GET /auth/login-config`** to drive the IdP flow.
3. **Demo role buttons not gated by `VITE_ENVIRONMENT`**.
4. **No platform-admin Tenants console** (list, detail, status, subscription).
5. **Learner list still client-side filters** (`useMemo` filter on the array). Need server filters + cursor.
6. **Learner detail still composes ~6 calls in `getLearnerWorkspace`** — switch to single `/learners/{id}/workspace`.
7. **No registration wizard** wired to `POST /learners` with the new validation.
8. **`AuthContext` has no MFA / silent-refresh handling.**
9. **`X-Tenant-ID` switching UI** absent for platform-admin.

### Infra gaps

1. **Azure AD B2C tenant not provisioned**.
2. **Key Vault entries for `DOPS_AZURE_B2C_*` missing** (and frontend env equivalents).
3. **No CORS list update for the B2C redirect URI(s)**.

## Plan For The Next Iteration

Order is dependency-driven; each step is shippable.

### Step 1 — Backend learner module on the normalised model (this iteration)

Updates limited to `dops-api`:

- New schemas: `LearnerStatus` lifecycle enum, `LearnerHoldType/Status`, `LearnerHold`, `LearnerWorkspace`, `PaginatedLearners`, `CreateLearnerRequest` validators, `WithdrawLearnerRequest`, `CreateLearnerHoldRequest`, `ResolveLearnerHoldRequest`.
- Refactor `LearnerService` to read/write the normalised columns. Old JSON fields are kept temporarily for backfill but become read-only.
- Add `LearnerService` methods: `list_learners(filters, cursor, page_size)`, `workspace`, `create_hold`, `resolve_hold`, `withdraw`, plus internal helpers `_emit_timeline`, `_emit_event`, `_audit`.
- Add lifecycle transition table and enforce on `update_learner` and `withdraw`.
- New router endpoints under `/learners`:
  - `GET /learners?cursor=&page_size=&status=&branch_id=&program_id=&instructor_id=&language=&behind_schedule=&certificate_ready=&payment_hold=&document_hold=&enrolled_from=&enrolled_to=&search=`
  - `GET /learners/{id}/workspace`
  - `POST /learners/{id}/holds`
  - `PATCH /learners/{id}/holds/{hold_id}/resolve`
  - `GET /learners/{id}/holds`
  - `POST /learners/{id}/withdraw`
- `Idempotency-Key` header honoured for the create + hold + withdraw routes via a service-layer helper.
- Audit row + timeline event + domain event written for every mutation.
- Update seed to populate normalised columns + a payment hold for "Emma Roy" and a documents hold for "Olivia Nguyen" so the new endpoints have realistic data.
- Tests in `app/tests/test_api.py` (or new files) covering: paged list with filters, workspace shape, hold lifecycle, withdraw, lifecycle transitions rejected, idempotency replay, demo-login disabled when `environment=production` (monkeypatched), B2C JWT verification accepts a token signed with a test key.

### Step 2 — Frontend learner module against the new endpoints

- `src/lib/contracts/learners.ts`: add `LearnerStatus` lifecycle, `LearnerHold`, `LearnerWorkspace`, `PaginatedLearners`.
- `src/lib/api/mappers/learners.ts`: switch to top-level fields from API.
- `src/lib/api/adapters/httpAdapter.ts`: replace multi-call workspace with `/learners/{id}/workspace`; `listLearners` accepts cursor + filter params and returns `{ items, nextCursor, pageSize }`.
- `src/features/learners/LearnersPage.tsx`: server filter chips + cursor "load more". Remove client filter `useMemo`.
- `src/features/learners/LearnerDetailPage.tsx`: render from workspace response, expose hold/withdraw actions.
- New `src/features/learners/RegisterLearnerWizard.tsx` six-step wizard.
- Add `<TenantStatusBoundary>` in `AppShell` that catches `423 tenant_suspended` and shows the lockout state.

### Step 3 — Frontend B2C plumbing

- Add `@azure/msal-browser` + `@azure/msal-react`.
- Read `GET /auth/login-config` on app load; if `provider === "azure_b2c"`, configure MSAL with returned authority/clientId/audience.
- `LoginPage` becomes a redirect-only screen except when `loginConfig.demoEnabled === true` (then the existing buttons are kept and the demo flow runs).
- Refresh handling: silent token renewal via MSAL; on 401 the http layer triggers `acquireTokenRedirect`.
- Logout redirects to `frontend_auth_logout_redirect_uri`.

### Step 4 — Frontend platform-admin Tenants console

- New `src/features/platform/PlatformTenantsPage.tsx` (lists `/platform/tenants`).
- Detail drawer with status changer and subscription editor.
- `X-Tenant-ID` switch shipped here for cross-tenant peeking (when `permissions` include `platform.tenants.read`).

### Step 5 — Infra & secrets

- New Bicep parameters for B2C authority/client-id/audience/jwks-url.
- Key Vault entries: `DOPS-AZURE-B2C-CLIENT-ID`, `DOPS-AZURE-B2C-AUDIENCE`, `DOPS-AZURE-B2C-AUTHORITY`, `DOPS-AZURE-B2C-JWKS-URL`.
- API Container App env wiring + frontend build env wiring.
- CORS allow-list update if B2C uses a separate sign-in subdomain.

## Implementation Status Snapshot — Step 1 (Backend Learner Module)

> Updated this iteration.

### Landed

- [`dops-api/app/schemas/learners.py`](../dops-api/app/schemas/learners.py): `LearnerStatus` lifecycle enum, `LIFECYCLE_TRANSITIONS` table, `LearnerHoldType`/`LearnerHoldStatus`, `LearnerProfile` with email + language regex validators, `LearnerHold`, `LearnerWorkspace*` shapes, `PaginatedLearners`, `LearnerListFilters`, `CreateLearnerRequest`, `UpdateLearnerRequest` (rejects withdrawn/transferred status as a direct assignment), `CreateLearnerHoldRequest`, `ResolveLearnerHoldRequest`, `WithdrawLearnerRequest`.
- [`dops-api/app/services/learner_service.py`](../dops-api/app/services/learner_service.py): full rewrite against the normalised columns. Reads & writes `first_name`, `last_name`, `email`, `phone`, `preferred_language`, `branch_id`, `program_id`, `program_name`, `enrolled_at`, `expected_completion_at`, `current_phase`, `overall/theory/practical_percent`, `completed_*`, `required_practical_lessons`, `retain_data_until`, `normalized_search`. Provides:
  - `list_learners(filters)` with status / branch / program / instructor / language / behind / cert-ready / payment-hold / document-hold / enrolled-range / search and cursor pagination (`enrolled_at desc, id asc`).
  - `get_learner` returns `LearnerDetail` with timeline events and active holds joined.
  - `workspace` composes upcoming sessions, outstanding invoices, certificates, and a derived recommendations list.
  - `create_learner`, `update_learner` (with lifecycle transition guard), `withdraw_learner`, `delete_learner`, `list_holds`, `create_hold`, `resolve_hold`. Every mutation writes `audit_logs`, an entry in `learner_timeline_events`, and a `domain_events` outbox row.
- [`dops-api/app/api/v1/endpoints/learners.py`](../dops-api/app/api/v1/endpoints/learners.py): new endpoints + camelCase query aliases + `Idempotency-Key` honoured for `POST /learners`, `POST /learners/{id}/withdraw`, `POST /learners/{id}/holds`.
- [`dops-api/app/services/operations.py`](../dops-api/app/services/operations.py): `IdempotencyStore.replay` returns stored response or raises `IdempotencyConflictError` on payload-hash mismatch; `save` is no-op when key absent.
- [`dops-api/app/db/seed.py`](../dops-api/app/db/seed.py): seeds learners against normalised columns, adds `LearnerTimelineEvent` rows, seeds payment hold on Emma and documents hold on Olivia, and seeds the `starter` plan + tenant subscription + completed onboarding tasks for the NorthStar tenant so the new platform/onboarding endpoints have data.
- [`dops-api/app/services/mappers.py`](../dops-api/app/services/mappers.py): `learner_to_schema` now delegates to the service helper for normalised reads; legacy detail mapper preserved for any non-service caller.
- [`dops-api/app/tests/test_api.py`](../dops-api/app/tests/test_api.py): 17 tests, including paged list, cursor pagination, branch + search filter, workspace endpoint shape, hold lifecycle, lifecycle transition guard (422), withdraw end-to-end, idempotent create + replay header + 409 on conflict, `login-config`, demo-login disabled outside `local|dev|test`, tenant block returns `423 tenant_suspended`, and platform tenants listing.

### Verification

- `python -m ruff check .` — clean.
- `python -m pytest -q` — `17 passed`.

### Behaviour notes

- `GET /api/v1/learners` response is now `{ items, nextCursor, pageSize }` — **breaking change for any consumer still expecting a plain array**.
- Lifecycle transitions enforced by `LIFECYCLE_TRANSITIONS`. Direct assignment of `withdrawn` / `transferred` rejected at schema level — use `POST /learners/{id}/withdraw`.
- Concept statuses (`active`, `behind_schedule`, `missing_documents`, `payment_due`, `waiting_for_practical`) are still accepted on read and seed for backfill compatibility, and are silently mapped onto the lifecycle when shown to clients.
- New seed places Emma Roy on a `payment` hold and Olivia Nguyen on a `documents` hold so the new endpoints have realistic data without manual setup.

## Hand-off Checklist For Each Step

Before merging a step:

- `python -m ruff check .` clean in `dops-api`.
- `python -m pytest -q` green in `dops-api`.
- `npm run lint` + `npm run typecheck` + `npm run build` clean in `dops-frontend`.
- API health on the dev Container App reports `database: ok`.
- Manual smoke: a fresh dev tenant can be registered via `/onboarding/register-tenant`, listed in `/platform/tenants`, and an owner can demo-log into it.
- Old JSON learner columns are not regressed (still present for read-only fallback during the same release).
