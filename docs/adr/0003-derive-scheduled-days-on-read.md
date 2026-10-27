# ADR-0003: Derive scheduled days on read instead of generating task rows

- **Status:** Accepted (2026-10-27)
- **Date:** 2026-10-27
- **Relates to:** TAD §6.1, §6.3, §6.4; PRD D2, FR-CHK-07, FR-TGT-03/06/07, BR-SCH

## Context

The brief says tasks are generated lazily when a user opens the app and missed days are backfilled as incomplete. The PRD (D2) found the flaw: a member who stops opening the app has no rows, which makes them invisible to the roll call, the group totals and the streaks, and they are exactly the people the group needs to see. The brief also rules out cron, and a daily job would have to run per member timezone.

Target rules also change over time (goal, weekdays; archive; later pause), and past days must keep the rules that were in force.

## Decision

Store only **durable facts**, and compute everything else on read with pure functions in `core/`:

- `targets` (with `start_date`), `target_revisions` (rules with an effective date), `target_inactive_periods` (archived now; paused later), `share_intervals`, and `entries` (one row per target and local date, only when progress is greater than zero).
- Scheduled, tracked, missed, rest, level and streak are all derived. A missing entry means incomplete. There are **no task rows, no backfill and no cron**.
- A rule change adds a revision effective from the next local day; history is judged on the revision in force that day.
- For polling, the immutable part of a group streak (days older than yesterday) is cached as a **derived prefix** that can be deleted and rebuilt at any time.

## Alternatives considered

| Option | Why not |
|---|---|
| Lazy generation on open plus backfill (the brief) | Wrong for members who do not open the app; every rule edit would need rewriting rows; write amplification. |
| A scheduled job that materialises tasks daily | Needs cron (not in the brief), per-member-timezone timing, and failure recovery. |
| SQL views using `generate_series` | Duplicates rule logic in SQL, which is harder to test across timezones and revision history. |
| Hybrid: materialise tasks only for the last two days | Two code paths for the same truth; the edit window already makes the derived path cheap. |

## Consequences

- **+** Correct for every member regardless of app usage; trivial writes; rule edits never rewrite history.
- **+** One set of pure functions backs every view, so the same member-day gives the same result everywhere (NFR-QUAL-02) and is unit-testable in many timezones (NFR-QUAL-01).
- **−** Reads cost CPU: about 3,640 schedule checks per member-year, about 55,000 for a 15-member heatmap, which is a few milliseconds. Polling avoids history scans through the prefix cache.
- **−** Every screen must use the shared functions; copying the logic would reintroduce drift. Enforced by folder rules (TAD §3).
- **−** Needs a revisions model, a "has history" test and prefix-cache invalidation rules (rejoin, timezone change, tombstone, algorithm version, repair scripts).
- **−** The cached streak must never disagree with a full recomputation. TAD §6.4 keeps a cache-free reference implementation as the oracle, and the test plan must enforce the equivalence (requirement TP-01, TAD §16).
