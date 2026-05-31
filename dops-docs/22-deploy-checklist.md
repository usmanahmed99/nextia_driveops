# 22 — Deploy Checklist

Concrete step-by-step deploy guide for the slices currently sitting unshipped on `main`. Everything that **can** be expressed in Bicep is in Bicep — what's left is genuinely operator work (Azure DevOps Library entries, Azure portal B2C clicks, three pipeline buttons).

## Slices stacked on dev as of this writing

1. Backend learner module on normalised columns (paged list, workspace, holds, withdraw, idempotency, audit + outbox).
2. Frontend learner module (paged + filtered list, workspace-driven detail, holds + withdraw dialogs, registration wizard, tenant-suspension boundary).
3. Frontend B2C SPA wiring (MSAL, login-config drives provider, demo gated by server flag, 401 silent-refresh).
4. Platform-admin Tenants console (list, create, status editor, subscription editor, tenant switcher).
5. Backend driving centers + service catalog + slot-gap (new tables, CRUD endpoints, seed update, tests).
6. Frontend Settings → Driving centers + Settings → Services (CRUD pages reachable from Settings landing).
7. **Infra slice (just shipped, this checklist)**: B2C parameters + Key Vault secrets + API container env vars wired through Bicep; `app/db/init_db.py` fixed so `Base.metadata.create_all` picks up `driving_centers` + `service_offerings`.

Status legend: ☐ pending — you have to do it · ☑ done · ⚠ manual click in a UI · 💻 single CLI command.

---

## 0. Pre-flight (one-time, ~5 min)

Run from your workstation, signed in to Azure (`az login`). Confirms you're pointed at the right subscription + service connection.

```bash
# 0.1 - confirm subscription + resource group
az account show --query "{subscription:name, id:id}" -o table
az group show --name rg-dops-dev --query "{name:name, location:location}" -o table

# 0.2 - confirm the existing Container Apps still resolve (sanity check)
az containerapp show --resource-group rg-dops-dev --name ca-dops-api-dev --query "properties.configuration.ingress.fqdn" -o tsv
az containerapp show --resource-group rg-dops-dev --name ca-dops-frontend-dev --query "properties.configuration.ingress.fqdn" -o tsv
```

If any of those error out, the resource group or Container Apps were renamed since the last deploy — fix that first.

---

## Path A — Demo-mode deploy (no B2C yet, ~25 min)

Goal: ship Slices 1–7 to the dev Container Apps with **demo login still on**. B2C secrets stay empty; `GET /auth/login-config` returns `provider: "demo"` and the SPA keeps showing demo role buttons. Learners, platform tenants, driving centers, services all light up end-to-end.

Run the steps **in order**. Stop and ping me if anything fails.

### A1. Reset the dev Postgres database (~2 min) ⚠

**Why**: the existing dev DB was created with the old schema. Migration `0002` and `0003` add new tables/columns; the API does not run alembic on startup, it only calls `Base.metadata.create_all`, which is a no-op on existing tables. The safe path for dev is to drop and let the new app rebuild from scratch. All data is demo data, so this is fine.

> ⚠ This deletes all dev data. If you've put anything important in dev you want to keep, stop and tell me.

```bash
# A1.1 - get the Postgres server FQDN + admin user from infra (if you don't remember)
PG_FQDN=$(az postgres flexible-server show \
  --resource-group rg-dops-dev \
  --name "$(az postgres flexible-server list --resource-group rg-dops-dev --query '[0].name' -o tsv)" \
  --query "fullyQualifiedDomainName" -o tsv)
echo "Postgres: $PG_FQDN"

# A1.2 - temporarily allow your client IP through the firewall
MY_IP=$(curl -s ifconfig.me)
az postgres flexible-server firewall-rule create \
  --resource-group rg-dops-dev \
  --name "$(az postgres flexible-server list --resource-group rg-dops-dev --query '[0].name' -o tsv)" \
  --rule-name allow-deploy-workstation \
  --start-ip-address "$MY_IP" --end-ip-address "$MY_IP"

# A1.3 - drop + recreate the dops database (requires psql installed locally)
# Replace <admin-password> with the value of POSTGRES-ADMIN-PASSWORD from Key Vault.
PGPASSWORD='<admin-password>' psql \
  "host=$PG_FQDN port=5432 dbname=postgres user=dopsadmin sslmode=require" \
  -c "DROP DATABASE IF EXISTS dops;" -c "CREATE DATABASE dops;"

# A1.4 - remove the firewall rule when done
az postgres flexible-server firewall-rule delete \
  --resource-group rg-dops-dev \
  --name "$(az postgres flexible-server list --resource-group rg-dops-dev --query '[0].name' -o tsv)" \
  --rule-name allow-deploy-workstation --yes
```

