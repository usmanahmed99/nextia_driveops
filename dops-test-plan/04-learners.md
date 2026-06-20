# 04 - Learners

Covers learner list, registration wizard, learner workspace, holds, documents, path progress,
credentials, portal invite, scheduling preferences, and lifecycle actions.

Primary roles: Owner, Admin, Receptionist.

Read roles: Billing and Scheduler can read where their permissions allow. Instructor should be
tested through own portal/session context, not as a full learner-roster user.

No staff access: Learner.

Preconditions:
- Settings doc 03 is complete enough to provide branches, services, license categories, checkpoints,
  and learning paths.

---

## 04-1 Learner List, Search, Filters, Pagination

Route: `/learners`

Steps:
1. Open the learner list.
2. Search by learner name.
3. Test chips: all, in practical, ready for practical, pending documents, pending payment,
   certificate ready.
4. If enough learners exist, use Load more.
5. Open a learner card.

Expected:
- List uses server-backed filters and cursor loading.
- Cards show name, status, hold badges, program/path, phase/progress, next session, and payment,
  document, and communication status where available.
- Create-capable roles see Register learner.
- Read-only roles do not see mutation controls.

---

## 04-2 Register Learner Happy Path

Route: `/learners/new`

Roles: Owner/Admin/Receptionist

Steps:
1. Start the wizard.
2. Step 1 Identity: enter Liam's name, DOB, and preferred language.
3. Step 2 Contact: enter email, phone, emergency contact, notification contact, communication
   preferences, and notes.
4. Step 3 Enrollment: choose branch, license category, and learning path.
5. Step 4 Transfer: leave transfer off.
6. Step 5 Pickup: enter pickup address and notes.
7. Step 6 Documents: add learner permit metadata and attach a small file if available.
8. Step 7 Confirm: submit.

Expected:
- Wizard prevents skipping required fields.
- Path list filters by selected license category.
- Learner is created and the app navigates to `/learners/:learnerId`.
- Document metadata is created and file upload is attempted after learner creation.
- Contact, enrollment, transfer, and document data appear on the learner workspace.

Current gap to verify:
- Pickup address/notes are collected in the UI, but the current submit payload does not appear to
  include pickup fields. Record whether pickup persists anywhere.

---

## 04-3 Transfer Learner

Roles: Receptionist/Admin

Steps:
1. Register Maya Manual.
2. Turn transfer on.
3. Enter previous school name, optional contact, started/transferred dates, notes.
4. Add completed service counts and completed checkpoints.
5. Submit.

Expected:
- Previous-school data appears on learner detail.
- Completed services/checkpoints influence progress display where implemented.
- Missing previous school name is rejected when transfer is enabled.

---

## 04-4 Minor/Guardian Edge Case

Roles: Admin/Receptionist

Steps:
1. Register Theo Teen.
2. Add parent/guardian as notification contact.
3. Complete registration.

Expected:
- Learner creation succeeds or shows clear age validation if implemented.
- Notification contact is stored for later notification routing.
- No crash on minor DOB.

---

## 04-5 Wizard Validation

Roles: Receptionist

Steps:
1. Try to continue with empty first/last name.
2. Enter invalid email.
3. Enter too-short phone.
4. Submit transfer without previous school when transfer is enabled.
5. Submit without a required learning path/category selection.

Expected:
- Field-level errors are shown.
- Confirm step cannot submit invalid data.
- User can return to earlier steps without losing valid entries.

---

## 04-6 Learner Workspace Overview

Route: `/learners/:learnerId`

Steps:
1. Open Liam.
2. Confirm the new top-down order (this page was reordered: progress-first, config/danger last):
   hero -> progress-detail card -> journey timeline -> recommendations -> sessions + payments ->
   attention items (holds, missing docs, WhatsApp, portal) -> profile/learning-path/scheduling/
   documents/checkpoints/credentials -> consent/archive/danger zone.
3. Confirm the header actions: Send message, Issue certificate, Print schedule, Reschedule.

Expected:
- Workspace loads from current learner workspace API.
- Profile and operational cards are section-level loaded; one failing card should not blank the page.
- Progress and checkpoint data match selected learning path.
- The old 4 metric tiles and the separate Quick actions card are gone (consolidated): theory/
  practical/payment now live in the progress-detail card; certificate issuing is a header action.

---

## 04-7 Edit Learner Profile

Roles: Owner/Admin/Receptionist

Steps:
1. Edit phone, email, notes, language, and other exposed profile fields.
2. Save and reload.

Expected:
- Changes persist.
- Read-only roles do not see edit controls or receive 403 on mutation.

---

## 04-8 Holds

Roles: Owner/Admin/Receptionist

Steps:
1. Add a payment hold with reason.
2. Add a document hold with reason.
3. Resolve one hold with a note.
4. Return to learner list.

Expected:
- Active holds are visible on the learner workspace and list card.
- Resolved holds are no longer active.
- Create/resolve actions require learner update permissions.

---

## 04-9 Documents and Document Review

Roles: Owner/Admin/Receptionist for upload; document reviewers in doc 08 for approval

Steps:
1. Upload a learner document with category, number, issue date, expiry date, and notes.
2. Confirm it appears in the learner Documents card.
3. If review is required, open `/reviews` as a reviewer and approve/reject in doc 08.
4. Delete a disposable document if the UI offers delete.

