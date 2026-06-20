# 07 - Invoices and Payments

Covers the current invoice page, payment checkout entry, and backend-supported billing operations.

> Certificates moved to their own document: see [13 - Certificate Templates, Issuance, and
> Approval](13-certificates-templates.md). The certificate cases below (07-8..07-13) are retained
> only as a short index pointer; use doc 13 for the real, current certificate coverage.

Roles:
- Owner/Admin: invoice/certificate management where UI or API exposes it.
- Billing Support: invoice/payment management and certificate read.
- Receptionist: certificate read.
- Learner: own payment/certificate data only where portal exposes it.

Preconditions:
- At least one learner exists.
- At least one priced service session has been created or invoice seed data exists.
- Payouts are configured if testing real Stripe checkout.

---

## Current Capability Map

| Area | Current state |
|---|---|
| Invoice list | Visible `/invoices` page lists invoices, status, due date, amount, balance, revenue chart, outstanding total. |
| Invoice create | Header "Create invoice" dialog (learner + line items + due date); label i18n bug fixed. |
| Payment checkout | Pay now appears when payments config is enabled and invoice has balance; surfaces the 409 reason on failure. |
| Record payment/refund | API and hooks exist; visible page exposes record-payment in the invoice detail dialog. |
| Auto-invoice | Booking priced firm sessions can create invoices backend-side. |
| Certificates | Moved to doc 13 (configurable templates + review/approve flow). |

---

## 07-1 Invoice List

Route: `/invoices`

Roles: Owner/Admin/Billing

Steps:
1. Open `/invoices`.
2. Inspect metrics, revenue chart, and invoice cards.
3. Confirm each card shows invoice number, learner, due date, balance/amount, and status.

Expected:
- Invoice list loads for permitted roles.
- Empty state is clear if no invoices exist.
- Outstanding total matches invoice balances.
- Page does not show invoice data to unauthorized roles.

---

## 07-2 Auto-Invoice From Booking

Roles: Admin/Scheduler plus Billing for verification

Steps:
1. Ensure service `in_car_60` has a fee.
2. Create a firm scheduled session using that service.
3. Open `/invoices`.
4. Open learner detail if invoice summary is exposed there.

Expected:
- Backend creates or updates invoice data for priced firm bookings where configured.
- Tentative sessions should not invoice until materialized as firm, per backend behavior.
- New invoice appears with learner and balance.

---

## 07-3 Pay Now / Checkout

Roles: Owner/Admin/Billing

Steps:
1. Open an invoice with balance greater than zero.
2. Confirm Pay now visibility.
3. Click Pay now in an environment with payment config enabled.
4. Repeat when Stripe/payouts are not configured.

Expected:
- Pay now appears only when payments config says enabled and balance is positive.
- Checkout redirects only when API returns an HTTP URL.
- If payouts are not connected, the API returns 409 `payouts_not_connected` and the dialog now shows
  the backend's friendly message (it no longer fails silently with only a console error).
- If the invoice has no balance, the API returns 409 `invoice_settled`.
- UI handles disabled/unconfigured payment state without crashing.

---

## 07-4 Create/Edit Invoice - API or Future UI

Roles: Owner/Admin/Billing

Steps:
1. Look for a working create invoice dialog or action on `/invoices`.
2. If no UI exists, test API manually only if in scope.
3. Create an invoice for Liam with line items and due date.
4. Patch invoice status/line data if API testing.

Expected:
- The Invoices header has a working Create invoice dialog (the earlier i18n bug where the button
  rendered an "object instead of string" error is fixed; the label reads "Create invoice").
- Creating an invoice with a learner + at least one priced line calculates totals (and taxes where
  applicable) and the new invoice appears in the list.
- Submitting with no learner or no priced line is blocked with a clear message.

---

## 07-5 Record Payment and Refund - API or Future UI

Roles: Owner/Admin/Billing

Steps:
1. Look for visible record-payment/refund controls.
2. If not visible, test API manually only if in scope.
3. Record a payment against Maya's invoice.
4. Refund a disposable payment.

Expected:
- Backend record payment reduces balance and updates status.
- Backend refund creates credit/refund records and restores balance as implemented.
- Visible `/invoices` page reflects API-created changes after refresh.

---

## 07-6 Overdue Policy

Roles: Billing/Admin

Steps:
1. Seed or create an invoice with due date in the past.
2. Open `/invoices`.

Expected:
- Invoice status or badge reflects overdue if backend marks it.
- Due date uses tenant policy where invoices are generated from policy-aware operations.

