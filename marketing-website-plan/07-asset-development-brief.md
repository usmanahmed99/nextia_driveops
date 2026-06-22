# 07 — Asset Development Brief (for the Design Team)

This brief tells the design team **exactly which visual assets to produce** for the
DrivingOps marketing site, the specs for each, and **how to edit the raw
screenshots into premium, framed device mockups** — including **laptop, iPhone
(iOS), and Android** views.

> **Read this with** [`03-asset-and-screenshot-plan.md`](./03-asset-and-screenshot-plan.md)
> (the shot list) and [`06-handoff-to-design.md`](./06-handoff-to-design.md)
> (where files go in the codebase).

---

## 0. Where the raw screenshots are

| Type | Location |
| ---- | -------- |
| **Raw desktop captures** (10) | `dops-marketing-website/public/screenshots/*.png` |
| **Capture notes / what's pending** | `dops-marketing-website/public/screenshots/CAPTURE-LOG.md` |
| **Raw mobile captures** (Android emulator) | `dops-marketing-website/public/screenshots/mobile/*.png` |
| **Brand logos** | `dops-marketing-website/public/brand/*` |
| **Finished/edited assets go to** | `dops-marketing-website/public/assets/<category>/…` |

Desktop captures: 1440×900, real product, seed tenant **NorthStar Driving School**.
Mobile captures: Android emulator (Pixel 8), demo-login session against the dev API.

---

## 1. Guiding principle: show the product on real devices

Every product visual on the marketing site should appear **inside a device frame**,
never as a bare screenshot. We want prospects to picture DrivingOps running on the
exact hardware they use:

- **Laptop / desktop frame** — for the admin workspace (dashboard, scheduling,
  billing, reports, settings). This is where owners and office staff work.
- **iPhone (iOS) frame** — for the learner and instructor portals / mobile app.
- **Android frame** — the same portal/app shown on Android.

> **Why both iOS and Android:** our learners and instructors are split across both
> platforms. Showing the app on an iPhone **and** an Android phone signals "works
> on your phone, whatever you carry" and removes a silent objection. Use them as a
> **pair** in mobile sections (one iOS, one Android, slightly overlapped), and
> pair a **laptop + phone** composite in hero / cross-device sections to say
> "office on the laptop, instructors and learners on their phones."

The site already ships lightweight CSS `BrowserFrame` and `PhoneFrame` components.
The design team's job is to produce **higher-fidelity, brand-polished** versions as
images (PNG/WebP) for hero and showcase slots, and optimized exports of the raw
screenshots for the in-page frames.

---

## 2. Device-frame editing guide

### 2.1 Laptop / desktop (admin workspace)

- **Source:** the 1440×900 desktop PNGs in `public/screenshots/`.
- **Frame:** a clean, modern laptop (MacBook-style silver or neutral) **or** the
  existing browser-chrome frame with `app.drivingops.ca/<route>` in the address bar.
- **Editing steps:**
  1. Crop to remove dev-state artifacts where noted in `CAPTURE-LOG.md` (e.g. the
     amber "WhatsApp configuration pending" strip on `automations.png`).
  2. Sharpen / ensure text is crisp at 2× export.
  3. Place on a **navy→ice gradient** backdrop with the soft shadow
     (`0 40px 80px -40px rgba(7,27,58,.45)`).
  4. Optional: subtle reflection or perspective tilt (≤ 8°) for hero only — keep
     all in-page showcases flat and legible.
- **Export:** 2× (e.g. 2880 wide), then downscale to WebP/AVIF + PNG fallback.

### 2.2 iPhone (iOS) — learner & instructor portal

- **Source:** the mobile captures in `public/screenshots/mobile/` (Android),
  re-skinned into an iOS frame, **or** re-capture on an iOS simulator if available.
- **Frame:** current-gen iPhone (notch/Dynamic Island, rounded corners, titanium
  or black bezel). Status bar showing a clean time, full signal, full battery.
- **Editing steps:**
  1. Mask the screenshot to the iPhone screen radius.
  2. Replace the Android status bar with an **iOS status bar** (or hide it under
     the notch/Island).
  3. Drop shadow + gradient backdrop matching the laptop treatment.
- **Aspect:** 1179×2556 (iPhone 15/16) screen area; export the framed asset at 2×.

### 2.3 Android — learner & instructor portal

- **Source:** the same mobile captures (already Android) in
  `public/screenshots/mobile/`.
- **Frame:** a clean modern Android phone (Pixel-style, centered punch-hole
  camera, flat display). Keep the Android status bar.
- **Editing steps:** mask to screen radius, gradient backdrop + shadow, same
  treatment as iOS so the pair looks consistent.
- **Aspect:** ~1080×2400 screen area; export at 2×.

### 2.4 Cross-device composites (the money shots)

Produce **2–3 hero composites** that combine devices:

1. **Hero (home):** laptop showing the **dashboard**, with an **iPhone** and an
   **Android** phone in front showing the **learner portal** and **instructor
   portal**. One image that says "the whole school, every device."
2. **For Learners / For Schools:** an iPhone + Android pair, overlapped, each
   showing a portal screen.
3. **Cross-device band:** laptop + single phone, side by side.

---

## 3. Asset checklist (with specs)

### A. Edited product mockups (highest priority)

