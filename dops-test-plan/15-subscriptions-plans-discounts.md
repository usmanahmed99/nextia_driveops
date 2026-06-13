# 15 - Subscriptions: Plans and Discounts

Covers platform-admin management of subscription plans (the offerings sold to schools) and
promotional discounts.

Source of truth:
- `../dops-api/app/services/plan_service.py`, `app/api/v1/endpoints/plans.py`
- `../dops-frontend/src/features/platform/PlatformPlansPage.tsx`, `PlatformDiscountsPage.tsx`
- `../dops-api/app/models/tenant.py` (`Plan`, `Discount`, `TenantSubscription`)

Preconditions:
- Sign in as a **Platform admin** (has `platform.plans.manage`).
- At least one tenant exists so the in-use guards can be exercised.

---

## Plans

Route: `/platform/plans`

### 15-1 Create a plan

Steps:
1. Open Plans & pricing → New plan.
2. Enter code `Growth Plan`, name `Growth`, active learner rate `1500` cents, currency `cad`,
   AI tokens/month `1000000`.
3. Save.

Expected:
- The plan appears with code normalized to `growth-plan` and currency upper-cased to `CAD`.
- The card shows the rate line and the AI tokens/month line.

### 15-2 Duplicate code rejected

Steps:
1. Create another plan with code `growth-plan`.

Expected:
- 409 conflict; the UI surfaces "a plan with this code already exists".

### 15-3 Edit a plan's AI quota (closes the loop with metering)

Steps:
1. Edit the plan; change AI tokens/month to `250000`.
2. Save, then check a tenant on this plan in `/ai-usage` (doc 14).

Expected:
- The plan's `entitlements.ai_monthly_tokens` updates.
- That tenant's monthly AI quota limit reflects the new value.

### 15-4 Delete guard (plan in use)

Steps:
1. Ensure a tenant's subscription references the plan.
2. Attempt to delete the plan.

Expected:
- 409 "move schools off this plan before deleting it"; the plan remains.
- A plan with no subscriptions deletes successfully (204).

### 15-5 Permission gating

Steps:
1. Sign in as a school admin (no `platform.plans.manage`).
2. Attempt `POST /platform/plans`.

Expected:
- 403; the Plans nav item is not visible to non-platform roles.

---

## Discounts

Route: `/platform/discounts`

### 15-6 Create a percentage discount

Steps:
1. Open Discounts → New discount.
2. Code `WELCOME20`, name `Welcome 20%`, type Percentage, value `2000` (basis points).
3. Save.

Expected:
- Created with code normalized to `welcome20`; the card reads `20%`.

### 15-7 Create a fixed discount

Steps:
1. New discount, type Fixed amount, value `500` (cents), currency CAD.

Expected:
- The card reads `$5.00 CAD`.

### 15-8 Validation

Steps:
1. Try a Percentage discount with value `20000` (200%).
2. Try a discount with an unsupported type.

Expected:
- Both return 400 (percent cannot exceed 100% / 10000 bps; invalid kind).

### 15-9 Edit and archive

Steps:
1. Edit a discount and set status to `archived`.

Expected:
- The status badge updates to archived; the discount stays listed.

### 15-10 Delete guard (discount applied)

Steps:
1. If a subscription references the discount (`discount_id`), attempt to delete it.

Expected:
- 409 "remove this discount from subscriptions before deleting it".
- An unused discount deletes successfully (204).

---

## Notes

- Stripe coupon mirroring (pushing a Discount onto the recurring Stripe Billing price) is **not yet
  wired** — discounts are modeled and CRUD-able but do not yet alter the Stripe-charged amount.
  Verify only the platform-side modeling here.
