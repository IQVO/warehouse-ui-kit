---
id: troubleshooting
title: Troubleshooting
sidebar_label: Troubleshooting
---

# Troubleshooting

Failure modes visible in the code and in how consumers build the kit. Each row is
symptom → cause → check → fix.

## Build and resolution

| Symptom | Cause | Check | Fix |
|---|---|---|---|
| Consumer `tsc`/Vite: `Cannot find module '@warehouse/ui-kit'` or its types | Consumers resolve the **built** package (`exports` point into `dist/`) and `dist/` is gitignored, so a fresh kit checkout has nothing to resolve | `ls warehouse-ui-kit/dist` should list `ui-kit.es.js`, `ui-kit.css`, `src/index.d.ts` | `npm ci && npm run build` in the kit, then `npm install` once in the consumer so the `file:` link exists |
| Consumer: `Module '"@warehouse/ui-kit"' has no exported member 'X'` for something that exists in `src/` | Stale `dist/`, or `X` is not exported from `src/index.ts` (only that file is public; `chartCommon` helpers, `Link` and `TimelineOrientation` are internal) | `grep -n "X" src/index.ts`; check `dist/src/index.d.ts` | Export it from `src/index.ts` (component **and** prop types), rebuild |
| Consumer CI fails right after a kit merge though the consumer did not change | Consumer CI builds the kit from `ref: develop`; a renamed/removed export or prop is a breaking change for every consumer | The failing step names the missing export/prop; `git log origin/develop` in the kit | Adapt the consumer (or restore the export); merge kit and consumer PRs back to back |
| Consumer CI passes but the change is not there | The kit PR is not merged into `develop` yet | `git branch -r --contains <sha>` in the kit | Merge first, or temporarily point the consumer workflow `ref:` at the kit branch |
| Docker build of a consumer: `Cannot find native binding ... @rolldown/binding-linux-*` | A host (macOS) `node_modules` or the macOS-authored lockfile reached the Linux build stage (npm/cli#4828) | `.dockerignore` here still lists `node_modules` and `dist`; the consumer Dockerfile's `uikit-build` stage runs `rm -rf node_modules package-lock.json && npm install --no-save` | Keep both; `facility-layout/web/Dockerfile` reuses the lockfile with `npm ci`, so align it with the other remotes if it hits this |
| `vitest`: `Cannot merge config in form of callback` | A `vite.config.ts` (here or in a consumer) was converted to `defineConfig(({ command }) => ...)` and a `vitest.config.ts` calls `mergeConfig` on it | Look at the first line of `export default` in `vite.config.ts` | Keep the object form; derive a remote's `base` statically |
| `tsc` error on a component's `import "./Foo.css"` (looks for `Foo.d.css.ts`) | `tsconfig.app.json` sets `allowArbitraryExtensions` | The import is beside the component instead of in `src/index.ts` | Import the stylesheet from `src/index.ts` after `tokens.css` |
| Test helpers or `*.test.tsx` types appear in `dist/src/` | A test path entered `tsconfig.app.json` (the project `vite-plugin-dts` reads) | `exclude` in `tsconfig.app.json` | Keep `src/**/*.test.ts(x)` and `src/test` excluded |
| CI `dependency-audit` fails | `npm audit --audit-level=high` found a high/critical advisory | `npm audit` locally | Bump the dependency (Dependabot opens weekly PRs); do not lower the level |
| CI `trivy-scan` fails | Trivy found a fixable CRITICAL/HIGH vulnerability in the filesystem scan | The job log lists the package and fixed version | Update the package/lockfile |

## Runtime in a consumer

| Symptom | Cause | Check | Fix |
|---|---|---|---|
| Components render unstyled; computed styles show unresolved `var(--wh-*)` | `@warehouse/ui-kit/tokens.css` (which is `dist/ui-kit.css`, tokens **and** component sheets) is not imported | `grep -rn "ui-kit/tokens.css" src` in the consumer | `import "@warehouse/ui-kit/tokens.css";` once in `main.tsx` |
| `Invalid hook call` or hooks misbehave only when the kit is used | Two React instances (React bundled into the kit, or not shared as a singleton) | `vite.config.ts` here must keep `react`, `react-dom`, `react/jsx-runtime` in `rollupOptions.external`; the consumer's federation `shared` must mark `react`/`react-dom` `singleton: true` | Restore the externals/singletons; never add React as a kit dependency |
| A remote renders with a different kit than it was built with | `@warehouse/ui-kit` is a federation singleton; only one bundled copy wins at runtime when the console and remotes were built from different kit commits | Compare the frontend image tags (`local-<hash>`); they share `uikit_source_hash` in `warehouse-infra/terraform/frontends.tf` | Rebuild and redeploy all frontends together (`terraform apply`) |
| Every nav click reloads the whole page and re-downloads `remoteEntry.js` | No `NavigationProvider`: the default link is a plain `<a href>` | React DevTools: is `NavigationProvider` above `AppShell`? | Wrap the tree in `<NavigationProvider link={RouterLink}>` (the console does this in `src/App.tsx`) |
| Links (nav, tiles, KPI drill-downs) lose focus/state and remount on every render | `link` passed to `NavigationProvider` is an inline arrow function, i.e. a new component type each render | Look at the `link={...}` expression | Declare the link component at module scope |
| `StatusPill` is gray where a color was expected | The status string is not in `STATUS_TONE` (unknown statuses fall back to `neutral`), or casing differs (`Pending` vs `PENDING` are different entries) | Inspect the pill: `data-status` and `data-tone="neutral"` | Add the status to `DomainStatus` and `STATUS_TONE` under its owning context, mirroring the Go enum; do not use `tone` as a permanent workaround |
| `KpiStat` shows `—` instead of `0` | `value` is `null`/`undefined` (by design `—` is "no reading"); `data-state="no-data"` | Inspect `data-state` on the tile | Pass a real `0` when zero is the measured value; pass `state="unavailable"` when the endpoint failed |
| A dead KPI looks calm | `state="unavailable"` not passed on fetch failure | `data-state` is `ok`/`no-data` while the request errors | Pass `state={error && !data ? "unavailable" : "ok"}` (renders dashed warning border and "no signal") |
| Timeline is vertical on a wide screen | `orientation` defaults to `"vertical"` | The `<ol>` lacks `wh-timeline--horizontal` | Pass `orientation="auto"` (horizontal above 900px) or `"horizontal"` |
| A Timeline stage says "no signal" with no reason | `unavailableReason` only renders when `state` is `unavailable` and it is set | The step object | Set `unavailableReason`, e.g. "inventory-storage did not respond" |
| Chart bars/lines invisible or black after a custom tweak | A color was set as an SVG presentation attribute (`fill="var(--x)"`), where `var()` is ignored | Inspect the element's attributes | Set `fill`/`stroke` through the `style` prop, as the kit's charts do |
| Chart shows `No data.` | `data`/`stages` is an empty array | Log the array passed in | Pass `emptyState` for a domain-specific message; the charts never fetch |
| Funnel shows `—` as every conversion | Entry stage value is `0` (conversion is measured against the first stage) | `stages[0].value` | Expected; check the upstream projection |
| `FreshnessBadge` shows "freshness unknown" | `lagSeconds` is `null` | The freshness response | Map the context's freshness field to `lagSeconds` |
| `FreshnessBadge` and a `KpiStat` with `format="duration"` write the same lag differently (`1.5h` vs `1h 30m`) | `FreshnessBadge` uses its own `formatLag`, not `formatDuration` from `utils/format` | `FreshnessBadge.tsx` | Known inconsistency; prefer one component per screen until it is consolidated |

## `useFetch`

| Symptom | Cause | Check | Fix |
|---|---|---|---|
| Screen never refreshes | `pollMs` has no default: without it `useFetch` fetches once | The call site options | Pass `pollMs`, only for side-effect-free endpoints |
| Reading a screen produces Kafka events | Polling an endpoint that publishes when read (wes-work-planning `/paths/{id}/telemetry` and `/rebalance`, per the `UseFetchOptions` comment) | Grep the consumer for `pollMs` on those URLs | Remove `pollMs` there; refresh on user action with `refetch()` |
| Polling stops when the tab is in the background | `pauseWhenHidden` defaults to `true` | `document.visibilityState` | Expected; it refetches and resumes on return. Pass `pauseWhenHidden: false` only if you must |
| Data is on screen but out of date, no visible error | A refresh failed: `data` keeps the last good payload and `stale` is `true` | `stale`, `error`, `lastUpdatedAt` from the hook | Render the stale state (e.g. `FreshnessBadge`, "updated N s ago" with `secondsSince`) |
| `error instanceof FetchError` is false for an outage | Only non-2xx responses throw `FetchError`; a network failure surfaces the original error from `fetch` (usually a `TypeError`) | `error.name` | Treat any non-`FetchError` error as "service unreachable"; use `FetchError.status` to tell 404 from 5xx |
| Nothing is fetched | `url` is `null` or `enabled: false` | The hook arguments | Expected; set the URL once the input exists |
| Spinner/skeleton flashes on every poll | It should not: background refreshes set `refreshing`, not `loading` | Is the screen keyed on `loading` from a *new* `url` each tick (e.g. a timestamp in the query string)? | Keep the URL stable; changing it resets state by design |

## Accessibility checks

| Symptom | Cause | Fix |
|---|---|---|
| No visible keyboard focus in a consumer | `tokens.css` not imported (the global `:focus-visible` ring lives there) or overridden by a consumer's `outline`/`box-shadow` | Import `tokens.css`; do not reset `:focus-visible` |
| Low-contrast faint text | `--wh-color-text-faint` was darkened | Keep `#78849a` (WCAG AA measured in `tokens.css`) |

## Known oddities (not failures)

- `DataTable` still renders its own `<style>` with `@keyframes wh-shimmer` per instance, even
  though `tokens.css` now defines the same keyframes globally (its comment says the per-table
  injection was removed). Duplicate identical keyframes are harmless.
- `chartCommon.ts` defines its own `NO_VALUE` and `formatChartValue`, duplicates of
  `NO_VALUE` and `formatNumber` in `utils/format.ts`; they produce the same output.
