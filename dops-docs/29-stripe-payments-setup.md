# 29 — Stripe Payments: Setup & Operations Guide

> **Status**: code shipped, configuration pending. The payment service, webhooks,
> refunds, credit notes, auto-invoicing on booking, the learner **Pay now** flow,
> **multi-tenant Stripe Connect (Express) payouts**, and **SaaS subscription
> billing** are all implemented and degrade gracefully when Stripe is unconfigured.
> This is the step-by-step to turn it on. See also
> [09-invoices-payments.md](./09-invoices-payments.md).

## The two money flows (multi-tenant)

DrivingOps is a multi-tenant platform with independent schools, so there are
**two** distinct Stripe relationships:

| Flow | Stripe product | Account that holds the money | Endpoints |
| --- | --- | --- | --- |
| **Learners → school** | **Connect (Express)** | The **school's** connected account | `/billing/payouts/*`, `/invoices/{id}/checkout-session` |
| **School → platform (SaaS)** | **Billing** (subscriptions) | The **platform's** account | `/billing/subscription/*` |

You operate **one platform Stripe account**. Each school onboards as an **Express
connected account** under it (Stripe-hosted KYC + dashboard), and learner payments
land in that school's balance — never pooled. Schools separately subscribe to a
DrivingOps plan, which charges them on your platform account.

**No platform fee is taken on learner payments today** (schools keep 100%); a
configurable hook (`STRIPE_PLATFORM_FEE_BPS`, basis points) lets you switch one on
later without code changes.

## What's already built (no further code needed)

| Piece | Where |
| --- | --- |
| Stripe wrapper (Checkout, webhook verify, refund, Connect, Billing) | `dops-api/app/services/stripe_service.py` |
| Connect + subscription orchestration | `dops-api/app/services/tenant_billing_service.py` |
| Pay / refund / config endpoints | `dops-api/app/api/v1/endpoints/invoices.py` |
| Payout onboarding + subscription endpoints | `dops-api/app/api/v1/endpoints/billing.py` |
| Signed webhook ingress (payments + Connect + subscriptions) | `dops-api/app/api/v1/endpoints/webhooks.py` → `POST /api/v1/webhooks/stripe` |
| Payment PSP columns + `credit_notes` table | `dops-api/app/models/invoice.py`, migration `0014_stripe_payments` |
| Tenant Connect/customer/subscription columns | `dops-api/app/models/tenant.py`, migration `0015_tenant_stripe_connect` |
| Auto-invoice on session booking | `dops-api/app/services/scheduling_service.py` (`_autoinvoice_session`) |
| Learner "Pay now" CTA + admin refund hooks | `dops-frontend` `InvoicesPage` |
| Settings → Payouts (Connect onboarding) | `dops-frontend` `PayoutsPage` (`/settings/payouts`) |
| Settings → Billing (subscribe + portal) | `dops-frontend` `BillingUsagePage` (`/settings/billing`) |

**Graceful degradation:** with no `STRIPE_SECRET_KEY`, `/invoices/payments/config`
returns `enabled: false`, the Pay-now button is hidden, and the manual
`record-payment` ledger path keeps working. Nothing breaks before you configure it.

## Design at a glance

We use **Stripe Checkout (hosted)** rather than in-app Elements: the learner is
redirected to Stripe's PCI-compliant page, pays, and is sent back. We never touch
card data. Reconciliation is webhook-driven so a closed browser tab still records
the payment.

```
Learner clicks "Pay now"
  → POST /invoices/{id}/checkout-session   (Checkout Session created ON the school's
                                            connected account; invoice_id in metadata)
  → 302 to Stripe hosted page → learner pays → funds land in the SCHOOL's balance
  → Stripe → POST /webhooks/stripe  (signed; checkout.session.completed)
       → verify signature → record Payment(method=card, psp_*), flip invoice to paid
  → learner returned to {FRONTEND_BASE_URL}/learner/invoices?paid={id}
```

The Checkout call is rejected with 409 `payouts_not_connected` until the school has
finished Connect onboarding (`charges_enabled`).

Refunds: `POST /invoices/payments/{payment_id}/refund` → Stripe refund → writes a
`credit_notes` row and restores the invoice balance. Manual (cash/Interac)
payments refund as a ledger credit note only (no Stripe call).

Auto-invoicing: booking a **priced** session (a service offering with
`base_fee_cents`, with a learner attached) issues an invoice automatically —
`status=issued`, tax at QC 14.975%, due in `invoicePaymentDueDays`. Tentative
(checkpoint-gated) bookings are **not** invoiced until they activate, so a session
that auto-cancels never bills.

