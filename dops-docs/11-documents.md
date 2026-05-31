# 11 — Documents

## Purpose

Securely capture learner permits, parental consents, payment receipts, and any other regulated uploads. Provide review state. Power the "Missing documents" learner status and automation flow.

## Current State

### Data model — [`dops-api/app/models/document.py`](../dops-api/app/models/document.py)

```python
class DocumentFile:
    owner_id, type, file_name, mime_type, size_bytes, uploaded_at, status
```

### API — [`dops-api/app/api/v1/endpoints/documents.py`](../dops-api/app/api/v1/endpoints/documents.py)

| Method | Path | Perm |
| ------ | ---- | ---- |
| GET | `/documents` | `learners.read` |
| POST | `/documents/upload-link` | `learners.update` |
| POST | `/documents` | `learners.update` |
| GET | `/documents/{id}` | `learners.read` |
| DELETE | `/documents/{id}` | `learners.update` |

### Behaviour

[`document_service.upload_link`](../dops-api/app/services/document_service.py) creates a `DocumentFile` row and returns a placeholder `http://localhost:8000/local-upload-placeholder/{id}` URL. **No file is ever actually uploaded or stored anywhere.**

## Gaps For Real Implementation

1. No real upload path → no real blob.
2. No virus scan / MIME-sniffing / size cap enforcement.
3. No review queue UI.
4. No retention / deletion policy.
5. No version history (re-upload overwrites silently).
6. No bilingual document type taxonomy.
7. Permissions are coarse — instructors can read documents because the route uses `learners.read` (instructors have that). Some documents (e.g. health) need stricter ACL.

## Real Implementation Plan

### A. Schema changes

```sql
documents
  ADD COLUMN tenant_id, learner_id null,            -- nullable for school-internal docs
  ADD COLUMN doc_type,                              -- enum: learner_permit | parental_consent | medical | id_card | invoice_pdf | certificate_pdf | other
  ADD COLUMN blob_path,                             -- 'documents/{tenant_id}/{owner}/{uuid}.{ext}'
  ADD COLUMN sha256, content_length,
  ADD COLUMN status,                                -- pending_upload | scanning | pending_review | approved | rejected | expired
  ADD COLUMN reviewed_by user_id null, reviewed_at null, review_note null,
  ADD COLUMN issued_at null, expires_at null,
  ADD COLUMN visibility,                            -- enum: tenant_staff | tenant_admins | learner_only
  ADD COLUMN uploaded_by user_id

document_audit_events (
  id pk, tenant_id, document_id,
  event,                                            -- requested_upload | uploaded | scanned | approved | rejected | downloaded | deleted
  actor user_id null, occurred_at, ip, user_agent
)
```

### B. Upload flow (real)

Use direct-to-storage upload with SAS URL.

```
POST /documents/upload-link
  body: { learner_id?, doc_type, file_name, mime_type, size_bytes, expires_at? }
  -> { document_id, upload_url (SAS write, 15 min), required_headers }

client PUTs the bytes to upload_url.

POST /documents/{id}/finalize
  body: { sha256, content_length }
  -> { document, status='scanning' }

Async: ClamAV scanner (sidecar) consumes a Service Bus message published on finalize.
  on clean → status = 'pending_review' + 'document.uploaded' event ([08-automations.md])
  on infected → status = 'rejected', purge blob.
```

### C. Download

```
GET /documents/{id}/download         learners.read + visibility check
  -> 302 to SAS read URL, 5 min TTL
```

Tenants where compliance is strict get IP-restricted SAS.

### D. Review queue UI

- New Documents section (under Learners or its own page) lists `pending_review` docs by tenant.
- Reviewer can approve / reject (with note). Rejection triggers the `missing_documents` flow with the note as context.
- Documents grouped per learner on the learner detail page.

### E. Retention

- Permits & evaluations: retain 7 years (matches QC Class 5 retention).
- Parental consents: retain until 7 years past learner age of majority.
- Background cron deletes blobs past their retention and marks the row `expired`.

### F. ACL

- `visibility = learner_only` — only the learner + admins can read.
- `visibility = tenant_admins` — owner + admins only (medical, complaints).
- `visibility = tenant_staff` — instructors included.

## Acceptance Criteria

- Learner uploads a permit from the portal via SAS URL; scan completes within 60 s; document moves to `pending_review`.
- Admin approves; the learner's `document_status` flips to `complete`; the missing-document flow stops.
- An expired retention period triggers blob deletion; the document row remains as `expired` (no PII exposed).
- Health documents are unreadable by instructors.

## Dependencies

- [04-learners.md](./04-learners.md) — `document_status` derived from open required docs.
- [08-automations.md](./08-automations.md) — missing-document nudges + on-upload notifications.
- [16-infrastructure.md](./16-infrastructure.md) — Storage container + SAS signing + virus-scan sidecar.
