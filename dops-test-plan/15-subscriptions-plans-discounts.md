# 15 - Subscriptions: Plans and Discounts

Covers platform-admin management of subscription plans (the offerings sold to schools) and
promotional discounts.

Source of truth:
- `../dops-api/app/services/plan_service.py`, `app/api/v1/endpoints/plans.py`
- `../dops-api/app/services/entitlement_service.py`, `app/api/v1/endpoints/platform.py` (limits + overrides)
- `../dops-api/app/services/tenant_service.py` (`update_subscription` discount apply, `compute_usage`)
- `../dops-frontend/src/features/platform/PlatformPlansPage.tsx`, `PlatformDiscountsPage.tsx`, `PlatformTenantsPage.tsx`
- `../dops-api/app/models/tenant.py` (`Plan`, `Discount`, `TenantSubscription`, `TenantEntitlementOverride`)

Preconditions:
- Sign in as a **Platform admin** (has `platform.plans.manage`, `platform.tenants.manage`).
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

### 15-3a Configure plan limits (enforced caps)

Steps:
1. Edit a plan. In **Limits**, set Max active learners `5`, Max branches `1`, Max instructors `2`.
   Leave a field blank to mean **unlimited**.
2. Save, then assign this plan to a test tenant (see 15-12) and sign in as that school's admin.
3. Create learners until the 6th, create a 2nd branch, create a 3rd instructor.

Expected:
- The plan persists `entitlements.max_learners=5` etc. (snake_case keys; blank fields are absent = unlimited).
- The over-limit create is **hard-blocked** with HTTP 402 `plan_limit_reached`, message naming the
  resource and `current/limit`. The UI surfaces that message inline.
- Only **active** learners count toward the learner cap (archived/withdrawn/completed do not).

### 15-3b Plan feature flags

Steps:
1. Edit a plan and uncheck **WhatsApp messaging** (or Public site / Courses).
2. Save and assign to a tenant.

Expected:
- `entitlements.whatsapp_enabled=false` persists. An **absent** flag = enabled (backward compatible).
- (Enforcement of feature flags is via `EntitlementService.enforce_feature`; verify the flag value
  is stored and reported in the tenant's plan-usage panel.)

### 15-3c Complimentary (internal) plan

Steps:
1. Create a plan with **Complimentary (internal) plan** checked.
2. Note it shows a "Complimentary" badge in the plans list.
3. Attempt a self-serve school signup (`POST /onboarding/register-school`) with that plan's code.

Expected:
- `isComplimentary=true` persists and the badge shows.
- Self-serve signup with a complimentary plan code is rejected **403** `plan_not_self_serviceable`.
- A platform admin **can** assign it to a tenant via the tenant manage dialog (15-12).

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

## Per-tenant plan, discounts, usage & overrides

Route: `/platform/tenants` → **Manage** a tenant. Source: `PlatformTenantsPage.tsx`,
`platform.py` (`/platform/tenants/{id}/subscription`, `/entitlements`, `/overrides`).

### 15-12 Assign a plan to a tenant (dropdown, not raw id)

Steps:
1. Manage a tenant → Subscription → **Plan** is a dropdown of real plans (name + code), not a
   free-text UUID box.
2. Pick a plan and Save subscription.

Expected:
- The subscription's `plan_id` updates; the dropdown preserves any unrecognised/legacy id.
- Create-tenant likewise offers a **plan dropdown**, defaulting to the first plan; it shows
  "No plans available — create one first" when none exist.

### 15-13 Apply a discount to a subscription

Steps:
1. In the tenant's Subscription panel, choose a discount from the **Discount** dropdown and Save.
2. Open the tenant's **usage** (doc 11 / `/tenants/{id}/usage`).

Expected:
- `subscription.discount_id` is set. Usage shows `monthly_subtotal_cents`, `discount_cents`,
  `monthly_total_cents` with the discount applied (percent = bps of subtotal; fixed = cents off).
- Selecting "No discount" + Save clears it (`clearDiscount`).
- Applying an **inactive / expired / wrong-plan / exhausted** discount is rejected 400 with a
  specific code (`discount_inactive` / `discount_expired` / `discount_plan_mismatch` /
  `discount_exhausted`). An expired discount left attached simply stops applying at billing time.

### 15-14 Plan-usage panel (tenant-wide + per branch)

Steps:
1. In the tenant manage dialog, view **Plan usage**.

Expected:
- Tenant-wide cards show current/limit for branches, instructors, active learners (red at cap),
  with `extra` from any active overrides folded into the limit.
- A **Per branch** breakdown appears **only** when the plan sets a per-branch cap
  (`max_instructors_per_branch` / `max_active_learners_per_branch`), listing each branch's
  current/limit.

### 15-15 Grant ad-hoc quota override (with period)

Steps:
1. In **Extra quota (overrides)**, choose a resource (e.g. Active learners), an amount (e.g. `5`),
   optional Start/End dates, an optional note. Click **Grant extra quota**.
2. As that school's admin, create records up to the new (plan + override) ceiling.

Expected:
- A new override row appears with its `+amount resource` and `start → end` period.
- The effective limit = **plan limit + sum of currently-active overrides**. An override only raises
  a **finite** plan limit; it never imposes a cap where the plan grants unlimited.
- For **AI tokens**, the monthly limit becomes plan + override while the window is active
  (verify in `/ai-usage`, doc 14).
- Invalid grants are rejected 400: unknown resource (`invalid_override_resource`), amount ≤ 0
  (`invalid_override_amount`), end-before-start (`invalid_override_period`).

### 15-16 Override period boundaries

Steps:
1. Grant an override with an **end date in the past** (or a start in the future).
2. View the plan-usage panel.

Expected:
- An out-of-window override contributes **0** extra (limit unchanged, `extra=0`).
- Deleting/revoking an override immediately lowers the effective limit back.

### 15-17 Overrides permission gating

Steps:
1. As a school admin (no `platform.tenants.manage`), `POST /platform/tenants/{id}/overrides`.

Expected:
- 403; override controls are not shown to non-platform roles.

---

## Notes

- Stripe coupon mirroring (pushing a Discount onto the recurring Stripe Billing price) is **not yet
  wired** — discounts are modeled, CRUD-able, applied to a subscription, and reflected in the
  computed `monthly_total_cents`, but do not yet alter the Stripe-charged amount. Verify the
  platform-side modeling + computed usage here.
- Entitlement limits are **absent = unlimited** and feature flags **absent = enabled**, so plans
  that predate these keys keep working unchanged (no tenant is suddenly blocked).
- New deployed-DB columns (`plans.is_complimentary`) + table (`tenant_entitlement_overrides`) ship
  Alembic migrations `0034`/`0035`; run `alembic upgrade head` on Postgres before testing there.
