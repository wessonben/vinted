# 03 - Vinted rate-limit and anti-bot posture

Type: research
Status:

Blocked by: 01

## Question

What does Vinted's unofficial API tolerate in terms of sustained polling? This determines the safe polling strategy for up to 3 searches at a configurable rate (default 60s).

Concretely:

- Does Vinted enforce rate limits or bot detection on the catalog search endpoint, and at what thresholds?
- What does an over-polling response look like (HTTP status, captcha, blocked)? How long do blocks last?
- What headers/cookies are required to look like a normal browser (User-Agent, accept headers, region cookies)?
- Recommended safe sustained cadence for a small number of searches, based on observed behaviour in open-source Vinted monitor projects.
- Backoff / jitter / retry patterns that work (exponential backoff, pause per search, staggering).
- Any per-search concurrency guidance (e.g. never parallelise polls on the same IP).

## Notes for the researcher

Use `/research` workflow. Primary sources: open-source Vinted notifier/monitor projects (their READMEs, issue threads, rate-limit discussions), and observed behaviour documented there. Blocked by 01 — the endpoint must be known before its tolerance can be assessed.
