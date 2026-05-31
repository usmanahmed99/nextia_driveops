# 04 — Learners

## Purpose

A learner is an enrolled student at a driving school. The system tracks their identity, program, enrollment date, progress (theory %, practical %, overall %), current phase, timeline of events, documents (learner permit + consents), payment status, communication status, and assigned instructor.

## Current State

### Data model — [`dops-api/app/models/learner.py`](../dops-api/app/models/learner.py)

```python
class Learner(Base, IdMixin, TenantScopedMixin, AuditMixin):
    status: str                     # behind_schedule | active | road_test_scheduled | …
    profile: dict                   # first_name, last_name, email, phone, preferred_language
    enrollment: dict                # program_id, program_name, enrolled_at, current_phase
    progress: dict                  # overall_percent, theory_percent, practical_percent,
                                    # completed_theory_modules, completed_practical_lessons,
                                    # required_practical_lessons
    instructor_id: str | None
    payment_status: str             # unpaid | partial | paid
    document_status: str            # missing | review_needed | complete
    timeline: list[dict]
    documents: list[dict]
```

Most rich data is collapsed into `JSON` columns — fine for the concept, not for production.

### API — [`dops-api/app/api/v1/endpoints/learners.py`](../dops-api/app/api/v1/endpoints/learners.py)

| Method | Path | Perm | Status |
| ------ | ---- | ---- | ------ |
| GET | `/learners?status=&search=` | `learners.read` | implemented |
| POST | `/learners` | `learners.create` | implemented |
| GET | `/learners/{id}` | `learners.read` | implemented |
| PATCH | `/learners/{id}` | `learners.update` | implemented |
| DELETE | `/learners/{id}` | `learners.delete` | implemented |
| GET | `/learners/{id}/timeline` | `learners.read` | implemented |
| GET | `/learners/{id}/documents` | `learners.read` | implemented |
| GET | `/learners/{id}/progress` | `learners.read` | implemented |

`LearnerService` also exposes a composed "workspace" used by [`dops-frontend/src/features/learners/LearnerDetailPage.tsx`](../dops-frontend/src/features/learners/LearnerDetailPage.tsx) that joins sessions + invoices + certificates + conversations into one payload. **Verify the workspace endpoint exists**; if not, it is computed client-side and should move server-side.

### UX

- `LearnersPage.tsx` — filterable list with quick filters (all, active, behind, missing docs, payment, certificate-ready) + search.
- `LearnerDetailPage.tsx` — header with status badges, metric cards (theory, practical, payment, certificate), journey timeline, WhatsApp preview, upcoming sessions, missing docs, notes, recommended next actions.

## Gaps For Real Implementation

1. **Normalize rich fields**. `profile`, `enrollment`, `progress`, `timeline`, `documents` are all `JSON`. Real impl moves them into proper tables or strict Pydantic schemas with column constraints.
2. **Progress is a snapshot, not a calculation**. Today `progress` is stored. Real impl derives it from completed sessions + completed theory modules — see [06-scheduling.md](./06-scheduling.md) and [07-school-schedule-templates.md](./07-school-schedule-templates.md).
3. **No `branch_id` on Learner**. Each learner should belong to a primary branch.
4. **No real Program model**. `enrollment.program_name` is a free string. Real impl needs a `Program` table with theory module list, required practical lessons count, evaluations, road test prep, and pricing — see [07-school-schedule-templates.md](./07-school-schedule-templates.md).
5. **No registration funnel**. The concept's "Register learner" button is a stub. The real funnel must capture: identity, contact (verified), preferred language, branch, program, learner permit number, parent/guardian (if minor), pickup address & notes, payment plan.
6. **No de-duplication / merge** for learners that re-enroll.
7. **No "blocked" reason model**. Today UI infers reasons; backend should hold `LearnerHold(type, reason, set_by, set_at, resolved_at)` so payment-block, document-block, behaviour-block are explicit.
8. **No GDPR/PII deletion path**. Required for prod.

## Real Implementation Plan

### A. Schema cleanup

New tables:

