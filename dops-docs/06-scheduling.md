# 06 — Scheduling & Sessions

## Purpose

Plan and run the day-to-day calendar of theory classes, practical lessons, evaluations, road-test prep, and admin follow-ups. Detect conflicts. Surface suggestions for replacement slots and waitlist fills. Generate learner schedules from a school template (see [07-school-schedule-templates.md](./07-school-schedule-templates.md)).

## Current State

### Data model — [`dops-api/app/models/scheduling.py`](../dops-api/app/models/scheduling.py)

```python
class ScheduledSession(Base, IdMixin, TenantScopedMixin, AuditMixin):
    type: str           # theory_module | practical_lesson | evaluation | road_test_prep | admin_follow_up
    status: str         # scheduled | confirmed | completed | cancelled | reschedule_requested |
                        # no_show | instructor_unavailable | conflict
    title: str
    starts_at, ends_at: datetime
    branch_id: str
    participants: list[dict]   # [{ user_id, role }]
    location: str
    learner_id, instructor_id: str | None
    conflict: str | None
```

### API — [`dops-api/app/api/v1/endpoints/scheduling.py`](../dops-api/app/api/v1/endpoints/scheduling.py)

| Method | Path | Perm | What it does today |
| ------ | ---- | ---- | ------------------ |
| GET | `/schedule/sessions` | `schedule.read` | list all sessions for tenant |
| POST | `/schedule/sessions` | `schedule.manage` | create one session |
| GET | `/schedule/sessions/{id}` | `schedule.read` | fetch one |
| PATCH | `/schedule/sessions/{id}` | `schedule.manage` | update times/status |
| POST | `/schedule/sessions/{id}/cancel` | `schedule.cancel` | sets status to cancelled (does not record reason) |
| POST | `/schedule/sessions/{id}/reschedule` | `schedule.reschedule` | sets new times + confirmed |
| POST | `/schedule/generate` | `schedule.manage` | **placeholder. returns empty sessions[] + hard-coded suggestion** |
| GET | `/schedule/suggestions` | `schedule.read` | returns hard-coded suggestions |
| GET | `/schedule/conflicts` | `schedule.read` | computes instructor overlap with O(n²) loop |

### UX — [`dops-frontend/src/features/scheduling/SchedulingPage.tsx`](../dops-frontend/src/features/scheduling/SchedulingPage.tsx)

- Instructor lanes for the week with the first 4 sessions each.
- Tabs: weekly view and "today" mobile view.
- Suggestion panel with hardcoded WhatsApp/replacement copy.
- Top metrics: theory class count, practical count, replacement slots (hardcoded 6), waitlist count.
- Buttons: **Generate schedule** and **Replacement** (no handlers wired).

## Gaps For Real Implementation

1. **`generate_schedule` is a stub**. See [`scheduling_service.py:66-70`](../dops-api/app/services/scheduling_service.py). It must consume a school template + program curriculum + learner availability + instructor availability and emit concrete sessions.
2. **Conflicts are O(n²) over all tenant sessions** with no time bounds and no learner-side conflict checks (only instructor overlap is detected).
3. **No cancellation reason is persisted** — `CancellationRequest.reason` is accepted but never stored. Required for cancellation policy enforcement and reporting.
4. **No reschedule reason persisted** either.
5. **No waitlist data model**. The suggestion talks about a waitlist; nothing backs it.
6. **No "replacement slot" data model**. Free instructor + free time slot calculations would have to be derived.
7. **No paged listing**. `/schedule/sessions` returns everything; admins with months of data will explode.
8. **No filtering by date range, instructor, learner, branch, status**.
9. **No status transitions guard**. Today any update can set any status; should enforce a state machine (scheduled → confirmed → completed; or → cancelled / no_show / reschedule_requested).
10. **No notifications wiring**. Cancel/reschedule should trigger automation flows (WhatsApp learner + email confirmation) — see [08-automations.md](./08-automations.md).
11. **No timezone discipline**. `starts_at`/`ends_at` are stored as `DateTime(timezone=True)` (good) but no per-branch timezone awareness when rendering UI for multi-province tenants.

