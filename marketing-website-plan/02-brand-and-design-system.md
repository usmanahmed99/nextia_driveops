# 02 — Brand & Design System

The marketing site reuses the **product brand** so the site and app feel like one
company. Source of truth for the app palette:
`dops-frontend/tailwind.config.ts` and `src/styles/*.css`.

## Logos (from `dops-frontend/public/brand/`)

Copied into the Astro site at `public/brand/`:

| Asset | Use |
| ----- | --- |
| `drivingops_primary_horizontal.svg` | Header on light backgrounds |
| `drivingops_reverse_white_horizontal_transparent.svg` | Header/footer on dark |
| `drivingops_icon_mark_transparent.svg` | Favicon source, compact spots |
| `drivingops_wordmark_only.svg` | Tight horizontal spaces |
| `drivingops_favicon_192.png/.svg` | Favicon |

## Colour

| Token | Value | Use |
| ----- | ----- | --- |
| Navy (ink) | `#071b3a` | Headings, dark sections, footer |
| Electric blue (primary) | `#0f4cff` | Primary buttons, links, accents |
| Ice | `#eef7ff` | Tinted section backgrounds, cards |
| Success | `#16a34a` | Positive stats, checks |
| Attention | `#f59e0b` | Highlights, "most popular" |
| White | `#ffffff` | Base background |
| Slate text | `#475569` | Body copy on light |

Gradients: subtle navy→electric for hero accents; `rgba(15,76,255,.05)` grid
overlay (`surface-grid`) reused from the app for a "product" texture.

## Typography

- **Inter** (matches the app's `font-sans`). Loaded via `@fontsource-variable/inter`
  (self-hosted, no external request — good for Cloudflare + privacy).
- Scale: display 3rem/3.5rem bold, h1 2.5rem, h2 2rem, h3 1.25rem, body 1.0625rem,
  small 0.875rem. Line-height generous (1.6 body).
- Tracking slightly tight on display headings.

## Spacing & layout

- Max content width `1200px`; comfortable section padding (`py-20`/`py-28`).
- 12-col mental model; cards on 1/2/3 grids responsive.
- Radius: `xl 0.9rem`, `2xl 1.2rem` (matches app). Cards `rounded-2xl`.
- Shadows: soft (`0 18px 45px -30px rgba(7,27,58,.45)`) and glow for primary CTAs.

## Components (built in Astro)

`Button`, `Section`, `Container`, `Card`, `FeatureCard`, `Stat`, `BrowserFrame`
(screenshot in a faux browser chrome), `PhoneFrame` (portal screenshots),
`Badge`, `Accordion` (FAQ), `PricingCard`, `Showcase` (alt image+text),
`LogoStrip`, `CtaBand`, `Header`, `Footer`, `Nav`.

## Imagery treatment

- Product screenshots always presented inside `BrowserFrame`/`PhoneFrame` with
  the soft shadow and a faint navy→ice gradient backdrop — never bare.
- Decorative blobs/grid kept subtle; the product shots are the hero.

## Motion

- Restrained: fade/slide-up on scroll (IntersectionObserver, respects
  `prefers-reduced-motion`), hover lifts on cards/buttons. No autoplay video in v1.

## Accessibility

- WCAG 2.1 AA target. Contrast checked for navy/electric on white & ice.
- Visible focus rings (`--ring` electric). Semantic landmarks, alt text on every
  product image (see asset plan), skip-to-content link, keyboard-operable
  accordion/menu.

## Premium cues (avoid "cheap")

- Real product screenshots (not stock laptops), consistent framing, generous
  whitespace, a tight 2-colour palette with one accent, crisp Inter type, subtle
  depth, and no clip-art. Stats and copy are specific, not vague.
</content>