> Alternative if you don't have `psql` installed: open the Postgres flexible server in the Azure portal → **Databases** → delete `dops` → click **+ Add database** → name `dops` → save. Then close the temporary firewall rule.

### A2. Push the code + run the three pipelines

The three pipelines auto-trigger on `main`. The order matters because the API pipeline expects ACR + Container App to exist, and the frontend bakes `VITE_API_BASE_URL` from the deployed API URL.

```bash
# A2.1 - push every repo's main
cd C:/Users/musma/NextiaPortfolio/DrivingOps/dops-api && git status
cd C:/Users/musma/NextiaPortfolio/DrivingOps/dops-frontend && git status
cd C:/Users/musma/NextiaPortfolio/DrivingOps/dops-infra && git status
```

Commit + push each in turn. **The infra repo must merge to `main` first**, then API, then frontend.

**A2.2. Pipeline: `DrivingOps Infra CI/CD`** (~5 min):

⚠ Watch the run. It executes three stages:

1. **Validate** — `az bicep build` + `az deployment group validate`. Confirms the new B2C params + Key Vault secrets compile.
2. **What-if Dev** — shows the diff. **Expected diffs**:
   - 4 new Key Vault secrets: `DOPS-AZURE-B2C-AUTHORITY`, `DOPS-AZURE-B2C-CLIENT-ID`, `DOPS-AZURE-B2C-AUDIENCE`, `DOPS-AZURE-B2C-JWKS-URL` (all with empty string values).
   - API Container App: 4 new `secrets` entries (azure-b2c-*) + 7 new env vars (AZURE_B2C_*, FRONTEND_AUTH_*). Empty values for the issuer/redirect ones; secrets resolve to empty strings.
   - Nothing else changes.
3. **Deploy Dev** — applies it.

If the diff shows anything else (e.g. a Container App revision wipe), pause and ping me.

**A2.3. Pipeline: `DrivingOps API CI/CD`** (~10 min):

Watch the run. Stages:

1. **Validate** — ruff + pytest (23 tests).
2. **BuildAndPushImage** — Docker build of the new API image with all the slice changes; pushes to ACR.
3. **DeployDev** — `az containerapp update --image ...` which swaps the revision.

**The API does not run alembic.** It relies on `Base.metadata.create_all` at startup. Since A1 dropped the database, the lifespan handler now runs `init_db()` and then `seed()` (because `SEED_DEMO_DATA=true` in dev). The seed creates the NorthStar tenant + 8 learners + 3 driving centers + 5 service offerings.

4. **VerifyDev** — polls `/api/v1/health` up to 30 times. Should pass on the first attempt.

If health fails: open the API Container App revision logs in Azure portal — look for tracebacks from `init_db()` or `seed()`. Stop and ping me with the log.

**A2.4. Pipeline: `DrivingOps Frontend CI/CD`** (~5 min):

Watch the run.

1. **Validate** — `npm ci`, `npm run lint`, `npm run typecheck`, `npm run build` with `VITE_API_BASE_URL` baked.
2. **BuildAndPushImage** — Docker build (passes the same `VITE_*` build args).
3. **DeployDev** — swap the frontend revision.
4. **VerifyDev** — curls the frontend URL.

### A3. Smoke test in the browser (~5 min)

Open `https://<frontend-fqdn>` (look it up in Step 0.2 output). Run through this list **in order**:

