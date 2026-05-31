# 03 — Authentication & Sessions

> **Updated**: auth is now **Microsoft Entra External ID only** (Azure AD B2C is closed to new tenants; password signup/login removed). Self-service registration creates a school owner; all other roles are invite-only. See [24-self-service-signup.md](./24-self-service-signup.md) for the signup/invite flows and [25-auth-entra-external-id.md](./25-auth-entra-external-id.md) for the identity-provider setup, token validation, and required values. Sections below describing password endpoints or B2C are historical.

## Purpose

Sign users in, return a session containing user, active tenant, memberships, and permissions, and provide a logout and refresh path.

## Current State

### Endpoints — [`dops-api/app/api/v1/endpoints/auth.py`](../dops-api/app/api/v1/endpoints/auth.py)

| Method | Path | Body | Returns |
| ------ | ---- | ---- | ------- |
| POST | `/api/v1/auth/demo-login` | `{ role: "PLATFORM_ADMIN" \| ... }` | `LoginResponse(session)` |
| GET  | `/api/v1/auth/session`    | — | `Session` |
| POST | `/api/v1/auth/logout`     | — | `LogoutResponse` |
| POST | `/api/v1/auth/refresh`    | — | `RefreshSessionResponse(session)` |

`AuthService.demo_login` picks a seeded user for the chosen role from the `school-northstar` tenant and returns a placeholder JWT (string only, not signed in any verifiable way).

### Frontend

- `src/features/auth/LoginPage.tsx` shows email/password fields (purely cosmetic) plus demo-role buttons.
- Session is stored in `localStorage` and re-hydrated on app load.
- The HTTP adapter sends `Authorization: Bearer <session.accessToken>` + `X-Tenant-ID: <activeTenant.id>` on every request.

## Gaps For Real Implementation

1. **No real identity provider**. The demo JWT is not signed or verified.
2. **No password / email / phone auth**, no MFA, no reset flow.
3. **No tenant-scoped invite + signup** flow (Owner invites Staff/Instructor; school registers a Learner who confirms email).
4. **No SSO** for school owners that want Google/Microsoft.
5. **No proper logout** — the API endpoint is a no-op, all state is client-side.
6. **No refresh token rotation**. `refresh` simply returns the current session.
7. **No anti-CSRF**. The Bearer model is fine but the cookie path (if we want SSR public pages) is undefined.
8. **No rate limiting / brute-force protection** on login.

## Real Implementation Plan

### A. Pick the identity provider

Recommended: **Azure AD B2C** (matches the rest of the Azure-based infra; user flow customization for sign-up/in, password reset, MFA; supports OIDC + social federation).

Alternative: **Auth0** or **Clerk** if B2C friction proves too high. Either way the API contract below stays the same.

The API does **not** issue user credentials. It only:

- accepts an OIDC ID token / access token from the IdP via `Authorization: Bearer <token>`,
- verifies its signature against the IdP JWKS,
- looks up or upserts the local `User` row by `(issuer, subject)`,
- resolves the user's `TenantMembership` rows,
- returns the same `Session` shape the frontend already consumes.

### B. New endpoints

```
GET  /api/v1/auth/session           (existing) — now verifies real JWT
POST /api/v1/auth/logout            invalidates server-side refresh token if used
POST /api/v1/auth/exchange-code     body: { code, redirect_uri }
                                    only used if frontend hands off to API (PKCE flow)
GET  /api/v1/auth/login-config      returns IdP discovery URLs the SPA should use
```

Remove `demo-login` from prod builds (gate by `settings.environment == "dev"`).

### C. Frontend changes

- `LoginPage` redirects to IdP using `oidc-client-ts` or the IdP-provided SDK.
- After redirect, exchange code for tokens, call `GET /auth/session`, store session.
- Token refresh handled by the OIDC client; the SPA calls `auth/refresh` only when it wants the server to re-evaluate membership/permissions (e.g. after invite acceptance).
- Demo-role login buttons hidden when `VITE_ENVIRONMENT !== "dev"`.

### D. Account provisioning flows

Three flows to implement, each backed by an IdP user-journey + an API onboarding endpoint:

1. **Self-service school registration** (public site → "Start free trial"): creates Tenant + Owner User + first Membership. API: `POST /api/v1/onboarding/register-tenant`.
2. **Staff/Instructor invite**: Owner invites by email; API stores a one-time `Invitation(token, tenant_id, role, expires_at)`. Email link → IdP sign-up → `POST /api/v1/onboarding/accept-invite { token }` to mint the membership. Endpoints belong under `users.manage`.
3. **Learner registration from the school**: school admin registers a learner; system creates `User` with `email_unverified=true` and a phone; sends a portal-onboarding link (WhatsApp/email). When the learner clicks, IdP claims the account and they reach the portal.

### E. Session refresh policy

- IdP access token TTL: 15 min.
- IdP refresh token TTL: 30 days, rotating.
- `POST /auth/refresh` is called when the API rejects a request with `401 token_expired` (the frontend's http layer handles this transparently). Otherwise the OIDC client refreshes silently.

### F. MFA & policy

- Require MFA for `SCHOOL_OWNER`, `SCHOOL_ADMIN`, and `PLATFORM_ADMIN`. Configure as a user-journey condition in B2C.
- Lock account after 5 failed attempts for 15 minutes (IdP policy).
- Password rules: 12+ chars, one symbol, one digit, NIST-style breach-list check.

## Acceptance Criteria

- A new owner can sign up from the marketing site and land in an empty tenant.
- An owner can invite a teammate by email; teammate signs up via the IdP and is correctly scoped to the tenant.
- A learner registered by the school receives a magic link and lands on the learner portal.
- All admin roles are forced through MFA on first login.
- Tokens issued by IdP are verified against JWKS on every API call; demo bypass disabled in stg/prod images.
- Logout revokes the refresh token and clears local session.
- Penetration test pass on common OWASP auth issues (CSRF, fixation, brute-force, redirect-uri tampering).

## Dependencies

- [02-multitenancy-rbac.md](./02-multitenancy-rbac.md) — invite/membership endpoints.
- Infrastructure: add Azure AD B2C tenant + secrets to Key Vault. Track in [16-infrastructure.md](./16-infrastructure.md).
