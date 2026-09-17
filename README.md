# rocketflare-plugin-analytics

Dashboards, cubes, fact tables and the drizzle-cube API for a [Rocketflare](https://github.com/rocketflare-dev/rocketflare) app.

**This is not an npm package.** A Rocketflare plugin is a git repository *copied into* your app —
exactly like the kit itself — so its code lands as ordinary source you can read, debug and edit,
translated into your app's own vocabulary on the way in. The price is that an upgrade is a patch
rather than a version bump, which the kit's tooling already knows how to do.

```bash
pnpm plugin add https://github.com/rocketflare-dev/rocketflare-plugin-analytics.git@1.0.0          # read the plan
pnpm plugin add https://github.com/rocketflare-dev/rocketflare-plugin-analytics.git@1.0.0 --apply  # then install
pnpm db:generate --name plugin-analytics-1.0.0 && pnpm db:migrate
```

A fresh kit installs it for you: it is in the kit's `.rocketflare.json` `defaultPlugins`, so
`bash scripts/bootstrap.sh` does the above as its `plugins` step. `pnpm bootstrap --no-plugins`
skips it, and `pnpm plugin remove analytics --apply` takes it back out.

**Requires kit `>=0.6.0 <1.0.0`.** Kit 0.6.0 is where analytics stopped being core.

## What it adds

| | |
|---|---|
| **Dashboards** | `analytics_pages` — a drizzle-cube `DashboardConfig` stored whole as jsonb, unique slug per tenant, restrictable to groups (D29). `/api/analytics/pages\|templates\|facts`, a list page, a view page with inline editing and autosave, and an explore page. |
| **Templates** | One ships — `tenant-overview` ("Organisation Overview") — copied into every organisation by `onTenantCreated` and repaired lazily on every `GET /pages`, so a template added later still reaches existing tenants. |
| **Cubes** | `ActivityEvents`, `TenantActivityDaily`, `TenantUsers`, `Users`, served by drizzle-cube at `/cubejs-api` and `/mcp`. Every cube scopes its own `sql()` by tenant; nothing else does it for them. |
| **Fact tables** | `analytics_tenant_activity_daily_facts`, rebuilt per tenant by the `15 * * * *` cron in one transaction per tenant. |
| **Permissions** | `Dashboard` (admin+ manage, member read) and `Analytics` (read for every role — the cube API is read-only by nature). |
| **CLI** | `rocketflare analytics pages list`, `check-facts` (exit 1 when stale), `refresh-facts`. |

## Contributing cubes from another plugin

The kit's core knows nothing about drizzle-cube, so a second plugin contributes through this one:

```ts
import { analyticsExtensions } from '@/plugins/analytics'

export const crmServer = {
  shared: crmShared,
  extensions: analyticsExtensions({
    cubes: [dealsCube],
    factTables: [dealsDailyFacts],
    dashboardTemplates: [pipelineTemplate],
    cubeIsolationCases: [dealsIsolationCase], // not optional in practice — see below
  }),
} satisfies ServerPlugin<typeof crmShared>
```

Declare `requires: { plugins: ['analytics'] }` in your `plugin.json`, so installing without this
plugin is refused rather than quietly doing nothing. Each list is zod-narrowed here and **throws
naming your plugin** when it cannot be parsed. `cubeIsolationCases` is how a contributed cube's
tenant scoping gets proven: the coverage assertion in `cube-isolation.test.ts` compares the case
keys to the whole registry, so a cube with no case fails the host's suite.

## After installing

Four things a plugin may not do for you, which `pnpm plugin add` prints as numbered steps — both
wrangler tomls (`[triggers].crons` and `[assets].run_worker_first`), the Vite dev proxy and two
`resolve` entries, the dependency install, and the migration. `docs/upgrades/1.0.0.md` has the
exact lines.

## Releasing

`node scripts/release.mjs X.Y.Z` — the kit's own release script, vendored here. It detects a
`rocketflare-plugin.json` with no `.rocketflare.json`, stamps that manifest's version and
`package.json`, folds `docs/upgrades/unreleased.md` into `docs/upgrades/X.Y.Z.md` and prepends the
`CHANGELOG.md` section. CI (`.github/workflows/ci.yml`) calls the kit's reusable `plugin-ci.yml`,
which clones the oldest and the newest kit inside `requires.kit`, installs this checkout into each
and runs the whole gate.

## Licence

MIT, the same as the kit. See `LICENSE`.

## The drizzle-cube CLI and Claude Code plugin (optional)

The semantic layer is Cube.js-compatible, so drizzle-cube's own CLI and its Claude Code plugin can
query it. They read `apps/web/.drizzle-cube.json` in the host app — a file the host creates, since
it sits outside the four directories a plugin may own, and one the kit already git-ignores:

```json
{
  "_comment": "Git-ignored. apiToken is a tenant API key from Settings → API keys (Bearer auth); it scopes every query to that tenant, exactly like any other request.",
  "serverUrl": "http://localhost:3001",
  "apiToken": "<TENANT_API_KEY>",
  "mode": "rest"
}
```

Verify: the CLI's `meta` lists `ActivityEvents`, `TenantActivityDaily`, `TenantUsers` and `Users`.
A browser MCP client additionally needs `mcp.allowedOrigins` in the plugin's `cube-api` route,
which it deliberately leaves unset — the default policy is loopback plus clients that send no
`Origin`, such as a desktop connector.
