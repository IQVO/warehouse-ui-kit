---
name: ship-ui-kit-change
description: Get a ui-kit change into the console and every remote, and verify it there. Covers the file dependency, why dist must be built first, the CI dual-checkout pattern consumers use, and the Docker named build context. Use when a ui-kit change must be seen by warehouse-console or a remote web/ app, when a consumer build or typecheck cannot find an export or types, or when editing a consumer CI job or Dockerfile that builds this package.
---

# How consumers pick up ui-kit changes

## The mechanism

- No registry publish; no version bump. `package.json` here is `private`, and
  consumers declare `"@warehouse/ui-kit": "file:../../warehouse-ui-kit"` (remote `web/`
  folders) or `file:../warehouse-ui-kit` (the `warehouse-console` shell). Sibling-checkout
  layout under `warehouse-systems/` is therefore required.
- Consumers resolve the BUILT package: `main`/`module`/`types` and `exports` in
  `package.json` all point into `dist/` (`dist/ui-kit.es.js`, `dist/src/index.d.ts`,
  `dist/ui-kit.css` as `@warehouse/ui-kit/tokens.css`). `dist/` is gitignored, so a fresh
  checkout has no types or JS until `npm run build` runs here. Symptom of forgetting:
  consumer `tsc`/vite cannot resolve `@warehouse/ui-kit` or a new export.

## Local loop

1. In this repo: `npm ci` (first time), edit, `make check`, then `npm run build`.
   While iterating, `npm run dev` (`vite build --watch`) rebuilds `dist/` on save.
2. In the consumer's `web/` (or the console root): `npm install` once so the `file:` link
   exists, then run its own lint, typecheck, test, build. Check `npm run` there first: some
   remotes have no `test` script (`CLAUDE.md` non-negotiable 5).
3. Do not edit `node_modules/@warehouse/ui-kit` or commit `dist/`.

## CI dual-checkout pattern (as used by consumers)

A consumer's workflow checks out itself AND this repo, builds the kit, then builds itself.
Reference implementations: `order-management/.github/workflows/ci.yml` (search for
`repository: IQVO/warehouse-ui-kit`) and `warehouse-console/.github/workflows/ci.yml`:

```yaml
- uses: actions/checkout@<sha>        # the consumer, path: <consumer-repo>
- uses: actions/checkout@<sha>
  with: { repository: IQVO/warehouse-ui-kit, ref: develop, path: warehouse-ui-kit }
- working-directory: warehouse-ui-kit
  run: npm ci && npm run build
# then npm ci / lint / tsc -b / npm test / npm run build in the consumer's web/
```

The kit is taken from `develop`, so a ui-kit change is only visible to consumer CI after it
is merged into `develop`. To test a consumer PR against an unmerged kit PR, point that
workflow's `ref:` at the kit branch temporarily and revert before merge. Merge the kit PR
first, then the consumer PR that uses the new export.

## Docker (consumer images)

The consumer supplies this checkout as a named context
(`docker build --build-context uikit=../../warehouse-ui-kit .`), then in a ui-kit stage runs
`rm -rf node_modules package-lock.json && npm install --no-save && npm run build` and copies
only `package.json` and `dist` onward (see `order-management/web/Dockerfile`). Details and
why: `.claude/rules/consumer-builds.md`. Base image must be `node:22-bookworm-slim`.

## Verifying in a consumer

Prefer the real build over assumption: build the kit, build the consumer, and grep the
consumer's output or run its tests for the new component. For a remote image, serve the
built image and fetch through the real gateway path rather than inspecting
`remoteEntry.js` (its small relative form is correct).

## Pitfalls

- Never change a consumer's `vite.config.ts` to callback form to work around this.
- Changing an export's name or props is a breaking change for up to nine consumers; grep
  them (`grep -rn "@warehouse/ui-kit" ../*/web/src ../warehouse-console/src`) before merging.
- Keep Action SHAs pinned when editing `.github/workflows/ci.yml` here.
