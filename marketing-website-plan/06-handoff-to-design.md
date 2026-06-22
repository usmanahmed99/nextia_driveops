# 06 — Handoff to the Design Team

The site is **built and functional** with real product screenshots. Your job is to
elevate it from "accurate and clean" to "premium and unmistakably DrivingOps".
Everything is structured so you can drop assets in without touching layout.

## What's already done

- Full site built in Astro + Tailwind on the product brand (navy `#071b3a`,
  electric `#0f4cff`, ice `#eef7ff`, Inter).
- 10 real product screenshots captured from the running app (mock seed data,
  NorthStar Driving School) — see `marketing-website/public/screenshots/` and its
  `CAPTURE-LOG.md`.
- Every screenshot is presented inside a `BrowserFrame` (desktop) or `PhoneFrame`
  (mobile) component with a soft shadow and gradient backdrop.

## Where to put assets

| Asset type | Drop into | Wired by |
| ---------- | --------- | -------- |
| Replacement product shots | `public/screenshots/<name>.png` (same filename) | Used automatically |
| Polished/branded assets | `public/assets/<category>/…` | Reference in the relevant page |
| OG/social images (1200×630) | `public/assets/og/<page>.png` | Set `ogImage` prop on each page's `<BaseLayout>` |
| Favicons | `public/brand/` | Already referenced in `BaseLayout` |
| Customer logos | `src/components/LogoStrip.astro` | Replace the placeholder names with `<img>` |
| Testimonial avatars | `public/assets/people/` | Add to the home testimonial |

## Priority deliverables

1. **Capture the pending screenshots** (schedule with data, certificates, learner
   & instructor mobile portals) — see `CAPTURE-LOG.md` for the API-mode recipe —
   or design polished mockups for them. The marketing pages currently show
   branded placeholder slots for these.
2. **Hero composite** for the home page: a refined version of `dashboard.png`
   (possibly with subtle callouts) and an optional secondary phone showing a
   portal, on the navy→ice gradient.
3. **Per-page OG images** (1200×630): home, features, pricing, for-schools,
   for-learners, each blog post.
4. **Optimize images**: export WebP/AVIF; the PNGs are large. Consider Astro's
   `<Image>` for responsive `srcset`.
5. **Customer logos & testimonials**: replace placeholders once pilots are signed.
6. **Two or three illustrations**: "spreadsheet chaos → one workspace" (home
   problem section) and a Canada/bilingual motif.
7. **Annotated feature shots** (optional): callout pins on key UI to guide the eye.

## Brand guardrails (keep it premium)

- Always frame product UI; never show a bare screenshot.
- Keep the 2-colour + 1-accent palette; resist adding new hues.
- Generous whitespace; let the product shots breathe.
- No stock clip-art. If using photography (About/blog), choose authentic,
  diverse, Canadian, in-car/instructor scenes.
- Specific copy beats vague hype — keep stats labelled as illustrative until real.

## Review loop

1. Drop assets into the folders above.
2. `npm run dev` and review at `http://localhost:4321`.
3. `npm run build` to confirm it still builds.
4. Run Lighthouse; target ≥ 95 on Performance / SEO / Accessibility.
</content>
