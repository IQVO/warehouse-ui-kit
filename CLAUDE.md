# Project: warehouse-ui-kit (@warehouse/ui-kit)

harness-template: v3

Shared design tokens and React component library for the `warehouse-systems`
micro-frontend console. Consumed by `warehouse-console` (the shell) and every
remote micro-frontend (`order-mgmt-mfe`/`inventory-mfe`/`planning-mfe`/
`fulfillment-mfe`/`workforce-mfe`/`facility-mfe`/`process-path-mfe`/
`labor-mfe`), each living inside its own bounded-context repo's `web/`.

> **Study project.** Personal DDD/hexagonal/micro-frontend learning exercise.
> Not a production system; no uptime or support guarantee.

## What this repo is the ONE place for

- Design tokens (color, spacing, type, motion): `src/tokens/tokens.css`
- Domain-status to visual-tone mapping (`StatusPill`, `src/components/StatusPill.tsx`):
  every context's Order/Task/WorkUnit/Reservation/Slot status renders identically
- Layout primitives (`Card`, `DataTable`, `Timeline`, `KpiStat`, `LaunchTile`) and
  dependency-free SVG charts (`BarChart`, `LineChart`, `FunnelChart`; shared helpers in
  `src/components/chartCommon.ts`). No charting library, deliberately
- The persistent app chrome (`AppShell`), shared formatters (`src/utils/format.ts`) and
  the `useFetch` hook (`src/hooks/useFetch.ts`)

A remote hand-rolling its own palette or status colors is a bug, not a style choice.
A new domain-status color or chart type belongs HERE, not in the consuming remote.

## Non-negotiables (read before touching a component or the build)

1. **No registry publish step (single-workspace, pre-1.0).** Every consumer references
   this package as `file:../../warehouse-ui-kit` (the shell: `file:../warehouse-ui-kit`).
   No version to bump: commit, merge, and the consumer's next `npm install` picks it up.
   Consumers read the BUILT `dist/` (`package.json` `exports`), so run `npm run build`
   here before a consumer's own build/typecheck. Skill: `ship-ui-kit-change`.
2. **Docker builds of a CONSUMING remote cannot use this package's committed lockfile or
   a host `node_modules`** (Linux native bindings go missing: `Cannot find native
   binding ... @rolldown/binding-linux-*`). Keep `node_modules`/`dist` in `.dockerignore`;
   details in `.claude/rules/consumer-builds.md`.
3. **Never convert a `vite.config.ts` to the callback form** (`defineConfig(({ command })
   => ...)`) in this repo or in a consuming remote. Remotes' `vitest.config.ts` does
   `mergeConfig(viteConfig, ...)`, which throws `Cannot merge config in form of callback`
   and kills the ENTIRE suite. Keep the object form (`vite.config.ts` here is one);
   derive a remote's `base` statically with `process.argv.includes("build")`.
4. **A built remote's `dist/remoteEntry.js` being a ~144-byte relative re-export is
   CORRECT.** Do not force an absolute publicPath; see `.claude/rules/consumer-builds.md`.
5. **A remote with no `test` script silently never runs added tests.** Check `npm run`
   in that remote's `web/` and that its CI job invokes `npm test`.
6. **react/react-dom are peer deps and stay `external` in `vite.config.ts`.** Bundling
   React would create two React instances at runtime and break hooks silently.
7. **Charts and tokens: no new dependency, no invented palette.** Statuses must mirror
   the owning context's Go domain enum; never invent UI-only status labels. Chart colors
   go through the `style` prop (CSS `var()` is invalid in SVG presentation attributes).
8. **Public API is `src/index.ts` only.** Consumers import from `@warehouse/ui-kit`,
   never from `src/`. A component that is not exported there does not exist for them.
9. **Tests live beside components (`*.test.tsx`) and are excluded from
   `tsconfig.app.json`** so they never leak into published `dist/` types.

## Commands

```bash
npm ci                # or: make install
make check-fast       # lint + typecheck, the agent Stop-hook gate (run after every change)
make check            # lint typecheck test build (mirrors ci.yml)
make check-all        # check + dependency-audit: run before pushing
make guide-lint       # lint these agent guides (CI job guide-lint is blocking)
make harness-test     # unit-test the agent hooks
npm run test:watch    # vitest watch mode (jsdom + React Testing Library)
npm run dev           # vite build --watch, for a linked consumer
```

CI (`.github/workflows/ci.yml` + `.github/workflows/codeql.yml`) runs lint, typecheck,
test, build, `npm audit --audit-level=high`, trivy and guide-lint on every push/PR.
Branch flow: GitFlow, PRs into `develop`; never push to develop/main (hooks block it).

## Skills (`.claude/skills/`) and rules (`.claude/rules/`)

- `add-ui-kit-component`: add a component or chart primitive, test it, export it.
- `ship-ui-kit-change`: how consumers pick up a change; build order, CI dual checkout.
- `ui-kit-fast-checks`: run and read lint/typecheck/test/build locally.
- `.claude/rules/components-and-tokens.md`: conventions for `src/**`.
- `.claude/rules/consumer-builds.md`: Docker/Vite/remoteEntry details for build files.

<!-- harness:scoped-rules:start (generated by tools/migrate_v3.py in warehouse-harness-template; do not hand-edit) -->
## Scoped rules and harness

Claude Code loads each rule below automatically when you touch the matching paths. OpenCode and Codex do NOT: read the rule BEFORE editing matching files.

| When touching | Read |
|---|---|
| `src/**` | `.claude/rules/components-and-tokens.md` |
| `.dockerignore`, `vite.config.ts`, `vitest.config.ts`, `package.json`, `.github/**`, `Makefile` | `.claude/rules/consumer-builds.md` |

Hooks (`scripts/harness/hook.py`, wired for Claude Code, Codex and OpenCode) block pushes to develop/main, `--no-verify`, bare `rm -rf`, and edits to generated files, and feed lint findings back after each edit. Before saying "done" run `make check-fast`; the full gate is `make check-all`. `HARNESS_OFF=1` disables the hooks when debugging the harness itself.
<!-- harness:scoped-rules:end -->
