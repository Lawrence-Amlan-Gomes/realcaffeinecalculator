# Real Caffeine Calculator

Caffeine half-life calculator at realcaffeinecalculator.com. Logs drinks, shows mg in system right now, projects when it's safe to sleep.

## Stack

- **Astro 6** — static site, no SSR
- **Tailwind CSS v4** — loaded via `@tailwindcss/vite` Vite plugin (NOT the Astro integration)
- **TypeScript** — components use `.ts` types; calculator script is plain JS
- **Cloudflare Workers Assets** — deployed via `wrangler deploy`

## Commands

```bash
npm run dev       # local dev server
npm run build     # astro build → dist/
wrangler deploy   # push dist/ to Cloudflare
```

## Project Structure

```
src/
  layouts/Layout.astro       # base HTML, meta, GA, theme FOUC prevention
  components/
    Header.astro             # sticky nav + theme toggle + mobile hamburger
    Footer.astro             # footer links
  pages/
    index.astro              # calculator — all logic is inline <script>
    how-long-does-caffeine-stay-in-your-system.astro  # SEO content page + mini calculator
    about.astro
    contact.astro
    privacy-policy.astro
    terms-and-conditions.astro
    404.astro / 500.astro
  styles/global.css          # CSS custom properties + shared component classes
public/                      # robots.txt, favicons, og-image.png
```

## Design System

Dark-first, full light mode. Vercel-inspired aesthetic (see DESIGN.md for full spec).

**CSS custom properties** (defined in `global.css`, overridden by `html.light`):

| Token | Dark | Light |
|---|---|---|
| `--bg` | `#000000` | `#f8f8f8` |
| `--bg-soft` | `#0a0a0a` | `#f0f0f0` |
| `--card` | `#0d0d0d` | `#ffffff` |
| `--border` | `#1a1a1a` | `#e5e5e5` |
| `--border-strong` | `#2e2e2e` | `#cccccc` |
| `--text` | `#ededed` | `#111111` |
| `--text-sec` | `#888888` | `#555555` |
| `--text-muted` | `#444444` | `#aaaaaa` |
| `--accent` | `#0070f3` | same |
| `--radius` | `8px` | same |
| `--radius-sm` | `6px` | same |
| `--radius-lg` | `12px` | same |

**Shared CSS classes** (in `global.css`):
- `.calc-card` — dark card with border + stacked shadow
- `.calc-input` — select/time/number inputs (height 40px, bg `#050505`)
- `.status-badge` — pill badge; variants: `neutral`, `safe`, `warn`, `danger`
- `.warning-banner` — red 400mg+ exceeded warning
- `.form-field` / `.form-control` — contact page form inputs (height 44px)

**Font**: Inter from Google Fonts (weights 400/500/600), loaded in `Layout.astro`.

### Light Mode

Light mode is toggled by adding `.light` to `<html>`. All overrides use `html.light` selector prefix. The theme is persisted in `localStorage('theme')`.

To avoid FOUC, `Layout.astro` includes an inline `is:inline` script in `<head>` that applies `.light` before paint:

```js
(function(){ var t = localStorage.getItem('theme'); if(t==='light') document.documentElement.classList.add('light'); })();
```

When theme changes, `Header.astro` dispatches a custom event:
```js
document.dispatchEvent(new CustomEvent('theme-change'));
```

The calculator listens for `theme-change` and re-renders the SVG chart with correct colours.

## Calculator Logic

All calculator JS lives in the `<script>` block at the bottom of `src/pages/index.astro`.

**Core formula**: `remaining_mg = dose_mg × (0.5 ^ (hours_elapsed / HALF_LIFE))`
- Half-life: **variable** — read from `#metabolism` select via `getHalfLife()`
  - Normal: 5hr | Fast metabolizer: 3hr | Slow: 8hr | Pregnant: 15hr | Smoker: 3hr
