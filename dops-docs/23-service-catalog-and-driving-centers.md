# 23 — Service Catalog, Driving Centers, and Slot-Gap Configuration

> **Status**: spec + first backend slice in flight (this iteration).

## Purpose

Real driving schools sell more than driving lessons. They run their own theory and practical lessons at branches, and they also offer **driving-test booking and ride-along services at government or partner test centers** (e.g. SAAQ service centers in Quebec). Each thing they sell has its own duration, location, fee, and confirmation message. Admins also need to set the **gap between consecutive slots** so an instructor can travel, refuel, or simply breathe.

This document captures three coupled features:

1. **Driving centers** — first-class records for any location where a service can take place: a school branch, a SAAQ service center, a partner test track, etc.
2. **Service catalog** — per-tenant configurable services (theory module, practical lesson, road-test prep, mock evaluation, official road-test booking, custom services). Each has duration, default location, fee, buffer minutes after, and other configurables.
3. **Slot-gap configuration** — school-level default minimum gap between consecutive sessions, with per-instructor overrides.

A successful booking emits a `booking.created` event that the automation worker turns into a bilingual confirmation message with configurable details.

## Why these belong together

- Sessions today only have `branch_id` + a free-text `location`. The scheduling engine cannot pick a service-appropriate venue when it has no concept of "this service runs at the SAAQ center."
- Sessions today derive `type` from a fixed enum (`theory_module | practical_lesson | …`). That cannot represent "Class 5 fast-track road-test prep" or any school-specific offering.
- The cancellation policy on schedule templates already controls one schedule-time guardrail; the slot-gap is the natural sibling.
- A booking confirmation requires (service, time, location, fees) all in one row — which is exactly what the joined model produces.

## Personas & Workflows

- **School owner / admin** opens Settings → **Services** to define what they sell, with prices and durations. Opens Settings → **Driving centers** to register SAAQ-style external test centers and verify their address + operating hours.
- **Scheduling page** is now service-aware. When generating or creating a session, the admin picks a service. The duration, default location, fee, and required vehicle flow automatically.
- **Learner portal** (later phase): browses bookable services. Clicks Book → confirmation message in their preferred language with: service, instructor, pickup, address, fee, cancellation policy.
- **Instructor portal** (later phase): sees a session card including the service name + center address + any pre-arrival notes (e.g. "bring health card").

## Data Model

> The first backend slice adds the tables and columns below. Tenant scoping enforced via `TenantScopedMixin`.

### `driving_centers`

```sql
driving_centers (
  id pk, tenant_id,
  name,
  type,                          -- enum: 'school_branch' | 'saaq_service_center' |
                                 --  'partner_center' | 'office' | 'other'
  address, city, province, postal_code,
  timezone,                      -- 'America/Toronto'
  phone null, email null,
  is_owned_by_tenant boolean,    -- true for school's own branches
  branch_id fk null,             -- links to branches table when owned
  is_active boolean default true,
  notes null,
  audit cols
)
```

A tenant's own branches are mirrored here with `type='school_branch'` and `branch_id` set (kept in sync by the service). External test centers have `is_owned_by_tenant=false`, `branch_id` null.

### `service_offerings`

```sql
service_offerings (
  id pk, tenant_id,
  code,                          -- 'class_5_theory' | 'class_5_practical' |
                                 --  'road_test_prep' | 'mock_road_test' |
                                 --  'official_road_test_booking' | etc.
                                 -- unique per tenant
  name, name_fr,                 -- bilingual display name
  description null, description_fr null,
  service_kind,                  -- enum: 'lesson' | 'evaluation' | 'road_test' | 'admin'
  duration_minutes int,
  buffer_after_minutes int default 0,    -- mandatory gap after the session
  default_location_type,         -- enum: 'branch' | 'driving_center' | 'on_road' | 'remote'
  default_branch_id fk null,
  default_driving_center_id fk null,
  base_fee_cents int default 0,
  currency default 'CAD',
  tax_code,                      -- enum: 'standard' | 'exempt'
  requires_vehicle boolean default false,
  requires_instructor boolean default true,
  bookable_directly boolean default true,
  confirmation_template_code null,        -- pairs with message_templates.code (Phase 3)
  active boolean default true,
  audit cols
)
```

### Slot-gap configuration

Two locations:

1. **Schedule template** ([07-school-schedule-templates.md](./07-school-schedule-templates.md)):
   ```sql
   schedule_templates ADD COLUMN default_slot_gap_minutes int default 15
   ```
2. **Instructor**:
   ```sql
   instructors ADD COLUMN slot_gap_minutes int null
   ```
   `null` → fall back to the active schedule template default.

The scheduling engine reads:

```
effective_gap(instructor, template) =
  instructor.slot_gap_minutes ?? template.default_slot_gap_minutes ?? 15
```

The **service's `buffer_after_minutes`** is the minimum block after a particular service (often longer for road-test-prep because of test-center logistics). The **slot gap** is the minimum block between any two sessions of any service on the same instructor. The engine uses `max(service.buffer_after_minutes, effective_gap)`.

### Sessions reference services

```sql
scheduled_sessions
  ADD COLUMN service_offering_id fk null,
  ADD COLUMN driving_center_id fk null,
  ADD COLUMN fee_cents int null,
  ADD COLUMN currency null,
  ADD COLUMN buffer_after_minutes int default 0,
  ADD COLUMN pickup_notes null
```

