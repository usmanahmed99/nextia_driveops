# 13 — Learner & Instructor Portals

## Purpose

Mobile-first self-service surfaces. Learners see their plan, next session, payments, documents, certificate, and a support chat. Instructors see their day, navigation, attendance, lesson notes, and a way to mark themselves unavailable.

## Current State

### UX

- [`LearnerPortalPage.tsx`](../dops-frontend/src/features/portals/LearnerPortalPage.tsx) — header, next session card, journey timeline, payments/documents/resources/certificate cards, sticky bottom bar with **Book** and **Support**.
- [`InstructorPortalPage.tsx`](../dops-frontend/src/features/portals/InstructorPortalPage.tsx) — header, next-lesson card (time, location, focus, pickup notes, **Open route**), today's session list, sticky bottom bar **Attendance / Lesson note / Unavailable**.
- Both use a hard-coded learner/instructor id (`lea-1`, `ins-1`) for the demo.

### API

There is no portal-specific API today. The pages call the same admin endpoints with a hard-coded id.

## Gaps For Real Implementation

1. **No auth scoping**. A logged-in learner should only see *their* data; an instructor should only see *their* sessions.
2. **No real "Book or reschedule" flow** from learner portal — it should call scheduling endpoints with policy checks.
3. **No real "Support" chat** — should open the learner's WhatsApp conversation thread.
4. **No real "Open route"** — should open `maps:` deep link with pickup coords.
5. **No attendance / lesson-note write endpoints**.
6. **No "Mark unavailable"** writes to instructor availability exceptions.
7. **PWA install** advertised in README but not implemented.
8. **No mobile-push** (out of scope) but in-app toasts when state changes (delivery receipts, new offers) would be needed.

## Real Implementation Plan

### A. Portal-scoped endpoints

```
# Learner portal (auth = LEARNER membership in tenant)
GET    /api/v1/portal/learner/me                       returns { learner, upcoming_sessions, balance, documents_status, certificate_status }
GET    /api/v1/portal/learner/sessions?from=&to=
POST   /api/v1/portal/learner/sessions/{id}/cancel     body: { reason }       enforces cancellation_policies
POST   /api/v1/portal/learner/sessions/{id}/reschedule body: { requested_starts_at_options[] }
POST   /api/v1/portal/learner/availability-windows     body: list of windows
POST   /api/v1/portal/learner/documents/upload-link    body: { doc_type, file_name, mime_type, size_bytes }
GET    /api/v1/portal/learner/invoices
POST   /api/v1/portal/learner/payments                 body: { invoice_id }   -> Stripe PaymentIntent client_secret
GET    /api/v1/portal/learner/certificates
GET    /api/v1/portal/learner/conversation             returns whatsapp_conversation
POST   /api/v1/portal/learner/conversation/messages    body: { body }         queued via worker

# Instructor portal (auth = INSTRUCTOR membership)
GET    /api/v1/portal/instructor/me
GET    /api/v1/portal/instructor/sessions?date=
POST   /api/v1/portal/instructor/sessions/{id}/attendance         body: { learner_attended: bool, instructor_attended: bool, notes? }
POST   /api/v1/portal/instructor/sessions/{id}/complete           body: { notes, lesson_focus, evaluation_score? }
POST   /api/v1/portal/instructor/availability-exceptions          body: { date, type, starts_at?, ends_at?, reason }
POST   /api/v1/portal/instructor/sessions/{id}/cancel             body: { reason }   when allowed by policy
```

`/portal/*` endpoints derive the acting learner/instructor from the session — no path id required. They reject if the session role mismatches.

### B. UX wiring

- Replace `useLearnerWorkspace("lea-1")` with `usePortalLearnerMe()`.
- Cancellation modal enforces `policy.window_hours` and shows fee if late.
- Reschedule shows up to three legal slot proposals from the engine, or opens a contact-school message.
- Documents card lets learners upload missing docs via the SAS flow.
- Payments card opens Stripe Elements inline.
- Support → opens the conversation thread; sends/receives via the worker.
- Open route → `https://www.google.com/maps/dir/?api=1&destination=<lat,lng>` from the session's pickup geocode.

### C. Push & PWA

- Add a real service worker + manifest with offline shell of /portal routes.
- Web Push (VAPID) for session reminders 2h before start — opt-in toggle in portal settings.

## Acceptance Criteria

- A learner logged in as Sofia sees only Sofia's data; switching tabs never leaks a sibling learner.
- A learner cancelling within the policy window sees a confirmation + no fee; outside the window the modal warns and applies the fee.
- An instructor marking themselves unavailable for tomorrow creates an availability exception and triggers reschedule flow for affected sessions.
- Lesson note + attendance save in < 1 s offline-first (queued by service worker, replayed on reconnect).

## Dependencies

- [03-authentication.md](./03-authentication.md) — real identity required.
- [06-scheduling.md](./06-scheduling.md), [05-instructors.md](./05-instructors.md) — write endpoints.
- [09-invoices-payments.md](./09-invoices-payments.md) — portal payments.
- [08-automations.md](./08-automations.md) — conversation thread.