```sql
learners (
  id pk, tenant_id, branch_id,
  status,                      -- enum: active | behind_schedule | road_test_scheduled | …
  preferred_language,
  preferred_pickup_location,
  pickup_notes,
  notes,
  registered_at,
  enrolled_program_id fk,
  current_phase,
  instructor_id fk null,       -- preferred / default
  emergency_contact_name,
  emergency_contact_phone,
  audit cols
)

learner_users (learner_id fk, user_id fk)     -- 1:1 for adult learners; can grow to 1:N for guardians

learner_holds (
  id pk, tenant_id, learner_id,
  type,                        -- enum: payment | documents | behaviour | health | other
  reason text,
  set_by user_id, set_at,
  resolved_by user_id null, resolved_at null
)

learner_timeline_events (
  id pk, tenant_id, learner_id,
  type,                        -- enum: registration | document_uploaded | document_approved |
                                --  theory_module_completed | practical_lesson_completed |
                                --  evaluation_completed | road_test_scheduled | road_test_passed |
                                --  certificate_issued | hold_added | hold_resolved
  payload jsonb,
  occurred_at,
  recorded_by user_id null
)

learner_progress_snapshots (   -- optional materialised view, recomputed nightly + on session complete
  learner_id pk, snapshot_at,
  overall_percent, theory_percent, practical_percent,
  completed_theory_module_ids jsonb,
  completed_practical_lesson_count
)
```

`Learner.profile` collapses into top-level columns; PII (phone, email, address) goes through the user row.

### A1. Learner lifecycle state machine

Learner status should be deterministic and transition through a controlled state machine, not arbitrary strings.

Recommended lifecycle:

```text
draft_registration
  -> pending_documents
  -> pending_payment
  -> ready_for_theory
  -> in_theory
  -> ready_for_practical
  -> in_practical
  -> ready_for_evaluation
  -> road_test_ready
  -> road_test_scheduled
  -> certificate_ready
  -> completed
```

Exceptional states:

```text
on_hold
withdrawn
inactive
transferred
```

Statuses shown in the current UI (`behind_schedule`, `missing_documents`, `payment_due`, `waiting_for_practical`) should become computed badges or active holds layered on top of this lifecycle.

### A2. Registration validation

The register-learner flow should validate:

- legal first and last name;
- preferred language;
- date of birth;
- minor/guardian requirements;
- email and phone format;
- branch;
- program;
- pickup address;
- learner permit/license number when required;
- consent and contract documents;
- payment plan or invoice rule.

Validation failures should return structured errors that the wizard can map to each step.

### B. Endpoints (added/changed)

```
POST   /api/v1/learners/{id}/holds                            learners.update
DELETE /api/v1/learners/{id}/holds/{hold_id}                  learners.update
POST   /api/v1/learners/{id}/timeline-events                  learners.update   (internal use; usually system-emitted)
GET    /api/v1/learners/{id}/workspace                        learners.read     (composed view: learner + upcoming sessions + outstanding invoices + active conversation + certificate state)
POST   /api/v1/learners/{id}/instructor                       learners.update   body: { instructor_id }
POST   /api/v1/learners/{id}/withdraw                         learners.delete   body: { reason, retain_data_until }
```

`workspace` is what the detail page currently consumes; consolidate it server-side.

### B1. Learner workspace response

The detail page should use one endpoint to avoid page-level orchestration and inconsistent loading states.

```text
GET /api/v1/learners/{id}/workspace
```

Response shape:

```json
{
  "learner": {},
  "journey": {
    "currentStage": "in_practical",
    "stages": [
      {"code": "registration", "status": "completed"},
      {"code": "documents", "status": "completed"},
      {"code": "theory", "status": "completed"},
      {"code": "practice", "status": "current"},
      {"code": "evaluation", "status": "blocked"},
      {"code": "certificate", "status": "not_started"}
    ]
  },
  "holds": [],
  "progress": {},
  "upcomingSessions": [],
  "recentSessions": [],
  "outstandingInvoices": [],
  "documents": [],
  "activeConversation": null,
  "certificate": null,
  "recommendedActions": [
    {
      "code": "schedule_next_practical",
      "priority": "high",
      "labelKey": "learners.actions.scheduleNextPractical"
    }
  ]
}
```

Performance target: p95 < 300 ms for a normal learner, p95 < 700 ms for a learner with large timeline/history.

### C. Registration funnel (UI)

Replace the Register Learner stub with a wizard:

