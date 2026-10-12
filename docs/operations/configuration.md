---
id: configuration
title: Configuration
sidebar_label: Configuration
---

# Configuration

## Runtime configuration: none

`@warehouse/ui-kit` reads **no environment variables** and has no runtime configuration.
A search of `src/`, `vite.config.ts` and `vitest.config.ts` finds no `import.meta.env`,
`process.env` or `VITE_*` reference. Everything that varies per consumer is passed in as
props or through React context:

| What varies | How a consumer sets it |
|---|---|
| API base URLs | The consumer builds the URL and passes it to `useFetch(url)`; the kit never prefixes it |
| Polling | `useFetch(url, { pollMs, enabled, pauseWhenHidden })`; no default interval |
| Router integration | `<NavigationProvider link={...}>` (or `renderLink` on `AppShell`) |
| Brand / site switcher | `AppShell` `brand` and `siteSwitcher` props |
| Freshness threshold | `FreshnessBadge` `staleThresholdSeconds` (default 120 s) |
| KPI thresholds / tone | `KpiStat` `tone` or `threshold` |
| Density | `data-surface="monitor"` on a subtree (see [Tokens](../tokens.md)) |

The consumers' own `VITE_*` variables (for example API base URLs) are documented in those
repos, not here.

## Package manifest (`package.json`)

| Field | Value | Notes |
|---|---|---|
| `name` | `@warehouse/ui-kit` | The specifier consumers import |
| `private` | `true` | `npm publish` refuses to publish it |
| `version` | `0.1.0` | Not bumped per change; see [Release](release.md) |
| `type` | `module` | ESM only |
| `main` / `module` | `./dist/ui-kit.es.js` | |
| `types` | `./dist/src/index.d.ts` | |
| `exports` | `"."` → `types`/`import`; `"./tokens.css"` → `./dist/ui-kit.css` | Only these two subpaths are importable |
| `files` | `["dist"]` | |
| `peerDependencies` | `react ^19.0.0`, `react-dom ^19.0.0` | No runtime `dependencies` at all |

## Build configuration (`vite.config.ts`)

Object form `export default defineConfig({...})`; keep it that way (a callback form breaks
`mergeConfig` in a `vitest.config.ts`).

| Option | Value |
|---|---|
| `plugins` | `react()`, `dts({ tsconfigPath: "tsconfig.app.json", insertTypesEntry: true })` |
| `build.lib.entry` | `src/index.ts` |
| `build.lib.name` | `WarehouseUiKit` |
| `build.lib.formats` | `["es"]` |
| `build.lib.fileName` | `` (format) => `ui-kit.${format}.js` `` |
| `build.rollupOptions.external` | `react`, `react-dom`, `react/jsx-runtime` |
| `build.rollupOptions.output.globals` | `react: "React"`, `react-dom: "ReactDOM"` (only meaningful for UMD/IIFE, which are not built) |
| `build.cssCodeSplit` | `false` (single `dist/ui-kit.css`) |

## Test configuration (`vitest.config.ts`)

`plugins: [react()]`, `environment: "jsdom"`, `setupFiles: ["./src/test/setup.ts"]`,
`include: ["src/**/*.test.{ts,tsx}"]`. See [Testing](../development/testing.md).

## TypeScript

`tsconfig.json` has no files of its own; it references three projects, so `tsc -b` builds
all of them:

| Project | Includes | Notable options |
|---|---|---|
| `tsconfig.app.json` | `src`, excluding `src/**/*.test.ts(x)` and `src/test` | `target es2023`, `lib ES2023 + DOM`, `types ["vite/client"]`, `moduleResolution bundler`, `jsx react-jsx`, `allowArbitraryExtensions`, `verbatimModuleSyntax`, `erasableSyntaxOnly`, `noUnusedLocals`, `noUnusedParameters`, `noFallthroughCasesInSwitch`, `noEmit`; also the project `vite-plugin-dts` generates `dist/` declarations from |
| `tsconfig.node.json` | `vite.config.ts`, `vitest.config.ts` | `module nodenext`, `types ["node"]` |
| `tsconfig.test.json` | `src/**/*.test.ts(x)`, `src/test` | Same strictness as the app project, `types ["node"]` |

Consequences for contributors:

- `verbatimModuleSyntax`: type-only imports must use `import type`.
- `erasableSyntaxOnly`: no `enum`, `namespace` or constructor parameter properties.
- `allowArbitraryExtensions`: a relative `import "./Foo.css"` resolves to a `Foo.d.css.ts`
  file, which is why component stylesheets are imported from `src/index.ts`.
- Build-info files go to `node_modules/.tmp/*.tsbuildinfo`.

Installed compiler: `typescript ~7.0.2`, with `@typescript/typescript6 ^6.0.2` also in
`devDependencies`.

## Lint (`.oxlintrc.json`)

`oxlint` with plugins `react`, `typescript`, `oxc`, `jsx-a11y`. Explicit rules:
`react/rules-of-hooks: error`, `react/only-export-components: warn`
(`allowConstantExport: true`), `jsx-a11y/prefer-tag-over-role: off`. The source contains a
few `eslint-disable-next-line` comments (`jsx-a11y/control-has-associated-label` in
`DataTable.tsx`, `react/set-state-in-effect` in `useFetch.ts`, `react/static-components` in
`NavigationContext.tsx`) and one file-level `eslint-disable react/only-export-components`
in `NavigationContext.tsx`, each with its justification beside it.

## Docker build context (`.dockerignore`)

The kit has no Dockerfile. Consumer images receive this checkout as a named build context
(`docker build --build-context uikit=../../warehouse-ui-kit .`). `.dockerignore` excludes
`node_modules` and `dist` (a macOS `node_modules` would carry only darwin native bindings
and break the Linux `vite build` with `Cannot find native binding ...
@rolldown/binding-linux-*`), plus `.vite`, `coverage`, logs, `.DS_Store`, `.git`,
`.gitignore`, `.github`, `docs` and `README.md`. Do not remove the `node_modules`/`dist`
entries.

## Dependency updates (`.github/dependabot.yml`)

Weekly Dependabot PRs for `npm` (root) and `github-actions`, up to 10 open each.
