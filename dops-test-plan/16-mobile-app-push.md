# 16 - Mobile App and Push Notifications

Covers the React Native + Expo mobile app (`../dops-mobile`): authentication, role-locked
navigation, the per-role screens, and push notifications.

Source of truth:
- `../dops-mobile/app/**` (expo-router screens), `../dops-mobile/src/lib/**`
- API: existing `/portal/*`, `/schedule/*`, `/learners`, `/documents/*`, `/invoices/*` endpoints,
  plus `POST /portal/devices` for push registration
- `../dops-api/app/services/expo_push_service.py`, `notification_service.py`

Preconditions:
- The app is running (see `../docs/ai-and-mobile-setup/02-run-the-mobile-app.md`). On this machine:
  launch on the **`Pixel_8-_DrivingOps`** AVD with Metro on port **8083** (the `Pixel_8_-_TaxReady`
  AVD is a separate project — do not target it). With both emulators up, open the dev URL on the
  DrivingOps device explicitly (`adb -s <drivingops-emulator> shell am start -a
  android.intent.action.VIEW -d exp://127.0.0.1:8083 host.exp.exponent` after `adb reverse tcp:8083
  tcp:8083`).
- `EXPO_PUBLIC_API_BASE_URL` points at a reachable dops-api with seed data **(the same instance the
  web flow in doc 17 used)**.
- `EXPO_PUBLIC_ALLOW_DEMO_LOGIN=true` for the demo-login tests, OR Entra configured for the
  Microsoft sign-in test.
- For push: a **physical device** (Expo push tokens are not issued on simulators/emulators) and
  `EXPO_PUSH_ENABLED=true` on the API.

---

## Branding and sign-in screen

### 16-0 App branding and sign-in copy

Steps:
1. Inspect the launcher icon and app name after install.
2. Cold-launch the app and watch the splash, then the sign-in screen.

Expected:
- Launcher shows the **DrivingOps app icon** (rounded navy plate with the blue "D" mark), not the
  default Expo icon; the label reads "DrivingOps".
- The splash is the navy brand field with the reverse-white wordmark.
- The sign-in screen shows the **DrivingOps horizontal logo** (blue mark + "Driving" navy / "Ops"
  blue), not a generic car glyph.
- The primary button reads **"Sign in"** (the Microsoft/identity-provider step is the *next*
  screen, so the button must not say "Sign in with Microsoft").
- The tagline reads "Your driving school, in your pocket."

---

## Authentication and role routing

### 16-1 Demo login routes to the correct stack

Steps:
1. Launch the app → Sign in screen.
2. Tap "Learner" under Demo login.
3. Repeat in fresh launches for "Instructor" and "Staff (School Admin)".

Expected:
- Learner lands on the learner tabs (Schedule / My path / Documents / Profile).
- Instructor lands on instructor tabs (Today / My learners / Availability / Profile).
- Staff lands on staff tabs (Schedule / Learners / Reviews / More).
- A learner can never see staff/instructor tabs (role-locked).

### 16-2 Session persists across relaunch

Steps:
1. Sign in, then fully close and reopen the app.

Expected:
- The app restores the session from secure storage and lands on the role stack without re-login.
- If the stored token is invalid, it falls back to the sign-in screen.

### 16-3 Microsoft (Entra) sign-in availability

Steps:
1. With Entra not configured, observe the **"Sign in"** button and the note beneath the card.
2. With Entra configured, tap **"Sign in"**.

Expected:
- The button is labelled "Sign in" (not "Sign in with Microsoft"); Microsoft/Entra is the IdP on
  the *next* screen once the flow is initiated.
- Disabled with a "Microsoft sign-in is not configured in this build yet" note when not configured;
  initiates the Entra flow when configured.

### 16-4 Sign out

Steps:
1. From Profile (or staff More), tap Sign out.

Expected:
- Returns to the sign-in screen; relaunch does not restore the session.

---

## Learner stack

### 16-5 Learner home

Expected:
- Greeting + program, overall progress, upcoming lessons with status badges.
- If invoices are outstanding and payments are enabled, an outstanding-balance card shows "Pay now".
- Pull-to-refresh refetches.

### 16-6 Pay now

Steps:
1. Tap "Pay now" on an outstanding invoice.

Expected:
- The device opens the Stripe Checkout URL in the browser.
- "Pay now" is hidden when payments are not enabled for the tenant.

### 16-7 Learning path and documents

Expected:
- Learning path shows theory/practical progress bars and recommendations.
- The Practical card now shows a "Remaining" count (required minus completed lessons).
- When the learner has outstanding invoices, an "Upcoming payment" card lists the next due
  installment(s) with amount (and due date when present); "Pay now" follows the existing flow once
  the invoice is issued.
- Documents lists submitted documents with review-status badges.

---

## Instructor stack

### 16-8 Today and attendance

Steps:
1. As an instructor with a session dated today, open Today.
2. Tap "Present" on the session; on another, tap "No-show".

Expected:
- Today lists today's sessions sorted by time with status badges.
- Present/No-show appear only for sessions today-or-past and not already terminal.
- After marking, the card's status updates (cache invalidated).
- Marking attendance for a session that is not yours, or before its day, is rejected by the API.

### 16-9 My learners (scoping)

Expected:
- Lists only learners the instructor shares a session with.
- Shows program, phase, and an overall-progress bar — **no contact or financial data**.

---

## Staff stack

### 16-10 Schedule and learners

Expected:
- Schedule shows today's sessions across instructors.
- For a staff member with `reports.read`, a "School summary" card appears with Revenue today,
  Tomorrow's sessions, Completed today, and Unpaid (from the branch-summary endpoint). It is hidden
  for staff without `reports.read`.
- Learners is searchable; results show status, payment, and document badges.

### 16-11 Document reviews

Steps:
1. Open Reviews with at least one pending document.
2. Tap Approve on one, Reject on another.

Expected:
- The queue lists pending documents with owner names.
- After a decision, the card drops out of the queue (cache invalidated).

### 16-12 School setup is web-only

Steps:
1. Open staff More → tap any setup area (Branding, Branches, Roles, Plans, Programs).

Expected:
- An interstitial appears explaining setup is in the web app, with a button that opens
  `EXPO_PUBLIC_FRONTEND_URL`. No in-app editor is shown.

---

## Push notifications

### 16-13 Permission prompt and registration

Steps:
1. On a physical device, sign in for the first time.

Expected:
- The OS notification-permission prompt appears.
- On grant, the app registers an Expo token (`POST /portal/devices` → 201). A `device_tokens`
  row exists for the user.
- On deny / on a simulator, the app continues normally with push off (no crash).

### 16-14 Receiving a push

Steps:
1. With `EXPO_PUSH_ENABLED=true`, trigger a session event for that user (e.g. cancel/reschedule a
   session the learner or instructor is on — doc 06).

Expected:
- A push arrives on the device with the event title/body.
- Tapping it opens the app.
- The corresponding `notifications` row has channel `expo_push` and status `sent`.

### 16-15 Push falls back gracefully

Steps:
1. Trigger an event for a user who has **no** registered device.

Expected:
- The `expo_push` notification row is recorded as `config_pending` (nothing to send to); other
  channels (email/WhatsApp) are unaffected.

### 16-16 Sign-out unregisters

Steps:
1. Sign out.

Expected:
- `DELETE /portal/devices/{token}` is called while still authenticated; the `device_tokens` row
  is removed, so the device stops receiving pushes for that account.