1. Identity (name, DOB, language)
2. Contact (email + phone with OTP verification stubbed; real OTP comes via Twilio/Azure Comm Services)
3. Branch + program selection (drives pricing + curriculum)
4. Pickup details (address, instructions)
5. Documents (learner permit upload via secure-upload link — see [11-documents.md](./11-documents.md))
6. Payment plan + first invoice (see [09-invoices-payments.md](./09-invoices-payments.md))
7. Confirmation + WhatsApp/email welcome (see [08-automations.md](./08-automations.md))

### D. Holds drive UI states

The badges on the Learners list (`Behind schedule`, `Missing documents`, `Payment due`, `Certificate ready`) should be derived from a deterministic rule set:

- `payment` hold → red "Payment due" badge; new sessions cannot be generated (configurable in tenant settings).
- `documents` hold → amber "Missing documents"; document-list block.
- `behind_schedule` is a system-computed status: enrolled > 30 days AND `progress.overall_percent < expected_for_days_since_enrol`.
- `certificate_ready` → `progress.overall_percent >= 100 AND all_required_evaluations_passed`.

### E. Privacy

- Soft-delete + retention window. `POST /learners/{id}/withdraw` sets `status=withdrawn` and `retain_data_until = now + 7 years` (per QC Class 5 retention).
- Hard-delete background job purges PII columns + audit-log scrubs after retention.

### F. Operator workflows

The learner feature should support complete office workflows, not only table updates:

1. Register learner and send welcome message.
2. Convert public-site lead into learner.
3. Assign or change program.
4. Assign preferred branch, instructor, vehicle type, and language.
5. Place learner on hold for missing documents, payment, medical, admin review, or learner request.
6. Resume learner from hold.
7. Generate schedule suggestions.
8. Confirm schedule and notify learner/instructor.
9. Reschedule or cancel one session.
10. Withdraw learner and stop future automations.
11. Transfer learner between branches.
12. Mark learner certificate-ready after final requirements are met.

Every workflow should create:

- learner timeline event;
- audit-log row with actor, before/after values, and reason;
- optional automation event;
- optional notification record;
- idempotency key for operations that can be retried by the UI or worker.

### G. Search, filters, and list performance

The learner list needs to be usable for a school with thousands of learners.

Required filters:

- lifecycle status;
- branch;
- program;
- instructor;
- language;
- payment hold;
- document hold;
- behind schedule;
- certificate ready;
- enrolled date range;
- next session date range;
- free-text search across learner name, phone, email, permit number, and public registration reference.

Implementation notes:

- use server-side paging, sorting, and filtering;
- default sort: most urgent operational issue first, then next session date;
- expose a `hasMore` cursor response for infinite-scroll UI;
- never return PII fields the current role cannot view;
- add indexes for `tenant_id`, `branch_id`, `status`, `program_id`, and normalized search fields.

### H. Metrics and reporting hooks

Learner writes should update reporting facts asynchronously through events, so dashboards do not depend on expensive live joins.

Minimum events:

- `learner.registered`;
- `learner.program_changed`;
- `learner.hold_added`;
- `learner.hold_removed`;
- `learner.session_completed`;
- `learner.evaluation_passed`;
- `learner.certificate_ready`;
- `learner.withdrawn`.

Metrics fed by those events:

- active learners by branch/program;
- average days from registration to first lesson;
- average days from registration to certificate-ready;
- learners blocked by payment/documents/admin hold;
- learner churn/withdrawal reasons;
- schedule adherence by learner cohort;
- certificate conversion rate.

## Acceptance Criteria

- Admin can run the full register-learner wizard and see the learner appear in the list with `status=registered`.
- All learner statuses on the UI come from deterministic backend rules, not free strings.
- Adding a payment hold blocks new schedule generation for that learner.
- The detail page is fed by one workspace endpoint that returns in < 300 ms p95.
- All write actions emit timeline + audit-log rows.
- Learner list supports server-side filtering, sorting, paging, and role-aware PII redaction.
- Learner lifecycle events feed reporting asynchronously.

## Dependencies

- [05-instructors.md](./05-instructors.md), [06-scheduling.md](./06-scheduling.md), [07-school-schedule-templates.md](./07-school-schedule-templates.md) — for assignment + progress derivation.
- [08-automations.md](./08-automations.md) — welcome flow on registration.
- [09-invoices-payments.md](./09-invoices-payments.md) — first invoice at registration.
- [11-documents.md](./11-documents.md) — permit upload.
