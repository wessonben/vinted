# 05 - Poller architecture and session handling

Type: grilling
Status:

Blocked by: 03

## Question

What is the poller's internal architecture — session bootstrap, cookie storage, polling loop, and process lifecycle?

Concretely:

- **Session bootstrap:** the poller fetches `https://www.vinted.co.uk/` on start, harvests `access_token_web` / `refresh_token_web`, and refreshes on 401/403 or JWT expiry. When the user pastes a pinned cookie (manual paste, optional), how does it combine with the anonymous bootstrap?
- **Cookie storage:** where the session cookies live on disk (alongside SQLite?), whether the pinned cookie is stored separately from ephemeral anonymous cookies, and how a stale pinned cookie is surfaced to the UI.
- **Polling loop:** per-search scheduling (up to 3 searches), the refresh-rate override model, and how the rate-limit findings from ticket 03 (backoff, jitter, staggering) shape the loop.
- **Process lifecycle:** how poller and viewer start together locally (dev script, ports, whether they share one `node` process or two), and how the viewer reaches the poller (localhost port).
- **Dedup/state:** how items seen in earlier polls are reconciled against new results (keyed by Vinted item id), and where "new since last viewed" is computed.

## Notes

HITL ticket — resolve through grilling, one question at a time. Depends on the rate-limit findings (03). Feed the locked architecture into the spec and ADRs.
