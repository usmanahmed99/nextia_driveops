# 25 — Authentication: Microsoft Entra External ID

> DrivingOps uses **Microsoft Entra External ID** for customer (driving-school) identity. This replaces Azure AD B2C, which Microsoft closed to new tenants on 2025-05-01. The app architecture is unchanged — only the identity-provider configuration, authority handling, and token validation were re-pointed. See [24-self-service-signup.md](./24-self-service-signup.md) for the signup/invite flows that sit on top of this.

## What changed vs. B2C (and what didn't)

| Aspect | Azure AD B2C | Microsoft Entra External ID |
| --- | --- | --- |
| Authority | `https://<t>.b2clogin.com/<t>.onmicrosoft.com/B2C_1_signupsignin` (policy in path) | `https://<t>.ciamlogin.com/<t>.onmicrosoft.com` (**no policy in path**) |
| User flow | `B2C_1_signupsignin` baked into the authority | A sign-up/sign-in user flow associated with the app in the portal |
| Domain | `b2clogin.com` | `ciamlogin.com` |
| Token validation | audience + JWKS + optional issuer | identical mechanism, re-pointed config |
| App architecture | MSAL SPA + JWT-validating API | **unchanged** |

The MSAL client already derives `knownAuthorities` from the authority host, so it works with `ciamlogin.com` without code changes. The backend already validated tokens against a configurable audience/issuer/JWKS, so only the config names and values changed.

## Required portal setup (one-time)

1. **Create an External ID tenant** — Azure portal → *Create a resource* → *Microsoft Entra External ID* → external/customer configuration. Note the tenant subdomain `<tenant>.onmicrosoft.com` and the tenant ID (GUID).
2. **Create a sign-up/sign-in user flow** — Entra admin center → *External Identities* → *User flows* → create a Sign up and sign in flow. Enable Email + password (and any social providers you want). Attributes to collect/return: at least **Email** and **Display Name**.
3. **Register the SPA app** — *App registrations* → New → platform **Single-page application**. Redirect URIs:
   - `http://localhost:5173` (local dev)
   - the deployed frontend origin, e.g. `https://app.drivingops.ca`
   Capture the **Application (client) ID** → `ENTRA_EXTERNAL_ID_CLIENT_ID` / `VITE_ENTRA_EXTERNAL_ID_CLIENT_ID`.
4. **Register the API app** — separate app registration for the backend API. Under *Expose an API*, set the Application ID URI (e.g. `api://<api-app-client-id>`) → this is `ENTRA_EXTERNAL_ID_AUDIENCE`. Add a delegated scope, e.g. `access_as_user`.
5. **Grant the SPA permission to the API scope** — on the SPA app → *API permissions* → add the API's `access_as_user` delegated permission and grant admin consent. The full scope string `api://<api-app-client-id>/access_as_user` is `ENTRA_EXTERNAL_ID_SCOPE` / `VITE_ENTRA_EXTERNAL_ID_API_SCOPE`.
6. **Associate the user flow** with both apps so customers hit the sign-up/sign-in experience.
7. **Add logout redirect URIs** matching the local + deployed origins.

## Required values

| Variable | Where it comes from |
| --- | --- |
| `ENTRA_EXTERNAL_ID_AUTHORITY` | `https://<tenant>.ciamlogin.com/<tenant>.onmicrosoft.com` (no policy segment) |
| `ENTRA_EXTERNAL_ID_CLIENT_ID` | SPA app registration → Application (client) ID |
| `ENTRA_EXTERNAL_ID_AUDIENCE` | API app registration → Application ID URI (`api://<api-app-client-id>`) |
| `ENTRA_EXTERNAL_ID_JWKS_URL` | `jwks_uri` from the tenant OIDC discovery document (below) |
| `ENTRA_EXTERNAL_ID_ISSUER` | the **exact** `iss` from a real decoded access token (below) |
| `ENTRA_EXTERNAL_ID_SCOPE` | `api://<api-app-client-id>/access_as_user` |
| `VITE_ENTRA_EXTERNAL_ID_AUTHORITY` | same as authority above |
| `VITE_ENTRA_EXTERNAL_ID_CLIENT_ID` | same as SPA client id |
| `VITE_ENTRA_EXTERNAL_ID_API_SCOPE` | same as scope above |

> Do not guess the authority/JWKS/issuer host. Confirm them from the tenant's OIDC discovery document and a real token. Microsoft has used both `ciamlogin.com` and `<tenant-id>.ciamlogin.com` forms; capture the exact values your tenant emits.

### Get the JWKS URL from OIDC discovery

