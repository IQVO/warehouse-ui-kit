---
id: quickstart
title: Quickstart
sidebar_label: Quickstart
---

# Quickstart

## Prerequisites

| Requirement | Why |
|---|---|
| Node.js 22 | The version every job in `.github/workflows/ci.yml` uses (`actions/setup-node`, `node-version: "22"`); consumer Dockerfiles build on `node:22-bookworm-slim` |
| npm | `package-lock.json` is committed; `npm ci` installs from it |
| Sibling checkout layout | Consumers reference the kit by relative path: `warehouse-systems/warehouse-ui-kit` next to `warehouse-systems/warehouse-console` and `warehouse-systems/<context>/web` |
| GNU make (optional) | `Makefile` wraps the npm scripts with the fleet's `make check` vocabulary |

No backend, database, Kafka broker or environment variable is needed to build or test the
kit.

## Build and check the kit

```bash
cd warehouse-systems/warehouse-ui-kit
npm ci                 # or: make install
make check             # lint + typecheck + test + build (mirrors ci.yml)
npm run build          # tsc -b && vite build -> dist/
```

| npm script | Command | `make` target |
|---|---|---|
| `dev` | `vite build --watch` (rebuilds `dist/` on save for a linked consumer) | none |
| `build` | `tsc -b && vite build` | `make build` |
| `lint` | `oxlint` | `make lint` |
| `typecheck` | `tsc -b --noEmit` | `make typecheck` |
| `test` | `vitest run` | `make test` |
| `test:watch` | `vitest` | none |
| (none) | `npm audit --audit-level=high` | `make dependency-audit` |

Bundles: `make check` = `lint typecheck test build`; `make check-all` = `check` +
`dependency-audit`; `make check-fast` = `lint typecheck`. `make help` prints the list.

After `npm run build`, `dist/` contains `ui-kit.es.js`, `ui-kit.css` and the declarations
under `dist/src/` (entry `dist/src/index.d.ts`). `dist/` is gitignored.

## Use it from a consumer

The package is `"private": true` and is never published to a registry. A consumer links the
sibling checkout with a `file:` dependency:

```jsonc
// warehouse-console/package.json
"@warehouse/ui-kit": "file:../warehouse-ui-kit"

// <context>/web/package.json
"@warehouse/ui-kit": "file:../../warehouse-ui-kit"
```

Then:

1. Build the kit first (`npm run build` here). Consumers resolve the **built** package
   (`exports` point into `dist/`), so a fresh kit checkout without `dist/` makes the
   consumer's `tsc`/Vite fail to resolve `@warehouse/ui-kit`.
2. Run `npm install` in the consumer once so the `file:` link exists under its
   `node_modules/@warehouse/ui-kit`.
3. While iterating on the kit, keep `npm run dev` running here; the consumer's dev server
   picks up the rebuilt `dist/`.

### Peer dependencies

`react` and `react-dom` `^19.0.0` are peer dependencies and are marked `external` in
`vite.config.ts`, so the kit never bundles React. The console and every remote declare
`react`/`react-dom` `^19.2.8` and share them as Module Federation singletons together with
`@warehouse/ui-kit` itself. Two React copies at runtime break hooks silently; never add
React as a regular dependency here.

### Import the stylesheet once

```tsx
// main.tsx of the console or a remote
import "@warehouse/ui-kit/tokens.css";
```

`@warehouse/ui-kit/tokens.css` maps to `dist/ui-kit.css`, which contains the tokens **and**
every component stylesheet (`cssCodeSplit: false`). Without it components render with
unresolved `var(--wh-*)` values. The console's `src/main.tsx` and every remote's
`web/src/main.tsx` import it on `origin/develop`.

### Inject the router link (hosts with a client-side router)

```tsx
import { Link as RouterLinkBase } from "react-router-dom";
import { AppShell, NavigationProvider, type LinkProps } from "@warehouse/ui-kit";

// Module scope: the reference must be stable across renders.
function RouterLink({ href, children, ...rest }: LinkProps) {
  return <RouterLinkBase to={href} {...rest}>{children}</RouterLinkBase>;
}

export function App() {
  return (
    <NavigationProvider link={RouterLink}>
      <AppShell nav={[{ id: "orders", label: "Orders", href: "/orders", active: true }]}>
        {/* routes */}
      </AppShell>
    </NavigationProvider>
  );
}
```

Without a `NavigationProvider` every kit link is a plain `<a href>` (a full document
navigation), which is the safe default for a remote running standalone. The
`warehouse-console` shell wraps its tree in `<NavigationProvider link={RouterLink}>`
(`src/App.tsx`).

## First screen

```tsx
import {
  Card, DataTable, FreshnessBadge, KpiStat, StatusPill, useFetch, formatInstant,
} from "@warehouse/ui-kit";

type Order = { orderId: string; status: string; receivedAt: string };

export function OrdersScreen() {
  // No default polling: opt in only for side-effect-free endpoints.
  const { data, loading, error, stale } = useFetch<Order[]>("/api/orders", { pollMs: 15000 });
  return (
    <Card title="Orders" actions={stale ? <FreshnessBadge lagSeconds={null} /> : null}>
      <KpiStat label="Open orders" value={data ? data.length : null}
               state={error && !data ? "unavailable" : "ok"} />
      <DataTable
        loading={loading}
        rows={data ?? []}
        rowKey={(o) => o.orderId}
        columns={[
          { key: "id", header: "Order", render: (o) => o.orderId },
          { key: "status", header: "Status", render: (o) => <StatusPill status={o.status} /> },
          { key: "at", header: "Received", render: (o) => formatInstant(o.receivedAt) },
        ]}
      />
    </Card>
  );
}
```

The URL above is illustrative; each remote calls its own context's API through the edge.
Every prop is listed in [Components](../components.md).

## Theming

The kit has a single dark theme defined by the `:root` custom properties in
`src/tokens/tokens.css` (`html { color-scheme: dark; }`). There is no light theme and no
theme provider. Two supported knobs:

- **Surface calibration**: set `data-surface="monitor"` on any element and every font-size
  token inside it scales up (e.g. `--wh-font-size-2xl` from 28px to 44px) for a wall
  monitor; colors and status meaning never change.
- **Token override**: a consumer can redefine a `--wh-*` property on a subtree, but the
  rule in this repo is that a remote must not invent its own palette; new colors belong
  in `tokens.css`. See [Tokens](../tokens.md).
