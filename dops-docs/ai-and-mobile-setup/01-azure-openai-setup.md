# 01 — Azure OpenAI: Setup & Operations

> **Status**: code shipped, configuration pending. The `AiService`, per-school/per-user token
> metering, monthly quota hard-block, the AI WhatsApp draft-reply feature, and the usage dashboards
> are all implemented and degrade gracefully when Azure OpenAI is unconfigured (`/ai/usage` reports
> `configured: false`; AI actions return 503). This is the step-by-step to turn it on.

## What's already built (no further code needed)

- `AiService` (`dops-api/app/services/ai_service.py`) — wraps Azure OpenAI (gpt-4o-mini), meters every
  call into `ai_usage_events` + `ai_usage_daily`, and enforces a monthly token quota.
- Endpoints: `GET /ai/usage` (school, `ai.usage.read`), `GET /platform/ai/usage` (platform admin).
- First AI feature: `POST /automations/whatsapp/conversations/{id}/draft-reply`.
- Web UI: **AI usage** (`/ai-usage`), **AI usage — all schools** (`/platform/ai-usage`), and the
  **Plans** page where `ai_monthly_tokens` per plan sets each school's quota.
- Infra: `dops-infra/infra/modules/azure-openai.bicep` provisions the account + a `gpt-4o-mini`
  deployment and grants the API's managed identity the **Cognitive Services OpenAI User** role,
  gated behind `deployAzureOpenAI` (default false).

## Decision: managed identity vs API key

Two ways to authenticate the API to Azure OpenAI. **Managed identity is preferred** (no secret to
store or rotate); the API key is a fallback for local dev.

| Mode | When | Config |
| --- | --- | --- |
| **Managed identity** (preferred) | Deployed in Azure | Set `AZURE_OPENAI_ENDPOINT` + `AZURE_OPENAI_DEPLOYMENT`; the Bicep role assignment does the rest |
| **API key** | Local dev / non-Azure host | Also set `AZURE_OPENAI_API_KEY` |

## Option A — Deploy via Bicep (recommended)

1. In the **`dops-dev`** pipeline variable group (these are **non-secret** config flags, so they go
   in `dops-dev`, not `dops-dev-secrets` — `dops-dev-secrets` is for Key-Vault-linked secrets like
   `DOPS-API-SECRET-KEY`), set:
   - `DOPS_DEPLOY_AZURE_OPENAI = true`
   - `DOPS_AZURE_OPENAI_LOCATION = eastus2` (a region where `gpt-4o-mini` is available; often differs
     from your resource-group region)
   - optionally `DOPS_AZURE_OPENAI_DEPLOYMENT = gpt-4o-mini`
   - optionally `DOPS_AI_MODEL_PRICES_JSON =
     {"gpt-4o-mini":{"prompt_per_1k":0.015,"completion_per_1k":0.06}}`
     (USD cents per 1k tokens — used to compute `cost_cents`)

   With the recommended managed-identity path there are **no secrets** in this feature — every AI
   pipeline variable above is non-secret. (Only the optional `AZURE_OPENAI_API_KEY` fallback below is
   a secret, and it's for local dev, not the deployed environment.)
2. Run the infra pipeline. It creates the Azure OpenAI account + `gpt-4o-mini` deployment, assigns
   the API identity **Cognitive Services OpenAI User** on the account, and sets
   `AZURE_OPENAI_ENDPOINT` / `AZURE_OPENAI_DEPLOYMENT` / `AZURE_OPENAI_API_VERSION` /
   `AI_MODEL_PRICES_JSON` on the API container app.
3. Redeploy the API (so it picks up the env vars and the migration `0031_ai_usage` has run).

That's it — no key is stored anywhere. The API authenticates with its managed identity.

## Option B — Manual portal setup

Use this if you are not deploying the Bicep, or you want to point at an existing account.

1. **Create the resource**: Azure Portal → *Create a resource* → **Azure OpenAI** → pick a region
   that offers `gpt-4o-mini` → pricing tier **Standard S0**. Enable a **custom subdomain** (required
   for Entra/token auth on the data plane).
2. **Deploy the model**: open the resource in **Azure AI Foundry / OpenAI Studio** → *Deployments* →
   *Create* → model `gpt-4o-mini` → deployment name `gpt-4o-mini` (or your own; it becomes
   `AZURE_OPENAI_DEPLOYMENT`).
3. **Grant access** (managed identity path): IAM on the account → *Add role assignment* →
   **Cognitive Services OpenAI User** → assign to the API container app's managed identity.
4. **Collect values**: the **Endpoint** (`https://<name>.openai.azure.com/`) and the deployment name.
   For the API-key fallback, copy **Key 1**.
5. **Set API env vars** (Key Vault or container app config):

   ```
   AZURE_OPENAI_ENDPOINT=https://<name>.openai.azure.com/
   AZURE_OPENAI_DEPLOYMENT=gpt-4o-mini
   AZURE_OPENAI_API_VERSION=2024-10-21        # default; override if needed
   # API-key fallback only (omit when using managed identity):
   AZURE_OPENAI_API_KEY=<key1>
   # optional pricing override (USD cents per 1k tokens):
   AI_MODEL_PRICES_JSON={"gpt-4o-mini":{"prompt_per_1k":0.015,"completion_per_1k":0.06}}
   ```

6. Restart the API. `GET /ai/usage` should now report `configured: true`.

## Local dev

In `dops-api/.env`:

```
AZURE_OPENAI_ENDPOINT=https://<name>.openai.azure.com/
AZURE_OPENAI_DEPLOYMENT=gpt-4o-mini
AZURE_OPENAI_API_KEY=<key1>
```

Managed identity won't resolve locally, so use the key for dev.

## Set per-plan AI quotas

The monthly token allowance is a **plan entitlement**, not a global setting:

1. Sign in to the web app as a **platform admin** → **Plans & pricing** (`/platform/plans`).
2. Edit a plan → set **AI tokens / month** (e.g. 250000) → Save.
3. Every school on that plan now has that monthly quota. When month-to-date usage reaches it, AI
   actions return **402 `ai_quota_exceeded`** until the next month or a plan change. (A missing
   entitlement defaults to 500,000 tokens so existing tenants aren't surprised.)

## Verify

1. `GET /ai/usage` → `configured: true`, quota numbers present.
2. In Automations, open a WhatsApp conversation with an inbound message → **Draft reply** → a
   suggestion appears (not auto-sent).
3. `GET /ai/usage` again → token count and the acting user's row increased; `cost_cents` reflects the
   price map.
4. Platform admin → `/platform/ai-usage` shows the school's usage in the cross-tenant view.

## Cost control

- Pricing is configurable without a deploy via `AI_MODEL_PRICES_JSON` (Key Vault).
- Overage is a **hard block** (402), not metered billing — predictable cost. The metered-billing hook
  is left for later.
- gpt-4o-mini is the cheapest capable model; keep `max_tokens` small on new AI features (the
  draft-reply caps at 200).
