---
paths:
  - ".dockerignore"
  - "vite.config.ts"
  - "vitest.config.ts"
  - "package.json"
  - ".github/**"
  - "Makefile"
---

# Consumer builds: Docker, Vite, remoteEntry, test wiring

Detail moved out of CLAUDE.md. The hard rules stay in CLAUDE.md's non-negotiables.

## Why `.dockerignore` excludes `node_modules` and `dist`

Nine sibling frontend images (the `warehouse-console` shell and the bounded-context
remotes) receive this checkout as a named Docker build context
(`docker build --build-context uikit=../../warehouse-ui-kit .`). A host `node_modules`
holds only darwin bindings (`@rolldown/binding-darwin-arm64`, `lightningcss-darwin-*`,
`@oxlint/binding-darwin-*`); npm treats them as installed, never fetches the linux ones,
and `vite build` dies with `Cannot find native binding ... @rolldown/binding-linux-*`
(npm/cli#4828). Do not remove those entries from `.dockerignore`; the header comment
there explains it.

## Fixing a remote's Dockerfile against this package

The remote's own Dockerfile (for example `order-management/web/Dockerfile`) must:

- use base `node:22-bookworm-slim`, NOT alpine (npm 11 rejects npm-10-authored lockfiles
  outright; glibc matches the published `*-gnu` bindings);
- in the ui-kit build stage run `rm -rf node_modules package-lock.json`, then
  `npm install --no-save && npm run build`;
- copy only `package.json` and `dist` from that stage into
  `/workspace/warehouse-ui-kit` of the remote's build stage (matches `exports` and `files`
  in this repo's `package.json`).

## vite config form

`vite.config.ts` stays in OBJECT form (`export default defineConfig({...})`). A callback
form breaks `mergeConfig(viteConfig, defineConfig({...}))` in a `vitest.config.ts`. Here
`vitest.config.ts` is deliberately separate (the library build runs vite-plugin-dts
against `tsconfig.app.json`; tests are outside that project). React/react-dom must stay in
`build.rollupOptions.external`.

## `remoteEntry.js` is small on purpose

A built remote with a `/mfes/<context>/` base has a ~144-byte `dist/remoteEntry.js` that
re-exports a relative `./assets/virtual_mf-REMOTE_ENTRY_ID...js`. Grepping it for
`/mfes/<context>/` returns nothing, and that is correct: `dist/index.html` carries the
absolute prefixed URLs, chunks use relative ESM imports, and the preload helper resolves
via `new URL("../" + e, import.meta.url)`. Do not force an absolute publicPath; verify by
serving the built image and fetching through the real gateway path.

## Missing `test` script in a remote

`npm test` in a remote with no `test` script fails with `Missing script: "test"` and CI
only notices if that repo's `web:` job runs `npm test`. When adopting this package in a
new remote, run `npm run` in its `web/` and confirm the job invokes the tests.

## CI in this repo

`.github/workflows/ci.yml` jobs: lint, typecheck, guide-lint (blocking), test, build,
dependency-audit (`npm audit --audit-level=high`), trivy-scan. Actions are pinned to
commit SHAs; keep them pinned when editing. `Makefile` targets mirror these jobs;
update both together.
