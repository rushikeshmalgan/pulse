# ADR-0004: Polling through one consolidated endpoint instead of realtime

- **Status:** Accepted (2026-10-27)
- **Date:** 2026-10-27
- **Relates to:** TAD §7.2, §7.3, §12.3, §12.4; PRD NFR-POLL-01…04, NFR-FRESH-01, FR-ROL-03, FR-FED-05

## Context

The brief says polling (about 20 seconds) with TanStack Query and no realtime service in v1. The PRD (rev 2) turns that into a budget: at most 180 requests per active user per hour, polling that stops after 3 minutes without interaction, nothing from hidden tabs, immediate resume on focus or interaction, and one request per refresh cycle.

Free-tier facts checked on 2026-10-27:

- Vercel Hobby: 1,000,000 function invocations, 4 CPU-hours of active CPU, 360 GB-hours of provisioned memory, 100 GB fast data transfer, 1 hour of runtime logs.
- Neon Free: 100 CU-hours, scale-to-zero after 5 idle minutes that cannot be disabled, 5 GB public network transfer.

## Decision

- **One endpoint** per open group view returns the roll call, each member's streak and their last two days, and page 1 of the feed with reactions (and in v1.1 the unread count). The client merges it into its cache.
- **Two states** driven by the time since the last interaction: polling every 20 s while the user interacted in the last 3 minutes, then stopped. A hidden tab never polls, and polling resumes immediately on focus or interaction. The hook mounts only on the group view.
- The endpoint reads a **bounded window** (members, shared structure, entries for at most four dates, cached streak prefixes, one feed page), with a budget of 20 KB read and 40 ms of CPU per poll.
- The server returns `pollHintMs` and `idleStopMin`, controlled by `PULSE_POLL_MODE`, so the cadence and the idle stop can be degraded (levels L0, L1, L2) without shipping a new client.

## Alternatives considered

| Option | Why not |
|---|---|
| Server-Sent Events or WebSockets from Vercel functions | Long-lived connections fit poorly with function duration limits and cost CPU and memory the whole time they are open. |
| A realtime service (Neon Realtime, Pusher, Ably, Supabase Realtime) | Excluded by the brief for v1 and adds a vendor and quotas. |
| Long polling | Holds a function open per viewer, with the same problems as streaming. |
| Refresh on focus and pull only | Cheapest, but misses the "friends can see if I showed up today" feel (goal G1). Kept as degradation level L2. |

## Consequences

- **+** Stateless and simple; fits Hobby comfortably on invocations, CPU and transfer (expected about 3% of invocations and about 9% of CPU).
- **−** Freshness is bounded by the cadence: 45 s p95 holds while polling is active; after the 3-minute stop the view refreshes on the next focus or interaction.
- **−** **Neon awake time does not depend on cadence.** Any query within five minutes keeps compute awake, so the number of distinct visits, not the cadence, drives usage: about 30–41% of the 100 CU-hours in the expected scenarios, rising with more visits per day (TAD §12.3). Shortening the idle stop (L1) and removing background polling (L2) are modest extra levers, both allowed by PRD rev 2, and the plan includes a weekly check of Neon usage in the first month.
- **−** The cap is per window: two visible windows for one user double their rate.
- **−** Client complexity: an idle-stop timer, merge by id, and mutation/poll races handled with query cancellation (TAD §7.4).
