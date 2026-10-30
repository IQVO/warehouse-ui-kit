# @warehouse/ui-kit

Shared design tokens and React component library for the
[warehouse-systems](https://github.com/IQVO) micro-frontend console. Consumed by
`warehouse-console` (the shell) and every remote micro-frontend in a bounded-context repo's
`web/` folder (13 on `develop`: `order_mgmt_mfe`, `inventory-mfe`, `planning-mfe`,
`fulfillment-mfe`, `workforce-mfe`, `facility-mfe`, `process-path-mfe`, `labor-mfe`,
`netfulfil-mfe`, `capacity-mfe`, `productmaster-mfe`, `nip-mfe`, `inbound-mfe`).

This package is the **one place** that decides:
- Design tokens (color, spacing, type, motion) — `src/tokens/tokens.css`
- Domain-status → visual-tone mapping (`StatusPill`) — every context's Order/Task/WorkUnit/
  Reservation/Slot status renders identically everywhere it appears
- Shared layout primitives (`Card`, `DataTable`, `Timeline`, `KpiStat`, `LaunchTile`)
- The persistent app chrome (`AppShell`) reused by the shell and by any remote run standalone
  in local dev

A remote hand-rolling its own palette or status colors instead of consuming these is a bug,
not a style choice — visual consistency across a micro-frontend boundary is the #1 place MFE
UX fails.

## Disclaimer

This project is a domain-driven design study. It is not associated with,
endorsed by or affiliated with any company, and it uses no company's code,
data, logos or trademarks. All names, figures and examples are illustrative
or synthetic.

## Documentation

Plain markdown under [`docs/`](docs/) (this repo has no docs site; GitHub Pages is not
enabled for it, so `https://iqvo.github.io/warehouse-ui-kit/` does not exist):

| Page | Contents |
|---|---|
| [Introduction](docs/overview/introduction.md) | What the kit owns, what it does not, who consumes it |
| [Architecture](docs/overview/architecture.md) | Source layout, every public export, styling model, build output, how consumers import it |
| [Quickstart](docs/overview/quickstart.md) | Prerequisites, npm scripts and `make` targets, linking from a consumer, peer deps, theming |
| [Components](docs/components.md) | Every component, hook and formatter with props and a usage example |
| [Tokens](docs/tokens.md) | Every `--wh-*` custom property, status lanes, `data-surface="monitor"` |
| [Testing](docs/development/testing.md) | Vitest setup, test inventory, CI jobs |
| [Configuration](docs/operations/configuration.md) | No runtime env vars; package, Vite, TypeScript, oxlint, Docker context settings |
| [Release](docs/operations/release.md) | No registry publish; how the console and remotes pick up a change |
| [Troubleshooting](docs/operations/troubleshooting.md) | Symptom, cause, check, fix |

## Development

```bash
npm install
npm run typecheck   # tsc -b --noEmit
npm run lint        # oxlint
npm test            # vitest run (jsdom + React Testing Library)
npm run build        # tsc -b && vite build -> dist/
```

Consumers reference this package via `file:../../warehouse-ui-kit` (the console:
`file:../warehouse-ui-kit`) in local development; there is no npm registry publish step yet
(single-workspace, pre-1.0). See [Release](docs/operations/release.md).

## Components

| Component | Purpose |
|---|---|
| `StatusPill` | Renders any domain status (Order, Task, WorkUnit, Reservation, Slot, shift) with a consistent tone |
| `Timeline` | Lifecycle stepper (vertical by default; `orientation="auto"` goes horizontal on wide screens) — powers the Order Lifecycle screen |
| `Card` | Bordered content panel with optional title/actions header |
| `DataTable` | Typed, column-driven table |
| `FreshnessBadge` | Surfaces analytics-projection lag from each service's `/reports/*/freshness` endpoint |
| `KpiStat` | Control-tower stat panel (big number, label, threshold-colored accent bar) |
| `LaunchTile` | Fiori-style launchpad tile linking into a bounded context's screen |
| `AppShell` | Persistent top nav + site-switcher slot; `Breadcrumbs` ships beside it |
| `BarChart` | Dependency-free SVG bar chart (horizontal or vertical) for comparing named categories |
| `LineChart` | Dependency-free SVG line chart / sparkline for one time series, with an optional target line |
| `FunnelChart` | Stage funnel with tapering connectors and per-stage leakage — powers the WMS order funnel |
| `useFetch` | Minimal typed-fetch hook shared by every remote |

### Charts

The chart primitives draw plain SVG and pull **no** charting dependency. This package is a
Module Federation shared singleton, so a 100 kB chart library here is a cost every remote pays
on every screen. They are dashboard-card primitives — no zoom, no legend, no crosshair — and
they render exactly the numbers passed in, fetching nothing themselves.

```tsx
<FunnelChart
  title="Order funnel"
  stages={[
    { label: "Received", value: 1000 },
    { label: "Allocated", value: 820 },
    { label: "Released", value: 700 },
  ]}
/>
```

Every chart takes a `{ label, value }[]`, an optional `formatValue`, and an `emptyState`
(defaulting to `No data.`) so no screen ships a silent blank box. All colors and spacing come
from `tokens.css`; because CSS custom properties are invalid inside SVG presentation
attributes, the charts set `fill`/`stroke` through `style` rather than as attributes.

## License

MIT — see [LICENSE](LICENSE).
