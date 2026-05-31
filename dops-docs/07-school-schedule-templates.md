# 07 — Typical School Schedule (Templates) & Program Curriculum

> See also [23-service-catalog-and-driving-centers.md](./23-service-catalog-and-driving-centers.md) — the service catalog + driving centers + slot-gap configuration that the templates and engine reference.

> **Status: NOT IMPLEMENTED in the current concept.** This document specifies the missing building block that the existing **Generate schedule** button and **Settings → Programs / Session types / Cancellation policy** placeholder cards already imply. Without it, [06-scheduling.md](./06-scheduling.md) cannot produce a real per-learner schedule.

## Why This Is Treated As "Real Implementation Of An Existing Feature"

The concept UI advertises:

- A **Generate schedule** button on the Scheduling page.
- "Smart rescheduling" / "Waitlist slot filling" / "Open slot suggestion" automation flow types ([`dops-api/app/db/seed.py`](../dops-api/app/db/seed.py), [`AutomationsPage.tsx`](../dops-frontend/src/features/automations/AutomationsPage.tsx)).
- Settings placeholder cards: "Programs", "Session types", "Cancellation policy".
- A `POST /api/v1/schedule/generate` endpoint already exposed by the API.

These cannot be made real without:

1. A **school-level typical week** (operating hours per branch).
2. A **closure / holiday calendar**.
3. **Class period definitions** (recurring blocks like Theory Mon/Wed 18:00–20:00).
4. **Program curriculum** (ordered list of theory + practical + evaluations).
5. **Cancellation policy** that drives generation and notifications.

This doc specifies these as a single, cohesive feature: **School Schedule Templates**.

## Concept Confirmation (Search Findings)

A keyword search across all four repos for `template`, `typical`, `operating_hours`, `business_hours`, `weekly_template`, `schedule_template`, `ScheduleTemplate` returns no matches in `dops-api` models/services or `dops-frontend` features. The only "template" hits are:

- Pipeline YAML templates ([`dops-api/pipelines/azure-pipelines-api.yml`](../dops-api/pipelines/azure-pipelines-api.yml)).
- Bicep templates ([`dops-infra/infra/*.bicep`](../dops-infra/infra)).
- Frontend i18n strings mentioning "templates" for messaging or programs in copy only ([`dops-frontend/src/i18n/en/common.json`](../dops-frontend/src/i18n/en/common.json)).

There is no domain template entity anywhere in the system today.

## Data Model To Add

```sql
schedule_templates (
  id pk, tenant_id, branch_id,
  name,                             -- "NorthStar Default Week"
  is_default boolean,
  effective_from date, effective_to date null,
  created_at, updated_at, created_by user_id
)

operating_hours (
  id pk, tenant_id, branch_id, template_id,
  weekday smallint,                 -- 1..7
  opens_at time, closes_at time,
  is_open boolean default true
)
  -- multiple rows per (template, weekday) are allowed (split shifts)

closures (
  id pk, tenant_id, branch_id null, -- null = all branches
  template_id null,                 -- null = global; pinned to template if branch+template specific
  starts_on date, ends_on date,
  type,                             -- enum: holiday | maintenance | weather | other
  label,
  created_by user_id
)

class_periods (
  id pk, tenant_id, branch_id, template_id,
  name,                             -- "Theory block A"
  session_type,                     -- enum: theory_module | practical_lesson | evaluation | road_test_prep
  weekday smallint, starts_at time, ends_at time,
  capacity int default 1,           -- theory blocks can host multiple learners; practical=1
  default_instructor_id null,       -- optional pinning
  default_vehicle_id null,
  language_lock null                -- 'en'|'fr'|null (any)
)

cancellation_policies (
  id pk, tenant_id,
  window_hours int default 24,      -- learner-initiated cancel must be > N hours before start
  late_fee_cents int default 0, currency,
  no_show_fee_cents int default 0,
  free_reschedules_per_program int default 1,
  applies_to,                       -- enum: practical_lesson | all
  created_at, updated_at
)
```

## Program Curriculum (Required Companion Feature)