---

## 07-7 Learner Payment Visibility

Roles: Learner

Steps:
1. Sign in as Liam.
2. Inspect learner portal for payment/invoice information.
3. Try deep-linking to `/invoices`.

Expected:
- Learner can only see own payment data where portal exposes it.
- Learner cannot access invoice management.
- Deep link to staff invoice page is blocked.

---

## 07-8 Certificates (moved)

Certificate coverage has moved to [13 - Certificate Templates, Issuance, and Approval](13-certificates-templates.md),
which replaces the old eligible/pending/generated columns + placeholder preview with the real flow:
configurable templates, four issuance triggers, a clickable certificates list, a review-and-approve
page showing the learner's course performance, and learner certificate viewing/printing. Use doc 13.

---

## 07-11 Learning-Path Payment Plan - Configure

Roles: Owner/Admin

Steps:
1. Settings -> Learning paths -> open a path with a package price (or service items that sum to one).
2. In the Payment plan section choose a mode: Pay upfront, Monthly, Quarterly, or Every X months.
3. For a non-upfront mode set installments (and interval months for "Every X months"), and an
   optional deposit. Save.
4. Reload the path and confirm the plan persisted.

Expected:
- Upfront keeps the legacy single-invoice behavior (no schedule built).
- A non-upfront plan stores mode/installments/interval/deposit and round-trips on reload.
- Validation: installments >= 1; interval months 1-12; deposit not negative.

---

## 07-12 Payment Plan - Schedule Built On Enrollment

Roles: Owner/Admin

Steps:
1. Assign the plan path (07-11) to a learner (learner detail -> Learning path, or registration).
2. `GET /learners/{id}/payment-schedule` (or the learner statement, doc 04) to inspect installments.

Expected:
- A schedule is created with the expected number of installments; amounts sum to the package total.
- When a deposit is set, the first row is the deposit (billed at enrollment) and the remainder splits
  across the installments; any rounding remainder lands on the last installment.
- Re-assigning the same path does not double-bill (a prior active schedule is superseded).
- Upfront/no-plan paths produce no schedule (endpoint returns null).

---

## 07-13 Payment Plan - Installment Invoicing Job

Roles: Owner/Admin (verify outcomes in UI/API)

Steps:
1. With a learner on a monthly plan, let the daily installment job run (or trigger
   `PaymentPlanService.issue_due_installments` in a dev/ops context with a future "now").
2. Re-read the payment schedule and the learner's invoices.

Expected:
- Installments whose due date is within the lead window flip from `pending` to `invoiced` and carry
  an `invoice_id`; a matching invoice appears for the learner.
- An upcoming-payment notification row is recorded (in-app `payment_due`).
- The job is idempotent: an already-invoiced installment is not re-billed.

---

## 07-14 Missed-Session Penalty - Configure And Auto-Charge

Roles: Owner/Admin for config; Owner/Admin/Billing/assigned instructor for attendance

Steps:
1. Settings -> Services -> open a service. Set a penalty: Fixed amount or Percentage of fee, plus a
   notice window (hours). Save.
2. Create a session for a learner on that service in the past (so attendance can be marked).
3. Mark the session `no_show`.
4. Separately, create two future sessions: cancel one outside the notice window and one inside it.

Expected:
- No-show: a penalty invoice is auto-created on the learner, linked to the session
  (`penaltyInvoiceId` set); fixed = the amount, percent = that % of the base fee.
- Cancellation outside the notice window: no penalty. Inside the window: penalty charged.
- A service with penalty "none" never charges.

---

## 07-15 Missed-Session Penalty - Waive

Roles: Owner/Admin/Billing (`invoices.manage`)

Steps:
1. On a session that has a penalty invoice (07-14), call the waive action
   (`POST /schedule/sessions/{id}/waive-penalty`).

Expected:
- The session is marked `penaltyWaived = true` and the penalty invoice is voided/refunded.
- The penalty is not re-charged afterward.
- A role without `invoices.manage` is blocked (403).

---

## RBAC Negative Checks

### 07-9 Receptionist/Scheduler/Instructor Cannot Access Invoices

Expected:
- No Invoices nav for roles without `invoices.read`.
- Deep link to `/invoices` is blocked or API returns 403.

### 07-10 Learner Cannot Manage Invoices

Expected:
- Learner sees only own payment data where the portal exposes it.
- Deep link to the staff `/invoices` page is blocked.