1. ☐ Login page renders. The Microsoft button is **not** present. Demo role buttons are visible (because `loginConfig.provider === "demo"`).
2. ☐ Click **Demo as School admin** → land on `/` (Dashboard). No 401s in the network tab.
3. ☐ Sidebar → **Learners** → list shows 8 seeded learners; pagination defaults to 25 per page; chips filter; search returns matches.
4. ☐ Click Sofia Martin → workspace loads (upcoming sessions, outstanding invoices, recommendations panel). Add a behaviour hold → it appears with a red badge. Resolve it.
5. ☐ Click **Register learner** on the list → wizard walks all 6 steps → on submit you land on the new learner's detail page.
6. ☐ Switch role to **Platform admin** (header dropdown). Sidebar gains **Platform tenants** entry. Header gets the **tenant switcher**.
7. ☐ Open **Platform tenants** → list shows NorthStar + Aurora. Click **Manage** → status + subscription editors open. Change Aurora to `suspended`, save. Click **Act as** Aurora in the row → the active tenant header changes.
8. ☐ While "Acting as Aurora" (now suspended), open any tenant-scoped page (e.g. **Learners**). You should see the **TenantSuspendedError boundary** — full-page locked state.
9. ☐ Switch back to NorthStar, set Aurora back to `trial`.
10. ☐ Sidebar → **Settings** → **Driving centers** → list shows 3 seeded centers (downtown branch, Laval branch, SAAQ Service Center Montreal). Add a partner center → it appears in active. Deactivate → it moves to inactive.
11. ☐ **Settings** → **Services & pricing** → list shows 5 seeded services (Class 5 theory, practical, mock road test, road-test prep, SAAQ chaperone). Edit one → change duration → save. Deactivate one → moves to inactive.

If every check passes, Path A is **DONE**. Tell me and we either tackle Path B or move on to the scheduling-page service picker.

If any check fails, paste the failing step + the browser console + Azure portal API logs.

---

## Path B — Switch the SPA to Azure AD B2C (~90 min including portal work)

Do this **after Path A is green** and you want real OIDC sign-in. The frontend already auto-detects `provider: "azure_b2c"` from the API's `login-config` and switches to the Microsoft sign-in button. **No redeploys needed once the secrets are populated** — just re-run the infra pipeline to push the new env values into the API Container App, and the next page load on the frontend flips automatically.

### B1. Provision the B2C tenant ⚠ Azure portal

1. ☐ Azure portal → **Create a resource** → search **"Azure Active Directory B2C"** → Create → **Create a new Azure AD B2C Tenant**.
   - Organisation name: `DrivingOps Identity`
   - Initial domain name: `drivingopsidentity` *(must be globally unique)*
   - Country/Region: `Canada`
   - Subscription: same as `rg-dops-dev`
   - Resource group: `rg-dops-dev`
2. ☐ Wait for "Your B2C tenant is created" (~3 min).
3. ☐ Click the directory switcher (top right) → switch into **drivingopsidentity** directory.

### B2. Create the sign-up/sign-in user flow ⚠ Azure portal

