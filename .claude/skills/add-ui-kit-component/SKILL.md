---
name: add-ui-kit-component
description: Add a new React component, chart primitive or domain status to @warehouse/ui-kit, test it with Vitest, and export it from src/index.ts so consumers can import it. Use when asked to add a component, chart type, status color, KPI or token to the shared kit, or when a remote needs a visual primitive that belongs here instead of in its own web/ folder.
---

# Add a component or chart primitive to ui-kit

Decide first whether it belongs here: anything about palette, domain-status color, shared
layout or a chart type does (`CLAUDE.md`). A one-off screen layout stays in the remote.

## Steps

1. **Model it on a sibling.** Plain component: `src/components/Card.tsx` or
   `src/components/KpiStat.tsx` (+ `src/components/KpiStat.css`). Chart: `src/components/BarChart.tsx`,
   `src/components/LineChart.tsx` or `src/components/FunnelChart.tsx`. Read
   `src/components/chartCommon.ts` and reuse `ChartDatum`, `ChartTone`, `CHART_TONE_COLOR`,
   `maxOf`, `ratioOfMax`, `truncateLabel`, `formatChartValue`; do not copy them.
2. **Write the component** as a named export with an exported `XxxProps` interface, return
   type `ReactElement`, doc comment stating what it does and does not do. Colors/spacing
   only via `--wh-*` tokens from `src/tokens/tokens.css`. Charts: plain SVG, no dependency,
   colors via the `style` prop, `role="img"` plus `aria-label`, an `emptyState` prop.
3. **Styles (only if needed):** create `src/components/Xxx.css` and add
   `import "./components/Xxx.css";` to `src/index.ts` AFTER the `tokens.css` import and
   beside the other component sheets. Do not import the CSS from the `.tsx` file.
4. **Export it** in `src/index.ts`: `export { Xxx } from "./components/Xxx";` and
   `export type { XxxProps } from "./components/Xxx";`, in the matching section (charts
   are under the "charts" comment). If it is not exported there, no consumer can use it.
5. **New domain status?** Edit `src/components/StatusPill.tsx` only: add the literal to
   `DomainStatus` under its owning context's comment and to `STATUS_TONE`. Mirror the Go
   domain enum exactly; never invent UI-only labels.
6. **Test it** in `src/components/Xxx.test.tsx` (pattern: `src/components/BarChart.test.tsx`,
   `src/components/KpiStat.test.tsx`): `render`, query by role/text, assert the empty
   state, the missing-value dash, and one behavior that would fail if the component broke.
   Setup is already global (`src/test/setup.ts`). Tests stay out of `tsconfig.app.json`.
7. **Verify:** `make check` (lint, typecheck, test, build). Confirm the export reached the
   built types: `grep -n Xxx dist/src/index.d.ts` after `npm run build`.
8. **Hand off to consumers:** follow the `ship-ui-kit-change` skill. A remote that wants
   the component must rebuild against the new `dist/`.

## Pitfalls

- Hard-coded hex colors, a new chart library, or `fill="var(--x)"` as an SVG attribute
  (silently ignored) are the usual review failures.
- `react/only-export-components` is a lint warning (`.oxlintrc.json`): keep constants and
  helpers out of component files unless they are constant exports.
- `import type` is required for type-only imports (`verbatimModuleSyntax`).
- Do not add a runtime dependency; `react`/`react-dom` are peers and stay external in
  `vite.config.ts`.
