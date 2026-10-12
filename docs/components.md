---
id: components
title: Components
sidebar_label: Components
---

# Components

Every component in `src/components/`, plus the hook, navigation context and formatters
exported from `src/index.ts`. Props are copied from the TypeScript types on
`origin/develop`; defaults are the destructuring defaults in each function. Import
everything from the package root:

```tsx
import "@warehouse/ui-kit/tokens.css";
import { StatusPill, DataTable /* ... */ } from "@warehouse/ui-kit";
```

Shared rules that hold for every component:

- Components render exactly the props they receive and fetch nothing (only `useFetch`
  performs HTTP).
- Missing values render as `NO_VALUE` (`—`), never as `0`.
- Colors come from `--wh-*` tokens; see [Tokens](tokens.md).

| Component | File | Has stylesheet | Has tests |
|---|---|---|---|
| [`StatusPill`](#statuspill) | `StatusPill.tsx` | no (inline styles) | no |
| [`Timeline`](#timeline) | `Timeline.tsx` | `Timeline.css` | `Timeline.test.tsx` |
| [`Card`](#card) | `Card.tsx` | no | no |
| [`DataTable`](#datatable) | `DataTable.tsx` | no | no |
| [`FreshnessBadge`](#freshnessbadge) | `FreshnessBadge.tsx` | no | no |
| [`KpiStat`](#kpistat) | `KpiStat.tsx` | `KpiStat.css` | `KpiStat.test.tsx` |
| [`LaunchTile`](#launchtile) | `LaunchTile.tsx` | `LaunchTile.css` | no |
| [`AppShell` and `Breadcrumbs`](#appshell-and-breadcrumbs) | `AppShell.tsx` | `AppShell.css` | no |
| [`BarChart`](#barchart) | `BarChart.tsx` | no | `BarChart.test.tsx` |
| [`LineChart`](#linechart) | `LineChart.tsx` | no | `LineChart.test.tsx` |
| [`FunnelChart`](#funnelchart) | `FunnelChart.tsx` | no | `FunnelChart.test.tsx` |

## StatusPill

Renders any domain status as a pill with a colored dot. `STATUS_TONE` in
`StatusPill.tsx` is the one status-to-tone table for the whole console.

| Prop | Type | Default | Meaning |
|---|---|---|---|
| `status` | `DomainStatus` | required | The status string exactly as the owning service returns it; also rendered as the label |
| `tone` | `"neutral" \| "progress" \| "success" \| "warning" \| "danger"` | derived from `STATUS_TONE` | Escape hatch to override the derived tone; prefer extending `STATUS_TONE` |
| `size` | `"sm" \| "md"` | `"md"` | `sm` uses `--wh-font-size-xs` and `3px 8px` padding; `md` uses `--wh-font-size-sm` and `5px 10px` |

The rendered `<span>` carries `data-status` and `data-tone` attributes. An unknown status
falls back to `neutral` instead of throwing (`DomainStatus` ends with `string & {}`), so a
new backend enum value never breaks a screen; gray where you expected a color means a
missing `STATUS_TONE` entry.

`DomainStatus` values and their tone, grouped by owning context as in the source:

| Owning context (source enum) | Status | Tone |
|---|---|---|
| order-management (`Order.Status`) | `Received` | progress |
| | `Allocated` | success |
| | `PartiallyAllocated` | progress |
| | `Backordered` | warning |
| | `Released` | success |
| | `PartiallyReleased` | success |
| | `Cancelled` | danger |
| order-management (`OrderLine.LineStatus`) | `Pending` | neutral |
| fulfillment-execution (`Task.Status`) | `PENDING` | neutral |
| | `CLAIMED` | progress |
| | `COMPLETED` | success |
| inventory-storage (StockUnit state) | `Available` | success |
| | `Reserved` | progress |
| | `Picked` | neutral |
| | `Removed` | danger |
| | `Unlocated` | danger |
| facility-layout (LocationSlot state) | `Occupied` | success |
| | `Free` | neutral |
| | `Decommissioned` | neutral |
| workforce-management (shift/associate state) | `OnShift` | success |
| | `OnBreak` | progress |
| | `OffShift` | neutral |
| process-path-management (`ProcessPath.Status`) | `ACTIVE` | success |
| | `DEACTIVATED` | neutral |

To add a status: add it to the `DomainStatus` union **and** to `STATUS_TONE`, under its
owning context, mirroring that context's Go domain enum. Never invent UI-only labels.

```tsx
<StatusPill status="Backordered" />
<StatusPill status="CLAIMED" size="sm" />
```

## Timeline

Lifecycle stepper, built for the Order Lifecycle screen's cross-service narrative.
Partial-failure tolerant: render whatever stages came back and mark the rest `pending`
(not reached) or `unavailable` (owning service did not answer).

| Prop | Type | Default | Meaning |
|---|---|---|---|
| `steps` | `TimelineStep[]` | required | Ordered stages |
| `orientation` | `"auto" \| "horizontal" \| "vertical"` | `"vertical"` | `horizontal` adds `wh-timeline--horizontal`; `auto` is horizontal above 900px viewport width and vertical at or below it (`@media (max-width: 900px)` in `Timeline.css`) |

`TimelineStep`:

| Field | Type | Required | Meaning |
|---|---|---|---|
| `id` | `string` | yes | React key |
| `context` | `string` | yes | Owning bounded context, shown as a small uppercase caption |
| `title` | `string` | yes | Stage name |
| `state` | `TimelineStepState` | yes | See below |
| `timestamp` | `string` | no | Rendered as-is (format it with `formatInstant` first) |
| `detail` | `ReactNode` | no | Free-form detail under the title |
| `warnings` | `string[]` | no | One inline warning row per entry |
| `unavailableReason` | `string` | no | Shown only when `state` is `unavailable`, e.g. "inventory-storage did not respond" |

`TimelineStepState` and how each renders (color and glyph, so state is never conveyed by
color alone; a visually hidden label is also rendered for screen readers):

| State | Glyph | Color token | Screen-reader label |
|---|---|---|---|
| `done` | ✓ | `--wh-color-status-success` | completed |
| `active` | • | `--wh-color-status-progress` | in progress |
| `pending` | (hollow dot) | `--wh-color-status-neutral` | not reached yet |
| `unavailable` | (dashed warning ring) | `--wh-color-status-warning` | no signal |
| `warning` | ! | `--wh-color-status-warning` | completed with warnings |
| `error` | ✕ | `--wh-color-status-danger` | failed |

Each `<li>` carries `data-state` and the class `wh-timeline__step--<state>`.

```tsx
<Timeline
  orientation="auto"
  steps={[
    { id: "recv", context: "order-management", title: "Received", state: "done", timestamp: formatInstant(o.receivedAt) },
    { id: "alloc", context: "inventory-storage", title: "Allocated", state: "unavailable",
      unavailableReason: "inventory-storage did not respond" },
    { id: "pick", context: "fulfillment-execution", title: "Picked", state: "pending" },
  ]}
/>
```

## Card

Bordered content panel (`<section>`) with an optional header.

| Prop | Type | Default | Meaning |
|---|---|---|---|
| `title` | `ReactNode` | none | A string renders as an `<h2>` at `--wh-font-size-md`; any other node renders as-is |
| `actions` | `ReactNode` | none | Right-aligned header content (buttons, a `FreshnessBadge`) |
| `children` | `ReactNode` | required | Body |
| `padded` | `boolean` | `true` | Body padding `--wh-space-5`; `false` for edge-to-edge content such as a `DataTable` |

The header renders only when `title` or `actions` is set. `Card`'s props are an inline
type; there is no exported `CardProps`.

```tsx
<Card title="Pick queue" actions={<FreshnessBadge lagSeconds={lag} />} padded={false}>
  <DataTable columns={cols} rows={rows} rowKey={(r) => r.id} />
</Card>
```

## DataTable

The one table component every remote should use: sticky header, `scope="col"` headers,
skeleton loading rows and a built-in empty state.

| Prop | Type | Default | Meaning |
|---|---|---|---|
| `columns` | `Column<T>[]` | required | Column definitions |
| `rows` | `T[]` | required | Data |
| `rowKey` | `(row: T) => string` | required | React key per row |
| `emptyState` | `ReactNode` | `No data.` | Rendered in a full-width cell when `rows.length === 0` |
| `loading` | `boolean` | `undefined` | Renders `loadingRowCount` shimmering skeleton rows (hidden from assistive tech) instead of the body and sets `aria-busy` on the table |
| `loadingRowCount` | `number` | `5` | Skeleton row count |
| `onRowClick` | `(row: T) => void` | none | Makes rows clickable (`cursor: pointer`) |

`Column<T>`:

| Field | Type | Required | Meaning |
|---|---|---|---|
| `key` | `string` | yes | React key for header and cells |
| `header` | `string` | yes | Header text (rendered uppercase) |
| `render` | `(row: T) => ReactNode` | yes | Cell content |
| `align` | `"left" \| "right" \| "center"` | no (`left`) | Applied to header and cells |
| `width` | `string` | no | CSS width of the header cell |

```tsx
<DataTable<Task>
  columns={[
    { key: "id", header: "Task", render: (t) => <span className="wh-numeric">{t.taskId}</span> },
    { key: "qty", header: "Qty", align: "right", render: (t) => formatNumber(t.quantity) },
    { key: "status", header: "Status", render: (t) => <StatusPill status={t.status} size="sm" /> },
  ]}
  rows={tasks ?? []}
  rowKey={(t) => t.taskId}
  loading={loading}
  onRowClick={(t) => openTask(t.taskId)}
/>
```

## FreshnessBadge

Renders analytics-projection lag, typically from a context's `/reports/<name>/freshness`
endpoint, so no dashboard silently presents lagging data as live.

| Prop | Type | Default | Meaning |
|---|---|---|---|
| `lagSeconds` | `number \| null` | required | `null` renders "freshness unknown" (neutral) |
| `staleThresholdSeconds` | `number` | `120` | Above it the badge is warning-toned; at or below it success-toned |

Text is `<lag> behind`, where lag is `Ns` under 60 s, `Nm` under 3600 s, otherwise hours with
one decimal (`1.5h`). This uses a local `formatLag`, not `formatDuration` from
`utils/format` (see [Troubleshooting](operations/troubleshooting.md)).

```tsx
const { data } = useFetch<{ lagSeconds: number | null }>("/api/<context>/reports/<name>/freshness");
<FreshnessBadge lagSeconds={data?.lagSeconds ?? null} staleThresholdSeconds={300} />
```

The response field name depends on the context's reports API; map it to `lagSeconds`.

## KpiStat

A control-tower stat panel: label, big monospace number, optional caption and delta, and a
left accent bar colored by tone. Renders as a link (through the injected link component)
when `href` is set.

| Prop | Type | Default | Meaning |
|---|---|---|---|
| `label` | `string` | required | Short label, rendered uppercase |
| `value` | `ReactNode \| number \| null` | required | `null`/`undefined` renders `—` and resolves the state to `no-data`; never `0` |
| `caption` | `string` | none | One line under the value |
| `tone` | `KpiTone` (`"neutral" \| "progress" \| "success" \| "warning" \| "danger"`) | none | Accent color; prefer a tone your backend computed |
| `threshold` | `KpiThreshold` | none | Presentational thresholds, used only when `tone` is not set and `value` is a finite number |
| `format` | `KpiFormat` (`"integer" \| "percent" \| "duration" \| "compact" \| (value: number) => string`) | grouped integer (`formatNumber`) | Applied only to numeric values; `percent` → `formatPercent`, `duration` → `formatDuration` (seconds), `compact` → `formatCompact` |
| `state` | `KpiState` (`"ok" \| "no-data" \| "unavailable"`) | `"ok"` | `unavailable` = the endpoint did not answer: value `—`, caption replaced by "no signal", warning accent, dashed warning border (`wh-kpi--unavailable`) |
| `delta` | `{ value: number; goodDirection: "up" \| "down" }` | none | Movement vs the previous window; shown only in state `ok`; arrow ↑/↓/→ plus good/bad/flat color |
| `href` | `string` | none | Renders the whole panel as a drill-down link |

`KpiThreshold` fields: `warnAbove`, `dangerAbove`, `warnBelow`, `dangerBelow` (all optional
numbers). Evaluation order: `dangerAbove`, `dangerBelow`, `warnAbove`, `warnBelow`, else
`success`. Tone resolution: `unavailable` → `warning`; otherwise `tone`, then the threshold
result, then `neutral` (accent blue). The root element carries `data-state` with the
resolved state.

```tsx
<KpiStat label="Pick queue" value={brief?.pendingTasks ?? null} caption="tasks pending"
         tone={brief?.overAlarmThreshold ? "danger" : undefined} href="/fulfillment/tasks" />
<KpiStat label="Projection lag" value={lag} format="duration"
         threshold={{ warnAbove: 120, dangerAbove: 600 }} />
<KpiStat label="Throughput" value={null} state={error ? "unavailable" : "ok"} />
```

## LaunchTile

A launchpad tile (SAP Fiori pattern) linking into a context's screen. Hover and focus are
CSS (`:hover`, `:focus-visible`), so keyboard users get the same affordance.

| Prop | Type | Default | Meaning |
|---|---|---|---|
| `context` | `string` | required | Bounded-context name, e.g. `order-management` (uppercase caption) |
| `title` | `string` | required | Human title, e.g. "Orders" |
| `description` | `string` | required | One-line description |
| `href` | `string` | required | Target; rendered through the injected link component |
| `badge` | `string` | none | Small live status, e.g. a KPI number or "3 sites" |

```tsx
<LaunchTile context="inventory-storage" title="Inventory" description="Stock units, reservations and adjustments"
            href="/inventory" badge={formatCompact(units)} />
```

## AppShell and Breadcrumbs

`AppShell` is the persistent top chrome reused by the console and by any remote running
standalone in dev: skip link ("Skip to content"), brand, primary nav, optional site
switcher and a focusable `<main id="wh-main">`.

| Prop | Type | Default | Meaning |
|---|---|---|---|
| `nav` | `NavItem[]` | required | Primary navigation items |
| `siteSwitcher` | `ReactNode` | none | Slot for the site/building switcher |
| `brand` | `ReactNode` | `"Warehouse Console"` | Wordmark; a standalone remote should name itself |
| `renderLink` | `LinkComponent` | none | Per-instance override of the injected link component (prefer `NavigationProvider`) |
| `children` | `ReactNode` | required | Page content |

`NavItem`: `id: string` (key), `label: string`, `href: string`, `active?: boolean` (adds
`wh-shell__nav-item--active` and `aria-current="page"`). The nav scrolls horizontally
instead of squashing labels when it overflows the 56px bar.

`Breadcrumbs` takes `items: { label: string; href?: string }[]`; items with `href` render
as links, the item without one renders as the current page (`aria-current="page"`).
Separators are `/`. `Breadcrumbs` does not accept `renderLink`; it always uses the context.

```tsx
<AppShell
  brand="Inventory (standalone)"
  nav={[
    { id: "home", label: "Home", href: "/" },
    { id: "inv", label: "Inventory", href: "/inventory", active: true },
  ]}
  siteSwitcher={<SiteSelect />}
>
  <Breadcrumbs items={[{ label: "Inventory", href: "/inventory" }, { label: "SU-0042" }]} />
  {children}
</AppShell>
```

## Charts: shared contract

`BarChart`, `LineChart` and `FunnelChart` draw plain SVG (fixed 320-unit-wide `viewBox`,
`width: 100%`) and pull no charting dependency. They share `chartCommon.ts`:

| Export | Kind | Meaning |
|---|---|---|
| `ChartDatum` | type (public) | `{ label: string; value: number }`; non-finite values are treated as missing |
| `ChartTone` | type (public) | `"neutral" \| "accent" \| "success" \| "warning" \| "danger"` |
| `CHART_TONE_COLOR` | internal | Tone → `--wh-color-status-*` / `--wh-color-accent` |
| `formatChartValue` | internal | Default formatter: grouped integer, one decimal for fractions, `—` for non-finite |
| `safeValue`, `ratioOfMax`, `maxOf`, `truncateLabel` | internal | Non-finite → 0, clamp to `0..1`, max floored at 0, ellipsis truncation |

Every chart has a `title` (rendered as an uppercase `<figcaption>` and used in the SVG
`aria-label`), a `formatValue` (default `formatChartValue`), a `tone` (default `accent`) and
an `emptyState` (default `No data.`) rendered when the data array is empty. Long labels are
truncated; the full `label: value` is in a native `<title>` tooltip.

## BarChart

Compares a handful of named categories.

| Prop | Type | Default | Meaning |
|---|---|---|---|
| `data` | `ChartDatum[]` | required | Categories in reading order |
| `title` | `string` | none | Caption |
| `formatValue` | `(value: number) => string` | `formatChartValue` | Per-bar readout |
| `orientation` | `"horizontal" \| "vertical"` | `"horizontal"` | Horizontal gives labels a real column (truncated at 14 chars); vertical suits ordered buckets |
| `tone` | `ChartTone` | `"accent"` | Bar color |
| `height` | `number` | `128` | Plot height in user units, vertical only (minimum 40); horizontal derives height from the row count (24 units per row) |
| `emptyState` | `ReactNode` | `No data.` | Shown when `data` is empty |

Bars share a `0..max` domain. Negative values draw a zero-length bar while the readout shows
the real number; any non-zero positive value gets at least a 2-unit sliver.

```tsx
<BarChart title="Orders by status" data={[{ label: "Released", value: 700 }, { label: "Backordered", value: 42 }]} />
```

## LineChart

Sparkline for a single time series.

| Prop | Type | Default | Meaning |
|---|---|---|---|
| `data` | `ChartDatum[]` | required | Chronological series; only the first and last labels are drawn (12 chars max) |
| `title` | `string` | none | Caption |
| `formatValue` | `(value: number) => string` | `formatChartValue` | Used in point tooltips |
| `threshold` | `LineChartThreshold` | none | Dashed reference line |
| `tone` | `ChartTone` | `"accent"` | Line and area color |
| `height` | `number` | `88` | Plot height excluding the label band (minimum 32) |
| `emptyState` | `ReactNode` | `No data.` | Shown when `data` is empty |

`LineChartThreshold`: `value: number` (included in the y-domain so the line is always on
screen), `label?: string` (drawn at the right end), `tone?: ChartTone` (default `warning`).

The y-domain is the data range (plus the threshold) padded by 10%; it does **not** start at
zero. A flat series is centered rather than dividing by zero; a single point renders only
the end dot.

```tsx
<LineChart title="Picks per hour" data={hourly} threshold={{ value: 1200, label: "target" }}
           formatValue={(v) => `${formatNumber(v)} /hr`} />
```

## FunnelChart

Stage funnel with tapering connectors; the gap between consecutive stages shows the
leakage (`−<drop> (<pct of previous>)`) in the danger color.

| Prop | Type | Default | Meaning |
|---|---|---|---|
| `stages` | `ChartDatum[]` | required | Ordered stages, widest first; never re-sorted |
| `title` | `string` | none | Caption |
| `formatValue` | `(value: number) => string` | `formatChartValue` | Stage counts and drop-off |
| `tone` | `ChartTone` | `"accent"` | Funnel body color; leakage is always `danger` |
| `emptyState` | `ReactNode` | `No data.` | Shown when `stages` is empty |

Widths scale against the largest stage (so a stage bigger than its predecessor is drawn,
not clipped); the conversion percentage on the right is always against the entry (first)
stage, `—` when the entry is 0. No leakage text is drawn when a stage grows.

```tsx
<FunnelChart title="Order funnel" stages={[
  { label: "Received", value: 1000 }, { label: "Allocated", value: 820 }, { label: "Released", value: 700 },
]} />
```

## useFetch and FetchError

`useFetch<T>(url: string | null, opts?: UseFetchOptions): FetchState<T>` — a minimal
typed GET hook (`fetch(url)` then `res.json()`).

`UseFetchOptions`:

| Option | Type | Default | Meaning |
|---|---|---|---|
| `pollMs` | `number` | none (no polling) | Poll interval. Deliberately no default: two wes-work-planning endpoints publish Kafka events when read, so polling must be an explicit opt-in |
| `enabled` | `boolean` | `true` | Skip fetching without unmounting; `url === null` has the same effect |
| `pauseWhenHidden` | `boolean` | `true` | Stop polling while `document.visibilityState` is `hidden`; refetch and resume on return |

`FetchState<T>`:

| Field | Meaning |
|---|---|
| `data` | Last good payload, kept across failed refreshes; `null` only before the first success (and after the URL changes) |
| `loading` | True only while there is nothing to show yet (first load) |
| `refreshing` | A request is in flight over data already on screen |
| `error` | Last error, cleared when a new request starts |
| `stale` | `data !== null && error !== null`: what is on screen is older than it looks |
| `lastUpdatedAt` | Epoch ms of the last success |
| `refetch` | Re-runs the request |

Non-2xx responses throw `FetchError` (`name: "FetchError"`, `status: number | null`,
`url: string`, message `Request failed with <status> <statusText>`), so a screen can tell
404 from 5xx. Network failures surface as the original `Error`. A newer request aborts the
older one, out-of-order responses are discarded, the in-flight request is aborted on
unmount, and state resets synchronously when `url` changes.

```tsx
const { data, error, stale, refetch } = useFetch<Order>(orderId ? `/api/orders/${orderId}` : null);
if (error instanceof FetchError && error.status === 404) return <p>Order not found.</p>;
```

## NavigationProvider and useLinkComponent

| Export | Signature | Meaning |
|---|---|---|
| `NavigationProvider` | `({ link: LinkComponent; children: ReactNode })` | Injects the host router's link component for every kit link below it. `link` must be a stable, module-scope reference |
| `useLinkComponent` | `() => LinkComponent` | Returns the injected component, or a plain-anchor default |
| `LinkProps` | `{ href: string; children: ReactNode; className?: string; "aria-current"?: "page" \| undefined; [key: string]: unknown }` | Props every injected link receives |
| `LinkComponent` | `ComponentType<LinkProps>` | |

Used by `AppShell` (nav items), `Breadcrumbs`, `KpiStat` (`href`) and `LaunchTile`.

## Formatters (`src/utils/format.ts`)

| Export | Signature | Output |
|---|---|---|
| `NO_VALUE` | `"—"` | Placeholder for "no reading" |
| `formatNumber` | `(value: number \| null \| undefined) => string` | `13,480`; one decimal for fractions (`12.5`); `—` for null/non-finite |
| `formatCompact` | same | `12.4K` (`Intl` compact notation, en-US, max 1 decimal) |
| `formatPercent` | same | `formatNumber(value)` + `%`, e.g. `98.2%` (the value is already a percentage, not a ratio) |
| `formatDuration` | `(seconds) => string` | `45s`, `12m`, `3h 04m`, `2d 06h`; negatives clamp to `0s` |
| `formatInstant` | `(iso: string \| null \| undefined) => string` | en-US local time, e.g. `Oct 12, 09:30 AM`; returns the raw string if unparseable; `—` when empty |
| `secondsSince` | `(iso, now = Date.now()) => number \| null` | Elapsed seconds (≥ 0), `null` when empty or unparseable |
