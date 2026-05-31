# 24 — Signup, Invites & Onboarding

> **Status**: implemented. This documents the shipped External-ID-only auth model, the invite flow, the ACS email wiring, dev login, and the one-time Microsoft Entra External ID portal setup needed to turn it on.

## The model

Authentication is **Microsoft Entra External ID only** in deployed environments. External ID proves identity; the app decides authorization (tenant + role). Password auth has been removed.

A freshly-authenticated External ID identity has **no tenant membership** and can do exactly two things:

1. **Create a school** → becomes `SCHOOL_OWNER` of a new tenant (self-service).
2. **Accept an invite** → gets the role the inviter assigned, on an existing tenant.

There is no path to self-assign `SCHOOL_ADMIN` / `INSTRUCTOR` / `LEARNER` / `PARENT_GUARDIAN`. So **staff and learners are invite-only by construction**, and "the only users that can self-register are school owners" holds.

The enabling primitive: `get_current_identity` ([app/api/deps.py](../dops-api/app/api/deps.py)) validates an Entra External ID token and finds/creates the `User` **without** requiring a membership. `get_current_session` still requires one. Register-school and accept-invite use the identity dependency; everything else uses the session dependency.

## Flow A — self-service school signup (owner)

```
/signup → "Sign in with Microsoft" → External ID sign-up/sign-in user flow
   → return to /signup, status = needs_provisioning (authenticated, no membership)
   → "Create your school" form (name, city, phone)
   → POST /api/v1/onboarding/register-school   (Depends: get_current_identity)
   → Tenant + SCHOOL_OWNER membership + trial subscription + 4 onboarding tasks + seeded learning paths
   → Session returned, dashboard.
```

Owner name/email come from the **Entra External ID token**, never the payload — the membership is bound to the authenticated subject. A second register-school for an identity that already has an active membership returns `409`.

## Flow B — invite (every other role)

```
Owner/Admin → Settings → Members → invite (email + role)
   → POST /api/v1/tenants/{id}/memberships   (require_permission users.manage)
   → invited membership + Invitation(token_hash) + ACS email with link:
        {FRONTEND_BASE_URL}/invite/accept?token=<raw>
Invitee → clicks link → /invite/accept
   → GET /api/v1/onboarding/invite?token=  (public preview: school, role, email)
   → "Continue with Microsoft" (login_hint = invited email)
   → return, status = needs_provisioning → auto POST /api/v1/onboarding/accept-invite
   → backend verifies invited email == authenticated External ID email, activates membership
   → Session returned, role landing page.
```

The email-match check (`accept_invite` in [tenant_service.py](../dops-api/app/services/tenant_service.py)) stops an invite being consumed by a different account. Mismatch → `403`.

## Backend surface

| Endpoint | Auth | Purpose |
| --- | --- | --- |
| `POST /onboarding/register-school` | identity | create school + owner membership |
| `GET /onboarding/invite?token=` | public | invite preview (no secrets) |
| `POST /onboarding/accept-invite` | identity | bind membership to identity |
| `POST /tenants/{id}/memberships` | `users.manage` | create invite + send email |
| `POST /auth/demo-login` | none (dev only) | role switcher, 404 in prod |
| `GET /auth/login-config` | public | `provider = entra_external_id` when configured, else `demo` |

Removed: `POST /auth/signup`, `POST /auth/login` (password), PBKDF2 hashing, `users.password_hash` (migration `0002_drop_user_password_hash`).

## ACS transactional email

Invites are delivered via Azure Communication Services Email ([app/services/email_service.py](../dops-api/app/services/email_service.py)). It degrades gracefully: when `ACS_CONNECTION_STRING` / `ACS_SENDER_ADDRESS` are unset (local dev) it logs the invite link instead of sending, so the flow is testable without a verified domain. The Azure SDK is imported lazily.

Config: `ACS_CONNECTION_STRING`, `ACS_SENDER_ADDRESS`, `FRONTEND_BASE_URL` ([app/core/config.py](../dops-api/app/core/config.py)).

