# ADR-0005: Store local calendar dates, not timestamps, for day logic

- **Status:** Accepted (2026-10-27)
- **Date:** 2026-10-27
- **Relates to:** TAD §5; PRD BR-DAY-1…6, FR-ACC-05, FR-CHK-06, FR-ROL-01/02, NFR-SEC-04, NFR-QUAL-01

## Context

Every rule in the product is about a member's own calendar day: "today", "yesterday", the edit window, weekday schedules, the 18:00 roll-call cutoff, week boundaries. Members live in different timezones, travel, and change timezone. History must never be re-dated when that happens (BR-DAY-4), and a member's current date must never move backwards (BR-DAY-5). Device clocks and device-supplied dates cannot be trusted (BR-DAY-1, NFR-SEC-04).

## Decision

- Every day-bearing record stores a **`DATE`** fixed at write time: entries, rule effective dates, share intervals, inactive periods, feed cards, streak prefixes. Timestamps are used only for ordering and audit.
- In code the type is `LocalDate`, a branded `'YYYY-MM-DD'` string. Domain arithmetic uses UTC-only date math in `core/time`, so results never depend on the process timezone.
- The **server resolves the day**: it derives it from the member's stored IANA zone and an injected server clock. The client sends `day: 'today' | 'yesterday'` plus an `expectedDate`; a mismatch is rejected as `STALE_DAY`.
- BR-DAY-5 is implemented with one nullable `users.date_floor`, written only when the timezone changes. `effectiveToday = max(floor, calendarDate(zone, now))`.
- Roll call evaluates each member on their own effective today using one bounded query over the few dates that can be "today" for anyone in the group.
- Only the repository layer converts between Prisma's JS `Date` (for `DATE` columns) and the string type.

## Alternatives considered

| Option | Why not |
|---|---|
| Store UTC instants, derive the local day at read time with the current zone | A timezone change would silently re-date history, violating BR-DAY-4. |
| Store an instant plus a zone/offset snapshot | Possible, but heavier, and daylight-saving ambiguity returns; the date is the fact we actually need. |
| Let the client send the date | Forgeable and subject to clock skew (NFR-SEC-04). |
| Use the server's timezone | Contradicts the brief ("never server time"). |

## Consequences

- **+** History is immutable under timezone changes; range queries and joins are plain date comparisons.
- **+** The hardest logic is pure and testable under many `TZ` values and a fake clock (UTC−12 to UTC+14, daylight saving, a skipped day, mid-day changes).
- **−** "Now" must be recomputed per member (roll call), and the BR-DAY-5 floor adds one column and one rule.
- **−** Prisma returns JS `Date` for `DATE` columns, which is a classic off-by-one source; confined to one layer and covered by tests.
- **−** `date-fns` functions that depend on the process timezone are banned from domain code (lint rule and test matrix).