## Step 1 — Create the Stripe account & keys

1. Sign up / sign in at <https://dashboard.stripe.com>. Set the account country to
   **Canada** and default currency **CAD**.
2. (Recommended) Stay in **Test mode** first — the toggle is top-right.
3. **Developers → API keys**. Copy:
   - **Publishable key** (`pk_test_…` / `pk_live_…`) — safe for the browser.
   - **Secret key** (`sk_test_…` / `sk_live_…`) — server only, treat as a password.
4. Enable card payments (on by default). For **Interac** in Canada, enable it under
   **Settings → Payment methods** (Checkout will offer it automatically when
   eligible).

## Step 2 — Configure the webhook

1. **Developers → Webhooks → Add endpoint**.
2. Endpoint URL: `https://<your-api-host>/api/v1/webhooks/stripe`
   (e.g. `https://ca-dops-api-dev.<region>.azurecontainerapps.io/api/v1/webhooks/stripe`).
3. Events to send:
   - `checkout.session.completed` — learner payments **and** subscription starts.
   - `account.updated` — Connect payout-onboarding status changes.
   - `customer.subscription.created` / `updated` / `deleted` — SaaS lifecycle.
4. **Enable "Listen to events on Connected accounts"** on the endpoint so
   `account.updated` and connected-account `checkout.session.completed` events reach
   you. (Stripe Dashboard → the endpoint → toggle Connect.)
5. After creating it, reveal the **Signing secret** (`whsec_…`). This is
   `STRIPE_WEBHOOK_SECRET` — the webhook **rejects unsigned/forged calls** without it.

> **Local testing:** use the Stripe CLI —
> `stripe listen --forward-to localhost:8000/api/v1/webhooks/stripe`
> prints a temporary `whsec_…`; export it as `STRIPE_WEBHOOK_SECRET` and run
> `stripe trigger checkout.session.completed`.

## Step 3 — Environment variables

The API reads these (see `dops-api/app/core/config.py`):

| Var | Required | Example |
| --- | --- | --- |
| `STRIPE_SECRET_KEY` | yes (enables everything) | `sk_test_…` |
| `STRIPE_PUBLISHABLE_KEY` | recommended | `pk_test_…` |
| `STRIPE_WEBHOOK_SECRET` | yes (for the webhook) | `whsec_…` |
| `STRIPE_CURRENCY` | no (default `cad`) | `cad` |
| `STRIPE_PLATFORM_FEE_BPS` | no (default `0` = no fee) | `250` (= 2.5%) |
| `STRIPE_BILLING_PRICES_JSON` | for SaaS subscriptions | `{"starter":"price_123","pro":"price_456"}` |
| `FRONTEND_BASE_URL` | yes (return URLs) | `https://app.drivingops.ca` |

**Local dev (`dops-api/.env`):**
```dotenv
STRIPE_SECRET_KEY=sk_test_xxx
STRIPE_PUBLISHABLE_KEY=pk_test_xxx
STRIPE_WEBHOOK_SECRET=whsec_xxx
STRIPE_CURRENCY=cad
STRIPE_PLATFORM_FEE_BPS=0
STRIPE_BILLING_PRICES_JSON={"starter":"price_xxx"}
FRONTEND_BASE_URL=http://localhost:5173
```

Install the SDK (already in `requirements.txt`): `pip install -r requirements.txt`.

## Step 4 — Deploy config (Azure)

Keep secrets in **Key Vault**, mirroring the ACS / worker-key pattern in
`dops-infra/infra/main.bicep`. For each of `STRIPE_SECRET_KEY` and
`STRIPE_WEBHOOK_SECRET`:

1. Add a Key Vault secret (e.g. `dops-stripe-secret-key`, `dops-stripe-webhook-secret`)
   in `modules/key-vault.bicep`, exposing a `*SecretName` output (copy the
   `acsConnectionStringSecretName` lines).
2. In `main.bicep`, add a Container App `secrets[]` entry with a `keyVaultUrl`
   pointing at each secret (copy the `acs-connection-string` block ~line 337), and
   an `env[]` entry mapping `STRIPE_SECRET_KEY` → `secretRef: 'stripe-secret-key'`
   (copy the `ACS_CONNECTION_STRING` block ~line 436).
3. `STRIPE_PUBLISHABLE_KEY` and `STRIPE_CURRENCY` are not secrets — add them as plain
   `env[]` values (or as params).
4. Populate the secret values via the same path you use for
   `DOPS-ENTRA-EXTERNAL-ID-*` / `DOPS-WORKER-API-KEY` (the `dops-dev-secrets`
   pipeline variable group).
