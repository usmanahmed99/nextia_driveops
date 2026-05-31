# 05 — Instructors

## Purpose

An instructor delivers theory and practical lessons. The system tracks identity, languages, vehicle, locations they teach in, weekly availability, current utilization, and today's assignment count.

## Current State

### Data model — [`dops-api/app/models/instructor.py`](../dops-api/app/models/instructor.py)

```python
class Instructor(Base, IdMixin, TenantScopedMixin, AuditMixin):
    status: str                  # available | fully_booked | away | conflict | inactive
    profile: dict                # first_name, last_name, email, phone, languages, vehicle, locations
    availability: list[dict]     # [{ weekday: 1..7, starts_at: "08:00", ends_at: "17:00", branch_id }]
    utilization_percent: int
    today_assignments: int
```

### Schema — [`dops-api/app/schemas/instructors.py`](../dops-api/app/schemas/instructors.py)

```
InstructorProfile { first_name, last_name, email, phone, languages, vehicle, locations }
InstructorAvailability { weekday: int, starts_at: str("HH:MM"), ends_at: str("HH:MM"), branch_id? }
```

### API — [`dops-api/app/api/v1/endpoints/instructors.py`](../dops-api/app/api/v1/endpoints/instructors.py)

| Method | Path | Perm | Notes |
| ------ | ---- | ---- | ----- |
| GET | `/instructors` | `instructors.read` | list |
| POST | `/instructors` | `instructors.manage` | create |
| GET | `/instructors/{id}` | `instructors.read` | detail |
| PATCH | `/instructors/{id}` | `instructors.manage` | partial update incl. availability replace |
| GET | `/instructors/{id}/availability` | `instructors.read` | |
| PUT | `/instructors/{id}/availability` | `instructors.manage` | full replace |

### UX

- `InstructorsPage.tsx` shows the roster, status, vehicle, languages, utilization.
- The Scheduling page lays out one lane per instructor and shows their first 4 sessions of the week.

## Gaps For Real Implementation

1. **Availability is a list of recurring weekday strips only**. It cannot express:
   - per-date overrides (sick day, training day, vacation),
   - varying weekly patterns (weekends only on alternate weeks),
   - branch-specific availability (an instructor splits Mon–Wed Downtown vs Thu–Fri Laval).
   The `branch_id` field exists but is single-valued per strip; the engine cannot reason about it because no engine exists yet.
2. **No vehicle model**. `profile.vehicle` is a free string. Practical lessons must match learner needs (manual vs automatic, accessible) — needs a `Vehicle` table.
3. **No qualifications / endorsements** (e.g. allowed to teach Class 5, Class 6 motorcycle, winter driving certified, accessibility-trained).
4. **Utilization is a stored integer**. Should be derived from sessions in a configurable window.
5. **No `INSTRUCTOR` role linkage**. `Instructor.profile.email` is free text. There must be a `User`-`Instructor` link so an instructor can log in to the portal.
6. **`today_assignments` is stored** rather than computed.

## Real Implementation Plan

### A. Schema changes

```sql
instructors (
  id pk, tenant_id, branch_id,            -- primary branch
  user_id fk,                              -- 1:1 with users.role=INSTRUCTOR
  first_name, last_name, email, phone,
  preferred_languages text[],              -- ['en','fr']
  status,                                  -- enum
  hourly_rate_cents, currency,
  notes,
  audit cols
)

instructor_branches (instructor_id, branch_id) -- many-to-many for split-branch staff

instructor_qualifications (
  instructor_id, code,                     -- enum: class_5 | class_6 | winter | accessibility | ...
  issued_at, expires_at
)

vehicles (
  id pk, tenant_id, branch_id,
  make, model, year, transmission,         -- manual | automatic
  plate, vin,
  accessible boolean,
  in_service_from, in_service_to,
  notes
)

instructor_vehicles (instructor_id, vehicle_id, is_primary boolean)

instructor_availability_rules (
  id pk, tenant_id, instructor_id, branch_id,
  weekday smallint,                        -- 1..7
  starts_at time, ends_at time,
  effective_from date, effective_to date null
)

instructor_availability_exceptions (
  id pk, tenant_id, instructor_id,
  date,
  type,                                    -- enum: off | extra | shifted
  starts_at time null, ends_at time null,  -- null when off
  reason
)
```

### B. Endpoint changes

Replace the current PUT-replace pattern with explicit rule/exception CRUD:

```
GET    /instructors/{id}/availability/rules
POST   /instructors/{id}/availability/rules
PATCH  /instructors/{id}/availability/rules/{rule_id}
DELETE /instructors/{id}/availability/rules/{rule_id}

GET    /instructors/{id}/availability/exceptions?from=&to=
POST   /instructors/{id}/availability/exceptions
DELETE /instructors/{id}/availability/exceptions/{exception_id}

GET    /instructors/{id}/effective-availability?from=&to=
  -> returns the materialised intervals after rules ∪ exceptions are merged
```

Add:

```
POST /instructors/{id}/qualifications              instructors.manage
GET  /vehicles  /  POST /vehicles  /  PATCH /vehicles/{id}     instructors.manage
```

`effective-availability` is the only endpoint the scheduling engine and portal consume; rules+exceptions are admin-only.

### C. Utilization & today's count

- `utilization_percent(window)` is computed by the API as `scheduled_minutes / available_minutes` over the requested window (default last 7 days).
- `today_assignments` is computed from `scheduled_sessions` where `starts_at` between today's local-tz bounds.
- Remove the stored columns.

### D. Instructor onboarding

- Owner invites instructor via the invitation flow ([03-authentication.md](./03-authentication.md)).
- On accept, the instructor completes a profile wizard: language(s), branches, qualifications, vehicle assignment, default weekly availability.
- The Instructor Portal becomes available to them immediately.

### E. UX changes

- Instructors page: keep the roster card. Add tabs for **Availability** and **Vehicles**. Availability uses a week-grid editor (Google-Calendar-like) bound to rules + exceptions.
- Settings → **Vehicles** as a new section.

## Acceptance Criteria

- Adding an exception (sick day) immediately removes that instructor from generation candidacy on that date.
- Effective availability endpoint returns merged intervals in < 100 ms for a 4-week window.
- Vehicle transmission/accessibility is matched against learner preference during scheduling.
- Instructor logs into the portal and sees only their tenant + their sessions + their branches.

## Dependencies

- [02-multitenancy-rbac.md](./02-multitenancy-rbac.md), [03-authentication.md](./03-authentication.md) — instructor invite/login.
- [06-scheduling.md](./06-scheduling.md) — engine consumes effective availability.
- [07-school-schedule-templates.md](./07-school-schedule-templates.md) — school-level operating hours bound instructor availability.
