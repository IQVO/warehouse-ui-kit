---
id: introduction
title: Introduction
sidebar_label: Introduction
---

# Introduction

`@warehouse/ui-kit` is the shared design-token and React component library of the
`warehouse-systems` fleet. It is not a bounded context: it owns no domain data, calls no
backend by itself and has no runtime of its own. It is a **frontend library** consumed by:

- `warehouse-console`, the Module Federation shell (host);
- every remote micro-frontend, each living in the `web/` folder of its own
  bounded-context repo. On `origin/develop` of those repos, thirteen `web/package.json`
  files declare `"@warehouse/ui-kit": "file:../../warehouse-ui-kit"`:

  | Repo | Remote package name |
  |---|---|
  | order-management | `order_mgmt_mfe` |
  | inventory-storage | `inventory-mfe` |
  | wes-work-planning | `planning-mfe` |
  | fulfillment-execution | `fulfillment-mfe` |
  | workforce-management | `workforce-mfe` |
  | facility-layout | `facility-mfe` |
  | process-path-management | `process-path-mfe` |
  | labor-performance | `labor-mfe` |
  | network-fulfillment | `netfulfil-mfe` |
  | warehouse-planning | `capacity-mfe` |
  | product-master | `productmaster-mfe` |
  | network-inventory-planning | `nip-mfe` |
  | inbound-receiving | `inbound-mfe` |

  `warehouse-ops-agent` and `slotting-optimization` have no `web/` remote.

Because every remote and the shell render through the same kit, it is the one place that
decides how the console looks and how domain facts are written on screen.

## What the kit owns

| Concern | Where | Why it lives here |
|---|---|---|
| Design tokens (color, spacing, radius, type, elevation, focus ring, motion, z-index) | `src/tokens/tokens.css` | A remote hand-rolling its own palette is a bug: visual consistency across a micro-frontend boundary is where MFE UX fails first. |
| Domain status to visual tone mapping | `src/components/StatusPill.tsx` (`STATUS_TONE`) | An `Order`, `Task`, `StockUnit`, `LocationSlot`, shift or `ProcessPath` status renders with the same word and color everywhere. |
| Layout primitives (`Card`, `DataTable`, `Timeline`, `KpiStat`, `LaunchTile`, `FreshnessBadge`) | `src/components/` | Loading, empty, "no data" and "unavailable" states are baked in once, so no screen ships a bare spinner or a silent blank box. |
| Persistent app chrome (`AppShell`, `Breadcrumbs`) | `src/components/AppShell.tsx` | The shell and any remote run standalone in local dev look like one product. |
| Dependency-free SVG charts (`BarChart`, `LineChart`, `FunnelChart`) | `src/components/*Chart.tsx`, `chartCommon.ts` | The kit is a Module Federation shared singleton; a charting library here would be paid by every remote on every screen. |
| Router-agnostic navigation (`NavigationProvider`, `useLinkComponent`) | `src/navigation/NavigationContext.tsx` | The host injects its router's `Link`; the kit never imports `react-router-dom`. |
| Data fetching (`useFetch`, `FetchError`) | `src/hooks/useFetch.ts` | One minimal typed-fetch hook with polling, stale-data retention and abort handling for every remote. |
| Formatters (`formatNumber`, `formatCompact`, `formatPercent`, `formatDuration`, `formatInstant`, `secondsSince`, `NO_VALUE`) | `src/utils/format.ts` | A quantity is written the same way everywhere; "no reading" is never rendered as `0`. |

## What it does not do

- It fetches nothing on its own: every component renders exactly the props it is given.
  Only `useFetch` performs HTTP, and only for the URL the caller passes.
- It reads no environment variables (`VITE_*` or otherwise) and has no runtime
  configuration; see [Configuration](../operations/configuration.md).
- It is not published to an npm registry: `package.json` is `"private": true` and
  consumers link it from a sibling checkout; see [Release](../operations/release.md).
- It ships no router, no state library and no charting library. React and React DOM are
  peer dependencies (`^19.0.0`).

## Documentation map

| Page | Contents |
|---|---|
| [Architecture](architecture.md) | Source layout (tokens, components, hooks, navigation, utils), build output, how consumers import it |
| [Quickstart](quickstart.md) | Install/link, peer dependencies, importing the stylesheet, theming with tokens, first screen |
| [Components](../components.md) | Every component with its props (from the TypeScript types) and a usage example |
| [Tokens](../tokens.md) | Every CSS custom property in `tokens.css`, the status lanes and the `data-surface` calibration |
| [Testing](../development/testing.md) | Vitest setup, every test file, `make` targets and CI jobs |
| [Configuration](../operations/configuration.md) | Build, TypeScript, lint and test configuration (the kit has no runtime env vars) |
| [Release](../operations/release.md) | Versioning, the absence of an npm publish step, how the console and the remotes pick up a change |
| [Troubleshooting](../operations/troubleshooting.md) | Symptom, cause, check and fix for the failure modes visible in the code |

Study-project disclaimer: like every repository in the `warehouse-systems` set, this is a
personal study project, not production software.
