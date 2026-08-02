# Research: Vinted UK (vinted.co.uk) catalog search API contract

- **Date of research:** 2 August 2026
- **Context:** Unofficial catalog-search API used by the Vinted UK web app, intended to back a Vinted UK listing monitor + search/filter UI.
- **Status:** Research only. No application code was written. Live requests made to `vinted.co.uk` / `vinted.fr` during research were read-only GETs.
- **Method:** Reverse-engineering against **primary sources**: (a) live HTTP calls to the web app's own API endpoints using a browser-cookie session, (b) the deployed Next.js JS bundle on `marketplace-web-assets.vinted.com` (the exact request builders the web app ships), and (c) widely-used open-source Vinted API clients. Everything below was **verified live against `https://www.vinted.co.uk` on 2026-08-02** unless explicitly marked "secondary source".

---

## 1. Executive summary

- The catalog search endpoint is **`GET https://www.vinted.co.uk/api/v2/catalog/items`** — JSON, no path versioning beyond `/api/v2`, same shape on all `www.vinted.<tld>` locales (verified on `co.uk` and `fr`).
- It is **not fully anonymous**: a request with no session returns **HTTP 401** with `{"code":100,"message":"Jeton d'authentification invalide","message_code":"invalid_authentication_token"}`. The web app authenticates via a cookie, not an `Authorization` header: first `GET https://www.vinted.co.uk/` returns a JWT cookie `access_token_web` (plus `refresh_token_web`), which is then sent with API calls.
- Filters are flat query params: `search_text`, `catalog_ids`, `brand_ids`, `status_ids`, `size_ids`, `color_ids`, `material_ids`, `price_from`, `price_to`, `country_ids`, `order`, `page`, `per_page`, `time`. The web app's own request builder flattens them as comma-joined values (e.g. `brand_ids=53,14`); bracket/repeated forms (`brand_ids[]=53`, `brand_ids=53&brand_ids=14`) also work.
- **Pagination is capped at 960 results total** (`total_entries` never exceeds 960; `total_pages = ceil(960 / per_page)`). `per_page` is capped at **96**. The `time` param (echoed from the previous response's `pagination.time`) is the web app's search-snapshot cursor that keeps page-to-page pagination stable.
- The response envelope is `{"code": 0, "items": [...], "pagination": {...}, "search_tracking_params": {...}}`. Each item carries id, title, price, brand_title, size_title, status (condition as a **string**), photos, user, URL — but **no colour field** and **no explicit created timestamp** (the listing time is recoverable from `photos[0].high_resolution.timestamp`).
- Lookup endpoints for an autocomplete UI all exist and are public-with-cookie: `/api/v2/brands`, `/api/v2/catalog/filters/facets`, `/api/v2/catalog/filters/search`, `/api/v2/catalog/faceted_categories`, `/api/v2/search_suggestions`. `/api/v2/catalog/initializers` (the old category-tree endpoint) is **dead — returns a 404 HTML page** on the live site.
- A newer, feature-flagged "OpenAPI gateway" set of routes (`/svc-catalogue/items`, `/svc-filters/filters`, `/svc-filters/filters/facets`, `/svc-filters/faceted_categories`, `/svc-filters/filters/search`) exists in the JS bundle behind a flag; it was **not** exercised live (base URL unconfirmed) — treat as in-flux.

---

## 2. The catalog search endpoint

### 2.1 Full URL and method

```
GET https://www.vinted.co.uk/api/v2/catalog/items
```

- Method: **GET** only.
- Base: `https://www.vinted.<domain>/api/v2` — the web app defines `API_BASE_PATH = "/api/v2"` in its bundle. [JS bundle]
- Domain per locale: `www.vinted.co.uk` (UK), `www.vinted.fr`, `www.vinted.com`, etc. All clients parameterise the domain and keep `/api/v2/catalog/items` fixed. [Pawikoski `vinted.py` `api_url = f"{self.base_url}/api/v2"`; vlymar1 `api/catalog.py` `f"{self.base_url}/api/v2/catalog/items"`]
- Content-Type of response: `application/json`.
- Envelope: `{"code": 0, "items": [...], "pagination": {...}, "search_tracking_params": {...}}`. `code: 0` = success; non-zero codes are error codes (see §8.5).

### 2.2 Query parameters

Verified live, and cross-checked against the web app's request builder (`ae`/`at` in bundle chunk `0mydq.q6aeo-k.js`) and the Pawikoski/vlymar1 client params:

| Param | Example | Meaning |
|---|---|---|
| `search_text` | `nike trainer` | Free-text query |
| `catalog_ids` | `1206` | Category (catalog) id; comma-join multiple |
| `brand_ids` | `53` or `53,14` | Brand ids |
| `status_ids` | `6` or `6,1` | Condition ids (see §3.3) |
| `size_ids` | `207` | Size ids (see §3.5) |
| `color_ids` | `1` | Colour ids (see §3.4) |
| `material_ids` | `5` | Material ids (verified accepted; not exercised) |
| `country_ids` | `GB` | Seller country filter |
| `price_from` / `price_to` | `10` / `100` | Price range (GBP on co.uk) |
| `currency` | `GBP` | Price currency |
| `order` | `newest_first` | Sort order (see §3.6) |
| `page` | `1` | 1-indexed page |
| `per_page` | `96` | Page size, **max 96** (web app constant `CATALOG_PER_PAGE = 96`) |
| `time` | `1785666545` | Search-snapshot cursor (see §4.3) |
| `popular` | `true` | Popular-search flag (from `isPopularCatalog`) |
| `catalog_from` | | "Browse" context flag |
| `disable_search_saving` | | Suppresses search-save side effects |
| `search_by_image_id` / `search_by_image_uuid` | | Image-search (reverse photo) mode |
| `search_session_id` / `global_search_session_id` / `global_catalog_browse_session_id` | uuid | Session correlation ids |
| `patterns_ids`, `video_game_platform_ids` | | Extra filter families accepted (Pawikoski client) |

The web app's `at()` builder appends `_ids` to each dynamic-filter code before serialising, so `attribute_ids.brand → brand_ids`, `attribute_ids.status → status_ids`, `attribute_ids.catalog → catalog_ids`, `attribute_ids.disposal → disposal_ids`, etc. Values are **comma-joined** into one value (`a.join(",")`). [JS bundle]

Serialisation forms that work **identically** (all verified live on co.uk):

```
brand_ids=53
brand_ids[]=53
brand_ids=53&brand_ids=14      (repeated key)
brand_ids[]=53&brand_ids[]=14
brand_ids=53,14                (comma-joined — what the web app sends)
```

The nested `attribute_ids[brand_ids]=53` form (used against the *new* `/svc-catalogue/items` gateway) is **ignored** by `/api/v2/catalog/items` — verified: it returned unfiltered items. [Live]

### 2.3 Sample working request

```http
GET https://www.vinted.co.uk/api/v2/catalog/items?search_text=nike&brand_ids=53&status_ids=6,1&size_ids=207&price_from=5&price_to=50&order=newest_first&per_page=20&page=1 HTTP/1.1
Host: www.vinted.co.uk
User-Agent: Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120.0.0.0 Safari/537.36
Accept: application/json, text/plain, */*
Accept-Language: en-GB,en;q=0.9
Referer: https://www.vinted.co.uk/catalog?search_text=nike
Cookie: access_token_web=<JWT>; refresh_token_web=<JWT>; anonymous-iso-locale=en-GB; anon_id=<uuid>
```

---

## 3. Search dimensions → API parameters

### 3.1 Brand

- Parameter: **`brand_ids`** (integers; comma-joined or repeated/bracket keys).
- ids come from `GET /api/v2/brands?search_text=<query>` (autocomplete) or from `GET /api/v2/catalog/filters/search?filter_search_code=brand&filter_search_text=<query>`. [Live]
- Verified live: `brand_ids=53` returns only Nike items; `brand_ids=53,14` returns Nike + adidas. [Live]
- Brands have globally-stable ids as far as checked: Nike=53, adidas=14 (from the live UK brands endpoint; also matches the FR reverse-engineering notes). Treat as "resolved from the API, not hard-coded". [Live, gist]

### 3.2 Category / style

- Parameter: **`catalog_ids`**.
- The web app's catalog URL is `/catalog/<id>-<slug>...` and the client extracts the leading id: `/catalog/1206-sneakers` → `catalog_ids=1206`. [vlymar1 `_extract_catalog_id`; live: `catalog_ids=1206` returned sneakers]
- Top-level category ids seen live: 1904=Women, 1193=Kids, 5=Men, 2309=Entertainment (from `/api/v2/catalog/faceted_categories`); drilling with `catalog_ids=1904` returns 4=Clothing, 16=Shoes, 1187=Accessories, 19=Bags, 146=Beauty. [Live]
- The web app also accepts a bare `catalog` param on the new gateway (`catalog=1206`), but on `/api/v2/catalog/items` only `catalog_ids` is honoured (bare `catalog=1206` was ignored — live). [Live, JS]

### 3.3 Condition (status)

- Parameter: **`status_ids`**.
- Current condition ids, **verified live and identical on co.uk and fr** (2026-08-02):

| id | UK label | FR label |
|---|---|---|
| 6 | New with tags | Neuf avec étiquette |
| 1 | New without tags | Neuf sans étiquette |
| 2 | Very good | Très bon état |
| 3 | Good | Bon état |
| 4 | Satisfactory | Satisfaisant |

- **Do not hard-code from older notes:** a popular reverse-engineering gist (FR, 2026-06) lists `1=Neuf avec étiquettes, 2=Neuf sans, 3=Très bon, 4=Bon, 5=Satisfaisant` — this is **out of date** (current live ids differ). Resolve at runtime from `/api/v2/catalog/filters/facets?filter_code=status`. [Live, gist]
- In item responses the condition is returned as a human-readable **string** (`status`, e.g. `"Very good"`), not the id. [Live]

### 3.4 Colour

- Parameter: **`color_ids`**.
- Verified live on co.uk (and identical on fr): 1=Black, 3=Grey, 12=White, 20=Cream, 4=Beige, 21=Apricot, 11=Orange, 22=Coral, … Each colour facet carries a `prefix.hex` (e.g. Black `000000`) useful for UI swatches. [Live]
- **The catalog search response contains no colour field per item** — colour is only available on the item-details endpoint. A monitor that needs item colour must fetch `/api/v2/items/{id}/details` (auth-gated; anonymous got 403 — see §8.3). [Live]

### 3.5 Size

- Parameter: **`size_ids`**.
- Size ids are **per category and per group** (Men's/Women's/Kids/Baby and, for footwear, UK/US/EU numerics). Resolve via `GET /api/v2/catalog/filters/facets?filter_code=size&catalog_ids=<category>` — returns grouped options, e.g. for `catalog_ids=1206` (sneakers): Men's sizes XS=206, S=207, M=208, L=209, XL=210, XXL=211, XXXL=212, … (group options have `type:"group"` and nested `options`). [Live]
- The search item carries only `size_title` (a human string, e.g. `"5 junior | EU 38"`), not the size id. [Live]

### 3.6 Free-text search

- Parameter: **`search_text`** (space-separated words; the web app joins with `+`). Empty/absent = unfiltered catalog. [Live]
- `GET /api/v2/search_suggestions?query=<text>` returns free-text autocomplete (title, `source:["taxonomy_data"]`, `entity_combination:["brand+parent_cat"]`). [Live]

### 3.7 Price range

- Parameters: **`price_from`**, **`price_to`** (decimal, GBP on co.uk). Both optional; verified filtering live. `currency` selects the price currency. [Live, Pawikoski]

### 3.8 Sort order

- Parameter: **`order`**.
- Values (web app enum `SortByOption`, plus live checks): `relevance` (default), `newest_first`, `price_low_to_high`, `price_high_to_low`. `oldest_first` is **accepted** by the API (verified) but is not in the web app's enum. [Live, JS]

---

## 4. Pagination

### 4.1 Parameters

- `page` — 1-indexed (default 1).
- `per_page` — default 96 in the web app (`CATALOG_PER_PAGE = 96`); **values above 96 are silently clamped to 96** (verified: per_page=97 and 100 both returned 96 items with `per_page: 96` echoed). [Live, JS]

### 4.2 Response block

```json
"pagination": {
  "current_page": 1,
  "total_pages": 10,
  "total_entries": 960,
  "per_page": 96,
  "time": 1785666562
}
```

### 4.3 The `time` cursor (important quirk)

- `pagination.time` is the server-side time of the search snapshot. The web app passes it back as the `time` query param on every subsequent page (`buildCatalogPaginationParams({page, perPage, time})`) — this pins pagination to the original snapshot and avoids page-shift as new items get listed. [JS bundle]
- Verified: requesting `page=2` with the `time` echoed from `page=1` returns a stable, non-overlapping page 2. Passing a fresh `time.time()` also works (that's what the Pawikoski wrapper does), but results can shift if new listings arrive between requests. [Live]
- **Cap:** `total_entries` never exceeds **960** and `total_pages = ceil(960 / per_page)` (verified: unfiltered co.uk search, which has millions of listings, returned `total_entries: 960`; with per_page=20, total_pages=48). A monitor that wants "everything new" therefore only ever sees the newest 960 items of any query. [Live]

### 4.4 Envelope

Top-level keys: `items`, `pagination`, `search_tracking_params`, `code`.

```json
"search_tracking_params": {
  "search_correlation_id": "a85a98ff-...",
  "global_catalog_browse_session_id": "9512872c-...",
  "search_session_id": "5e338203-..."
}
```

---

## 5. Item response shape

### 5.1 Field reference (verified live on 2026-08-02)

| Field | Type | Notes |
|---|---|---|
| `id` | int | Global item id |
| `title` | string | |
| `price` | object `{amount, currency_code}` | Amount as **string** ("7.0"); object shape observed both with and without `X-Money-Object: true` on this date |
| `is_visible` | bool | Visible/active flag (a sold/removed item is dropped from results entirely rather than flagged) |
| `brand_title` | string | e.g. "Nike" (brand id is **not** in the item payload) |
| `size_title` | string | e.g. "5 junior \| EU 38" |
| `status` | string | Condition label, e.g. "Very good" (id not included) |
| `user` | object | `{id, login, profile_url, photo, business}` |
| `url` | string | `https://www.vinted.co.uk/items/<id>-<slug>` |
| `path` | string | `/items/<id>-<slug>` |
| `photos` | array | All item photos (see below); `photos[0]` is the main photo |
| `photo` | object | Convenience = main photo (`photos[0]`) |
| `favourite_count`, `view_count` | int | |
| `is_favourite` | bool | (false for anonymous sessions) |
| `service_fee` | object | Buyer Protection fee, `{amount, currency_code}` |
| `total_item_price` | object | price + buyer protection |
| `conversion` | object\|null | Populated when seller's currency ≠ buyer's, e.g. `{seller_price, seller_currency, buyer_currency, fx_rounded_rate, ...}` — so co.uk results can include non-GBP sellers |
| `promoted` | bool | |
| `content_source` | string | e.g. `"search_promoted_items"`, `"realtime"` |
| `discount`, `badge`, `icon_badges` | mixed | Optional |
| `item_box` | object | UI helper: `{first_line, second_line, item_id, accessibility_label, exposures}` |
| `search_tracking_params` | object | Per-item `{score}` |

Photo object: `{id, image_no, width, height, dominant_color, dominant_color_opaque, url, is_main, thumbnails[], high_resolution{id, timestamp, orientation}, is_suspicious, full_size_url, is_hidden, extra}`.

### 5.2 Timestamps — there is no `created_at`

Search items have **no created/timestamp field**. The de-facto listing timestamp used by every open-source monitor is `photos[0].high_resolution.timestamp` (photo upload timestamp, epoch seconds) — this is what vlymar1's `CatalogItem.created_at_ts` is derived from. It is the photo time, which on re-listing ("bump") reflects the newest photo, not necessarily original creation. [Live, vlymar1 `models/item.py`]

### 5.3 Available/sold flags

- No explicit `sold` flag in search results. Active listings have `is_visible: true`; sold/closed/removed items are excluded from search.
- On the **details** endpoint (`/api/v2/items/{id}/details`) the payload has `is_closed`, `is_reserved`, `is_hidden`, `can_buy`, etc. (secondary-source field list from Pawikoski `models/items.py` `DetailedItem`; the endpoint itself was 403 for an anonymous curl — see §8.3).

### 5.4 Sample item (real response, trimmed thumbnails)

`GET /api/v2/catalog/items?search_text=nike&per_page=2` → `items[0]`:

```json
{
  "id": 9552573524,
  "title": "Nike Trainer - UK 5",
  "price": { "amount": "7.0", "currency_code": "GBP" },
  "is_visible": true,
  "brand_title": "Nike",
  "path": "/items/9552573524-nike-trainer-uk-5",
  "user": {
    "id": 98870513,
    "login": "jwilliamsx",
    "profile_url": "https://www.vinted.co.uk/member/98870513-jwilliamsx",
    "photo": {
      "id": 203487659,
      "url": "https://images1.vinted.net/t/02_026cd_5CGzJPUdphtpA16aCE3AJFdC/f800/1723402160.jpeg?s=5ca195e795f1d4b904774be0925265307b97655b",
      "dominant_color": "#31abc2",
      "high_resolution": { "id": "02_026cd_5CGzJPUdphtpA16aCE3AJFdC", "timestamp": 1723402160, "orientation": null },
      "full_size_url": "https://images1.vinted.net/tc/02_026cd_5CGzJPUdphtpA16aCE3AJFdC/1723402160.jpeg?s=1f4ee5c0558f27b84bb69515bcf659671c7697cd"
    },
    "business": false
  },
  "conversion": null,
  "url": "https://www.vinted.co.uk/items/9552573524-nike-trainer-uk-5",
  "promoted": true,
  "photos": [
    {
      "id": 40426740611,
      "image_no": 1,
      "width": 600,
      "height": 800,
      "url": "https://images1.vinted.net/t/02_010d4_VLJQB9z5o8H1Dyg9u6w2YoHf/f800/1785665755.jpeg?s=a6dea1bc500d907335450453601462f9ddda056f",
      "is_main": true,
      "high_resolution": { "id": "02_010d4_VLJQB9z5o8H1Dyg9u6w2YoHf", "timestamp": 1785665755, "orientation": null },
      "is_suspicious": false,
      "full_size_url": "https://images1.vinted.net/tc/02_010d4_VLJQB9z5o8H1Dyg9u6w2YoHf/1785665755.jpeg?s=e7976027ea456991aac45d7abe7f9a1f68712113"
    },
    { "id": 40426740624, "image_no": 2, "is_main": false, "url": "https://images1.vinted.net/t/04_00068_ozQCccKXHCynf72rGPama29Z/f800/1785665755.jpeg?s=f0628d566bff74a0b9b199941a7a3aa1024695db" }
  ],
  "favourite_count": 9,
  "is_favourite": false,
  "service_fee": { "amount": "0.86", "currency_code": "GBP" },
  "total_item_price": { "amount": "7.86", "currency_code": "GBP" },
  "view_count": 0,
  "size_title": "5 junior | EU 38",
  "content_source": "search_promoted_items",
  "status": "Very good",
  "item_box": {
    "first_line": "Nike",
    "second_line": "5 junior | EU 38 · Very good",
    "item_id": 9552573524
  },
  "search_tracking_params": { "score": 1 }
}
```

(Image host is `images1.vinted.net`; CDN URLs are signed with an `?s=` hash.)

---

## 6. Authentication, cookies and headers

### 6.1 Anonymous behaviour

- **No cookies at all → HTTP 401** (verified live):

```json
{"code":100,"message":"Jeton d'authentification invalide","message_code":"invalid_authentication_token"}
```

- **After one homepage visit**, the API returns 200. The web app and all clients follow the same bootstrap: `GET https://www.vinted.<tld>/` first, harvest cookies, then call the API. [Pawikoski `fetch_cookies()`; vlymar1 `session.refresh_cookies()`]

### 6.2 Cookies set by the homepage (verified live, vinted.co.uk)

| Cookie | Notes |
|---|---|
| `access_token_web` | HttpOnly JWT; `purpose: access`, `scope: public`, `aud: fr.core.api.vinted.com`; carries the `exp` the clients check for expiry |
| `refresh_token_web` | HttpOnly JWT, `purpose: refresh` |
| `anonymous-iso-locale` | `en-GB` on co.uk — the region/locale signal |
| `anon_id` | uuid |
| `v_udt` | opaque session value |
| `non_dot_com_www_domain_cookie_buster` | domain-sharding helper |
| `__cf_bm` | Cloudflare bot-management cookie |

Authentication is **cookie-based** (no `Authorization: Bearer` on this endpoint). vlymar1's `AuthManager` decodes `access_token_web` as a JWT and refreshes it when `exp` passes. [vlymar1 `auth.py`, live]

### 6.3 Headers

Verified working set (a browser UA is the one thing every client insists on):

```
User-Agent: Mozilla/5.0 (...) Chrome/120.0.0.0 Safari/537.36
Accept: application/json, text/plain, */*
Accept-Language: en-GB,en;q=0.9        (locale-tagged; co.uk → en-GB)
Referer: https://www.vinted.co.uk/      (or the /catalog page)
X-Requested-With: XMLHttpRequest        (sent by Pawikoski)
X-Money-Object: true                    (historical "prices as objects" header; vlymar1)
DNT: 1
Cache-Control: max-age=0
```

Notes: with a browser UA + cookie, `Accept-Language`, `Referer`, `DNT` and `X-Money-Object` are all **optional** — the endpoint served identical responses without them. [Live] The `User-Agent` matters because Vinted is fronted by Cloudflare (`__cf_bm` cookie) and the clients use TLS-fingerprint spoofing (`curl_cffi ... impersonate="chrome"`, or `cloudscraper`). [vlymar1 `session.py`, Pawikoski `vinted.py`]

### 6.4 Region pinning

- Region is pinned by **domain** — `www.vinted.co.uk` serves the UK marketplace (GBP, `anonymous-iso-locale=en-GB`). Sellers are not strictly UK-only: cross-currency sellers appear with a `conversion` object (§5.1). [Live]
- An explicit `country_ids=GB` filter exists and is accepted; whether the default co.uk scope already excludes non-UK sellers was not fully verified (search payload carries no country field; item-details, which would confirm `localization`, was 403 anonymously). [Live, §8.3]

---

## 7. Lookup endpoints (autocomplete UI)

All **verified live** on co.uk with the session cookie:

### 7.1 Brand autocomplete

```
GET /api/v2/brands?search_text=<q>
```

```json
{ "brands": [
  { "id": 14, "title": "adidas", "slug": "adidas",
    "favourite_count": 6650537, "item_count": 38674977,
    "is_visible_in_listings": true, "requires_authenticity_check": true,
    "is_luxury": true, "is_hvf": true,
    "path": "/brand/14-adidas", "url": "https://www.vinted.co.uk/brand/14-adidas",
    "is_favourite": false }
], "code": 0 }
```

### 7.2 Filter-embedded search (brand inside the filter panel)

```
GET /api/v2/catalog/filters/search?filter_search_code=brand&filter_search_text=nik
```

```json
{ "options": [
  {"id": 53, "title": "Nike", "type": "default"},
  {"id": 2703, "title": "Jordan", "type": "default"},
  {"id": 5977, "title": "Nike Air", "type": "default"}
], "is_selection_highlighted": false, "selected_filters": [] }
```

### 7.3 Facet options (size, colour, condition, …)

```
GET /api/v2/catalog/filters/facets?filter_code=<size|color|status|material>&catalog_ids=<cat>&search_text=<q>
```

- `filter_code=size` → grouped options (Men's/Women's/Kids/Baby + footwear numerics) with `type:"group"` nesting and `items_count`.
- `filter_code=color` → options with `prefix: {hex, code, type:"color"}` and `items_count`.
- `filter_code=status` → the 5 condition options (§3.3).
- The web app calls these facets lazily (filters render with `is_lazy: true`, `options: []` and fetch options on open). [Live, JS]

### 7.4 Category tree / faceted categories

```
GET /api/v2/catalog/faceted_categories?catalog_ids=<id>&search_text=<q>
```

Returns `categories: [{id, title, url, photo, item_count}]`; call with a `catalog_ids` value to drill into its children (Women `1904` → Clothing `4`, Shoes `16`, Accessories `1187`, Bags `19`, Beauty `146`). [Live]

### 7.5 Search-suggestion autocomplete

```
GET /api/v2/search_suggestions?query=<text>
```

```json
{ "search_suggestions": [
  { "title": "nike kids", "total_score": 518, "origin_id": -9126055927491251485,
    "params": [{ "title": ["nike kids"], "search_signals": ["prefix_exact"],
                 "source": ["taxonomy_data"], "entity_combination": ["brand+parent_cat"],
                 "total_score": ["518"] }],
    "suggestion_id": null, "suggestion_type": null }
], "code": 0 }
```

### 7.6 Filters metadata

```
GET /api/v2/catalog/filters?search_text=<q>&catalog_ids=<id>
```

Returns `filters` (id/code/title/display_type/selection_type/position/translations) and `selected_filters`. Options come back empty (`is_lazy: true`) — this endpoint is metadata only. [Live]

---

## 8. Known quirks, dead routes, error codes

### 8.1 Quirks (all verified live)

1. **Result cap of 960.** Any query returns at most 960 `total_entries`. (Monitor implication: page depth is bounded.)
2. **`per_page` clamps to 96.**
3. **No `created_at` / no colour / no condition-id in search items** — timestamps come from `photos[0].high_resolution.timestamp`; colour and condition id need the details endpoint.
4. **`time` cursor** — echo `pagination.time` on subsequent pages for stable pagination.
5. **Comma-joined vs bracket serialisation both accepted**; the nested `attribute_ids[...]` form is *not* accepted on `/api/v2/catalog/items`.
6. **Condition ids differ from older blog/gist values** — resolve from `/api/v2/catalog/filters/facets?filter_code=status` at runtime.
7. Prices were returned as `{amount, currency_code}` objects on 2026-08-02 **even without** `X-Money-Object: true`; older notes say the header toggles string→object. Keep the header; don't rely on either shape blindly.
8. No JSONP, no API key, no `Authorization` header — cookie auth only.
9. 401 (expired/invalid token) and 429 (rate limit) are real: clients retry-by-refreshing-cookies on 401/403 and raise on 429. [Pawikoski `response_codes.py`, vlymar1 `session.py`]

### 8.2 Feature-flagged "OpenAPI gateway" routes (in the JS bundle, NOT exercised live)

New client `ApiGatewayClient` targets `/svc-catalogue/items`, `/svc-filters/filters`, `/svc-filters/filters/facets`, `/svc-filters/faceted_categories`, `/svc-filters/filters/search`, activated by feature flags (`catalogue_items_open_api_gateway`, `faceted_categories_on_openapi_gateway`, `filter_on_openapi_gateway`, `web_catalog_dynamic_blocks`). These use the `catalog`/`attribute_ids[code]=...` param style. Base URL of this gateway is unconfirmed. Treat as in-flux; build against `/api/v2`. [JS bundle]

### 8.3 Dead / gated routes observed live

| Route | Result on live co.uk |
|---|---|
| `GET /api/v2/catalog/initializers` | **404 HTML** (dead; old category-tree endpoint, still referenced by older clients/gists) |
| `GET /api/v2/catalog/categories` | 404 HTML |
| `GET /api/v2/sizes` | 404 HTML |
| `GET /api/v2/catalog/blocks` | 404 HTML (flag-gated) |
| `GET /api/v2/items/{id}` | 404 HTML (old item endpoint is deprecated → `/details`) |
| `GET /api/v2/items/{id}/details` | **403** (HTML bot-block) for an anonymous curl; open-source clients use it with the same cookie session and it works for them |
| `GET /api/v2/search-bar/v2/suggestions` and `/web/api/search-bar/v2/suggestions` | 404 (search-bar gateway base URL is elsewhere) |

### 8.4 Versioning

- All routes under `/api/v2/`. No `v1` equivalents documented by the reverse-engineering community; the only official API (Vinted Pro seller integrations, `pro-docs.svc.vinted.com`) is a separate product (OpenAPI + HMAC webhooks), unrelated to the catalog search endpoint. [pro-docs]

### 8.5 Error codes

`code` field / HTTP mapping (Pawikoski `response_codes.py`, secondary source unless noted): `0` OK (live), `100` invalid token (live, HTTP 401), `21` login required, `102` IP blocked, `104` not found, `105` generic error, `106` access denied, `-99` session-from-token error. Rate limiting surfaces as HTTP **429**. [Pawikoski `response_codes.py`, vlymar1 `session.py`]

---

## 9. Recommended client strategy (summary of implications)

1. Session bootstrap: `GET https://www.vinted.co.uk/` → keep `access_token_web` (+ `refresh_token_web`, `anon_id`, `anonymous-iso-locale`); refresh on 401/403/expired JWT.
2. Send a browser `User-Agent`; prefer a TLS-impersonating HTTP client (curl_cffi/cloudscraper) to pass Cloudflare.
3. Search: `GET /api/v2/catalog/items` with flat params; echo `pagination.time` for paging; `per_page=96`; treat `total_entries` as capped at 960.
4. For a "new listings" monitor: `order=newest_first`, page 1, dedupe by item `id`, timestamp via `photos[0].high_resolution.timestamp`.
5. Autocomplete/filter UI: `/api/v2/brands`, `/api/v2/catalog/filters/facets` (size/color/status), `/api/v2/catalog/faceted_categories`, `/api/v2/search_suggestions`.
6. Item colour/condition-id and any sold-state detail require `/api/v2/items/{id}/details` (may be more strictly gated).

---

## Sources

### Primary — live API behaviour (verified via curl, 2026-08-02, vinted.co.uk unless noted)

- `GET https://www.vinted.co.uk/api/v2/catalog/items` — response envelope, item shape, all filter params, pagination cap/`time` cursor, `per_page` clamp, 401 anonymous error (this document §2–§6, §8).
- `GET https://www.vinted.co.uk/api/v2/brands?search_text=nik` — brand autocomplete shape (adidas=14, Nike=53).
- `GET https://www.vinted.co.uk/api/v2/catalog/filters/facets?filter_code={size|color|status}` — facet option shapes; status ids 6/1/2/3/4; colour hex table.
- `GET https://www.vinted.co.uk/api/v2/catalog/filters/search?filter_search_code=brand&filter_search_text=nik` — filter-embedded brand search.
- `GET https://www.vinted.co.uk/api/v2/catalog/faceted_categories?catalog_ids=1904` — category drill-down (1904→4/16/1187/19/146).
- `GET https://www.vinted.co.uk/api/v2/catalog/filters?search_text=nike` — filter metadata (lazy options).
- `GET https://www.vinted.co.uk/api/v2/search_suggestions?query=nike` — suggestion autocomplete shape.
- `GET https://www.vinted.co.uk/` — homepage cookie bootstrap (`access_token_web` JWT, `refresh_token_web`, `anonymous-iso-locale=en-GB`, `anon_id`, `__cf_bm`).
- Dead/gated routes: `/api/v2/catalog/initializers`, `/api/v2/catalog/categories`, `/api/v2/sizes`, `/api/v2/catalog/blocks`, `/api/v2/items/{id}`, `/api/v2/items/{id}/details` (403).
- `GET https://www.vinted.fr/api/v2/catalog/filters/facets?filter_code=status` and `...=color` — cross-locale id parity (status and colour ids identical to co.uk).

### Primary — Vinted web bundle (Next.js, `marketplace-web-assets.vinted.com`, fetched 2026-08-02)

- Chunk `0mydq.q6aeo-k.js` — `API_BASE_PATH="/api/v2"`, `CATALOG_PER_PAGE=96`, `SortByOption` enum, `/catalog/items`, `/catalog/filters`, `/catalog/filters/facets`, `/catalog/filters/search`, `/catalog/faceted_categories`, `/catalog/blocks`; request builders `ae`/`at` (param flattening, comma-join, `{code}_ids`), `buildCatalogPaginationParams` (time cursor), new `/svc-catalogue/items` + `/svc-filters/*` gateway under feature flags.
- Other chunks from the same page — `CURRENT_USER_API_ENDPOINT=/users/current`, `/search-bar/v2/suggestions`, `/search-bar/v2/users/{id}/previous_searches`, `/images/public/api/v2/images` (search-bar/image endpoints).

### Primary — official (unrelated, but authoritative re: what is official)

- Vinted Pro Integrations API docs (the only official Vinted API; separate product, OpenAPI + HMAC webhooks) — https://pro-docs.svc.vinted.com/

### Secondary — open-source clients (reverse-engineered contracts)

- Pawikoski/vinted-api-wrapper — https://github.com/Pawikoski/vinted-api-wrapper — `vinted/vinted.py` (search/catalog_filters/catalogs_list params, headers, cookie bootstrap, `time.time()` cursor), `vinted/endpoints.py` (`/catalog/items`, `/catalog/filters`, `/catalog/initializers`, `/items/{id}/details`, `/search_suggestions`), `vinted/models/{search,items,filters,money,photos}.py` (response DTOs incl. `DetailedItem`), `vinted/models/other.py` (`SortOption`, domains), `vinted/response_codes.py` (error-code table).
- vlymar1/vinted-api-kit — https://github.com/vlymar1/vinted-api-kit — `vinted/api/catalog.py` (URL→param mapping, `catalog/<id>-slug` extraction), `vinted/session.py` (headers, cookie refresh, 401/429 handling), `vinted/auth.py` (JWT `access_token_web` expiry), `vinted/constants.py` (locales, `SortOrder`, `DEFAULT_HEADERS`), `vinted/models/item.py` (`CatalogItem`, timestamp-from-photo logic). Also https://pypi.org/project/vinted-api-kit/
- tayeb4534, "Vinted.fr API Reference — Reverse Engineering" gist, 2026-06-10 — https://gist.github.com/tayeb4534/f4fbf2898944abce53a5a85f16cde2bb — corroborates `/api/v2/catalog/items`, headers, `/oauth/token` anonymous client_credentials, and the (now-stale) status-id table; **noted in this document where it disagrees with live data**.
- teloryfrozy/Vinted-API-RevEng — https://github.com/teloryfrozy/Vinted-API-RevEng — confirms the SSR/anti-automation picture and points at the official Pro docs.
- GitHub topic `vinted-api` (survey of clients: vinted-discord-bot, vinted-rs, vinted-api-kit, vinted-api-wrapper, vinted-mcp-server, etc.) — https://github.com/topics/vinted-api
- Pawikoski wrapper on PyPI — https://pypi.org/project/vinted-api-wrapper/

### Could not verify / notable gaps

- **`/api/v2/items/{id}/details`** (item colour, condition id, sold/reserved flags, `localization`) returned HTTP 403 for an anonymous curl; its field list is from the Pawikoski `DetailedItem` DTO (secondary source), not a live capture. It works for the open-source clients but may be more strictly gated.
- **New `/svc-catalogue/items` + `/svc-filters/*` gateway** — seen in the bundle, base URL not confirmed, never exercised live.
- **Country default scope** — whether plain `vinted.co.uk` search already excludes non-UK sellers (vs. needing `country_ids=GB`) was not conclusively verified; cross-currency (`conversion`) sellers do appear in results.
- **Colour id table** — only ids 1/3/4/11/12/20/21/22 captured; full table lives behind `/api/v2/catalog/filters/facets?filter_code=color`.
- **`material_ids`, `patterns_ids`, `video_game_platform_ids`** — accepted per client code, not individually exercised live.
- **Cloudflare/rate-limit thresholds** — 429 observed in clients' code, not reproduced here (a handful of requests only).
- All live requests were made from one residential IP on 2026-08-02; behaviour (especially caps, ids, and gating) may change over time.
