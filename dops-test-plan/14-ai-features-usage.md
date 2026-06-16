# 14 - AI Features and Token Usage

Covers the Azure OpenAI–backed AI features and the per-school / per-user token metering.

Source of truth:
- `../dops-api/app/services/ai_service.py`, `app/api/v1/endpoints/ai.py`, `app/api/v1/endpoints/automations.py`
- `../dops-frontend/src/features/platform/AiUsagePage.tsx`, `PlatformAiUsagePage.tsx`
- `../dops-api/app/models/ai_usage.py`

Preconditions:
- A tenant exists with at least one staff user holding `ai.usage.read` (owner/admin/billing roles do).
- A WhatsApp conversation with at least one inbound message exists (doc 08) for the draft-reply test.
- **AI must be configured for the live-call tests** (Azure OpenAI env set — see
  `../docs/ai-and-mobile-setup/01-azure-openai-setup.md`). When AI is OFF, the feature surfaces a
  503 / "not configured" and the usage dashboard reports `configured: false` — those degraded states
  are themselves test cases below.

---

## AI Usage Dashboard (school)

Route: `/ai-usage`

Role: any staff role with `ai.usage.read`

### 14-1 Usage dashboard with AI disabled

Steps:
1. With Azure OpenAI **not** configured on the API, sign in as a staff user.
2. Open `/ai-usage`.

Expected:
- The page loads with a banner / note that AI features are not enabled.
- Token, cost, request, and remaining cards render (zeros are acceptable).
- No error toast; the screen degrades gracefully.

### 14-2 Usage dashboard with AI enabled and usage present

Steps:
1. With AI configured, generate at least one AI draft (test 14-5).
2. Open `/ai-usage` and pull-to-refresh / reload.

Expected:
- "Tokens this month", "Estimated cost", "Requests", and "Remaining" reflect real numbers.
- The monthly quota bar fills proportionally; it turns amber ≥ 80% and red ≥ 100%.
- "Usage by user" lists the acting user with a non-zero token count.

### 14-3 Permission gating

Steps:
1. Sign in as a role **without** `ai.usage.read` (e.g. a custom role with it unchecked).
2. Attempt to open `/ai-usage`.

Expected:
- The nav item is hidden and the route is not accessible.
- A direct API call to `GET /ai/usage` returns 403.

---

## Platform AI Usage (cross-tenant)

Route: `/platform/ai-usage`

Role: Platform admin

### 14-4 Platform-wide AI usage

Steps:
1. Sign in as a platform admin.
2. Open `/platform/ai-usage`.

Expected:
- Totals aggregate across all schools for the current month.
- "Usage by school" lists tenants with a per-school percentage-of-quota badge.
- A non-platform-admin gets 403 on `GET /platform/ai/usage`.

---

## AI WhatsApp Draft Reply

Surface: Automations → WhatsApp conversation → draft reply (endpoint
`POST /automations/whatsapp/conversations/{id}/draft-reply`)

Role: staff with `automations.manage`

### 14-5 Draft a reply (AI enabled)

Steps:
1. Open a WhatsApp conversation that has an inbound learner message.
2. Trigger "Draft reply" (optionally set a tone).

Expected:
- A suggested reply returns and is shown for review — it is **not** auto-sent.
- The reply is in the learner's language and is concise.
- A new `AiUsageEvent` is recorded (verify via `/ai-usage` count increasing) for feature
  `whatsapp_draft`, attributed to the acting user.

### 14-6 Draft reply with AI disabled

Steps:
1. With AI not configured, trigger "Draft reply".

Expected:
- The request returns 503 with an "AI not enabled" message; no usage is recorded.

### 14-7 Quota hard-block

Steps:
1. As a platform admin, set the tenant's plan `ai_monthly_tokens` entitlement to a very low value
   (e.g. 10) via Plans (doc 15).
2. Consume the quota (one or two drafts).
3. Trigger another draft.

Expected:
- Once month-to-date tokens ≥ the limit, the draft request returns 402 `ai_quota_exceeded`
  with a friendly upgrade message.
- `/ai-usage` shows remaining tokens at 0 and the quota bar red.
- Raising the entitlement restores access on the next attempt.

### 14-7a Ad-hoc AI token override extends the monthly limit

Steps:
1. With the tenant at/near its AI limit, grant an **ad-hoc override** for `ai_monthly_tokens`
   (e.g. +500) via the tenant manage dialog → Extra quota (doc 15, 15-15), optionally for a period.
2. Trigger another draft.

Expected:
- The effective monthly limit becomes plan `ai_monthly_tokens` **+ override** while the override
  window is active; draft succeeds and `/ai-usage` reflects the higher ceiling.
- An out-of-window (expired/future) override does not raise the limit. Revoking it restores the
  plan-only limit.

---

## Metering integrity

### 14-8 Per-user and per-school attribution

Steps:
1. Have two different staff users each generate a draft.
2. Open `/ai-usage`.

Expected:
- The tenant total equals the sum of both users' tokens.
- Each user appears in "Usage by user" with their own subtotal.
- The platform view's school total matches the school's tenant total.