- Chart sleep-safe threshold line: **50 mg**
- Hero "safe until" threshold: **25 mg** (conservative; uses reference cup of 95mg)
- FDA daily max: **400 mg**
- Reference cup for cutoff calc: **95 mg**

**Drink database** (`DRINK_GROUPS` array, flattened to `DRINKS` for lookup):
- Grouped by category: Coffee, Energy Drinks, Tea & Other, Custom
- Rendered as `<optgroup>` elements in the drink select dropdown
- All current entries are `fixed: true` (size-specific names, no multipliers needed)
- `custom: true` — user enters mg directly; size selector hidden
- Size multiplier logic (`fixed: false`, small 0.75×/medium 1.0×/large 1.4×) still exists in code for future use

**Time handling**:
- Drink times: if > 1 hour in the future, assumed to be yesterday (`ms - 86400000`)
- Bedtime: if already past, assumed to be tomorrow (`ms + 86400000`)

**Key outputs**:
- `#safe-until` — hero display: latest time for a 95mg reference drink to stay below 25mg at bedtime. Always shown (even without drinks logged). Red if "None today", muted if "Window passed".
- `#sleep-risk-badge` — sleep disruption risk badge (Low/Moderate/High). Shown only when drinks are logged. Uses `.status-badge` classes (safe/warn/danger).
- `#mg-at-bed` — mg at bedtime shown inline with the sleep risk badge.
- `#clears-at` — time caffeine drops below 50mg.
- `#mgkg-val` — mg/kg load; only shown when weight is entered.

**Chart**: Inline SVG rendered via `updateChart(entries, halfLife)`. Samples 240 points over a 24-hour window from the first drink. Colour zones: green (<50mg), amber (50–150mg), red (>150mg). 50mg threshold line (orange dashed). Vertical markers for "now" (white dashed) and "bedtime" (purple dashed). Updates every 60 seconds via `setInterval(calculate, 60000)`.

**mg/kg row**: Only visible when body weight (lbs) is provided. Colour-coded: green (<3 mg/kg), amber (3–6), red (>6).

**Sources section**: Between FAQ and footer. Cites NIH, FDA, Drake et al. 2013 (JCSM), AASM, USDA FoodData Central. Numbered list, linked.

## Deployment Notes

- Google AdSense: `ca-pub-8206356034322089` — script loaded at end of `<body>` (after `<Footer />`), NOT in `<head>`. Moving it to `<body>` fixed a mobile LCP regression (7.1s → 3.2s). Do not move it back to `<head>`.
- Ad slot placeholders use `<div class="ad-slot" style="min-height: 280px; background: var(--bg-soft); border: 1px solid var(--border);">` — no `<ins>` code until AdSense account approved.
- `workers_dev = false` in `wrangler.toml` — prevents the `.workers.dev` subdomain from serving the site (avoids duplicate content indexing)
- `not_found_handling = "404-page"` — Cloudflare serves `dist/404.html` for unmatched routes
- Sitemap auto-generated by `@astrojs/sitemap` at build time; `site` is set in `astro.config.mjs`
- Google Analytics: `G-NTQMBGZE6N`, loaded in `Layout.astro`

## Adding New Pages

1. Create `src/pages/your-page.astro`
2. Use `<Layout title="..." description="...">` wrapper
3. Follow existing inline-style patterns for sections or add classes to `global.css`
4. Include light-mode overrides (`html.light .your-class`) if adding new colour values

## Adding New Drinks

Edit the `DRINK_GROUPS` array in `src/pages/index.astro`. Add to an existing group or create a new one:
```js
// Inside the relevant group's `drinks` array:
{ id: 'unique-id', name: 'Display Name (size)', mg: 100, fixed: true }
// fixed: true  → no size selector shown (preferred — use size-specific names)
// fixed: false → small/medium/large size selector shown (multipliers: 0.75×/1.0×/1.4×)
// custom: true → user enters mg directly; size selector hidden
```
`DRINKS` (flat lookup array) is derived automatically from `DRINK_GROUPS` via `.flatMap()`.