Expected:
- Documents are scoped to the learner.
- Upload/list metadata works even if dev storage does not provide a downloadable file.
- Review notifications appear for users with `documents.review`.

---

## 04-10 Portal Access

Roles: Owner/Admin

Steps:
1. Open Liam's learner detail page.
2. Use the portal invite card.
3. Resend the invite.
4. Accept the invite as Liam in doc 10.

Expected:
- Invite is linked to Liam's profile id.
- Existing active or invited membership is detected.
- If no learner email exists, the card blocks invite with a clear message.

---

## 04-11 Learning Path Assignment and Preview

Roles: Owner/Admin/Receptionist

Steps:
1. Inspect the assigned path preview.
2. Change path or category if the UI exposes it.
3. Open schedule preview from path.
4. Mark a checkpoint complete and preview again.

Expected:
- Preview shows service rows, checkpoint rows, warnings, blocked or tentative rows, and gate details.
- Completing checkpoints changes blocked/tentative behavior where applicable.
- Changes are reflected in scheduling doc 06.

---

## 04-12 Scheduling Preferences

Roles: Owner/Admin/Receptionist/Scheduler

Steps:
1. Add weekly learner availability.
2. Set instructor gender preference to any, female, male, then specific instructor.
3. Set preferred instructors when specific is selected.
4. Set transmission preference.
5. Set max per week, min days between, max per day.
6. Select preferred times of day.
7. Add notes and time-off exclusions.
8. Save and reload.

Expected:
- Availability, preferences, and exclusions persist.
- Specific instructor preference is available only when instructors exist.
- Generator uses these constraints where implemented.

---

## 04-13 Checkpoints

Roles: Owner/Admin/Receptionist

Steps:
1. Open learner checkpoints card.
2. Mark `learner_permit_obtained` complete.
3. Mark it pending again.

Expected:
- Checkpoint status persists.
- Schedule preview/generator changes contingent/tentative rows based on checkpoint status.

---

## 04-14 Credentials

Roles: Owner/Admin/Receptionist

Steps:
1. Add or edit a learner credential from the credential catalog.
2. Set expiry.
3. Mark it complete.
4. Clear or delete a disposable credential.

Expected:
- Credential state persists.
- Expiry and completion are visible.
- Required credentials can drive document/progress state where implemented.

---

## 04-15 Consent

Roles: Owner/Admin/Receptionist

Steps:
1. Open learner consent card.
2. Grant or renew consent.
3. Reload.

Expected:
- Consent state persists with date/version where shown.
- Renewal updates the visible consent state.

---

## 04-16 Archive and Withdraw

Roles: Owner/Admin for destructive lifecycle actions

Steps:
1. Archive a disposable learner.
2. Confirm list/detail behavior.
3. Withdraw another disposable learner with a reason shorter than 3 characters.
4. Withdraw with a valid reason.

Expected:
- Archive succeeds and changes status/visibility as implemented.
- Short withdraw reason is blocked.
- Valid withdraw returns to the learner list.
- Receptionist cannot perform delete/withdraw-only actions without permission.

---

## 04-17 Manual Message from Learner Detail

Roles: Owner/Admin

Steps:
1. Open learner detail.
2. Click Send message.
3. Choose WhatsApp, SMS, and email in separate attempts if available.
4. Send a short message.

Expected:
- Empty body is blocked.
- Send uses the automations message endpoint.
- WhatsApp may be `config_pending` unless provider is configured.

---

## 04-20 Progress Detail Card (Theory/Practical + Phases + Payment)

Roles: Owner/Admin/Receptionist (`learners.read`)

Steps:
1. Assign a learning path whose items carry programme phases (Settings -> Learning paths, set a
   phase 1-4 per item; QC's 4-phase curriculum). Mark a few sessions completed.
2. Open the learner; find the Progress detail card near the top.

Expected:
- Theory and Practical each show done/required counts and a "N left" remaining figure.
- When items are grouped into named phases, a By phase rollup lists each phase with its completed/
  required; an unphased path shows no empty "—" row.
- Payment progress shows paid vs total with a remaining amount; this renders even when the path has
  no schedulable items but the learner has invoices.
- Backed by `GET /learners/{id}/progress-detail`.

---

## 04-21 Printable Payment Statement

Route: `/learners/:learnerId/payments/print`

Roles: Owner/Admin/Billing

Steps:
1. From the learner's Payments card click Print statement (opens the print route in a new tab).
2. Review the statement, then use Print (browser print-to-PDF).

Expected:
- Header (school + learner + printed date/timezone) and Paid / Outstanding / Total tiles.
- Payment history lists recorded payments (date, method, amount).
- Upcoming payments: when an installment schedule exists, its unpaid installments are listed with
  due dates. When there is NO schedule but an outstanding balance exists, the unpaid invoices are
  listed instead (due date, or "On request" when none) so the balance always appears here.
- Toolbar (Back/Print) is hidden when printing; only the statement prints.

---

## RBAC Negative Checks

### 04-18 Read-only Roles Cannot Mutate

Roles: Billing, Scheduler

Expected:
- Can read permitted learner data.
- Cannot register, edit, hold, withdraw, archive, or upload documents unless permissions allow it.
- Direct API mutations return 403.

### 04-19 Learner Cannot Access Staff Learner Pages

Roles: Learner

Steps:
1. Deep-link to `/learners`.
2. Deep-link to another learner detail route.

Expected:
- Blocked or redirected.
- No other learner data is visible.

