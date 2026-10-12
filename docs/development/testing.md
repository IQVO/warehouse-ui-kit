---
id: testing
title: Testing
sidebar_label: Testing
---

# Testing

The kit is a component library, so its test pyramid is short: component and hook tests in
Vitest + React Testing Library on jsdom, plus static checks (oxlint, `tsc -b`), a
production build, a dependency audit, Trivy and CodeQL in CI. There are no integration,
contract, BDD, mutation or end-to-end suites in this repo; consumers exercise the kit in
their own `web/` test suites and builds.

## Tooling

| Piece | Version range (`package.json`) | Configured in |
|---|---|---|
| Vitest | `^5.0.2` | `vitest.config.ts` |
| jsdom | `^30.1.1` | `test.environment: "jsdom"` |
| @testing-library/react | `^16.3.3` | used directly in test files |
| @testing-library/jest-dom | `^7.0.1` | `src/test/setup.ts` imports `@testing-library/jest-dom/vitest` |
| @vitejs/plugin-react | `^6.1.0` | `plugins: [react()]` |

`vitest.config.ts`:

```ts
test: {
  environment: "jsdom",
  setupFiles: ["./src/test/setup.ts"],
  include: ["src/**/*.test.{ts,tsx}"],
}
```

It is deliberately separate from `vite.config.ts`: the library build runs
`vite-plugin-dts` against `tsconfig.app.json`, and tests are kept out of that project so
they never leak into the published `dist/` types. Test files are type-checked by
`tsconfig.test.json` (included through the `tsconfig.json` project references, so
`npm run typecheck` covers them).

`src/test/setup.ts` registers the jest-dom matchers and calls RTL's `cleanup()` after each
test (RTL does not auto-clean without Vitest globals mode).

## Test inventory

Real output of `npx vitest run --reporter=verbose` on this branch: **6 test files, 45 tests,
all passing**.

| File | Tests | What it pins down |
|---|---|---|
| `src/components/BarChart.test.tsx` | 8 | One labelled bar per datum with formatted values; widths scale against the largest value; caller `formatValue`; vertical orientation with baseline and labels; default and custom empty state; no NaN geometry when every value is 0; negative values clamp to a zero-length bar but still show the number |
| `src/components/FunnelChart.test.tsx` | 8 | Every stage with value and conversion against entry; leakage between consecutive stages; narrowing with tapering connectors; no leakage readout when a stage does not shrink; caller `formatValue` for stages and leakage; placeholder conversion (not NaN) when entry is 0; default and custom empty state |
| `src/components/KpiStat.test.tsx` | 7 | Thousands grouping; missing value renders as an em dash, never 0; `unavailable` marked distinctly from a real zero; tone derived from `threshold`; `format` applied to numbers; delta direction exposed to assistive tech; plain `div` without `href`, link with it |
| `src/components/LineChart.test.tsx` | 9 | Polyline through every point with first/last labels; higher value plotted above lower; threshold line with caption; threshold outside the data range stays on screen; per-point `<title>` tooltip; single point = dot, no polyline; flat series centred; default and custom empty state |
| `src/components/Timeline.test.tsx` | 6 | `unavailable` distinct from `pending`; names the service that went quiet; state exposed to assistive tech, not by color alone; ordered list with one item per step; horizontal orientation; defaults to vertical |
| `src/hooks/useFetch.test.tsx` | 7 | Loads once and reports the payload; keeps last good data on a failed refresh and marks it `stale`; no `loading` on background refresh; HTTP status surfaced (404 vs 503); never polls without `pollMs`; polls at the requested interval; no fetch when `url` is `null` or `enabled` is false |

Not covered by any test file: `StatusPill`, `Card`, `DataTable`, `FreshnessBadge`,
`LaunchTile`, `AppShell`/`Breadcrumbs`, `NavigationContext` and `src/utils/format.ts`
(the formatters are exercised indirectly through `KpiStat`). The console's
`src/shell/RouterLink.test.tsx` (in `warehouse-console`) covers `NavigationProvider` with
`AppShell` and `LaunchTile` from the consumer side.