## Real Implementation Plan

### A. Schema additions

```sql
scheduled_sessions
  ADD COLUMN program_id fk null,
  ADD COLUMN program_item_id fk null,       -- which item of the program curriculum
  ADD COLUMN vehicle_id fk null,
  ADD COLUMN generated_from_run_id null,    -- traceability back to a generate-schedule run
  ADD COLUMN cancellation_reason text null,
  ADD COLUMN cancellation_actor user_id null,
  ADD COLUMN reschedule_reason text null,
  ADD COLUMN pickup_location text null

session_status_history (
  id pk, tenant_id, session_id,
  from_status, to_status,
  actor_user_id, occurred_at, reason
)

waitlist_entries (
  id pk, tenant_id, learner_id,
  requested_session_type,                   -- enum
  preferred_branch_id, preferred_instructor_id,
  earliest, latest,                         -- date range
  status,                                   -- open | offered | accepted | declined | expired
  created_at, last_offered_at
)

replacement_offers (
  id pk, tenant_id,
  source_session_id null,                   -- the session that opened the slot
  starts_at, ends_at, branch_id, instructor_id,
  offered_to learner_id null,
  offered_at, expires_at, accepted_at, declined_at
)

schedule_generation_runs (
  id pk, tenant_id,
  requested_by user_id, requested_at,
  scope jsonb,                              -- { learner_ids, date_from, date_to, template_id }
  status,                                   -- pending | committed | rolled_back
  sessions_created int, conflicts_count int, suggestions_count int,
  log jsonb                                 -- per-decision trail
)
```

### B. Endpoint additions & changes

```
GET  /schedule/sessions?from=&to=&instructor_id=&learner_id=&branch_id=&status=&page=&size=
POST /schedule/sessions                  -- body adds vehicle_id, pickup_location, program_item_id
POST /schedule/sessions/{id}/cancel      -- body { reason, notify_participants }     -> persists
POST /schedule/sessions/{id}/reschedule  -- body { starts_at, ends_at, reason }      -> persists
POST /schedule/sessions/{id}/no-show     -- body { learner_attended, instructor_attended, notes }
POST /schedule/sessions/{id}/complete    -- body { notes, lesson_focus, evaluation_score? }

POST /schedule/generate
  body: {
    template_id,                    -- school template (07-school-schedule-templates.md)
    learner_ids,                    -- subset to schedule
    date_from, date_to,
    dry_run: true|false             -- when true, returns proposal without persisting
  }
  returns: GenerateScheduleResponse { run_id, sessions, conflicts, suggestions }

POST /schedule/generation-runs/{run_id}/commit       -- promotes a dry-run proposal to real sessions
POST /schedule/generation-runs/{run_id}/rollback     -- deletes sessions created by a run

GET    /schedule/waitlist
POST   /schedule/waitlist                            -- add learner to waitlist
DELETE /schedule/waitlist/{id}

GET    /schedule/replacement-offers
POST   /schedule/replacement-offers                  -- system-driven mostly; manual override allowed
POST   /schedule/replacement-offers/{id}/accept
POST   /schedule/replacement-offers/{id}/decline

GET    /schedule/conflicts?from=&to=                 -- bounded; learner + instructor + vehicle conflicts
```

### C. Conflict detection — production version

`SchedulingService.conflicts(tenant_id, from, to)`:

- Pull `scheduled_sessions WHERE tenant_id AND starts_at <= to AND ends_at >= from AND status in ('scheduled','confirmed')`.
- Build per-instructor sweep: instructor double-booking → severity=blocking.
- Build per-learner sweep: learner double-booking → severity=blocking.
- Build per-vehicle sweep: vehicle double-booking → severity=blocking.
- Cross-check against instructor effective-availability ([05-instructors.md](./05-instructors.md)) → severity=warning if outside availability.
- Cross-check against school operating hours ([07-school-schedule-templates.md](./07-school-schedule-templates.md)) → severity=blocking.
- Persist most-recent conflicts on a `session_conflicts` table for quick lookup; recompute on session writes.

### D. Generation algorithm (orchestrator)

