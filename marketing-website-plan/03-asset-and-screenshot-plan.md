# 03 — Asset & Screenshot Plan

All product screenshots are captured from **dops-frontend running locally in mock
mode** (`VITE_DATA_MODE=mock`, port 5174) using the seeded **NorthStar Driving
School** data. Captured via Playwright at desktop **1440×900** and mobile
**390×844** (portal screens). Files land in
`marketing-website/public/screenshots/` (raw) and are referenced by pages.

> **For the design team:** these are *real, accurate* product captures meant as the
> basis for polished assets. See [`06-handoff-to-design.md`](./06-handoff-to-design.md)
> for what to produce from each (framed mockups, annotated callouts, OG images,
> hero composites).

## Screenshot shot list

| File | Screen / route | Viewport | Used on |
| ---- | -------------- | -------- | ------- |
| `dashboard.png` | `/` Dashboard "Today at NorthStar" | 1440×900 | Home hero, Features, For Schools |
| `schedule.png` | `/schedule` Scheduling board | 1440×900 | Home showcase, Features |
| `learners.png` | `/learners` Learner list | 1440×900 | Features |
| `learner-detail.png` | `/learners/:id` Learner workspace | 1440×900 | Features, For Schools |
| `instructors.png` | `/instructors` Instructors | 1440×900 | Features |
| `automations.png` | `/automations` Automations/flows | 1440×900 | Home showcase, Features |
| `invoices.png` | `/invoices` Billing | 1440×900 | Features, Pricing |
| `certificates.png` | `/certificates` Certificates | 1440×900 | Features |
| `reports.png` | `/reports` Reports & dashboard | 1440×900 | Features, For Schools |
| `public-site-builder.png` | `/public-website` Site builder | 1440×900 | Features |
| `assistant.png` | `/assistant` AI assistant | 1440×900 | Features |
| `settings.png` | `/settings` Settings hub | 1440×900 | Features |
| `learner-portal.png` | `/portal/learner` (mobile) | 390×844 | Home portals, For Learners |
| `instructor-portal.png` | `/portal/instructor` (mobile) | 390×844 | Home portals, For Learners |
| `onboarding.png` | `/onboarding` Setup checklist | 1440×900 | For Schools (onboarding) |

If a route is gated/empty in mock mode it is skipped and noted in the capture log
(`marketing-website/public/screenshots/CAPTURE-LOG.md`).

## Non-screenshot assets the site needs (design team to create)

| Asset | Spec | Notes |
| ----- | ---- | ----- |
| OG / social image | 1200×630 | Per major page; brand + dashboard composite |
| Favicon set | 16/32/180/192/512 | From `drivingops_icon_mark` |
| Hero background texture | SVG | Subtle navy→electric grid (reuse `surface-grid`) |
| Customer logos | mono SVG, ~120×40 | Placeholder until pilots secured |
| Testimonial avatars | 96×96 | Placeholder portraits |
| Team photos | 400×400 | About page, placeholder |
| Illustration: "spreadsheet chaos vs one workspace" | — | Problem section |
| Map / Canada motif | SVG | Bilingual/Canada band |
| Blog cover images | 1200×675 | 3 seed posts |
| Icons | Lucide (same as app) | Inline SVG, no icon font |

## Photography / illustration direction

- Prefer **product UI** over stock photography. If photography is used (About,
  blog), choose authentic Canadian, diverse, in-car / instructor scenes — not
  generic stock-smile imagery.
- Illustrations: simple, line + brand fills, never clip-art.

## Naming & placement

- Raw captures: `public/screenshots/<name>.png`
- Polished/design assets: `public/assets/<category>/<name>.<ext>`
- Brand logos: `public/brand/*` (copied from product)
</content>