Existing `type` enum stays for backwards compatibility — for sessions created from a service offering, `type` is derived from `service_kind`. New sessions go through the service-offering path; the old `CreateSessionRequest` continues to work.

## API Surface (first slice — backend)

### Driving centers

```
GET    /api/v1/driving-centers?type=&is_active=
POST   /api/v1/driving-centers              settings.manage
GET    /api/v1/driving-centers/{id}
PATCH  /api/v1/driving-centers/{id}         settings.manage
DELETE /api/v1/driving-centers/{id}         settings.manage   (soft delete by is_active=false if referenced)
```

### Service offerings

```
GET    /api/v1/services?service_kind=&active=
POST   /api/v1/services                     settings.manage
GET    /api/v1/services/{id}
PATCH  /api/v1/services/{id}                settings.manage
DELETE /api/v1/services/{id}                settings.manage   (soft delete)
```

### Slot-gap

Already part of the schedule-template body (added when `0003` migrates `default_slot_gap_minutes`) and the instructor update payload (`slot_gap_minutes`).

### Bookings (future slice — depends on automation worker)

```
POST   /api/v1/services/{id}/bookings       learners.update | learner_portal_token
  body: { learner_id, starts_at, instructor_id?, vehicle_id?, driving_center_id? }
  -> Persists a ScheduledSession with service_offering_id, fee, location resolved from
     service defaults + body overrides. Emits booking.created event.
```

## Booking confirmation flow

When a session is created through the booking path:

1. The service writes the session.
2. The service emits `booking.created` with payload:
   ```json
   {
     "session_id": "...",
     "service_offering_id": "...",
     "learner_id": "...",
     "instructor_id": "...",
     "starts_at": "...",
     "ends_at": "...",
     "location_name": "SAAQ Service Center Montreal",
     "location_address": "855 Boulevard Henri-Bourassa O.",
     "fee_cents": 12500,
     "currency": "CAD",
     "confirmation_template_code": "booking_confirmation_road_test"
   }
   ```
3. The worker ([08-automations.md](./08-automations.md)) consumes the event, resolves the `confirmation_template_code` for the learner's preferred language, renders, and dispatches via WhatsApp + email.
4. Delivery state lands back on `automation_runs` with a `session_id` correlation.

All confirmation copy is per-service-template, configurable from Settings → Message Templates (which is part of Phase 3).

## Frontend Surface (next slice after backend lands)

- **Settings → Driving centers** — list + add/edit. Map preview (Leaflet/Maps optional, address text required).
- **Settings → Services** — list + add/edit. Fields: code, bilingual name, kind, duration, buffer, default location (branch picker or driving-center picker), fee, currency, tax code, requires-vehicle, requires-instructor, confirmation template code.
- **Settings → Schedule template** — new field `default_slot_gap_minutes`.
- **Instructors detail** — new optional field `slot_gap_minutes`.
- **Scheduling page** — when creating a session, select a service first; duration/location/fee/buffer auto-populate; admin can still override per-session.
- **Learner portal** — service catalog browser (later phase).

## Gaps / Open Questions

- **Package pricing per learning path**: in scope for learning paths/programs. The school can either set a fixed package price for the whole path or enter a discount applied to the total of all service items. More complex per-learner pricing tiers can come later as `service_pricing_overrides`.
- **Vehicle requirements per service**: today `requires_vehicle` is a boolean. If a service needs a *specific* transmission (manual vs auto) or an accessible vehicle, that belongs on the service offering. Out of scope for this first slice.
- **Capacity > 1**: theory classes typically host multiple learners per session. Right now each session has one `learner_id`. The plan in [07-school-schedule-templates.md](./07-school-schedule-templates.md) already mentions a participants array; we'll lean on it when the scheduling engine work happens.
- **External center calendars** (e.g. SAAQ slots): the engine can't see them today. The school records bookings manually; integration is a long-term roadmap item.
- **Multi-day services**: not in scope. All offerings are single-block.

## Slice plan

### Backend (this iteration)

- [x] Migration `0003_driving_centers_service_offerings` adds tables + columns + indexes.
- [x] SQLAlchemy models, Pydantic schemas, repositories.
- [x] Services + endpoint wiring with audit logging + domain events.
- [x] Seed updates for NorthStar (3 driving centers, 5 service offerings).
- [x] Tests covering CRUD + tenant scoping + soft delete.

### Frontend (next iteration)

- Settings → Driving centers page (list + dialog).
- Settings → Services page (list + dialog).
- Slot-gap field on schedule-template + instructor edit.
- Scheduling page "service picker" prefilling new sessions.

### Booking confirmation (later — couples with Phase 3 automation worker)

- Add `confirmation_template_code` field to service offering form.
- Worker consumer `handlers/booking_created.py` that renders the configured template.
- Settings → Message Templates UI.

## Dependencies

- [02-multitenancy-rbac.md](./02-multitenancy-rbac.md) — tenant scoping enforced by repository pattern.
- [05-instructors.md](./05-instructors.md) — `slot_gap_minutes` column lands on the instructors table.
- [07-school-schedule-templates.md](./07-school-schedule-templates.md) — `default_slot_gap_minutes` lands on schedule templates; cross-references the service catalog.
- [08-automations.md](./08-automations.md) — booking confirmation flow hooks into the worker spec.
- [15-settings.md](./15-settings.md) — new left-nav sections for Driving centers + Services.
