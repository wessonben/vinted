# 01 - Vinted UK catalog API contract

Type: research
Status: resolved

Blocked by:

## Question

What is the exact shape of the unofficial Vinted UK (vinted.co.uk) catalog search API the poller will call? Map the endpoints, request parameters, response structure, pagination, and session-cookie requirements.

Concretely:

- The catalog search endpoint(s) used by the vinted.co.uk web app (e.g. `/api/v2/catalog/items`) — full URL, method, query parameters.
- How each of our search dimensions maps to parameters: brand (catalog id), category (style), condition (status ids), size (size id), colour (colour id), free-text search, price range.
- Pagination: page/cursor params, page size, total count, response envelope.
- Response shape for an item: id, title, brand, size, colour, condition, price, photo URLs, URL, sold/available flags, timestamps.
- What cookies/headers the endpoint expects (region pinning for UK, session cookie), and behaviour when called anonymously.
- Whether brand/category/colour/size lookups (for the UI autocomplete) share the same host and their endpoint shapes.

## Notes for the researcher

Use `/research` workflow. Primary sources: the vinted.co.uk web app's own network traffic and existing open-source Vinted API clients. Capture findings with URLs and exact parameter names.

## Answer

Full contract captured in `research/01-vinted-api-contract.md` (515 lines). Key facts the rest of the map depends on:

- **Endpoint:** `GET https://www.vinted.co.uk/api/v2/catalog/items` — JSON, cookie-authenticated. Bare request → HTTP 401 `code:100`; a homepage visit sets `access_token_web` (JWT) + `refresh_token_web` cookies, then API works. No Authorization header.
- **Filter params (flat, comma-join or bracket):** `search_text`, `catalog_ids`, `brand_ids`, `status_ids`, `size_ids`, `color_ids`, `material_ids`, `price_from`/`price_to`, `currency`, `order`, `page`, `per_page`, `time`, `country_ids`.
- **Dimensions:** brand=`brand_ids` (ids from `/api/v2/brands?search_text=`); category=`catalog_ids` (drill via `/api/v2/catalog/faceted_categories`); condition=`status_ids` (live ids 6=New with tags, 1=New without, 2=Very good, 3=Good, 4=Satisfactory — differ from older gists, resolve at runtime); size=`size_ids` (per-category via `/api/v2/catalog/filters/facets?filter_code=size`); colour=`color_ids` (facet hex via `filter_code=color`); free text=`search_text`.
- **Pagination:** `page` 1-indexed, `per_page` max 96 (clamped), results capped at **960 total**; `pagination.time` echoed back as `time` param pins the snapshot. `order=newest_first` for a new-items monitor.
- **Item shape:** id, title, price `{amount,currency_code}`, `is_visible`, `brand_title`, `size_title`, `status` (string), user, url, photos. **No created_at, no colour, no condition-id in search items** — listing timestamp from `photos[0].high_resolution.timestamp`; colour/condition-id/sold state need `/api/v2/items/{id}/details` (auth-gated, 403 anonymous).
- **Cookies:** `access_token_web`, `refresh_token_web`, `anonymous-iso-locale=en-GB`, `anon_id`, `__cf_bm` (Cloudflare). Browser User-Agent required; TLS-impersonating client (curl_cffi/cloudscraper) recommended for Cloudflare.
- **Lookups for autocomplete:** `/api/v2/brands`, `/api/v2/catalog/filters/facets`, `/api/v2/catalog/filters/search`, `/api/v2/catalog/faceted_categories`, `/api/v2/search_suggestions` all work. `/api/v2/catalog/initializers` is dead (404).
- **Errors:** `0` OK; `100` invalid token (401); `429` rate limit.
- **Unverified gaps:** `/details` field list from secondary source (403 live); new `/svc-catalogue/items` gateway in flux; whether plain co.uk scope excludes non-UK sellers unconfirmed.