Lives in `dops-api/app/services/scheduling/generator.py`. Inputs:

- school template + operating hours + closures (template_id)
- program curriculum for each learner's enrollment ([07-school-schedule-templates.md](./07-school-schedule-templates.md))
- learner.preferred_language, learner.preferred_pickup_location
- learner_availability_windows (new — see "Learner availability" below)
- instructor effective-availability + qualifications + languages + vehicles
- existing scheduled sessions in [date_from, date_to]
- holds on learner (skip if `payment` or `documents` hold present, unless override flag)

Output: list of proposed `ScheduledSession` rows.

Algorithm sketch:

1. Group learners by program; for each learner determine which curriculum items are still due in window.
2. Build a constraint problem per learner: each item is a slot to fill with (date, time, instructor_id, vehicle_id) satisfying:
   - inside school operating hours,
   - inside instructor effective-availability,
   - inside learner availability window,
   - language match (instructor.preferred_languages ⊇ learner.preferred_language),
   - vehicle transmission match,
   - precedence (theory N must happen before practical N+1; road test prep only after practical >= required),
   - branch match,
   - no instructor/learner/vehicle double-booking against existing sessions,
   - inter-session minimum gap (configurable per tenant).
3. Use a deterministic greedy fill with conflict back-tracking. Iterate learners in priority order: (a) approaching road-test-prep, (b) behind_schedule, (c) registered_at ascending.
4. Return proposal, write `schedule_generation_runs` row, return run_id.

The MVP can be a heuristic; design the interface so a CP-SAT solver (`ortools`) can replace the heuristic later without changing endpoints.

### E. Learner availability

New table:

```sql
learner_availability_windows (
  id pk, tenant_id, learner_id,
  weekday smallint,
  starts_at time, ends_at time,
  effective_from date, effective_to date null,
  notes
)
```

Captured during registration (wizard step) and editable from the learner detail page or portal.

### F. State machine for session status

```
   scheduled ── confirm ──▶ confirmed
       │ │                      │
       │ └── cancel ───▶ cancelled                  (terminal)
       │                        │
       │                        └── reschedule_requested ─▶ reschedule_requested ─▶ confirmed/cancelled
       │
       └── reschedule (admin) ─▶ scheduled (new times)

   confirmed ── complete ─▶ completed   (terminal)
   confirmed ── no_show ──▶ no_show     (terminal)
```

Implement in `SchedulingService` with a `Transition` table; reject unknown transitions with 422.

### G. UX changes

- Replace the four hard-coded metric cards with derived counts (range = current week).
- **Generate schedule** opens a modal: pick learners, date range, dry-run; opens a review screen showing the proposal grouped by day with conflicts/suggestions; commit or cancel.
- Each session card surfaces a context menu: Cancel (with reason), Reschedule, Mark complete, Mark no-show, Reassign instructor, Reassign vehicle.
- Conflicts pane (collapsible) lists open conflicts in window with one-click resolutions.
- Replacement-slots panel: shows open `replacement_offers`, lets admin send a WhatsApp offer (calls automation flow).
- Waitlist panel: list, add/remove, "offer the cancelled slot" CTA.

## Acceptance Criteria

- All endpoints honour pagination + filtering.
- Cancel + reschedule persist reasons; status history is visible on a session detail drawer.
- `POST /schedule/generate` with dry-run produces a proposal in < 3 s for 50 learners over 30 days; commit creates exactly those rows and writes one `schedule_generation_runs` row.
- Rolling back a run deletes only the sessions created by that run.
- No DB query returns sessions across tenants.
- State machine enforced and unit-tested.
- Cancel triggers an automation flow run (mocked transport in dev).
- The Schedule page never renders hard-coded sample data after the migration.

## Dependencies

- [07-school-schedule-templates.md](./07-school-schedule-templates.md) — engine inputs.
- [05-instructors.md](./05-instructors.md) — effective availability + vehicles + qualifications.
- [04-learners.md](./04-learners.md) — holds + learner availability windows.
- [08-automations.md](./08-automations.md) — outbound notifications on cancel/reschedule/offer.