## Running tests

```bash
npm test                     # vitest run (CI's command)
npm run test:watch           # vitest in watch mode
npx vitest run --reporter=verbose
npx vitest run src/components/KpiStat.test.tsx
make test                    # same as npm test
```

## Writing a test

Conventions from `.claude/rules/components-and-tokens.md` and the existing files:

- Put `Foo.test.tsx` beside `Foo.tsx` in `src/`; it is picked up by the
  `src/**/*.test.{ts,tsx}` glob and excluded from `tsconfig.app.json` (and so from
  `dist/` types).
- Query by role and accessible name (`getByRole("img", { name: /bar chart/ })`,
  `getByRole("link")`), and assert on the `data-state` / `data-tone` / `data-status`
  attributes the components expose, rather than on inline colors.
- For SVG charts, assert on geometry attributes (`width`, `points`) and on the `<title>`
  tooltip text; check that no attribute contains `NaN`.
- For `useFetch`, stub fetch with `vi.stubGlobal("fetch", vi.fn())` (undo with
  `vi.unstubAllGlobals()`) and use `vi.useFakeTimers()` + `vi.advanceTimersByTimeAsync`
  for polling, as `useFetch.test.tsx` does.

## Make targets and local gates

| Target | Runs |
|---|---|
| `make lint` | `npm run lint` (oxlint with `.oxlintrc.json`: plugins `react`, `typescript`, `oxc`, `jsx-a11y`) |
| `make typecheck` | `npm run typecheck` (`tsc -b --noEmit` over the app, node and test projects) |
| `make test` | `npm test` |
| `make build` | `npm run build` (`tsc -b && vite build`) |
| `make dependency-audit` | `npm audit --audit-level=high` |
| `make check` | `lint typecheck test build` (run after every change) |
| `make check-all` | `check` + `dependency-audit` (run before pushing) |
| `make check-fast` | `lint typecheck` (the agent Stop-hook gate) |
| `make guide-lint` | `python3 scripts/harness/guide_lint.py` (agent guides) |
| `make harness-test` | `python3 scripts/harness/test_hook.py` |

## CI

`.github/workflows/ci.yml` runs on push to `develop`/`main` and on pull requests into
`develop`/`main`. Every Node job uses `actions/setup-node` with Node 22 and `npm ci`.
Actions are pinned to commit SHAs.

| Job | Runs |
|---|---|
| `lint` | `npm run lint` |
| `typecheck` | `npm run typecheck` |
| `guide-lint` | `python3 scripts/harness/guide_lint.py`, `repo_lint.py`, `test_hook.py`, `test_repo_lint.py` (blocking, 10-minute timeout) |
| `test` | `npm test` |
| `build` | `npm run build` |
| `dependency-audit` | `npm audit --audit-level=high` |
| `trivy-scan` | `aquasecurity/trivy-action` filesystem scan, `CRITICAL,HIGH`, `ignore-unfixed`, fails on findings |

Other workflows:

| Workflow | Trigger | Runs |
|---|---|---|
| `.github/workflows/codeql.yml` (`CodeQL`) | push/PR on `develop`/`main`, weekly cron `19 3 * * 1` | CodeQL `javascript-typescript`, `build-mode: none` |
| `.github/workflows/ai-review.yml` (`AI Review (advisory)`) | PRs into `develop` touching `internal/domain/**`, `internal/application/**`, `apis/**`, `charts/**` | Advisory architecture review; this repo has none of those paths, so it never triggers here |

`.github/dependabot.yml` opens weekly update PRs for `npm` and `github-actions` (up to 10
each).

## Consumer-side verification

A kit change is only proven when a consumer builds against it. Every consumer CI checks
out `IQVO/warehouse-ui-kit` at `ref: develop` next to itself and runs
`npm ci && npm run build` in it before its own lint/typecheck/test/build (see
[Release](../operations/release.md)). Locally: `npm run build` here, then run the
consumer's `npm run typecheck`/`npm test`/`npm run build` in its `web/` (or the console
root).
