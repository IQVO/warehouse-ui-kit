---
paths:
  - "src/**"
---

# Components, tokens and charts (`src/**`)

- **Tokens are the only palette.** Use `--wh-*` custom properties from
  `src/tokens/tokens.css`; never hard-code a hex value in a component. `--wh-color-text-faint`
  was raised for WCAG AA contrast (it carries table headers, timestamps, chart axes): do
  not darken it.
- **Status colors:** `STATUS_TONE` in `src/components/StatusPill.tsx` is the single
  status-to-tone table. Add a status to the `DomainStatus` union AND `STATUS_TONE`, grouped
  under its owning context, mirroring that context's Go domain enum. Unknown strings fall
  back to neutral gray; gray where you expected color means a missing entry.
- **Five tone lanes only:** neutral, accent/progress, success, warning, danger. Charts use
  `CHART_TONE_COLOR` in `src/components/chartCommon.ts`; KpiStat has its own `tone`.
- **Charts are plain SVG.** No charting dependency (this package is a Module Federation
  shared singleton: weight is paid on every screen). Set `fill`/`stroke` via the `style`
  prop, not attributes, because `var()` is ignored in SVG presentation attributes. Reuse
  `maxOf`, `ratioOfMax`, `truncateLabel`, `formatChartValue` from `chartCommon.ts`; give
  long labels a `<title>` tooltip; every chart takes `{ label, value }[]`, `formatValue`
  and an `emptyState`. Charts render exactly the numbers passed in and fetch nothing.
- **Missing data is not zero.** Show `NO_VALUE` (an em dash) for null/non-finite values;
  `KpiStat` has an `unavailable` state that must read louder than a healthy number.
- **Stylesheets** for a component are imported in `src/index.ts`, in cascade order after
  `tokens.css`, not beside the component (`allowArbitraryExtensions` in
  `tsconfig.app.json` makes a relative `./Foo.css` import resolve to a `Foo.d.css.ts`).
  `cssCodeSplit: false` folds everything into `dist/ui-kit.css`.
- **Public surface = `src/index.ts`.** Export the component and its prop types there; the
  library entry in `vite.config.ts` is that file.
- **No router import.** Navigation is injected via `NavigationProvider` in
  `src/navigation/NavigationContext.tsx`; the kit never imports react-router.
- **Tests:** `*.test.tsx` beside the component, Vitest + React Testing Library, jsdom;
  setup in `src/test/setup.ts` (does `cleanup()` after each test). Test files must not
  enter `tsconfig.app.json`.
- **TypeScript:** `erasableSyntaxOnly`, `verbatimModuleSyntax` (use `import type`), and
  `noUnusedLocals` are on; oxlint config in `.oxlintrc.json`.