5. `az bicep build --file infra/main.bicep --outfile $env:TEMP\proof.json` for each
   environment file before committing (per workspace CLAUDE.md).

> The publishable key reaches the SPA at runtime via `GET /invoices/payments/config`
> — no frontend rebuild needed when you flip Stripe on.

## Step 4b — Enable Connect (multi-tenant payouts)

1. In the Stripe Dashboard, go to **Connect → Get started** and enable it for the
   **platform** account. Choose the **platform/marketplace** model.
2. Set your platform business details and the Connect branding (logo/colour shown on
   the schools' Express onboarding screens).
3. No per-school keys are needed — the API creates each school's Express account on
   demand (`POST /billing/payouts/onboarding-link`) and stores the `acct_…` id on the
   tenant. The school finishes KYC + bank details on Stripe's hosted page, then
   `account.updated` flips `charges_enabled` so learner Pay-now turns on.
4. Optional platform fee: set `STRIPE_PLATFORM_FEE_BPS` (e.g. `250` = 2.5%); it's
   added as `application_fee_amount` on each learner payment and swept to the platform.

In the app: **Settings → Payouts** drives this. A school owner clicks **Connect
payouts** → Stripe onboarding → returns to `/settings/payouts?connected=1`, which
re-syncs status. Until charges are enabled, the learner **Pay now** button returns
409 `payouts_not_connected`.

## Step 4c — Set up SaaS subscription plans (schools pay you)

1. In the Stripe Dashboard (**platform** account), **Product catalog → Add product**
   for each DrivingOps plan (Starter, Pro, …) with a **recurring** Price.
2. Copy each Price id (`price_…`) and map your plan codes:
   `STRIPE_BILLING_PRICES_JSON={"starter":"price_123","pro":"price_456"}`.
3. (Optional) Enable the **Customer Billing Portal** in Stripe settings so schools
   can self-serve plan/payment changes — the API mints portal links.

In the app: **Settings → Billing** shows usage and a **Subscribe / change plan** +
**Manage subscription** action. Checkout runs on your platform account; the webhook
(`checkout.session.completed` subscription-mode + `customer.subscription.*`) keeps
`TenantSubscription.status` and the tenant status in sync.

## Step 5 — Verify

1. Confirm config is live: `GET /api/v1/invoices/payments/config` → `{"enabled": true, "publishableKey": "pk_…"}`.
2. **Connect a school's payouts:** Settings → Payouts → **Connect payouts** → complete
   Stripe's test onboarding (use Stripe's test KYC values) → return → status shows
   "connected". `GET /api/v1/billing/payouts/status` → `chargesEnabled: true`.
3. Book a priced session for a learner → an invoice appears (`status: issued`).
4. Open **Invoices** → an open invoice now shows **Pay now**. Click it → Stripe
   Checkout opens **on the school's connected account**. Pay with test card
   **4242 4242 4242 4242**, any future expiry, any CVC.
5. Webhook flips the invoice to **paid** within seconds; the funds appear in the
   *school's* Stripe balance (check `stripe.payment.recorded` + the connected
   account's Events).
6. From the admin, refund the payment → invoice balance restored, a `credit_notes`
   row written, `payment.refunded` logged (refund issued on the connected account).
7. **Subscription:** Settings → Billing → **Subscribe** → pay on Stripe → the webhook
   sets `TenantSubscription.status = active` and the tenant becomes `active`.

## Going live

1. Toggle the Stripe dashboard to **Live mode**, regenerate `pk_live_…` / `sk_live_…`.
2. Re-create the webhook against the production API host; capture the live `whsec_…`.
3. Update the Key Vault secrets and redeploy.
4. Complete Stripe's account activation (business details, bank account for payouts).

## Notes & gotchas

- **Idempotency:** the webhook dedupes by `psp_payment_intent_id`, so Stripe
  re-deliveries don't double-record. Stripe expects a `2xx` quickly — the handler
  is intentionally light.
- **Currency is in cents.** All amounts already flow as `Money.amount_cents`; do not
  divide before handing to Stripe.
- **Tax:** auto-invoicing applies QC 14.975% (GST 5% + QST 9.975%). For other
  provinces, override in the invoice items / a future Tax settings page
  ([09-invoices-payments.md](./09-invoices-payments.md) §C).
- **Partial payments:** a Checkout pays the full outstanding balance. Partial/manual
  payments still go through `record-payment`.
- **Permissions:** creating a checkout session needs `invoices.read` (a learner
  paying their own invoice); refunds need `invoices.manage`.
```
