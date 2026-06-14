# 02 — Run the Mobile App (emulator + your own device)

How to run **`dops-mobile`** (React Native + Expo, SDK 51) locally on an emulator and on your own
phone. The app is a thin client over `dops-api`; it ships a **demo login** so you can explore it
before identity is wired.

## Prerequisites

- **Node 18+** and npm.
- A running **dops-api** reachable from your machine/phone (local `uvicorn` or a deployed URL).
- One of:
  - **Expo Go** app on your phone (iOS App Store / Google Play) — fastest for development, **but**
    note push notifications do **not** work in Expo Go on SDK 53+; on SDK 51 remote push still works
    in Expo Go for now. For reliable push, use a dev build (guide 03).
  - **Android Studio** (Android emulator) and/or **Xcode** (iOS Simulator, macOS only).

## 1. Install

```bash
cd dops-mobile
npm install
```

## 2. Configure the API endpoint (and demo login)

Config is read from `EXPO_PUBLIC_*` environment variables (public — no secrets in the bundle). Create
`dops-mobile/.env`:

```
EXPO_PUBLIC_API_BASE_URL=http://localhost:8000/api/v1
EXPO_PUBLIC_ENVIRONMENT=dev
EXPO_PUBLIC_ALLOW_DEMO_LOGIN=true
EXPO_PUBLIC_FRONTEND_URL=https://app.drivingops.ca
# Entra (optional — enables "Sign in with Microsoft"; leave blank to use demo login):
EXPO_PUBLIC_ENTRA_AUTHORITY=
EXPO_PUBLIC_ENTRA_CLIENT_ID=
EXPO_PUBLIC_ENTRA_API_SCOPE=
```

> **`localhost` from a phone won't reach your computer.** When running on a physical device (or
> sometimes the Android emulator), set `EXPO_PUBLIC_API_BASE_URL` to your machine's LAN IP, e.g.
> `http://192.168.1.50:8000/api/v1`. Android emulator can also use `http://10.0.2.2:8000/api/v1`.
> Make sure dops-api's CORS / host binding allows it (run uvicorn with `--host 0.0.0.0`).

> **Pointing at the deployed API instead of localhost** is fine — set
> `EXPO_PUBLIC_API_BASE_URL` to the API container app URL (e.g.
> `https://ca-dops-api-dev.<region>.azurecontainerapps.io/api/v1`). Confirm it's reachable with
> `GET /api/v1/health` (should return 200) and `GET /api/v1/auth/login-config` (shows the live
> Entra `authority`, `clientId`/`audience`, and `scopes`).

## 2b. Sign in with Microsoft (Entra External ID) — optional

Demo login needs none of this. To enable the **Sign in with Microsoft** button, set the three Entra
vars so they match what the **deployed API advertises** at `GET /auth/login-config`:

```
EXPO_PUBLIC_ENTRA_AUTHORITY=https://<tenant>.ciamlogin.com/<tenant>.onmicrosoft.com
EXPO_PUBLIC_ENTRA_CLIENT_ID=<the SPA/public-client app id>
EXPO_PUBLIC_ENTRA_API_SCOPE=api://<api-app-id>/access_as_user
```

> **Common mix-ups** (verify against `/auth/login-config`):
> - The **authority** is the `.ciamlogin.com/<tenant>.onmicrosoft.com` form with **no `/v2.0`** suffix
>   — not a GUID-based `…/v2.0` URL.
> - The **client id** is the **SPA/public-client** app id (the `clientId` field from login-config),
>   **not** the API/audience app id. The API/audience id only appears inside the **scope**
>   (`api://<api-app-id>/access_as_user`).

**Register the mobile redirect URI in Entra.** The app uses the authorization-code + PKCE flow and a
custom-scheme redirect. In the Entra app registration for the SPA/public client, under
**Authentication → Add a platform → Mobile and desktop applications**, add the redirect URIs the app
uses:

- `dops://` — the standalone / dev-build scheme (from `app.config.ts` `scheme: "dops"`).
- For **Expo Go** during development, also add the Expo proxy / `exp://…` URI that the app logs at
  sign-in (it prints the exact `redirectUri`). Expo Go can't use a custom scheme, so a dev build
  (`eas build`) is the most reliable way to test the real Microsoft flow end to end.

The flow obtains an **access token** for `access_as_user`, then calls `GET /auth/session` with it as a
Bearer; the API validates the token (audience/issuer/JWKS) and returns the session. No client secret
is stored in the app (public client + PKCE).

## 3. Start the dev server

```bash
npm start
```

This opens the Expo dev tools with a QR code and key shortcuts.

### On an emulator / simulator

- Press **`a`** to open the **Android emulator** (Android Studio must have an AVD running or
  installed).
- Press **`i`** to open the **iOS Simulator** (macOS + Xcode only).

### On your own phone (Expo Go)

1. Install **Expo Go** from the App Store / Play Store.
2. Ensure the phone and computer are on the **same Wi‑Fi**.
3. Scan the QR code from the terminal (iOS: Camera app; Android: the Expo Go scanner).
4. The app loads over the local network. If it can't connect, switch the dev server to **Tunnel**
   mode: `npx expo start --tunnel` (works across networks/firewalls, slightly slower).

## 4. Sign in and explore

- With `EXPO_PUBLIC_ALLOW_DEMO_LOGIN=true`, the sign-in screen offers **Learner / Instructor /
  Staff** demo logins (no account needed). Each routes to its own role-locked tab stack.
- With Entra configured, use **Sign in with Microsoft** instead.

## Running on your device via expo.dev (EAS) — shareable builds

`npm start` requires your computer. To put the app on a device **without** a running dev server (e.g.
for a tester), build with **EAS** and install via **expo.dev**:

1. Create a free **Expo account** at <https://expo.dev> and `npm install -g eas-cli`, then
   `eas login`.
2. From `dops-mobile`, run `eas init` — this creates an **EAS project** and writes its
   `projectId` into the app config (also needed for push — see guide 03).
3. Build a shareable internal build:

   ```bash
   # Android (APK you can sideload or share):
   eas build --profile preview --platform android

   # iOS (needs an Apple Developer account + registered device UDID):
   eas build --profile preview --platform ios
   ```

4. When the build finishes, **expo.dev** shows the build page with a **QR code / install link**.
   Open it on the device to install. Android installs the APK directly; iOS installs via the
   ad‑hoc provisioning profile for registered devices.
5. Set the build's API URL by defining `EXPO_PUBLIC_API_BASE_URL` in an
   [EAS build profile / environment](https://docs.expo.dev/eas/environment-variables/) so the
   standalone build points at the right backend.

> `preview` is an internal-distribution profile (no app store). For store submission use
> `eas build --profile production` then `eas submit`. A minimal `eas.json` with a `preview` and
> `production` profile is the only addition needed; create it with `eas build:configure`.

## Troubleshooting

| Symptom | Fix |
| --- | --- |
| App can't reach the API | Use LAN IP (not `localhost`); bind uvicorn to `0.0.0.0`; check firewall |
| QR scan won't connect | Same Wi‑Fi, or use `npx expo start --tunnel` |
| Demo login missing | Set `EXPO_PUBLIC_ALLOW_DEMO_LOGIN=true` and restart `npm start` |
| Metro cache weirdness | `npx expo start -c` (clears the cache) |
| Type errors | `npm run typecheck` |