## Observability

All services log through the reusable factory `get_logger(__name__)` / `log_event(...)` ([app/core/logging.py](../dops-api/app/core/logging.py)). Logs are single-line JSON in deployed envs with stable keys (`logger`, `module`, `func`, `location`, `request_id`, `tenant_id`, `user_id`) stamped automatically from contextvars ([app/core/log_context.py](../dops-api/app/core/log_context.py)). App Insights export auto-configures when `APPLICATIONINSIGHTS_CONNECTION_STRING` is present ([app/core/telemetry.py](../dops-api/app/core/telemetry.py)). Auth events: `identity.created`, `school.registered`, `invite.created`, `invite.accepted`. The frontend mirrors this with `createLogger(scope)` ([src/lib/observability/logger.ts](../dops-frontend/src/lib/observability/logger.ts)).

## Dev / test login

`demo-login` is kept, env-gated to `local|dev|test`, hard-disabled (404) in prod. `get_current_session`/`get_current_identity` accept the internal HS256 demo token **only** in non-prod, so the role switcher works in dev even when External ID is configured, and can never be used against production. The login page shows the demo role buttons whenever `login-config.demoEnabled` is true.

## Identity provider setup (one-time)

DrivingOps uses **Microsoft Entra External ID**. The full portal checklist (tenant, sign-up/sign-in user flow, SPA + API app registrations, exposing the API scope, redirect URIs) and the required values live in **[25-auth-entra-external-id.md](./25-auth-entra-external-id.md)**. Key difference from B2C: the authority is `https://<tenant>.ciamlogin.com/<tenant>.onmicrosoft.com` with **no** user-flow segment in the path, and issuer must be pinned in production.

## Infra wiring

- External ID secrets are gated by `externalIdConfigured` in [infra/main.bicep](../dops-infra/infra/main.bicep): all four of authority/clientId/audience/jwksUrl must be populated before the API receives them. Until then the API reports `provider: demo`. Populate via the `DOPS_ENTRA_EXTERNAL_ID_*` env vars consumed by [parameters/dev.bicepparam](../dops-infra/infra/parameters/dev.bicepparam).
- ACS Email is always provisioned ([modules/communication-email.bicep](../dops-infra/infra/modules/communication-email.bicep)); the connection string flows into Key Vault and onto the API automatically. Uses an Azure-managed sender domain by default (swap to a verified custom domain for prod).
- Set `DOPS_FRONTEND_BASE_URL` to the deployed frontend origin so invite links resolve. Register that origin (and the `/invite/accept` path is covered by the SPA root redirect) in the External ID SPA app.
- The frontend reads identity-provider config at runtime from `/auth/login-config`, so flipping demo→External ID needs no frontend rebuild — only the infra params + registered redirect URIs.

## Verification

Local (demo, no External ID):
- `cd dops-api && .\.venv\Scripts\python.exe -m pytest -q` (fresh DB) — auth/invite tests pass.
- Frontend `npm run typecheck && npm run lint`.
- Demo role buttons log you in as each role.
- Invite from Settings → Members; the API log prints the `/invite/accept?token=...` link (ACS stub). Open it in another browser profile, accept (demo token), confirm the role landing.

Deployed (real Entra External ID):
1. Populate the four External ID params + `DOPS_FRONTEND_BASE_URL`, deploy infra, confirm `/auth/login-config` returns `provider: entra_external_id`.
2. `/signup` → Microsoft → create school → land on dashboard as owner.
3. Invite a teammate → they receive the ACS email → accept → land in the right role.
4. Confirm `demo-login` returns 404 in prod.

## Deliberately not covered

- Email verification, password reset, MFA — handled by the External ID user flow / conditional access.
- Plan selection / Stripe — `planCode` captured, subscription created `trial`; no money flows.
- Learner activation magic links beyond the invite flow.
- Custom per-tenant domains.
