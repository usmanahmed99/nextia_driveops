# 06 - Scheduling

Covers schedule calendar, service-aware manual booking, click-to-place booking, generation
preview/commit, conflicts, attendance, cancellation requests, staff cancellation, tentative
contingencies, and current API-only gaps.

Scheduling is staff-driven. Learner self-service booking is not currently the main workflow.

Primary staff roles: Owner, Admin, Receptionist, Scheduler.

Read/own roles: Billing can read; Instructor and Learner should be tested through own portal
surfaces in doc 10.

Preconditions:
- Profile timezone, driving centers, opening hours, closures, license categories, checkpoints,
  services, learning paths, vehicles, learners, instructors, availability, and preferences exist.

---

## Current Capability Map

| Capability | Current state |
|---|---|
| Calendar views | UI supports day, week, and month. |
| Filters | UI supports learner and instructor filters. |
| Manual session creation | UI supports service-aware create session. |
| Click free time band | UI can open create dialog prefilled from calendar resource/time. |
| Batch generation | UI supports dry-run preview, regenerate, and commit. Inputs: learner(s) + date range + optional service scope (empty = the learner's whole learning path). See the Scheduling Scenario Matrix below. |
| Rollback generation | Backend endpoint exists; current dialog does not expose rollback. |
| Attendance | UI supports Present and No-show on day-of or past sessions. |
| Cancellation | UI supports request cancellation and staff approve/cancel/decline controls. |
| Reschedule | Backend endpoint exists; current visible staff reschedule UI should be verified and logged if absent. |
| Tentative reconcile | Backend/worker endpoint exists; test as API/ops unless UI appears. |

---

## 06-1 Calendar Loads

Route: `/schedule`

Steps:
1. Open schedule as Owner/Admin.
2. Switch day, week, and month views.
3. Use previous, today, next, and date picker controls.
4. Apply learner filter.
5. Apply instructor filter.

Expected:
- Calendar loads from `/calendar/availability`.
- Day/week show resource grid with free bands and booked sessions.
- Month shows day density.
- Filters scope sessions/availability without crashing.
- Empty days show usable empty state, not a blank page.

---

## 06-2 Create Session from Header

Roles: Owner/Admin/Receptionist/Scheduler

Steps:
1. Click Create session.
2. Select service `in_car_60`.
3. Confirm duration, buffer, fee, and default location values populate from the service.
4. Select Liam, Ivan, start time, and auto vehicle.
5. Save.

Expected:
- Active services are available.
- Inactive services are not available.
- Duration/buffer/fee/default driving center fields update when service changes.
- Learner and active instructor selectors work.
- Session appears on the calendar.
- If service is priced and firm, invoice automation may create invoice data for doc 07.

---

## 06-3 Create Session from a Free Calendar Band

Roles: Owner/Admin/Receptionist/Scheduler

Steps:
1. Open week or day view.
2. Click an available free band for Ivan.
3. Complete the create session dialog.

Expected:
- Dialog opens with instructor and time prefilled.
- Saving creates a session at the selected time.
- Cancel/close does not create a session.

---

## 06-4 Explicit Vehicle and Location

Roles: Admin/Scheduler

Steps:
1. Create an in-car session with explicit Toyota Corolla.
2. Create a road-test-prep session with driving center location if the service requires it.
3. Create a theory/admin service that does not require a vehicle.

Expected:
- Explicit vehicle is stored and shown on session card.
- Driving center field appears when service/location requires it.
- Vehicle is optional or hidden for services that do not require vehicles.

---

## 06-5 Conflict Detection

Roles: Admin/Scheduler

Steps:
1. Book Ivan in one slot.
2. Try to book Ivan in an overlapping slot.
3. Try to book Liam in overlapping sessions.
4. Try to reuse a vehicle in overlapping sessions.

Expected:
- Double-booking is blocked or flagged.
- Conflict details identify instructor, learner, or vehicle cause where available.
- Existing sessions remain unchanged after failed create.

---

## 06-6 Generate Schedule Dry Run

Roles: Owner/Admin/Receptionist/Scheduler

Steps:
1. Click Generate.
2. Select Liam and Maya.
3. Use date from tomorrow and date to about 30 days from today.
4. Run preview.

Expected:
- Preview groups proposed sessions by day.
- Proposed sessions are not in the past.
- Unscheduled list appears for items that cannot be placed.
- Conflict count appears.
- Vehicle name is shown on practical sessions when matched.
- Maya's specific-instructor preference constrains her proposals to Ivan where possible.
- Closures, instructor exclusions, vehicle blackouts, category, transmission, availability, and
  learner spacing preferences are respected where implemented.

---

## 06-7 Tentative Checkpoint-Gated Proposals

Roles: Admin/Scheduler

Steps:
1. Ensure Liam has an unmet checkpoint that gates in-car lessons.
2. Generate a path-based schedule.
3. Inspect proposed sessions after that checkpoint.

Expected:
- Gated sessions are marked tentative or clearly warned instead of silently scheduled as firm.
- The preview explains what checkpoint/gate is assumed.
- Completing the checkpoint in learner detail changes future preview behavior.

---

## 06-8 Commit Generated Schedule

Roles: Admin/Scheduler

Steps:
1. From a successful preview, click Commit.
2. Return to calendar.
3. Inspect generated sessions.

Expected:
- Proposed sessions become real scheduled sessions.
- Tentative sessions include auto-cancel timing where applicable.
- Calendar reflects the committed run.
- Learner and instructor portal schedules reflect the committed sessions for their own profiles.

---

## 06-9 Regenerate Preview

Roles: Admin/Scheduler

Steps:
1. Run a dry preview.
2. Click Regenerate.

Expected:
- A fresh preview replaces the previous proposal.
- No real sessions are created until Commit.

---

## 06-10 Generation Rollback - API/Ops Gap

Roles: Admin or API tester

Steps:
1. Commit a generation run in a disposable tenant.
2. Look for a rollback button in the UI.
3. If no UI button is present, test backend rollback with the generation run endpoint if needed.

Expected:
- Current Generate dialog may not expose rollback.
- Backend rollback should remove sessions materialized by that generation run.
- Record missing UI rollback as a current gap, not a failed backend test.

---

## 06-11 Generator No-Coverage Case

Roles: Admin/Scheduler

Steps:
1. Use Theo or another learner whose category has no valid instructor/vehicle coverage.
2. Generate across a valid range.

Expected:
- The item is listed as unscheduled.
- The reason mentions unavailable instructor/vehicle/category coverage where available.
- No invalid session is created.

---

## 06-12 Attendance

Roles: Owner/Admin/Receptionist/Scheduler and assigned Instructor in portal

Steps:
1. Open a session on today's date or in the past.
2. Mark Present.
3. Mark No-show on another session.
4. Open a future session.

Expected:
- Present sets status to completed.
- No-show sets status to no_show/no show.
- Attendance buttons are unavailable for future and terminal sessions.
- Instructor can only mark own assigned sessions.
- Learner has no attendance controls.

---

## 06-13 Staff Cancellation

Roles: Owner/Admin/Receptionist/Scheduler

Steps:
1. Open a future scheduled session.
2. Click staff cancel.
3. Choose a reason code such as learner_request, instructor_unavailable, weather, vehicle_issue,
   no_show, holiday, or other.
4. Add reason text and submit.

Expected:
- Session status becomes cancelled.
- Reason and actor are persisted where shown.
- Notifications are created for affected recipients.
- Terminal sessions cannot be cancelled again.

---

## 06-14 Cancellation Request and Decline

Roles: Learner/Instructor for request, staff for approval/decline

Steps:
1. As learner in portal, request cancellation for own future session.
2. As instructor in portal, request cancellation for own assigned session.
3. As staff, inspect the pending request.
4. Decline one request.
5. Approve/cancel another request.

Expected:
- Portal users can request cancellation only for own sessions.
- Request does not immediately cancel the session.
- Staff can decline, clearing the request.
- Staff can approve by cancelling the session.
- Notifications appear in doc 08.

---

## 06-15 Cancellation Window Policy

Roles: Learner/Instructor and staff

Steps:
1. Set policy cancellation window to 24 hours.
2. Request cancellation less than 24 hours before a session.

Expected:
- UI/API reflects late-cancellation behavior where implemented.
- If only backend enforces it, record exact API response.
- If no warning/enforcement appears, log as a policy enforcement gap.

---

## 06-16 Reschedule - Current UI/API Check

Roles: Staff and assigned Instructor

Steps:
1. Look for reschedule controls on session cards, learner detail quick actions, and schedule page.
2. If visible, reschedule a future session to Tuesday 15:00 with reason `instructor sick`.
3. If not visible, verify backend reschedule endpoint separately if API testing is in scope.

Expected:
- If UI is present, new time persists and old slot is freed.
- Notifications are created.
- Instructor reschedule capability is own-session scoped.
- If no UI is present, record as current UI gap while noting backend endpoint exists.

---

## 06-17 Terminal State Guard

Roles: Admin/Scheduler

Steps:
1. Try to cancel, reschedule, or mark attendance again on completed/cancelled/no-show sessions.

Expected:
- Terminal sessions are locked from invalid state changes.
- UI hides controls or API returns a clear error.

---

## 06-18 Contingency Reconcile - API/Ops

Roles: Admin/API tester

Steps:
1. Create or seed tentative sessions gated by an unmet checkpoint.
2. Move time past auto-cancel deadline or use suitable test data.
3. Trigger `/schedule/reconcile-contingencies` or the worker/internal endpoint if configured.
4. Repeat after marking checkpoint complete.

Expected:
- Met checkpoint promotes/keeps sessions active as implemented.
- Unmet checkpoint past deadline cancels contingent sessions.
- Upcoming unmet contingencies can produce reminders.
- Worker automation requires environment key/config; record environment blockers.

---

## 06-19 Path Preview from Learner Detail

Roles: Owner/Admin/Receptionist/Scheduler

Steps:
1. Open learner detail.
2. Use schedule preview from the assigned path.
3. Compare preview rows to schedule generator output.

Expected:
- Preview shows services, checkpoints, blocked/tentative rows, and warnings.
- Completing checkpoints changes preview output.

---

## Scheduling Scenario Matrix

The generator takes **learner(s) + date range + optional service scope**. Leaving the service scope
empty generates the learner's **whole learning path**; selecting services generates only those.
Run each scenario below, record the proposed/unscheduled/conflict counts, then Commit one and verify
the calendar + both portals.

| # | Scenario | Inputs | What it isolates | Expected |
|---|---|---|---|---|
| A | Full learning-path generation | One learner with a full-course path; **no** service scope; range = next 30 days | End-to-end path planning | Proposals cover the path's services in sequence; checkpoints appear as gates; spacing/weekly caps respected. |
| B | Practice-only path | A `practice_only` learner; no service scope | Path-kind shaping | Only practical lessons proposed; no theory/checkpoints beyond what the path defines. |
| C | Service-scoped generation | One learner; pick a single service (e.g. `in_car_60`) | Service filter | Only that service is proposed; other path items are ignored. |
| D | Multi-learner batch | Two-three learners at once; no service scope | Shared resource contention | Proposals avoid instructor/vehicle double-booking across learners; conflicts surfaced, not silently dropped. |
| E | Specific-instructor preference | Maya (prefers Ivan); no service scope | Preference honouring | Maya's practical proposals use Ivan where possible; note any fall-back to others. |
| F | Checkpoint-gated path | Learner with an unmet checkpoint that gates in-car | Gating | Sessions after the gate are tentative/contingent or warned, not firm; preview explains the assumed checkpoint. |
| G | Course-in-path | Path includes a Course item (doc 12) whose exam marks a checkpoint that gates later steps | Course->checkpoint->gate chain | Later steps stay gated until the course exam is passed; after passing, regenerate shows them unblocked. |
| H | No-coverage learner | Learner whose category has no valid instructor/vehicle (e.g. Theo) | Coverage failure | Item listed as unscheduled with a coverage reason; no invalid session created. |
| I | Closure/holiday in range | Range spans a school closure (doc 03) | Closure respect | No proposals land on the closed date(s); branch-scoped closures only block that branch. |
| J | Instructor time-off in range | An instructor has an availability exclusion in range | Exclusion respect | No proposals use that instructor during the exclusion. |
| K | Narrow availability | Learner availability limited to e.g. Sat 09:00-12:00 | Availability windows | Proposals fall only within the learner's available windows. |
| L | Long horizon | Range = 90 days | Spacing/cap over time | Weekly/daily caps and minimum spacing hold across the whole horizon; no clustering. |
| M | Buffer / slot gap | Service buffer-after + per-instructor slot gap set (doc 03) | Spacing between sessions | Consecutive sessions for an instructor honour buffer + slot gap (no back-to-back without the gap). |

For each committed scenario also verify:
- The committed sessions appear on `/schedule` and in the **learner** and **instructor** portals
  for the right profiles only.
- Tentative/contingent sessions show their auto-cancel timing.
- Re-running Generate after Commit does not duplicate already-placed sessions.

---

## 06-19b Manual vs Generated - Side by Side

Roles: Admin/Scheduler

Steps:
1. For one learner, **manually** create two sessions (06-2 / 06-3).
2. For the same learner, run **path generation** (Scenario A) over an overlapping range.

Expected:
- The generator accounts for the manually created sessions (no double-book of the same
  learner/instructor/vehicle/time).
- Manual sessions and generated sessions coexist on the calendar and are distinguishable where the
  UI marks generation provenance.
- Counts in the preview exclude already-satisfied manual bookings where the generator dedupes.

---

## 06-19c Instructor Session Clustering (Best-Effort Adjacency)

Roles: Admin/Scheduler

Steps:
1. Set up one instructor with a wide availability window on a day and a learner needing several
   sessions that the same instructor can teach.
2. Run path generation over that range and inspect the proposed/committed sessions for that
   instructor and day.

Expected:
- The generator prefers placing the instructor's sessions back-to-back (adjacent, allowing the
  service buffer) instead of scattering them across the day, when slots allow.
- Adjacency is best-effort only: it never violates instructor/learner availability, school opening
  hours, closures, or the per-day/per-week caps. If a contiguous block is impossible, sessions are
  still placed (just not clustered).
- Same-instructor continuity across the path is preserved (a learner keeps one familiar instructor
  where possible).

---

## RBAC Negative Checks

### 06-20 Billing Cannot Manage Calendar

Expected:
- Billing can read schedule where permitted.
- No Create, Generate, staff cancel, attendance, or reschedule controls.
- Direct mutations return 403.

### 06-21 Instructor Cannot Manage as Staff

Expected:
- Instructor uses own portal/schedule context.
- No staff Create, Generate, staff cancel/approve controls.
- Own attendance/cancellation/reschedule capabilities are object-scoped.

### 06-22 Learner Cannot Manage Calendar

Expected:
- Learner sees own portal schedule only.
- No staff schedule route management.
- Cannot see other learners' or instructors' sessions.

