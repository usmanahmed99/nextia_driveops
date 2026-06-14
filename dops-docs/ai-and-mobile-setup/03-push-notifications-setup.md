# 03 — Push Notifications: Setup & Verification

> **Status**: code shipped, configuration pending. Device registration
> (`POST /portal/devices`), the `expo_push` delivery channel in `NotificationService`, the
> `ExpoPushService`, and the mobile registration flow are all implemented. Push is **off by default**
> (`EXPO_PUSH_ENABLED=false`) and other notification channels (email/WhatsApp) are unaffected. This is
> how to turn it on and verify end to end.

## How it works

1. After login, the mobile app asks for notification permission, mints an **Expo push token**, and
   calls `POST /portal/devices` to register it (stored in `device_tokens`, one row per device, keyed
   by the unique token). Sign-out calls `DELETE /portal/devices/{token}`.
2. When a session event fires (cancel / reschedule / reminder), `NotificationService` adds an
   `expo_push` channel (when enabled) alongside email/WhatsApp and sends to the recipient's
   registered devices via **Expo's push service** (`ExpoPushService`). Expo needs **no secret** — a
   valid token is the credential.
3. A recipient with no registered device records the `expo_push` notification as `config_pending`;
   nothing breaks.

## API configuration

The single API switch is the env var `EXPO_PUSH_ENABLED=true`. There is no Expo secret to store.
Ensure migration `0033_device_tokens` has run (it ships in the API's Alembic chain).

**This is already wired into the infra pipeline** (param `expoPushEnabled` in
`dops-infra/infra/main.bicep`, fed from `dops-dev.bicepparam`). To turn it on, set this **non-secret**
pipeline variable in the **`dops-dev`** variable group (not `dops-dev-secrets` — that's for
Key-Vault-linked secrets), then redeploy infra:

| Variable | Secret? | Maps to |
| --- | --- | --- |
| `DOPS_EXPO_PUSH_ENABLED` | no | `EXPO_PUSH_ENABLED` env var on the API container app (default `false`) |

It is safe to leave unset — push simply stays off and other notification channels are unaffected. For
a non-Bicep / local API, set `EXPO_PUSH_ENABLED=true` directly in the environment.

## Mobile configuration — EAS project id (required for real tokens)

`expo-notifications` needs an **EAS project id** to mint a push token on a device (Expo Go in
development can work without it, but standalone/dev builds require it):

1. `npm install -g eas-cli && eas login` (free Expo account at <https://expo.dev>).
2. From `dops-mobile`: `eas init` — this creates the EAS project and records its `projectId` in the
   app config. The app reads it via `Constants.expoConfig.extra.eas.projectId` /
   `Constants.easConfig.projectId` in `src/lib/push.ts`.
3. Rebuild / restart the app.

### iOS extra step (production push)

For real iOS push on a standalone build you need an **Apple Push Notification key (.p8)** uploaded to
Expo: `eas credentials` → iOS → Push Key → let EAS create/manage it. Expo Go and Android need no
extra credential for testing.

## Verify end to end

You need a **physical device** — Expo push tokens are not issued on simulators/emulators.

1. Build/run the app on a device (guide 02) with the EAS project id present and the API reachable.
2. Sign in. Accept the notification-permission prompt.
3. Confirm registration: a `device_tokens` row exists for your user
   (`SELECT * FROM device_tokens` or check the API logs for the `POST /portal/devices` 201).
4. Trigger an event for that user — e.g. as staff, **cancel or reschedule** a session the
   learner/instructor is on (test plan doc 06 / 16-14).
5. A push should arrive on the device with the event title/body. Tapping it opens the app.
6. The `notifications` row for that event has channel `expo_push` and status `sent`.

### Quick manual push (sanity check)

You can hit Expo directly with a token you registered to confirm device + token health:

```bash
curl -X POST https://exp.host/--/api/v2/push/send \
  -H "Content-Type: application/json" \
  -d '[{"to":"ExponentPushToken[xxxxxxxx]","title":"Test","body":"Hello from DrivingOps","sound":"default"}]'
```

A `"status":"ok"` ticket means the token + device are good; the issue (if any) is then in the
API wiring or the `EXPO_PUSH_ENABLED` flag.

## Troubleshooting

| Symptom | Likely cause / fix |
| --- | --- |
| No permission prompt | Already answered once; reset notification permission in OS settings, or reinstall |
| Token not registered | Running on a simulator (use a real device); missing EAS `projectId`; API unreachable |
| Notification row `config_pending` | `EXPO_PUSH_ENABLED` is false, **or** the recipient has no registered device |
| Notification row `failed` | Expo rejected the token (expired / `DeviceNotRegistered`) — sign out/in to re-register |
| Push works in Expo Go but not standalone | Missing EAS project id or (iOS) APNs key in EAS credentials |
