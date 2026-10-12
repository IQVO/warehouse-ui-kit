---
id: tokens
title: Design tokens
sidebar_label: Tokens
---

# Design tokens

`src/tokens/tokens.css` is the single source of truth for color, spacing, type, elevation,
focus, motion and stacking across the console shell and every remote. Consumers load it
(together with every component stylesheet) with one import:

```tsx
import "@warehouse/ui-kit/tokens.css"; // -> dist/ui-kit.css
```

Rules (from the file header and `.claude/rules/components-and-tokens.md`):

- A remote that hand-rolls its own palette instead of these variables is a bug, not a style
  choice. New colors belong here.
- Status colors use the same vocabulary as each service's domain status enum; every status
  maps onto one of five lanes via `StatusPill`'s `STATUS_TONE` (see
  [Components](components.md#statuspill)).
- Never hard-code a hex value in a component; reference `var(--wh-*)`.

All values below are the `:root` values on `origin/develop`.

## Color: neutrals

| Token | Value | Used for |
|---|---|---|
| `--wh-color-bg` | `#0b0f14` | Page background (`html`, `body`) |
| `--wh-color-bg-raised` | `#121822` | Cards, KPI tiles, launch tiles, shell header, table header |
| `--wh-color-bg-sunken` | `#060a0e` | Not referenced by any kit component; available for recessed surfaces in consumers |
| `--wh-color-border` | `#232b38` | Card/tile borders, table header rule, timeline connector |
| `--wh-color-border-subtle` | `#1a212c` | Row separators, chart tracks, skeleton rows |
| `--wh-color-text` | `#e6edf5` | Body text, values |
| `--wh-color-text-muted` | `#8f9bad` | Secondary text, labels, captions |
| `--wh-color-text-faint` | `#78849a` | Table headers, timestamps, chart axis labels. Raised from `#5c6779` for WCAG AA (5.09:1 on `--wh-color-bg`, 4.72:1 on `--wh-color-bg-raised`); do not darken it |

## Color: brand / accent

| Token | Value | Used for |
|---|---|---|
| `--wh-color-accent` | `#4d8dff` | Brand mark, active nav underline, focus ring, `neutral` KPI accent, chart `accent` tone |
| `--wh-color-accent-hover` | `#6ba0ff` | Not referenced by any kit component (hover states in the kit change the border to `--wh-color-accent`) |
| `--wh-color-accent-muted` | `#1c2b47` | `LaunchTile` badge background |

## Color: semantic status (the five lanes)

| Lane | Foreground token | Value | Background token | Value |
|---|---|---|---|---|
| neutral | `--wh-color-status-neutral` | `#8f9bad` | `--wh-color-status-neutral-bg` | `#1b212c` |
| progress | `--wh-color-status-progress` | `#4d8dff` | `--wh-color-status-progress-bg` | `#16233d` |
| success | `--wh-color-status-success` | `#3dd68c` | `--wh-color-status-success-bg` | `#123729` |
| warning | `--wh-color-status-warning` | `#f5b942` | `--wh-color-status-warning-bg` | `#3a2c10` |
| danger | `--wh-color-status-danger` | `#f4614c` | `--wh-color-status-danger-bg` | `#3a1712` |

Consumers of the lanes: `StatusPill` (all five, fg + bg), `FreshnessBadge` (neutral,
success, warning), `Timeline` dot colors, `KpiStat` accent and delta colors, and the chart
`ChartTone` map (`neutral`, `success`, `warning`, `danger`, plus `accent` →
`--wh-color-accent`).

## Spacing (4px base scale)

| Token | Value |
|---|---|
| `--wh-space-1` | `4px` |
| `--wh-space-2` | `8px` |
| `--wh-space-3` | `12px` |
| `--wh-space-4` | `16px` |
| `--wh-space-5` | `24px` |
| `--wh-space-6` | `32px` |
| `--wh-space-7` | `48px` |
| `--wh-space-8` | `64px` |

## Radius

| Token | Value |
|---|---|
| `--wh-radius-sm` | `4px` |
| `--wh-radius-md` | `8px` |
| `--wh-radius-lg` | `12px` |
| `--wh-radius-pill` | `999px` |

## Type

| Token | Value |
|---|---|
| `--wh-font-sans` | `-apple-system, BlinkMacSystemFont, "Segoe UI", Inter, Roboto, sans-serif` |
| `--wh-font-mono` | `"SF Mono", "JetBrains Mono", Menlo, Consolas, monospace` |
| `--wh-font-size-xs` | `12px` |
| `--wh-font-size-sm` | `13px` |
| `--wh-font-size-md` | `14px` (body) |
| `--wh-font-size-lg` | `16px` |
| `--wh-font-size-xl` | `20px` |
| `--wh-font-size-2xl` | `28px` (KPI values) |
| `--wh-font-weight-regular` | `400` |
| `--wh-font-weight-medium` | `500` |
| `--wh-font-weight-semibold` | `600` |
| `--wh-line-height-tight` | `1.15` (headline numerals, KPI values) |
| `--wh-line-height-snug` | `1.3` (table cells, pills, dense rows) |
| `--wh-line-height-normal` | `1.5` (prose; set on `body`) |
| `--wh-letter-spacing-eyebrow` | `0.06em` (uppercase micro-labels) |
| `--wh-letter-spacing-wide` | `0.04em` |

No web font is loaded; the stacks fall back to system fonts.

## Elevation

| Token | Value |
|---|---|
| `--wh-shadow-sm` | `0 1px 2px rgba(0, 0, 0, 0.24)` |
| `--wh-shadow-md` | `0 4px 12px rgba(0, 0, 0, 0.32)` |
| `--wh-shadow-lg` | `0 12px 32px rgba(0, 0, 0, 0.4)` |

## Focus

| Token | Value | Used for |
|---|---|---|
| `--wh-focus-ring` | `0 0 0 2px var(--wh-color-bg), 0 0 0 4px var(--wh-color-accent)` | Global `:focus-visible` box-shadow (two-stop ring visible on both bg and bg-raised) |
| `--wh-focus-ring-inset` | `inset 0 0 0 2px var(--wh-color-accent)` | Shell nav items, where an outer ring would be clipped |

## Motion

| Token | Value |
|---|---|
| `--wh-motion-fast` | `120ms` |
| `--wh-motion-base` | `180ms` |
| `--wh-motion-ease` | `cubic-bezier(0.4, 0, 0.2, 1)` |

Under `prefers-reduced-motion: reduce`, every animation and transition is cut to `0.01ms`
and smooth scrolling is disabled.

## Z-index scale

| Token | Value |
|---|---|
| `--wh-z-nav` | `100` (sticky shell header) |
| `--wh-z-dropdown` | `200` |
| `--wh-z-modal` | `300` |
| `--wh-z-toast` | `400` (also the skip link) |

## Surface calibration: `data-surface="monitor"`

One token set serves two modes. Set `data-surface="monitor"` on any subtree (for a
glanceable wall monitor) and only the font sizes inside it are recalibrated; colors and
status meaning never change.

| Token | Default | `monitor` |
|---|---|---|
| `--wh-font-size-xs` | `12px` | `13px` |
| `--wh-font-size-sm` | `13px` | `15px` |
| `--wh-font-size-md` | `14px` | `16px` |
| `--wh-font-size-lg` | `16px` | `20px` |
| `--wh-font-size-xl` | `20px` | `28px` |
| `--wh-font-size-2xl` | `28px` | `44px` |

```tsx
<section data-surface="monitor">
  <KpiStat label="Pick queue" value={queue} />
</section>
```

## Global rules and utility classes shipped with the tokens

| Selector | Effect |
|---|---|
| `*, *::before, *::after` | `box-sizing: border-box` |
| `html` | `color-scheme: dark` and the page background, painted before the JS module graph loads (no white flash) |
| `body` | No margin, `--wh-color-bg` background, `--wh-color-text`, `--wh-font-sans`, `--wh-font-size-md`, `--wh-line-height-normal`, antialiasing |
| `:focus-visible` | No outline; `--wh-focus-ring` box-shadow and `--wh-radius-sm` |
| `.wh-numeric` | Monospace with tabular figures, so polled numbers do not twitch |
| `.wh-visually-hidden` | Screen-reader-only text (used by `Timeline` and `KpiStat`) |
| `@keyframes wh-shimmer` | Opacity pulse used by `DataTable` skeleton rows |

Because these are global selectors, importing `tokens.css` restyles `html`, `body` and every
focusable element of the consuming app. That is intended: the console and every remote
share one look.

## Component-scoped custom properties

Two components set a private custom property inline and read it from their stylesheet:

| Property | Set by | Read by |
|---|---|---|
| `--wh-timeline-color` | `Timeline` (per step, from the state) | `.wh-timeline__dot` |
| `--wh-kpi-accent` | `KpiStat` (from the resolved tone) | `.wh-kpi__accent` |

## Tokens no kit component references

Defined for consumers but not used inside `src/` (checked with a `var(--wh-...)` search
over every component file and stylesheet): `--wh-color-bg-sunken`,
`--wh-color-accent-hover`, `--wh-space-7`, `--wh-font-size-xl`,
`--wh-font-weight-regular`, `--wh-shadow-sm`, `--wh-shadow-md`, `--wh-shadow-lg`,
`--wh-z-dropdown`, `--wh-z-modal`. `--wh-font-sans` is used only by the `body` rule.

Some inline-styled components still hard-code values that have a token: the chart
`<figcaption>`s use `letterSpacing: "0.06em"` and `fontWeight: 600`, and `DataTable`
headers use `letterSpacing: "0.04em"`, instead of `--wh-letter-spacing-eyebrow`,
`--wh-letter-spacing-wide` and `--wh-font-weight-semibold`.

## Adding or changing a token

1. Edit `src/tokens/tokens.css` (keep the `--wh-` prefix and the section grouping).
2. Reference it from the component with `var(--wh-...)`; in SVG, set it through the
   `style` prop, not a presentation attribute.
3. `make check`, then rebuild (`npm run build`) so consumers see it in `dist/ui-kit.css`.
4. Verify contrast for any text color change (the `--wh-color-text-faint` comment records
   the measured ratios).
