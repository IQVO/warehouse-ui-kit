---
name: ui-kit-fast-checks
description: Run and interpret the local quality gates for warehouse-ui-kit (make check-fast, make check, make check-all, guide-lint, harness tests) and fix the usual failures. Use when you finish an edit and need to say done, when a Stop-hook or CI job (lint, typecheck, test, build, dependency-audit, guide-lint) fails, or after a fresh clone where node_modules is missing.
---

# Fast checks for warehouse-ui-kit

Every `Makefile` target mirrors a CI job in `.github/workflows/ci.yml`.

| Command | Runs | Use |
|---|---|---|
| `make install` | `npm ci` | first time / after lockfile change |
| `make check-fast` | lint + typecheck | after every edit; the agent Stop hook runs this |
| `make check` | lint typecheck test build | before committing |
| `make check-all` | `check` + `npm audit --audit-level=high` | before pushing |
| `make guide-lint` | `scripts/harness/guide_lint.py` | after editing CLAUDE.md, `.claude/**` |
| `make harness-test` | `scripts/harness/test_hook.py` | after touching hooks (do not edit managed hook files) |
| `npx vitest run src/components/KpiStat.test.tsx` | one test file | tight loop |

## If it fails

- **Tools not found / `Cannot find module`:** `node_modules` is missing (it is gitignored).
  Run `npm ci` (needs network). Use Node 22 (CI uses `node-version: "22"`).
- **lint (oxlint, `.oxlintrc.json`):** `react/rules-of-hooks` is an error;
  `react/only-export-components` is a warning (keep non-component exports out of component
  files). `jsx-a11y` rules are on: charts need `role="img"` and an aria-label.
- **typecheck (`tsc -b --noEmit`):** `verbatimModuleSyntax` wants `import type`;
  `noUnusedLocals`/`noUnusedParameters`/`erasableSyntaxOnly` are on (no enums, no parameter
  properties). Tests are checked by `tsconfig.test.json`, not `tsconfig.app.json`.
- **test (vitest, jsdom):** config in `vitest.config.ts`, setup in `src/test/setup.ts`,
  tests are `src/**/*.test.{ts,tsx}`. RTL `cleanup()` runs via setup; a `getByText` that
  matches twice usually means two renders in one test.
- **build:** `tsc -b && vite build`, output `dist/` (`dist/ui-kit.es.js`, types, CSS).
  A build error about an export means it is missing from `src/index.ts`.
- **dependency-audit:** high/critical advisory; fix by updating the dependency (Dependabot
  PRs land here), not by loosening `--audit-level`.
- **guide-lint:** read the message; it names the file and line. Backticked repo paths in
  CLAUDE.md/rules/skills must exist; skills need `name` == directory and a description with
  a "Use when" clause (max 1024 chars).

## Done criteria

State which of `make check-fast` / `make check` / `make check-all` you actually ran and show
its result. If tools were missing and `npm ci` was impossible, say that instead of claiming
a pass. Never bypass hooks (`--no-verify` is blocked) and never push to develop or main.