1. ☐ In the B2C directory: **Azure AD B2C** → **User flows** → **+ New user flow**.
2. ☐ Choose **Sign up and sign in** → **Recommended version**.
3. ☐ Name: `signin` *(the full name will be `B2C_1_signin`)*.
4. ☐ Identity providers: check **Email signup** (add Microsoft/Google later if needed).
5. ☐ Multifactor authentication: **Conditional** for now (we'll harden later).
6. ☐ User attributes and token claims: collect **Display Name**, **Given Name**, **Surname**, **Email Address**. Return claims: same four + **User's Object ID** + **Email Addresses**.
7. ☐ **Create**.

### B3. Register the API app ⚠ Azure portal

1. ☐ **Azure AD B2C** → **App registrations** → **+ New registration**.
2. ☐ Name: `dops-api`.
3. ☐ Supported account types: **Accounts in any identity provider or organisational directory (for authenticating users with user flows)**.
4. ☐ Redirect URI: skip — APIs don't need one.
5. ☐ **Register**.
6. ☐ On the new app's overview: copy the **Application (client) ID** — call this `API_APP_ID`.
7. ☐ Left rail → **Expose an API** → **+ Add a scope** → keep the default Application ID URI (`https://drivingopsidentity.onmicrosoft.com/<API_APP_ID>` or use friendlier `api`) → **Save and continue**.
   - Scope name: `access_as_user`
   - Admin consent display name: "Access DrivingOps API as the signed-in user"
   - State: **Enabled**
   - **Add scope**
8. ☐ Copy the **Application ID URI** (the bit before `/access_as_user` if you accepted the default). This is **`AZURE_B2C_AUDIENCE`**.

### B4. Register the SPA app ⚠ Azure portal

1. ☐ **App registrations** → **+ New registration**.
2. ☐ Name: `dops-frontend-spa`.
3. ☐ Supported account types: same as B3.
4. ☐ Redirect URI: select **Single-page application (SPA)** → enter `https://<frontend-fqdn>/`. Add another for local dev: `http://localhost:5173/`. Add **Logout URL**: `https://<frontend-fqdn>/login`.
5. ☐ **Register**.
6. ☐ Copy the **Application (client) ID** — this is **`AZURE_B2C_CLIENT_ID`**.
7. ☐ Left rail → **API permissions** → **+ Add a permission** → **My APIs** → `dops-api` → **Delegated permissions** → check `access_as_user` → **Add permissions** → click **Grant admin consent for drivingopsidentity** → confirm.

### B5. Capture the remaining URLs

```
Authority           = https://drivingopsidentity.b2clogin.com/drivingopsidentity.onmicrosoft.com/B2C_1_signin
JWKS URL            = https://drivingopsidentity.b2clogin.com/drivingopsidentity.onmicrosoft.com/B2C_1_signin/discovery/v2.0/keys
Issuer              = (decode a real access token at jwt.ms after first sign-in; copy iss exactly)
```

Replace `drivingopsidentity` with the actual initial-domain you chose in B1.

For **Issuer**: easiest way is to grab it from the OpenID discovery doc once the tenant is up:

```bash
curl https://drivingopsidentity.b2clogin.com/drivingopsidentity.onmicrosoft.com/B2C_1_signin/v2.0/.well-known/openid-configuration | jq -r '.issuer'
```

That value is **`AZURE_B2C_ISSUER`**.

### B6. Push the values into the pipeline ⚠ Azure DevOps Library

Open **Azure DevOps** → your project → **Pipelines** → **Library** → variable group **`dops-dev-secrets`**. Add four secret variables linked to Key Vault (or set them directly as secret variables — they'll get persisted into Key Vault by the next infra deploy):

| Variable name | Value | Make secret? |
| --- | --- | --- |
| `DOPS-AZURE-B2C-AUTHORITY` | Authority from B5 | ✅ |
| `DOPS-AZURE-B2C-CLIENT-ID` | SPA client id from B4 | ✅ |
| `DOPS-AZURE-B2C-AUDIENCE` | API audience from B3 | ✅ |
| `DOPS-AZURE-B2C-JWKS-URL` | JWKS URL from B5 | ✅ |

Then open variable group **`dops-dev`** (non-secret) and add:

| Variable name | Value |
| --- | --- |
| `AZURE_B2C_ISSUER` | Issuer from B5 |
| `FRONTEND_AUTH_REDIRECT_URI` | `https://<frontend-fqdn>/` |
| `FRONTEND_AUTH_LOGOUT_REDIRECT_URI` | `https://<frontend-fqdn>/login` |

> The Bicep already declared these as parameters in `main.bicep`, the bicepparam reads them from env vars, and the pipeline already forwards both variable groups into the deployment env. So no further code edits are needed.

### B7. Re-run the Infra pipeline (~5 min)

Run `DrivingOps Infra CI/CD` again. The **What-if Dev** stage diff should show:

- The 4 Key Vault secrets gain real values (currently empty strings).
- The API Container App env vars `AZURE_B2C_ISSUER`, `FRONTEND_AUTH_REDIRECT_URI`, `FRONTEND_AUTH_LOGOUT_REDIRECT_URI` switch from empty to the URLs from B6.
- Nothing else changes.

Approve, deploy. The API revision will roll automatically and pick up the new secrets.

### B8. Verify (~5 min)

1. ☐ `curl https://<api-fqdn>/api/v1/auth/login-config` → returns `provider: "azure_b2c"` with the authority, client id, audience populated.
2. ☐ Open `https://<frontend-fqdn>` in an incognito window → Login page now shows only **Sign in with Microsoft** (no demo buttons unless `ENVIRONMENT=dev` is still in the API env, which it is — both can coexist for dev).
3. ☐ Click sign-in → land on the B2C `B2C_1_signin` flow → create a test account → returns to the app → dashboard loads.
4. ☐ Browser dev tools → Application → Session Storage → look for `msal.*` entries with valid token claims.
5. ☐ Sign out → redirects through B2C end-session → returns to `/login`.

### B9. (Later) Hardening

- ☐ Flip `ENVIRONMENT` to `staging`/`production` on stg/prod Container Apps so `/auth/demo-login` returns `404` and the demo buttons disappear from the SPA. For dev, leaving it as `dev` is fine — both auth paths work side by side.
- ☐ MFA-on-admin-roles via B2C custom policy (out of scope for this checklist; see [03-authentication.md](./03-authentication.md)).
- ☐ Add the production frontend FQDN(s) to the SPA app's redirect URI list before promoting beyond dev.

---

## What changed in Bicep / code for this checklist

Files touched in this slice (already committed locally):

| File | Change |
| --- | --- |
| [`dops-api/app/db/init_db.py`](../dops-api/app/db/init_db.py) | Imports `app.models` package directly so every model (including `driving_center`, `service_offering`) is on `Base.metadata` for `create_all`. |
| [`dops-infra/infra/modules/key-vault.bicep`](../dops-infra/infra/modules/key-vault.bicep) | Added 4 optional B2C secrets (`DOPS-AZURE-B2C-AUTHORITY/CLIENT-ID/AUDIENCE/JWKS-URL`). Always created; empty values mean demo mode. |
| [`dops-infra/infra/main.bicep`](../dops-infra/infra/main.bicep) | Added 7 optional params (4 secure B2C + issuer + 2 redirect URIs); passes them to the keyVault module; references the 4 KV secrets from the API Container App and adds 7 env vars (`AZURE_B2C_*`, `FRONTEND_AUTH_*`). |
| [`dops-infra/infra/parameters/dev.bicepparam`](../dops-infra/infra/parameters/dev.bicepparam) | Reads `DOPS_AZURE_B2C_*` and `DOPS_FRONTEND_AUTH_*` env vars (all default to empty). |
| [`dops-infra/pipelines/azure-pipelines-infra.yml`](../dops-infra/pipelines/azure-pipelines-infra.yml) | Sanitises empty inputs, then forwards `DOPS_AZURE_B2C_*` and `DOPS_FRONTEND_AUTH_*` to all three deployment stages from variable groups `dops-dev` + `dops-dev-secrets`. |

Verified locally with:

- `az bicep build --file infra/main.bicep` → exit 0.
- `cd dops-api && ruff check . && pytest -q` → 23/23 ✓.
- `cd dops-frontend && npm run lint && npm run typecheck && npm run build` → all green.

## Troubleshooting log — issues hit and resolved

### Issue 1 — Container App deploy fails: `Unable to get value … for secret azure-b2c-*`

**Symptom**: `DrivingOps Infra CI/CD → Deploy Dev` fails with messages like:

```
Invalid value: "azure-b2c-audience": Unable to get value using Managed identity
id-dops-api-dev for secret azure-b2c-audience. Error: unable to fetch secret
'azure-b2c-audience' using Managed identity ...
```

**Root cause**: Azure Container Apps refuses to bind a `keyVaultUrl` secret reference whose underlying Key Vault secret value is an empty string. The Key Vault accepted the empty values fine, but the API revision aborts on provisioning when those empty secrets are wired into `secrets[]` and `env.secretRef`. This only happens when B2C hasn't been configured yet (Path A scenario).

**Fix shipped**: [`main.bicep`](../dops-infra/infra/main.bicep) now conditionally includes the four B2C entries in the API Container App's `secrets[]` and `envVars[]` — gated by a new `b2cConfigured` variable that's true only when all four B2C params (`azureB2cAuthority`, `azureB2cClientId`, `azureB2cAudience`, `azureB2cJwksUrl`) are non-empty. When B2C is not yet configured, the API Container App ships without those bindings; `/auth/login-config` returns `provider: "demo"` and the SPA stays on demo login.

When you later populate the four B2C secrets in `dops-dev-secrets` (Path B), `b2cConfigured` flips to true and the next infra deploy adds the bindings + env vars automatically. No code change required.

**To recover**: re-run the `DrivingOps Infra CI/CD` pipeline. The bicep change is committed.

### Issue 2 — Postgres admin password silently reset to `todo` (or stale value) on every infra deploy

**Symptom**: After a successful infra deploy, `psql` / pgAdmin / VS Code authentication fails with `FATAL: password authentication failed for user "dopsadmin"`, even though Key Vault's `POSTGRES-ADMIN-PASSWORD` looks correct.

**Root cause**: Two compounding issues.

1. The pipeline bootstrap reads the password from variable group `dops-dev-secrets`. If `POSTGRES-ADMIN-PASSWORD` is unset/blank, it falls back to the legacy secret name `POSTGRES-PASSWORD`. If that legacy secret holds a stale placeholder (it did — value `todo`), the pipeline uses `todo`.
2. [`postgres.bicep`](../dops-infra/infra/modules/postgres.bicep) unconditionally set `administratorLoginPassword: administratorPassword` on the server resource. ARM reapplies this on every deploy, so the server's password gets clobbered to whatever Bicep was given — `todo` in our case.

The user-visible effect: after a manual password rotation, the next infra deploy silently undoes the rotation.

**Fix shipped**:

- [`postgres.bicep`](../dops-infra/infra/modules/postgres.bicep) now declares `param rotateAdminPassword bool = false` and uses `union()` to include `administratorLoginPassword` in the resource properties **only when the flag is true**. ARM `PUT` with the property absent preserves the existing server password.
- [`main.bicep`](../dops-infra/infra/main.bicep) forwards the flag; `postgresAdminPassword` is now optional (default `''`).
- [`dev.bicepparam`](../dops-infra/infra/parameters/dev.bicepparam) reads `DOPS_ROTATE_ADMIN_PASSWORD` from env (defaults to `false`).
- [`azure-pipelines-infra.yml`](../dops-infra/pipelines/azure-pipelines-infra.yml) sanitises and forwards `DOPS_ROTATE_ADMIN_PASSWORD` to all three deploy stages.

**How to use**:

- **Routine deploys** (almost always): do nothing. Bicep skips `administratorLoginPassword`, the server keeps its current password.
- **First deploy of a new server** or **intentional rotation**: queue the infra pipeline with the runtime variable `DOPS_ROTATE_ADMIN_PASSWORD=true`. Azure DevOps → run pipeline → Variables → Override → add `DOPS_ROTATE_ADMIN_PASSWORD` = `true`. Make sure `POSTGRES-ADMIN-PASSWORD` in `dops-dev-secrets` is the password you want applied.

**Additional one-time hygiene step** (do once, then forget):

- Make sure both Key Vault secrets `POSTGRES-ADMIN-PASSWORD` and `POSTGRES-PASSWORD` hold the SAME value, and that's the value the variable group resolves to. Otherwise the legacy fallback in the bootstrap script could still mislead a future deploy:

```bash
KV=$(az keyvault list --resource-group rg-dops-dev --query "[0].name" -o tsv)
PW=$(az keyvault secret show --vault-name "$KV" --name POSTGRES-ADMIN-PASSWORD --query value -o tsv)
az keyvault secret set --vault-name "$KV" --name POSTGRES-PASSWORD --value "$PW" -o none
```

## Rollback plan

If anything in Path A breaks at the smoke-test stage:

1. **API revision misbehaving** — `az containerapp revision deactivate --revision <new>` then `az containerapp revision activate --revision <previous>`. The previous revision still works against the (now-empty) dev DB because the seed runs fresh on startup of either image.
2. **DB in a weird state** — repeat Step A1 to drop + recreate `dops`; then restart the API revision: `az containerapp revision restart --resource-group rg-dops-dev --name ca-dops-api-dev --revision <active>`.
3. **Bicep deploy failed mid-way** — re-running the pipeline is safe; ARM is idempotent. If a Key Vault secret got created with a bad value (unlikely since we sanitise empties), Azure portal → Key Vault → set the value manually.

## Open items (next slices)

- ☐ Scheduling-page service picker (frontend, ~half day) — when creating a session, pre-populate duration/location/fee from a selected service offering.
- ☐ Instructor edit — slot-gap field (frontend, ~2h).
- ☐ Real Alembic-driven migrations (rather than `create_all`) for non-dev environments — requires adding a Container Apps Job that runs `alembic upgrade head` from the API image, plus a pipeline step that fires it before the revision swap.
- ☐ Booking confirmation flow (worker repo, blocked on Phase 3 automation worker).
- ☐ MFA-on-admin-roles via B2C custom policy.