| # | Asset | Devices | Source | Output |
| - | ----- | ------- | ------ | ------ |
| A1 | Hero cross-device composite | Laptop + iPhone + Android | `dashboard.png` + portals | 2400×1500 WebP+PNG, transparent or gradient bg |
| A2 | Dashboard laptop mockup | Laptop | `dashboard.png` | framed, 2× |
| A3 | Scheduling laptop mockup | Laptop | `schedule.png` (re-capture w/ data — see §5) | framed, 2× |
| A4 | Automations laptop mockup | Laptop | `automations.png` (crop dev banner) | framed, 2× |
| A5 | Invoices / billing laptop mockup | Laptop | `invoices.png` | framed, 2× |
| A6 | Reports laptop mockup | Laptop | `reports.png` | framed, 2× |
| A7 | Learner portal — iOS + Android pair | iPhone + Android | `mobile/learner-portal-*.png` | framed pair, 2× |
| A8 | Instructor portal — iOS + Android pair | iPhone + Android | `mobile/instructor-portal-*.png` | framed pair, 2× |
| A9 | Public site builder laptop mockup | Laptop | `public-site-builder.png` | framed, 2× |

### B. Brand / marketing assets

| # | Asset | Spec |
| - | ----- | ---- |
| B1 | OG / social images (per page) | 1200×630 PNG — home, features, pricing, for-schools, for-learners, each blog post. Brand + device composite. |
| B2 | Favicon set | 16, 32, 180 (apple-touch), 192, 512 — from `drivingops_icon_mark` |
| B3 | Customer logos | Mono SVG, ~120×40, for `LogoStrip` (replace placeholder text) |
| B4 | Testimonial avatars | 96×96, real or licensed portraits |
| B5 | Team photos | 400×400, About page |
| B6 | Illustration: "spreadsheet chaos → one workspace" | Vector, brand fills, home problem section |
| B7 | Canada / bilingual motif | Vector, subtle, for the Canada band |
| B8 | Blog cover images (3) | 1200×675 each |

### C. Optimization pass

| # | Task |
| - | ---- |
| C1 | Export all product PNGs to **WebP + AVIF** with PNG fallback |
| C2 | Generate responsive sizes (1×/2×) or migrate frames to Astro `<Image>` |
| C3 | Keep each hero asset < ~250 KB after compression |

---

## 4. Brand guardrails (keep it premium, not cheap)

- 2-colour + 1-accent palette only: navy `#071b3a`, electric `#0f4cff`,
  ice `#eef7ff`, accent attention `#f59e0b`. Don't introduce new hues.
- Inter typeface everywhere (matches the product).
- Always frame product UI; consistent shadow + gradient backdrop across all shots.
- iOS and Android frames must look like a **matched set** (same scale, shadow,
  backdrop) when shown together.
- No stock clip-art. Photography (if any) = authentic, diverse, Canadian, in-car /
  instructor scenes.
- Specific copy/labels beat vague hype; keep stats labelled "illustrative" until
  real data exists.

---

## 5. Screenshots still to capture (and how)

Some product screens don't populate in the frontend mock adapter. Capture these
against the **dev API with seed data** (the deployed dev API is live; the mobile
app's demo login already uses it):

| Screen | Why pending | How |
| ------ | ----------- | --- |
| Scheduling board with sessions | Mock adapter has no instructor availability | Run frontend in `api` mode (or use dev API) with seeded availability; navigate to the seeded week (seed sessions are ~late May 2026) |
| Certificates (issued list) | Mock seeds none | Approve a certificate in api mode, then capture |
| Learner portal (mobile) | Needs a linked learner session | Captured on the Android emulator via demo login (see `mobile/`) — re-skin to iOS for A7 |
| Instructor portal (mobile) | Needs a linked instructor session | Same — see `mobile/` |

**Mobile capture recipe (used for the populated Android shots):**

`dops-mobile` has a built-in **screenshot mode** that skips login and serves seed
data so portals are fully populated without a linked backend account.

1. Boot the `Pixel_8-_DrivingOps` Android AVD.
2. Start Expo with the screenshot flags (PowerShell):
   `$env:EXPO_PUBLIC_SCREENSHOT_MODE="true"; $env:EXPO_PUBLIC_SCREENSHOT_ROLE="LEARNER"; npx expo start --port 8083`
   (use `INSTRUCTOR` for the instructor portal).
3. `adb -s emulator-5554 reverse tcp:8083 tcp:8083` then open
   `exp://127.0.0.1:8083` on the emulator. Dismiss the Expo dev menu (Back).
4. `adb -s emulator-5554 exec-out screencap -p > mobile/<name>.png`
   **from a binary-safe shell (Git Bash)** — a PowerShell `>` corrupts the PNG.
5. Fixtures live in `dops-mobile/src/lib/demo/screenshotData.ts` — edit there to
   change names, lessons, progress, etc. Never ship these flags in a real build.

For **iOS** frames, either re-capture on an iOS simulator (Mac) the same way, or
re-skin the Android captures into an iPhone frame per §2.2.

---

## 6. Definition of done

- [ ] A1–A9 produced and dropped into `public/assets/mockups/` (and raw
      screenshots replaced where applicable).
- [ ] Mobile sections show an **iOS + Android pair**; hero shows a
      **laptop + phones** composite.
- [ ] OG images set per page (`ogImage` prop in each page's `<BaseLayout>`).
- [ ] Images optimized (WebP/AVIF), hero assets < ~250 KB.
- [ ] Placeholder logos/testimonials replaced.
- [ ] `npm run build` passes; Lighthouse ≥ 95 Performance / SEO / Accessibility.
</content>
