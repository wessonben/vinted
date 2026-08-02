# 04 - Data model

Type: grilling
Status:

Blocked by: 01

## Question

What is the SQLite schema for the feed-with-memory store, and the poller's read/write contract with the viewer?

Concretely:

- Tables for: searches (up to 3, each with filters, refresh rate, last-viewed timestamp), items (Vinted id, title, brand, category, size, colour, condition, price, photo URL, listing URL, available/sold flag, first-seen, last-seen), price history (7-day window, lightweight), and any poller state.
- How "new since last viewed" is computed (per-search last-viewed timestamp vs item first-seen).
- How price-drop detection works against the 7-day history.
- The API surface the poller exposes to the viewer (endpoints, payloads) for: search CRUD, current items per search, price-drop/new/sold flags, brand autocomplete, refresh-now, rate override.
- Which process owns writes (poller owns SQLite writes; does the viewer write anything, e.g. searches/last-viewed?).

## Notes

HITL ticket — resolve through grilling, one question at a time. Use `/domain-modeling` to pin terms (search, item, panel, new, drop, sold). Feed the locked schema into the spec and this repo's CONTEXT.md glossary.
