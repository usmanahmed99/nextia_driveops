# 10 — Certificates

## Purpose

Generate, approve, and issue official program-completion certificates (e.g. Quebec Class 5 Complete Program). Track approval, revocation, downloads.

## Current State

### Data model — [`dops-api/app/models/certificate.py`](../dops-api/app/models/certificate.py)

```python
class Certificate:
    certificate_number, learner_id, program, status,
    generated_at, approved_by, issued_at, download_url
```

`status` is a free string (concept uses values like `pending_approval`, `generated`).

### API — [`dops-api/app/api/v1/endpoints/certificates.py`](../dops-api/app/api/v1/endpoints/certificates.py)

| Method | Path | Perm |
| ------ | ---- | ---- |
| GET | `/certificates` | `certificates.read` |
| POST | `/certificates/generate` | `certificates.approve` |
| GET | `/certificates/{id}` | `certificates.read` |
| POST | `/certificates/{id}/approve` | `certificates.approve` |
| POST | `/certificates/{id}/void` | `certificates.approve` |

### UX

- `CertificatesPage.tsx` (verify) — list with approval CTAs and download placeholder.

## Gaps For Real Implementation

1. **No PDF rendering**. `download_url` is a fixed placeholder string.
2. **No storage**. Storage account + `certificates` container exist; nothing is written.
3. **No eligibility check**. Generation should require all program items completed and all evaluations passed.
4. **No certificate number sequence**.
5. **Approval is one-step**. Real flow: generate (draft) → reviewed by school owner → approved → emailed → issued.
6. **No signed download URL** (use SAS).
7. **No revocation log** beyond a status flag.
8. **No QR / verification page** for third parties (SAAQ inspectors).
9. **No bilingual template**.

## Real Implementation Plan

### A. Schema additions

```sql
certificates
  ADD COLUMN tenant_id, program_id fk,
  ADD COLUMN status enum,            -- draft | pending_approval | approved | issued | void | revoked
  ADD COLUMN generated_by user_id,
  ADD COLUMN revoked_by user_id null, revoked_at null, revocation_reason null,
  ADD COLUMN pdf_blob_path null, pdf_sha256 null,
  ADD COLUMN qr_token unique,        -- random URL-safe token for the public verification page
  ADD COLUMN issued_to_email, issued_to_phone

certificate_number_sequences (tenant_id pk, current_value bigint)

certificate_audit_events (
  id pk, tenant_id, certificate_id,
  event,                              -- generated | approved | issued | downloaded | revoked
  actor user_id null, occurred_at, payload jsonb
)
```

### B. Endpoint additions

```
POST /certificates/{id}/issue           certificates.approve   -- after approve, emails the learner with download link
GET  /certificates/{id}/download        certificates.read      -- redirects to SAS URL (logged event)
GET  /verify/{qr_token}                 public                 -- public page: name, program, issued_at, valid/revoked
```

### C. Eligibility check (server-side)

`POST /certificates/generate` body: `{ learner_id }`. Service must verify:

- learner.enrolled_program_id present,
- every non-optional `program_items` for that program has a `scheduled_session.status='completed'` (or `evaluation_passed=true` for evaluations),
- no open `learner_holds`,
- no outstanding `invoices.balance > 0` for that learner (configurable).

Refuse with 422 `not_eligible` and explanatory reasons.

### D. PDF + storage

- Template: bilingual HTML in `dops-api/app/services/certificates/templates/` rendered via WeasyPrint.
- Variables: learner name, program, locale, certificate number, issued date, QR code (rendered to PNG bytes, embedded).
- Output stored at `certificates/{tenant_id}/{certificate_id}.pdf` in the existing storage account container.
- `pdf_sha256` recorded for tamper-detection.

### E. Verification page

`GET /verify/{qr_token}` is **public** (no auth) and returns:

```json
{ "valid": true, "issued_to": "Maya H.", "program": "Class 5 Complete Program",
  "tenant": "NorthStar Driving School", "issued_at": "2026-05-25T16:00:00Z" }
```

If revoked, returns `{ "valid": false, "revoked_at": "..." }`. No PII beyond first name + last initial.

### F. UX changes

- Certificates page lanes: **Draft → Pending → Issued → Revoked**, with bulk approve.
- Each card surfaces a real download link after issue.
- Learner portal certificate card shows status and download link when issued.

## Acceptance Criteria

- Generating a certificate is rejected when prerequisites are not met.
- Approving issues a PDF in storage with the correct SHA-256 recorded.
- The public verification page works for issued certificates and shows revoked state cleanly.
- Revocation persists `revoked_at`, the verification page reflects it within 1 minute (cache invalidation).
- Learner receives an email + WhatsApp message on issue (linked into [08-automations.md](./08-automations.md)).
- All actions emit `certificate_audit_events`.

## Dependencies

- [07-school-schedule-templates.md](./07-school-schedule-templates.md) — program curriculum drives eligibility.
- [06-scheduling.md](./06-scheduling.md) — completion + evaluation rows.
- [09-invoices-payments.md](./09-invoices-payments.md) — outstanding-balance check.
- [08-automations.md](./08-automations.md) — `certificate_ready` flow.
