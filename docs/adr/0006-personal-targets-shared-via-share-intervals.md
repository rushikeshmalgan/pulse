# ADR-0006: Personal targets shared to groups through share intervals

- **Status:** Accepted (2026-10-27)
- **Date:** 2026-10-27
- **Relates to:** TAD §6.2, §6.3, §10, §14; PRD D1, D8, FR-VIS-01…07, BR-SHARE-1…7, BR-VIS, NFR-SEC-02

## Context

The brief says a target belongs to a user within a group. The PRD changed this (D1): a target is owned by the member and shared to zero or more groups, and one check-off counts in every group it is shared to. Group views (heatmap, roll call, streaks, feed) count only targets shared with that group **on the relevant day**. Sharing is not retroactive, unsharing and archiving take effect the next local day, past history stays, removal is immediate, and a member who rejoins gets their earlier shared history back.

A simple boolean "is shared" flag cannot express "shared from Oct 1 to Oct 20, again from Nov 3".

## Decision

- A target has exactly one owner. Visibility is recorded in **`share_intervals`**: one row per `(target, group)` period with an inclusive `from_date` and a nullable `to_date` (null means still shared).
- **Share** inserts an interval from the owner's effective today (or reopens one closed today). **Unshare** sets `to_date` to today, so today still counts and tomorrow does not. **Removal** closes all of that member's open intervals at yesterday and sets the membership to `REMOVED`; **rejoin** does not reopen intervals.
- A group's view of a member on date D uses exactly the targets whose interval covers D. Personal and group views call the same day-state functions with a different "visible targets" set (TAD §6.3).
- Progress is stored once per `(target, local date)`, so one check-off feeds every group.
- SQL constraints (added by hand in the migration): at most one open interval per `(target, group)`, and no overlapping intervals.

## Alternatives considered

| Option | Why not |
|---|---|
| Target belongs to a group (the brief) | Duplicates habits across groups and needs separate check-offs. |
| A boolean shared flag per target and group | Loses history: unsharing would rewrite the past, contradicting FR-VIS-05. |
| Copy a target into each group on share | Duplicated entries that can drift apart. |
| A group "subscribes" to all of a member's targets | Contradicts the explicit-sharing principle and leaks private targets. |
| Daily snapshots of what was visible | Heavy, and needs a job to produce them. |

## Consequences

- **+** The history rules of the PRD hold by construction; no data is rewritten when sharing changes.
- **+** v1.1 pause, restore and future start dates need no new structure.
- **−** Interval maintenance logic and hand-written SQL constraints must be tested.
- **−** Every group query must start from intervals (indexed on group and dates). A query that skipped them would leak private targets, so the design adds three guards: the compile-time `GroupAccess` requirement, explicit DTO mapping, and mandatory leak tests with a private-target fixture.
- **−** Streak prefixes must be invalidated on rejoin, because rejoin brings earlier intervals back into scope.
