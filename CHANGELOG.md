# Changelog

Every release of `rocketflare-plugin-analytics`, newest first. The porting instructions for each
live in `docs/upgrades/<version>.md`; this file is the index. `pnpm plugin upgrade analytics` in a
host app reads those notes and walks the `previous` chain, so a release with no note is a permanent
gap every copy has to step over.

## 1.0.0 — 2026-09-17

The first release: analytics as a plugin. Dashboards (`analytics_pages`), four tenant-scoped cubes
served by drizzle-cube at `/cubejs-api` and `/mcp`, one example fact table rebuilt hourly, the
`Dashboard` and `Analytics` CASL subjects, three UI routes, three CLI commands, and a contribution
seam for other plugins. Everything here was the Rocketflare kit's core until kit 0.6.0.

Requires kit `>=0.6.0 <1.0.0`. Porting note: `docs/upgrades/1.0.0.md`.
