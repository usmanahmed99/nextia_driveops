# 09 — Invoices & Payments

## Purpose

Issue invoices for program enrollment, lesson packages, road-test prep, and ad-hoc charges. Track balance. Record payments. Send dunning. Reconcile.

## Current State

### Data model — [`dops-api/app/models/invoice.py`](../dops-api/app/models/invoice.py)

```python
class Invoice:        learner_id, number, status, items, subtotal, tax, total, balance, due_at
class Payment:        invoice_id, amount, method, status, paid_at
```

Money fields are stored as `JSON` blobs `{ amount_cents, currency }`. Items are an unconstrained JSON list.

### API — [`dops-api/app/api/v1/endpoints/invoices.py`](../dops-api/app/api/v1/endpoints/invoices.py)

| Method | Path | Perm |
| ------ | ---- | ---- |
| GET | `/invoices` | `invoices.read` |
| POST | `/invoices` | `invoices.manage` |
| GET | `/invoices/{id}` | `invoices.read` |
| PATCH | `/invoices/{id}` | `invoices.manage` |
| POST | `/invoices/{id}/record-payment` | `invoices.manage` |

### UX

- `InvoicesPage.tsx` — list with statuses, balance display, manual "record payment" button.

## Gaps For Real Implementation

1. **No payment processor**. Recording is a manual ledger entry; no card/Interac/cash-handling.
2. **No tax model**. `tax` is a JSON blob; Canadian QC needs GST (5%) + QST (9.975%) with split reporting.
3. **No PDF rendering**. The frontend offers a "preview" placeholder; no real PDF is generated or stored.
4. **No dunning flow** wired to automations.
5. **No refund / credit-note** model.
6. **No customer accounts / receipts portal** for learners.
7. **No invoice number sequence** per tenant.
8. **No reconciliation** between bank statement / PSP payout and `Payment` rows.
9. **Money in JSON** prevents SQL-side sums for reporting.

## Real Implementation Plan

### A. Schema cleanup

```sql
invoices (
  id pk, tenant_id, learner_id,
  number,                                -- generated per-tenant: 'DOPS-{tenant_seq}-{year}-{n}'
  status,                                -- draft | issued | partially_paid | paid | void | refunded
  issued_at, due_at, paid_at null, voided_at null,
  subtotal_cents, gst_cents, qst_cents, total_cents, balance_cents, currency,
  notes, footer,
  created_at, updated_at
)

invoice_items (
  id pk, tenant_id, invoice_id, ordinal,
  description, description_fr,
  quantity numeric, unit_amount_cents,
  tax_code,                              -- enum: standard | exempt
  link_program_id null, link_session_id null
)

payments (
  id pk, tenant_id, invoice_id,
  amount_cents, currency,
  method,                                -- card | interac_etransfer | cash | other
  status,                                -- pending | succeeded | failed | refunded
  paid_at null,
  psp_provider null, psp_payment_id null, psp_payout_id null,
  recorded_by user_id,
  created_at
)

credit_notes (
  id pk, tenant_id, invoice_id null, learner_id,
  amount_cents, currency, reason,
  issued_at, applied_invoice_id null
)

tenant_invoice_sequences (tenant_id pk, current_value bigint)
```

### B. Payment processor integration

Pick **Stripe (Canada region)** for cards + Interac. For cash recording, keep the manual path.

- Create a `payment_intents` flow:
  - `POST /invoices/{id}/payment-intents` → creates a Stripe PaymentIntent, returns `client_secret` for SCA-3DS confirmation in the learner portal.
  - Stripe webhook → `POST /webhooks/stripe` (signature verified) → upserts `payments` row, transitions invoice.
- Manual cash: existing `record-payment` endpoint stays.
- Refunds: `POST /payments/{id}/refund` → Stripe refund → updates state.

### C. Tax handling

Canada QC defaults:

- GST 5% (federal), QST 9.975% (provincial).
- `invoice_items.tax_code = 'standard'` → both apply. `'exempt'` → neither.
- Settings → **Tax** to override defaults per tenant (other provinces have HST blends).

### D. PDF generation

- Render via `weasyprint` (server-side HTML → PDF) inside the API container.
- Store PDF in Azure Storage `certificates` … no, we add a `documents` container scope already exists — use `documents/invoices/{tenant_id}/{invoice_id}.pdf`.
- `GET /invoices/{id}/pdf` returns a short-lived SAS URL. Regenerate on amendment.

### E. Dunning

Tie into [08-automations.md](./08-automations.md):

- Cron at 02:00 local: scan `invoices WHERE status in ('issued','partially_paid') AND due_at < now`.
- Emit `invoice.overdue` events with `days_overdue` bucket: 1, 3, 7, 14.
- `payment_followup` automation flow handles each bucket with escalating templates and (configurably) auto-creating a `learner_holds(type='payment')` row at day 14 — see [04-learners.md](./04-learners.md).

### F. Learner portal payments

- "Payments" card surfaces outstanding invoice with **Pay now** CTA → Stripe Elements in portal.
- Receipts visible after success.

### G. Reporting

- Once money is in columns, the Reports page revenue chart becomes a real SQL aggregate by month, branch, program ([14-reports-dashboard.md](./14-reports-dashboard.md)).

## Acceptance Criteria

- Issuing an invoice generates a sequence number and renders a PDF.
- Learner pays in portal via card → PSP webhook flips status to `paid` and creates `payments` row within 30 s.
- Cash payment recorded in admin UI shows on the next reload, with audit log.
- Refund from admin reduces balance and notifies learner.
- Overdue scanner emits events; dunning flow sends bilingual reminders.
- All amount aggregates in reports use `*_cents` integer columns; no JSON arithmetic.

## Dependencies

- [04-learners.md](./04-learners.md) — payment hold + learner enrolment relation.
- [08-automations.md](./08-automations.md) — dunning + receipt emails.
- [07-school-schedule-templates.md](./07-school-schedule-templates.md) — cancellation policy late fees.
- [16-infrastructure.md](./16-infrastructure.md) — Stripe webhooks, secrets, public webhook ingress.