```
curl https://<tenant>.ciamlogin.com/<tenant-id>/v2.0/.well-known/openid-configuration
```

Use the `jwks_uri` field from that JSON as `ENTRA_EXTERNAL_ID_JWKS_URL`, and the `issuer` field as a starting point — but still verify against a real token (issuer can vary by token version).

### Get the exact issuer from a real token

After a successful SPA login, grab the access token (DevTools → network → the API call's `Authorization: Bearer …`, or `sessionStorage`) and decode it:

```bash
# header.payload.signature — decode the payload (2nd segment)
echo "<jwt>" | cut -d. -f2 | base64 -d 2>/dev/null | python -m json.tool
```

Use the `iss` value verbatim as `ENTRA_EXTERNAL_ID_ISSUER`.

## Validate the token has what we expect

From the decoded payload, confirm:

- `aud` == `ENTRA_EXTERNAL_ID_AUDIENCE` (the API app id URI / client id).
- `iss` == `ENTRA_EXTERNAL_ID_ISSUER`.
- `scp` contains `access_as_user` (proves it's an **access token** for the API, not just an ID token).
- A usable identity claim: one of `oid` / `sub`, plus `email` / `emails` / `preferred_username`, and ideally `name`.

The backend reads these defensively in `AuthService.user_from_claims` (subject from `sub`/`oid`, email from `emails`/`email`/`preferred_username`, name from `name`). RBAC and tenant/membership mapping are unchanged — identity is keyed on `(issuer, subject)` with an email fallback.

## How the app consumes config

- **Backend** ([app/core/config.py](../dops-api/app/core/config.py)) exposes resolved `external_id_*` properties that prefer `ENTRA_EXTERNAL_ID_*` and fall back to the deprecated `AZURE_B2C_*` during migration. [app/core/security.py](../dops-api/app/core/security.py) `decode_external_id_token` validates audience + JWKS + issuer; it fails closed in production when no issuer is pinned. `/auth/login-config` reports `provider="entra_external_id"` once an authority is set, and advertises the API scope so MSAL requests an access token.
- **Frontend** ([src/lib/api/config.ts](../dops-frontend/src/lib/api/config.ts), [src/lib/auth/msal.ts](../dops-frontend/src/lib/auth/msal.ts)) builds the MSAL client for the External ID authority and requests the API scope. Runtime config comes from `/auth/login-config`; `VITE_*` vars are a build-time fallback.

## Infra / CI

- Bicep params: `entraExternalIdAuthority|ClientId|Audience|JwksUrl|Issuer|Scope` ([infra/main.bicep](../dops-infra/infra/main.bicep)). Stored in Key Vault as `DOPS-ENTRA-EXTERNAL-ID-*` and surfaced to the API container as `ENTRA_EXTERNAL_ID_*` (the four secrets only when `externalIdConfigured`).
- Pipeline reads `DOPS_ENTRA_EXTERNAL_ID_*` (from `dops-dev-secrets` variable group). The bicepparam falls back to the old `DOPS_AZURE_B2C_*` values if the new ones are unset.
- **Manual step**: add `DOPS-ENTRA-EXTERNAL-ID-AUTHORITY|CLIENT-ID|AUDIENCE|JWKS-URL`, `ENTRA_EXTERNAL_ID_ISSUER`, `ENTRA_EXTERNAL_ID_SCOPE` to the `dops-dev-secrets` variable group (empty until you have real values).

## Local development

No External ID config needed locally: leave the values empty and use the demo role buttons (`/auth/demo-login`, env-gated to local/dev/test). To exercise the real flow locally, set the `VITE_ENTRA_EXTERNAL_ID_*` (frontend) and `ENTRA_EXTERNAL_ID_*` (backend) values and run both apps; redirect URIs must include `http://localhost:5173`.

## Remaining B2C references (intentional, during migration)

- Backend `azure_b2c_*` settings + the resolved-property fallbacks ([config.py](../dops-api/app/core/config.py)) — **fallback aliases**; remove once all environments set `ENTRA_EXTERNAL_ID_*`.
- Frontend `VITE_B2C_*` reads in [config.ts](../dops-frontend/src/lib/api/config.ts) — **fallback aliases**.
- Pipeline `DOPS_AZURE_B2C_*` exports + bicepparam fallback — **fallback aliases**.
- `authProvider` type union keeps `azure_ad_b2c` for any historical session payloads; new logins use `entra_external_id`.
- Existing `users.auth_provider = "azure_b2c"` rows in the DB are historical labels and are harmless; new sign-ins write `entra_external_id`.

Everything else (provider value, decode function names, env var names, Key Vault secret names, login-config) now uses External ID naming.
