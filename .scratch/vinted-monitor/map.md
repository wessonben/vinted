# Map: Vinted UK monitor

## Destination

A written spec (`docs/spec.md`) plus `CONTEXT.md` and ADRs for a local-first Vinted UK item monitor: a Hono poller + SQLite store + Vite/React viewer, monitoring up to 3 user-defined searches with feed-with-memory (new / price-drop / sold detection), configurable refresh rate, and TypeScript 7 (`tsgo`) used where the toolchain supports it. The map is complete when every open decision is locked and the way to build is clear.

## Notes

- Domain: Vinted UK (vinted.co.uk) unofficial web API; garment search by brand, category (style), condition, size, colour + optional free text.
- Skills: every session should consult `/grilling` and `/domain-modeling`. Research tickets resolve via `/research`.
- Stack so far: Hono (poller/API) + Vite/React SPA (viewer), SQLite, plain Node runtime, two workspaces (`poller/`, `viewer/`).
- Standing preferences: anonymous-first with optional pinned UK session cookie (manual paste); hotlink CDN images (no local photo storage); 7-day lightweight price history; per-panel last-viewed timestamps define "new"; viewer only, no alerts; page-1 polls with on-demand load-more.
- TypeScript 7 is a deliberate experiment — prefer `tsgo` where it works, document `tsc` fallbacks.

## Decisions so far

<!-- the index — one line per closed ticket: enough to judge relevance, then zoom the link for the detail the ticket holds -->

- [Vinted UK catalog API contract](issues/01-vinted-uk-catalog-api-contract.md) — Endpoint is `GET vinted.co.uk/api/v2/catalog/items`, cookie-authenticated (homepage visit sets `access_token_web` JWT); flat filter params `brand_ids/catalog_ids/status_ids/size_ids/color_ids` + `search_text`/price; `per_page` caps at 96, results at 960; no `created_at`/colour/condition-id in search items (timestamp from `photos[0].high_resolution.timestamp`); lookup endpoints (`/api/v2/brands`, facets, faceted_categories) all work.
- [tsgo toolchain integration](issues/02-tsgo-toolchain-integration.md) — TS 7.0 stable (8 Jul 2026), installed as `typescript`, invoked as `tsc` (native Go compiler); Vite transpiles with Oxc (type-check = `tsc --noEmit`); Hono checks cleanly; Node runs `.ts` natively via type stripping — no build step; keep `@typescript/typescript6` alias for API-consuming tooling (typescript-eslint).

## Not yet specified

- Spec assembly steps: exact sections and order for `docs/spec.md`.
- Panel UI fidelity details (card density, badge placement, pause/rate control placement).

## Out of scope

- Alerts / notifications (desktop, push, email) — viewer only.
- Storing photos locally — hotlink only.
- Cloud / server deployment — local-first; architected so a later move is a config change, not a rewrite.
- Multi-user auth / locking down the local UI.
- Browser-extension cookie capture — manual paste only.
- More than 3 searches.
- Logging into Vinted as the user (signed-in session reuse).