`learners.enrollment.program_id` already exists as a string. To make generation possible, programs must be modelled:

```sql
programs (
  id pk, tenant_id,
  code,                             -- 'CLASS_5'
  name, name_fr,                    -- bilingual
  description, description_fr,
  price_cents, currency,            -- optional fixed package price
  discount_cents,                   -- optional discount applied to service-item total when no fixed price is set
  required_practical_lessons int,
  required_theory_modules int,
  road_test_prep_count int,
  evaluations_count int,
  active boolean,
  created_at, updated_at
)

program_items (
  id pk, tenant_id, program_id,
  ordinal smallint,                 -- order within program
  item_type,                        -- enum: theory_module | practical_lesson | evaluation | road_test_prep | admin_follow_up
  name, name_fr,
  duration_minutes int default 90,
  prerequisite_item_ids uuid[]      -- precedence (theory N before practical N+1, etc.)
)
```

A `Learner.enrolled_program_id` foreign-keys to `programs`. The set of `program_items` not yet attached to a `completed` `scheduled_session` for that learner is the set the generator must place.

## API Surface

### Templates

```
GET    /api/v1/schedule/templates                      schedule.read
POST   /api/v1/schedule/templates                      settings.manage    body: { name, branch_id, is_default, effective_from, effective_to }
GET    /api/v1/schedule/templates/{id}                 schedule.read      returns template + operating_hours + class_periods + closures
PATCH  /api/v1/schedule/templates/{id}                 settings.manage
DELETE /api/v1/schedule/templates/{id}                 settings.manage    soft-delete unless referenced

POST   /api/v1/schedule/templates/{id}/operating-hours        settings.manage   bulk replace per weekday
POST   /api/v1/schedule/templates/{id}/class-periods          settings.manage   bulk replace
POST   /api/v1/schedule/templates/{id}/preview?from=&to=      schedule.read     returns the materialised week-grid the engine will see
```

### Closures

```
GET    /api/v1/schedule/closures?from=&to=             schedule.read
POST   /api/v1/schedule/closures                       settings.manage
PATCH  /api/v1/schedule/closures/{id}                  settings.manage
DELETE /api/v1/schedule/closures/{id}                  settings.manage
```

### Programs

```
GET    /api/v1/programs                                settings.manage
POST   /api/v1/programs                                settings.manage
GET    /api/v1/programs/{id}                           settings.manage
PATCH  /api/v1/programs/{id}                           settings.manage
DELETE /api/v1/programs/{id}                           settings.manage

POST   /api/v1/programs/{id}/items                     settings.manage   bulk replace curriculum
```

### Cancellation policy

```
GET    /api/v1/settings/cancellation-policy            settings.manage
PUT    /api/v1/settings/cancellation-policy            settings.manage
```

### Generation (now real)

`POST /api/v1/schedule/generate` ([06-scheduling.md](./06-scheduling.md)) becomes the integration point. Its `template_id` is the template the engine uses to define operating hours, closures, and class periods.

## UX

### Settings → Schedule Templates

Replaces the placeholder "Session types" + "Cancellation policy" cards.

- List of templates per branch with a default badge.
- Template editor:
  - Operating hours: 7-day editor (open/closed per weekday + open/close time per day; allow split shifts).
  - Class periods: visual week grid; add block (type, time, capacity, default instructor/vehicle, language lock).
  - Closures: range picker (national holidays, school closures, weather days).
  - Effective range (effective_from / effective_to) to support seasonal templates.
  - Preview tab: select a date range, see the materialised week-by-week grid that the engine will treat as legal scheduling space.

### Settings → Programs

Replaces the placeholder "Programs" card.

- Per-program editor: bilingual name, price, required counts, items list (drag-to-reorder, duration, prerequisites).
- Program archive vs delete (preserve historical enrollments).

### Settings → Cancellation policy

Single editor form bound to `PUT /settings/cancellation-policy`.

### Scheduling → Generate flow

The **Generate schedule** modal now needs:

- Template picker (defaults to the branch's default template).
- Date range.
- Learner picker (defaults: all learners with status `active | behind_schedule | waiting_for_practical | road_test_scheduled`).
- Dry-run preview before commit.

## Generation Algorithm — Inputs Wired

Generator code path (lives in `dops-api/app/services/scheduling/generator.py`):

```text
generate(tenant_id, payload):
  template       = templates.get(payload.template_id)
  operating      = operating_hours.list(template.id) - closures.in_window(payload.date_from, payload.date_to)
  periods        = class_periods.list(template.id)
  policy         = cancellation_policies.get(tenant_id)
  learners       = learners.list(payload.learner_ids)
  for each learner:
    program      = programs.get(learner.enrolled_program_id)
    items_open   = program_items - learner.completed_program_items
    candidate_slots = enumerate (date, period, instructor, vehicle) such that
        period.weekday in operating
        period.starts_at in operating[date].opens..closes
        instructor.effective_availability covers period
        instructor.languages ⊇ learner.preferred_language
        vehicle.transmission == learner.preferred_transmission
        vehicle.in_service at date
        not in closures
        not double-booking instructor/learner/vehicle
        respects program_item.prerequisite_items
    select earliest feasible slots greedily in priority order
  return proposal
```

### Worked Mini-Example

NorthStar:

- Template "Default Week" at Downtown branch.
- Operating: Mon–Fri 08:00–20:00, Sat 09:00–17:00, Sun closed.
- Closures: 2026-06-24 (St-Jean-Baptiste), 2026-07-01 (Canada Day).
- Class periods: Theory blocks Mon/Wed/Fri 18:00–20:00 (capacity 8), Practical slots every 2h between 08:00–18:00 weekdays + 09:00–15:00 Sat.
- Program "Class 5": 12 theory modules, 15 practical lessons, 2 evaluations, 2 road-test prep — total 31 items.
- Learner Sofia: French preferred, automatic transmission, Mon/Wed evenings + Sat morning available.

A call to `generate(template=Default Week, learner=Sofia, date_from=2026-05-26, date_to=2026-07-31)` should produce a proposed schedule that picks Mon/Wed 18:00–20:00 theory blocks first (filling remaining theory items), then weekend practical slots for the practical items, skipping 2026-06-24 and 2026-07-01, with `instructor_id` resolved to an instructor whose `preferred_languages` includes `fr` and whose vehicle is automatic and who is available in those windows.

## Acceptance Criteria

- An owner can build a template with operating hours, class periods, and closures, and persist it.
- An owner can build a program with curriculum items and prerequisites.
- The Generate-Schedule dry-run for a learner returns sessions only on dates the template considers open.
- The generator produces zero blocking conflicts on the proposal (otherwise it returns the conflict list to the UI).
- Cancellation policy is enforced when a learner triggers a cancel from the portal.
- Templates support effective ranges, so a school can have a "Summer Hours" template active 2026-06-15 → 2026-09-01 alongside the default.
- Programs and items can be archived without losing historical session links.
- All endpoints honour tenant scoping and the existing permission model.

## Risks / Open Questions

- **Capacity > 1 for theory blocks.** Need to ensure the generator treats them as multi-learner sessions (one `ScheduledSession` with N learner participants) rather than N separate sessions.
- **Holidays should probably ship with a seeded provincial calendar** per tenant province; until then, owners enter their own closures.
- **Concurrent generation runs** can race for the same slot. Take a tenant-scoped advisory lock during commit.
- **Generator complexity.** MVP is heuristic; aim to allow swapping in OR-Tools later. Persist `schedule_generation_runs.log` for explainability.

## Dependencies

- [06-scheduling.md](./06-scheduling.md) — host for sessions and engine endpoints.
- [05-instructors.md](./05-instructors.md) — effective availability + qualifications + vehicles.
- [04-learners.md](./04-learners.md) — availability windows + program enrollment + holds.
- [15-settings.md](./15-settings.md) — UX home for templates/programs/policies.
- [09-invoices-payments.md](./09-invoices-payments.md) — cancellation fees rely on `cancellation_policies`.
