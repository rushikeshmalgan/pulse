# Pulse — Test Plan

| | |
|---|---|
| **Status** | Approved rev 0 (merges applied) |
| **Date** | 2026-10-28 |
| **Inputs** | `docs/PRD.md` (rev 3), `docs/TAD.md` §5.8, §16, `docs/DB_SCHEMA.md`, `docs/API_SPEC.md`, `docs/TASKS.md` |
| **Companion** | `docs/TASKS.md` (owning tasks are T-numbers) |

**Scope.** How v1.0 is verified: the nine test-plan items of TAD §16, the Vitest unit and integration suites, the manual checklist, the gate each milestone must pass, and what is deliberately left untested.

---

## 1. Principles and tools

- **Test the logic where it is pure.** Day, schedule, share-interval, level, streak, roll-call and polling logic live in pure modules (`core/`, a few `features/*` modules) and are tested without a database, a browser or a framework.
- **Test the rules where they are enforced.** Authorization, transactions, constraints and feed-card bookkeeping are tested through the real Server Actions against a real PostgreSQL (§2).
- **Look at the rest by hand.** Screens, phones, time and installation are checked with the manual checklist (§6).
- **Tools:** Vitest 5.0.5 (the only test library approved; TAD §0.1). No Playwright, no Testing Library, no property-testing library: randomised tests use a hand-written seeded generator. Nothing in this plan needs a new library.
- **Time is always injected.** Every test sets the clock through `lib/clock`; none depends on the real date.

---

## 2. Environments, data and how tests run

### 2.1 Unit tests

`npm run test` runs the `unit` project: no database, no network, no environment variables. `npm run test:tz` runs the same project once per time zone (Pacific/Kiritimati, Etc/GMT+12, Asia/Kolkata, Australia/Lord_Howe, America/New_York, Pacific/Apia, Africa/Casablanca, UTC), by setting `TZ` in a child process, so no domain code can depend on the machine's time zone (TAD §5.8). Both run in CI on every push.

### 2.2 Integration tests (Server Actions against a real PostgreSQL)

- **Where.** The `pulse-dev` Neon project (Free), never production. The `integration` Vitest project creates a **separate Postgres schema per test file**, builds it from the migration SQL (including the hand-written constraints; `btree_gist` is created once at database level), runs the file, and drops the schema. The client uses the same `PrismaNeon` adapter as the app, with its `schema` option.
- **Why a separate Neon project.** Branches share their project's 100 compute hours, so integration runs on the production project would spend production hours. Ten runs a day of about five minutes, each followed by Neon's five-minute tail, keep the dev compute awake roughly 100 minutes a day: about 12 CU-hours a month at the assumed 0.25 CU minimum, inside the dev project's own allowance.
- **Alternative considered.** A local Postgres would remove the network, but the app's adapter is Neon's, so tests would need a second adapter (`@prisma/adapter-pg`): a library outside the approved list. We ask before adding it if Neon-based runs prove too slow.
- **Calling actions.** Tests import the exported action and call it with raw input, as Next would. `@clerk/nextjs/server` (`auth`, `clerkClient`), `@clerk/nextjs/webhooks` (`verifyWebhook`), `next/headers` (`cookies`), `next/navigation` and `server-only` are replaced in the setup file; a helper `asUser(userId)` sets the signed-in user.
- **Data.** Fixture builders (`makeUser`, `makeGroup`, `makeMembership`, `makeTarget`, `makeShare`, `makeEntry`) create exactly the rows a test needs; the large seeded group exists only for the budget and manual checks.
- **Concurrency.** Race tests start many calls with `Promise.all`, each on its own connection from the adapter pool, and are repeated ten times; a flaky concurrency test is a bug, never retried away.
- **Run.** `npm run test:db` (all) or `npm run test:db -- <file>`; needs `.env.test` with the dev database URLs. Run before merging any task that touches the database. Integration tests are not part of CI in v1.0.

### 2.3 Manual QA

Real devices and real accounts: a phone (Android Chrome and iPhone Safari), a desktop browser, and at least two Clerk accounts in different time zones (§6). Manual checks are run at the milestone named in each row, and the whole list runs again in the launch rehearsal.

---

## 3. Test-plan items TP-01 to TP-09

Consolidated from TAD §16. Each has a method, data, a pass criterion and an owning task.

### TP-01 — Cached streaks equal a full recomputation

- **Source:** TAD §6.4, §16
- **Method:** Differential test. After every operation, compare `groupStreak` (with the closed-days prefix) with `fullGroupStreak` (no cache) for every member, and again after advancing the clock to the next local day. Randomised runs use an in-memory store behind the same interface; fixed regression cases and 50 random sequences also run on the real database.
- **Data:** A seeded generator (seeds 1 to 1,000), a fake clock, members in at least three zones including a hold. Operations: record progress (today or yesterday), share, unshare, archive, change a rule, advance the clock, change time zone, remove, rejoin, write an old entry directly with and without invalidation, bump `ALGO_VERSION`. One fixed case per scenario 1 to 7 of TAD §16.
- **Pass criterion:** Zero differences across at least 1,000 randomised sequences and every fixed case; the “write directly without invalidation” cases must disagree until invalidated and `verifyStreakCache` must report them.
- **Owning task(s):** T-050, T-051, T-052

### TP-02 — Day logic passes the time-zone matrix

- **Source:** TAD §5.8, NFR-QUAL-01
- **Method:** Unit tests with a fake clock, run by `npm run test:tz` under eight process time zones; plus an integration roll call across members in different zones.
- **Data:** Zones: Pacific/Kiritimati (+14), Etc/GMT+12, Asia/Kolkata, Australia/Lord_Howe, America/New_York (both DST days), Pacific/Apia (the skipped day of 2011-12-30), Africa/Casablanca, UTC. Cases: zone change forward and backward, the TAD §5.4 hold table, week boundaries, the edit window at local midnight, 17:59 and 18:00 cutoffs, the PRD streak and level examples.
- **Pass criterion:** All unit tests green in all eight runs; the integration roll call returns the right state for every member at one instant.
- **Owning task(s):** T-005, T-006, T-054

### TP-03 — A deleted member and a deleted owner never break group pages

- **Source:** TAD §4.5, NFR-REL-02
- **Method:** Integration suite over every group surface, plus manual check M-11 against a real Clerk deletion.
- **Data:** Fixtures: a group with a deleted member (with progress, a card and a reaction) and a deleted owner; the tombstone transaction applied; a second group untouched.
- **Pass criterion:** Every group surface (overview, members, pulse, feed, reactions) responds without error, never shows the deleted users, their cards and reactions are hidden, joining works, `ownerless` is true; M-11 shows the same in the browser.
- **Owning task(s):** T-019, T-066

### TP-04 — A private target never influences any group view

- **Source:** TAD §10, NFR-SEC-02
- **Method:** Integration suite: build the same group twice, with and without a member's private and other-group data, and assert every group-visible value is identical.
- **Data:** A member with a private target, a target shared only with group A, and one shared with group B; progress on all three; the viewer is in group B.
- **Pass criterion:** Overview cells, roll-call state and counts, group streak, pulse fields, feed cards and task lists, totals and ordering are identical in both builds.
- **Owning task(s):** T-065

### TP-05 — The 15-member cap holds under concurrent joins

- **Source:** TAD §4.2, FR-GRP-03, NFR-REL-01
- **Method:** Integration test: fire many `joinGroup` calls at once with `Promise.all` against a group that has room for exactly one more.
- **Data:** A group of 14 active members; 20 distinct users joining at once; repeated 10 times.
- **Pass criterion:** Exactly one join succeeds each time; the other 19 get `GROUP_FULL`; the group never exceeds 15 active members.
- **Owning task(s):** T-022

### TP-06 — The poll handler stays within its budget

- **Source:** TAD §7.2, §12.3
- **Method:** Integration measurement on the seeded group (spike S3), kept as a test.
- **Data:** 15 members, 150 targets, 90 days of entries, feed page 1 populated.
- **Pass criterion:** At most 40 ms of CPU per poll (95th percentile) and 20 KB read from the database; the measured values replace the assumptions in TAD §12.3.
- **Owning task(s):** T-070

### TP-07 — Rapid taps are ordered and applied once; failures roll back; a stale day is detected

- **Source:** TAD §7.4, §5.5, FR-CHK-06, FR-CHK-08
- **Method:** Unit tests with a real `QueryClient` and a fake action, plus manual check M-06.
- **Data:** Ten taps in a row on one entry; an action that fails once; an action that returns `STALE_DAY`.
- **Pass criterion:** Calls arrive in tap order; a retried tap is applied once; the previous value is restored on failure with a visible message and retry; `STALE_DAY` refetches Today and shows a notice, with no rollback toast.
- **Owning task(s):** T-039, T-040

### TP-08 — The whole v1.0 flow works in an ordinary browser tab

- **Source:** TAD §1.4, API_SPEC §7.1, FR-PWA-05
- **Method:** Manual check M-13, run in a normal desktop tab and a normal phone browser tab, never installed.
- **Data:** Two accounts on production (or the preview).
- **Pass criterion:** Sign-up, onboarding, group, join, targets, check-off, heatmap, roll call, feed and reactions all work with the app not installed and no service worker registered (DevTools, Application, Service workers is empty).
- **Owning task(s):** T-067, T-072

### TP-09 — Concurrent overlapping writes are serialised (only if the btree_gist fallback is adopted)

- **Source:** DB_SCHEMA §3.1
- **Method:** Integration test with two real connections: two transactions write overlapping share intervals (or inactive periods) for the same key at once.
- **Data:** One target and group; two transactions started together, each inserting an overlapping interval.
- **Pass criterion:** One commits, the other fails with SQLSTATE `23P01`. **Created only if spike S1 adopts the fallback**; otherwise recorded as not needed.
- **Owning task(s):** T-008

---

## 4. Unit tests (Vitest, `unit` project)

| ID | Suite | Files | Key cases | Serves | Owner |
|---|---|---|---|---|---|
| U-01 | Date arithmetic | `tests/core/time/local-date.test.ts` | month and year boundaries, leap day, ISO weekday and Monday-start week, DST day, same results under all eight `TZ` | FR-CHK-06, FR-HM-01 | T-005 |
| U-02 | Zones and effective today | `tests/core/time/zone.test.ts, effective-today.test.ts` | BR-DAY-5 hold table, forward and backward changes, DST days, skipped day, floor update, `resolveDay`, `afterCutoff` 17:59 and 18:00, offset-zone rejection | FR-ACC-05, FR-CHK-06, FR-ROL-02 | T-006 |
| U-03 | Invite codes | `tests/core/invite-code.test.ts` | alphabet, 10,000 generated codes, normalisation of `abcd-2345` | FR-GRP-02, FR-GRP-03 | T-010 |
| U-04 | Zod primitives | `tests/lib/primitives.test.ts` | the primitive assertions of API_SPEC §10: impossible dates, offset and legacy zones, handle normalisation, `nextPath` open-redirect cases | FR-ACC-03, FR-ACC-04, FR-GRP-01 | T-010 |
| U-21 | Target and action input schemas | `tests/features/targets/schema.test.ts` | the input assertions of API_SPEC §10: COUNT 1 and 50, DURATION 5 to 480 in steps of 5, CHECKBOX with and without a goal, duplicate weekdays, seven weekdays become DAILY, progress value 0, 1440 and 1441 | FR-TGT-01, FR-TGT-02, FR-TGT-03, FR-VIS-01, FR-CHK-05 | T-028 |
| U-05 | Scheduling and completion | `tests/core/schedule/is-scheduled.test.ts` | revisions, DAILY and SPECIFIC_DAYS, archive from tomorrow, ARCHIVED over PAUSED, goal and completion, history predicate | FR-TGT-03, FR-TGT-07, FR-CHK-05, FR-CHK-07 | T-027 |
| U-06 | Share-interval decisions | `tests/core/schedule/share-interval.test.ts` | share counts from today, reopen when closed today, unshare through today, close-out at yesterday or delete when it began today | FR-VIS-02, FR-VIS-04, FR-VIS-05, FR-VIS-06, FR-GRP-07 | T-018 |
| U-07 | Heatmap level and range | `tests/core/level.test.ts, heatmap-range.test.ts` | 1/4, 1/3, 3/4, 4/5, 0 and 1/1; Monday-start columns for 52, 13 and 8 weeks | FR-HM-01, FR-HM-02 | T-041 |
| U-08 | Day state | `tests/core/schedule/day-state.test.ts` | personal versus group scope, sharing not retroactive, unshare keeps the past, REST versus UNTRACKED, today pending | FR-HM-02, FR-VIS-03, FR-VIS-07 | T-042 |
| U-09 | Streak fold | `tests/core/streak.test.ts` | PRD worked example, bridging, today pending, restart after a miss, start only on an active day, yesterday restored | FR-STK-01, FR-STK-02, FR-STK-03, FR-VIS-07 | T-048 |
| U-10 | Roll-call classification | `tests/core/rollcall.test.ts` | five states, 18:00 cutoff in the member's zone, hold never SLACKING, partial progress counts as active | FR-ROL-01, FR-ROL-02 | T-049 |
| U-11 | Default order | `tests/features/heatmap/order.test.ts` | streak, then 7-day percentage, then name; members with nothing scheduled last | FR-HM-06 | T-055 |
| U-12 | Heatmap summary and day list | `tests/features/heatmap/summary.test.ts` | generated sentence, month grouping, state labels | FR-HM-03, FR-HM-04 | T-046 |
| U-13 | Poll state machine | `tests/features/rollcall/poll-machine.test.ts` | fake timers: 3-minute and 1-minute stops, interaction resets, hidden tab, resume, null hint, at most 180 requests in an hour | FR-ROL-03, FR-FED-05 | T-058 |
| U-14 | Progress mutations | `tests/features/checklist/mutations.test.ts` | serial per entry, optimistic patch, rollback, retry once, STALE_DAY refetch (TP-07) | FR-CHK-02, FR-CHK-03, FR-CHK-04, FR-CHK-08, FR-CHK-09 | T-039 |
| U-15 | Pulse merge | `tests/features/rollcall/apply-pulse.test.ts` | patches members and last two cells, never reorders, member-set change asks for a refresh, feed page 1 merged by id | FR-ROL-03, FR-FED-05, FR-HM-05 | T-059 |
| U-16 | Result, errors, limits, templates | `tests/lib/*.test.ts, tests/core/templates.test.ts` | Zod-to-error mapping, templates equal the PRD table, limits equal the PRD numbers | FR-TGT-01, FR-TGT-02 | T-009 |
| U-17 | Manifest | `tests/app/manifest.test.ts` | name, start URL, standalone display, 192, 512 and maskable icons | FR-PWA-01, FR-PWA-03 | T-067 |
| U-18 | Import boundaries | `tests/lint/boundaries.test.ts` | fixture files that break each rule of TAD §3 must fail ESLint | FR-GRP-11 | T-004 |
| U-19 | Handle suggestion | `tests/features/accounts/handle.test.ts` | valid format, collisions, unusual names | FR-ACC-03 | T-013 |
| U-20 | Feed cursor | `tests/features/feed/cursor.test.ts` | encode and decode, malformed cursors rejected | FR-FED-03 | T-061 |

---

## 5. Integration tests (Vitest, `integration` project, real PostgreSQL)

| ID | Suite | Files | Key scenarios | Serves | Owner |
|---|---|---|---|---|---|
| I-01 | Constraints and indexes | `tests/integration/constraints.test.ts` | the 68 checks of DB_SCHEMA §10 on the real database: formats, one active owner, no overlap, restrict and cascade, one card per member-day, one reaction per emoji | FR-VIS-04, FR-VIS-05, FR-VIS-06, FR-FED-01, FR-REA-01, FR-TGT-08 | T-008 |
| I-02 | ensureUser and requireUser | `tests/integration/ensure-user.test.ts` | first request creates the row, concurrent first requests create one, tombstoned user is signed out, handle collision | FR-ACC-01, FR-ACC-02 | T-013 |
| I-03 | Webhook and tombstone | `tests/integration/webhook.test.ts, tombstone.test.ts` | upsert never overwrites the handle, ordering, duplicates, 400 and 500, member and owner tombstone | FR-ACC-02, FR-ACC-04 | T-019 |
| I-04 | Access helpers | `tests/integration/access.test.ts` | member, non-member, removed, deleted group give the same NOT_FOUND; owner-only gives FORBIDDEN | FR-GRP-11 | T-020 |
| I-05 | Groups | `tests/integration/groups-*.test.ts` | create, join (including the cap race, TP-05), already member, rejoin, remove, select, ownerless | FR-GRP-01, FR-GRP-02, FR-GRP-03, FR-GRP-05, FR-GRP-07 | T-022 |
| I-06 | Onboarding and profile | `tests/integration/onboarding.test.ts, profile.test.ts` | handle conflict, idempotence, `next` sanitising, zone change sets the floor and clears prefixes | FR-ACC-03, FR-ACC-04, FR-ACC-05 | T-026 |
| I-07 | Targets | `tests/integration/targets-*.test.ts` | create (start date is today), edit (in place versus tomorrow, mode lock), archive, delete, history rules | FR-TGT-01, FR-TGT-02, FR-TGT-03, FR-TGT-04, FR-TGT-06, FR-TGT-07, FR-TGT-08, FR-VIS-01 | T-030 |
| I-08 | Sharing | `tests/integration/sharing-*.test.ts` | share, unshare, same-day reshare, later reshare, non-member group | FR-VIS-01, FR-VIS-02, FR-VIS-04, FR-VIS-05, FR-VIS-06 | T-032 |
| I-09 | recordProgress | `tests/integration/record-progress.test.ts` | STALE_DAY, NOT_SCHEDULED, caps, absolute and idempotent, zero deletes, edit window, groups returned | FR-CHK-02, FR-CHK-03, FR-CHK-04, FR-CHK-05, FR-CHK-06, FR-CHK-08, FR-CHK-09 | T-037 |
| I-10 | Today data | `tests/integration/today.test.ts` | scheduled tasks, creation order, yesterday section, empty states; the personal streak cases are added by the personal-streak task | FR-CHK-01, FR-CHK-07, FR-CHK-10, FR-CHK-11, FR-STK-03 | T-036 |
| I-11 | Streak cache | `tests/integration/streak-cache.test.ts, tests/integration/tp01-fixed.test.ts, tests/streak/*` | prefix lifecycle and the TP-01 suite | FR-STK-01, FR-STK-02, FR-STK-03 | T-052 |
| I-12 | Roll call across time zones | `tests/integration/rollcall.test.ts` | members in four zones at one instant, hold, removed and tombstoned excluded | FR-ROL-01, FR-ROL-02, FR-ROL-03, FR-VIS-03 | T-054 |
| I-13 | Overview and pulse | `tests/integration/overview.test.ts, pulse.test.ts` | shape, group-scope cells, hint modes L0 to L2, uniform 404, default order | FR-HM-05, FR-HM-06, FR-ROL-03, FR-STK-01, FR-VIS-07, FR-GRP-11 | T-057 |
| I-14 | Feed | `tests/integration/feed-sync.test.ts, feed-read.test.ts` | one card per member-day, updated in place, deleted at zero, newly shared group, paging without duplicates, removed members hidden | FR-FED-01, FR-FED-03, FR-FED-04, FR-FED-05, FR-VIS-05 | T-061 |
| I-15 | Reactions | `tests/integration/reactions.test.ts` | toggle, five emojis, own card, removed reactor hidden, cascade | FR-REA-01, FR-REA-03 | T-062 |
| I-16 | Leak suite (TP-04) | `tests/integration/leak.test.ts` | see TP-04 | FR-VIS-02, FR-VIS-03 | T-065 |
| I-17 | Deleted-user suite (TP-03) | `tests/integration/deleted-user.test.ts` | see TP-03 | FR-ACC-02, FR-GRP-11 | T-066 |
| I-18 | Poll budget (TP-06) | `tests/integration/pulse-budget.test.ts` | see TP-06 | FR-ROL-03 | T-070 |

Every Server Action (A-01 to A-13) and route handler (R-01 to R-04) is exercised by at least one suite above; the page loaders are covered through the queries they call.

---

## 6. Manual QA checklist

Tick each item in the pull request that runs it. “When” says the first milestone at which it can run and when to repeat it.

### M-01 — First-run and invite flow on a phone

- **Owner / when:** T-023; after M1; again before launch
- **Serves:** FR-ACC-01, FR-ACC-02, FR-ACC-03, FR-GRP-03, FR-TGT-01
- **Steps:**
  1. On the owner's account create a group and copy the invite link.
  2. On a phone, in a private tab, open the link signed out.
  3. Sign up (once with Google, once with email), confirm the profile, return to the invite page.
  4. Tap **Join group**.
  5. Create a first target from a template with the group pre-ticked and tick it.
- **Expected:** Nothing joins until the tap; the group page shows the new member as Active; the whole path takes under about a minute after sign-up (goal G3).

### M-02 — Two accounts in different time zones: roll call

- **Owner / when:** T-059; after M4
- **Serves:** FR-ROL-01, FR-ROL-02, FR-ROL-03, FR-VIS-03
- **Steps:**
  1. Account A in Asia/Kolkata, account B in America/Los_Angeles, both in one group with a shared daily target.
  2. Set B's profile zone to one where the local time is 17:50, wait ten minutes, reload the group page as A.
  3. Tick B's target and watch A's page.
- **Expected:** B shows Not yet before 18:00 local and Slacking after; ticking moves B to Active within one poll; a member with no shared target shows No targets shared; one with nothing scheduled shows Rest day.

### M-03 — Midnight rollover

- **Owner / when:** T-040; after M2; again after M4
- **Serves:** FR-CHK-06, FR-ACC-05, FR-STK-03, FR-CHK-07
- **Steps:**
  1. Set the profile zone to one whose local time is about 23:55 (use the seeded zones or a nearby offset).
  2. Tick a task, leave Today open, wait past local midnight, tap another task.
  3. Reload.
- **Expected:** The stale screen gets a notice and refetches (STALE_DAY); the new day starts empty; yesterday's tasks are editable in the Yesterday section; a task not completed yesterday shows as a miss unless completed in time; streak does not break while today is pending.

### M-04 — Time-zone change and the date hold

- **Owner / when:** T-026; after M1
- **Serves:** FR-ACC-05
- **Steps:**
  1. Note today's date on Today.
  2. Change the profile zone to one that is a calendar day behind.
  3. Reload Today, then again after the new zone reaches the held date.
- **Expected:** Today keeps its date (never goes backwards); after the new zone catches up the date advances normally; recorded progress keeps its original dates.

### M-05 — Unshare, remove and rejoin

- **Owner / when:** T-025; after M1; again after M5
- **Serves:** FR-VIS-04, FR-VIS-05, FR-VIS-06, FR-GRP-07, FR-GRP-03
- **Steps:**
  1. Owner shares a target with the group for three days of history, then unshares it.
  2. Check the group heatmap, roll call and feed as the other member.
  3. Owner removes the member; the member opens the group; the member rejoins with the link and re-shares.
- **Expected:** History up to the unshare day stays and later days stop counting from tomorrow; the removed member sees the uniform not-found page; after rejoin the earlier shared history reappears and the absence days are not tracked.

### M-06 — Failed save: offline

- **Owner / when:** T-040; after M2
- **Serves:** FR-CHK-08, FR-CHK-09
- **Steps:**
  1. On Today open DevTools, Network, Offline.
  2. Tap a checkbox and a count.
  3. Go back online and tap Retry.
  4. Double-tap a count quickly.
- **Expected:** The tap shows instantly, then reverts with a visible message and a Retry; nothing is silently lost; the retry applies once; rapid taps end at the right number.

### M-07 — Check-off on a phone

- **Owner / when:** T-040; after M2; again in the accessibility pass
- **Serves:** FR-CHK-02, FR-CHK-03, FR-CHK-04, FR-CHK-05, FR-CHK-06, FR-CHK-11
- **Steps:**
  1. From the home screen icon open Today.
  2. Tick a checkbox, step a count, add 30 minutes, open Yesterday and tick an item.
- **Expected:** A checkbox task is two taps from opening the app; every control responds instantly and is at least 44 px; the summary and personal streak update.

### M-08 — Heatmap: appearance and accessibility

- **Owner / when:** T-047; after M3; again in the accessibility pass
- **Serves:** FR-HM-01, FR-HM-02, FR-HM-03, FR-HM-04
- **Steps:**
  1. View Today with seeded data in light and dark and at 360 px and 1280 px.
  2. Press Tab through the page; open the day-by-day list with Enter.
  3. Tap a cell on a phone; hover on desktop.
  4. Run NVDA or VoiceOver over the heatmap.
- **Expected:** 13 weeks narrow and 52 wide; missed, nothing scheduled, not tracked and today look different without relying on colour; the heatmap is one Tab stop; the list gives date, X of Y, percentage and state; the tooltip shows the same facts.

### M-09 — Polling behaviour

- **Owner / when:** T-059; after M4
- **Serves:** FR-ROL-03, FR-FED-05
- **Steps:**
  1. Open a group page with DevTools, Network, filter `pulse`.
  2. Interact for a minute, then leave the tab visible and untouched for 4 minutes.
  3. Switch to another tab for a minute, switch back.
  4. Move the mouse.
- **Expected:** Requests every 20 s while interacting; none after 3 minutes without interaction; none while the tab is hidden; resumes immediately on return or movement.

### M-10 — Feed, reactions and the group page across two accounts

- **Owner / when:** T-064; after M5
- **Serves:** FR-FED-01, FR-FED-03, FR-FED-04, FR-FED-05, FR-REA-01, FR-REA-03, FR-HM-05, FR-HM-06, FR-STK-01, FR-STK-02
- **Steps:**
  1. Account A ticks a task; watch account B's group page.
  2. B reacts 🔥; A watches the count.
  3. A removes all progress for the day.
  4. Scroll the feed to older days.
- **Expected:** B sees A's card appear and update in place within one poll; reactions update; the card disappears when progress is removed; older pages load without repeats; rows are ordered by group streak, then 7-day percentage, then name.

### M-11 — Deleted account in Clerk

- **Owner / when:** T-066; after M5
- **Serves:** FR-GRP-11, FR-ACC-02
- **Steps:**
  1. In the Clerk dashboard delete a test user who is a member, then one who owns a group.
  2. As another member reload the group, members, feed and pulse.
- **Expected:** Every page keeps working; the deleted person is gone from every view; for the deleted owner the group has no owner controls but joining still works.

### M-12 — Install on Android and iOS

- **Owner / when:** T-067; spike S4; again before launch
- **Serves:** FR-PWA-01, FR-PWA-03
- **Steps:**
  1. Android Chrome: open the HTTPS URL, menu, Install app, launch from the icon.
  2. iPhone Safari: Share, Add to Home Screen, launch from the icon.
  3. Sign in, tick a task, close and reopen.
- **Expected:** Both install; they open full-screen on /today with the Pulse icon; the session persists.

### M-13 — Ordinary-tab run (FR-PWA-05, TP-08)

- **Owner / when:** T-067; spike S4; again at launch
- **Serves:** FR-PWA-05
- **Steps:**
  1. In a normal desktop tab and a normal phone browser tab (not installed) run M-01, M-07, M-09 and M-10 with two accounts.
  2. DevTools, Application, Service workers: confirm the list is empty.
- **Expected:** Everything works without installing; no service worker is registered.

### M-14 — Targets management

- **Owner / when:** T-035; after M2
- **Serves:** FR-TGT-01, FR-TGT-02, FR-TGT-03, FR-TGT-04, FR-TGT-06, FR-TGT-07, FR-TGT-08, FR-VIS-01, FR-CHK-01, FR-CHK-10
- **Steps:**
  1. Create one target from each of the six templates and one custom, ticking different groups.
  2. Edit a goal; check the “takes effect tomorrow” note and that today's goal is unchanged.
  3. Archive one; try to delete one with history (refused) and one without (allowed).
- **Expected:** Defaults match the PRD; the edit applies from tomorrow; archived targets keep their past in every view; delete is refused for a target with history.

### M-15 — Group management

- **Owner / when:** T-024; after M1
- **Serves:** FR-GRP-01, FR-GRP-02, FR-GRP-03, FR-GRP-05
- **Steps:**
  1. Create a group, copy the link, type its code by hand as `abcd-2345` style (lower case, hyphen) into Join with a code.
  2. Join with 15 accounts or seeded members, then try a 16th.
  3. Switch between two groups and reload.
- **Expected:** The hand-typed code works; the 16th is told the group is full; the last selected group is remembered.

### M-16 — Sign-out and profile

- **Owner / when:** T-026; after M1
- **Serves:** FR-ACC-04, FR-ACC-06
- **Steps:**
  1. Edit name and handle; change the image in the Clerk account settings and reopen Pulse.
  2. Sign out and press Back.
- **Expected:** Changes show everywhere; the new image appears; after sign-out no private data is on screen.

### M-17 — Production smoke test

- **Owner / when:** T-071; once after the production deploy
- **Serves:** FR-ACC-01, FR-ACC-02, FR-PWA-01
- **Steps:**
  1. Deploy to production; sign up a fresh account; complete M-01 quickly.
  2. Edit the name in Clerk and watch it sync; check Neon and Vercel usage pages.
- **Expected:** Migrations ran, the webhook reaches production, the region is right, install works, and usage is a few percent of the monthly limits.

---

## 7. Milestone gates

A milestone is finished when the suites listed for it are green and the manual items were run. Earlier suites stay green throughout.

| Milestone | Must be green | Also |
|---|---|---|
| M0 Spikes and setup | U-01, U-02, U-18, I-01 | S1 and S5 recorded; `npm run test`, `test:tz` and `test:db` green; the preview shows “DB ok”. |
| M1 Auth and groups | U-03, U-04, U-16, U-06, U-19, I-02, I-03, I-04, I-05, I-06, M-01, M-04, M-15, M-16 | TP-05 passes; TP-03 database leg (tombstone) passes. |
| M2 Targets and checklist | U-05, U-14, U-21, I-07, I-08, I-09, I-10, M-03, M-06, M-07, M-14 | TP-07 passes. |
| M3 Heatmap | U-07, U-08, U-12, M-08 | The seeded account renders correctly in both themes. |
| M4 Roll call and streaks | U-09, U-10, U-11, U-13, U-15, I-11, I-12, I-13, M-02, M-09 | TP-01 and TP-02 pass. |
| M5 Feed and reactions | U-20, I-14, I-15, I-16, I-17, M-05, M-10, M-11 | TP-03 and TP-04 pass. |
| M6 PWA and launch prep | U-17, I-18, M-12, M-13, M-17 | TP-06 and TP-08 pass; the whole of §6 passes on production (launch rehearsal). |

---

## 8. What we deliberately do not test in v1.0, and why

- **Automated end-to-end browser tests.** Playwright is not an approved library, there are about fifteen users, and the manual checklist covers the flows. Revisit after launch if regressions in screens start to cost time.
- **Component and DOM rendering tests.** Testing Library is not approved. Client logic (mutations, polling, merging, summaries) is extracted into pure modules and tested there; the rendering itself is checked by hand.
- **Visual regression and screenshots.** No tooling approved and the interface is small; the manual checks include light, dark and 360 px.
- **Automated accessibility scans.** No scanner is approved. The accessibility pass (M-07, M-08) uses the keyboard, devtools contrast checks and a screen reader.
- **Clerk's own flows.** Google OAuth, email verification and password reset belong to the provider. We test our boundary (mocked `auth`, mocked `verifyWebhook`) and run the real sign-up in M-01 and M-17.
- **Webhook retries and signature internals.** Clerk and Svix test them. We test that our handler is idempotent, order-tolerant and returns 400 or 500 correctly.
- **Load and stress beyond the poll measurement.** Groups hold at most 15 members; spike S3 measures one poll at that size. Re-measure after real use (spike S6), not before.
- **A full browser matrix.** We test the latest Chrome (desktop and Android) and Safari on iPhone. Firefox and Edge get a smoke run of M-01 and M-07 only; NFR-COMPAT-01 is met by relying on standard features.
- **Offline use and service workers.** v1.0 has none (FR-PWA-04 is v1.1). The only offline behaviour tested is the failed save in M-06 and the no-service-worker check in M-13.
- **Penetration testing.** The security controls (membership on every read, signature checks, no secrets in clients) have automated tests (I-04, I-16, U-18); `npm audit` is report-only. A professional test is out of proportion for a friends-only app.
- **Soak tests with months of data.** The read volume is bounded by design (52 weeks, 15 members); spike S3 uses 90 days. Re-measure after six months of real data.
- **Database failover and restore drills.** Free-tier restore is 6 hours; we take one snapshot before the first production migration and do not rehearse restores.
- **v1.1 features.** Only the shapes v1.0 already supports (for example the PAUSED rule in `isScheduled`) are tested now; the features arrive with their own tests.

---

## 9. Coverage check

Generated from the suite data and verified by a script against PRD rev 3.

- **v1.0 requirements:** 58; **with at least one test or manual check:** 58.
- **Suites:** 21 unit, 18 integration, 17 manual, 9 test-plan items; every suite has an owning task, every TP item has an owner, and every suite is in a milestone gate.
- **FR-PWA-05** (the one requirement with no dedicated surface) is verified by TP-08 and M-13.

| FR | Title | Tests |
|---|---|---|
| FR-ACC-01 | Sign up and sign in | I-02, M-01, M-17 |
| FR-ACC-02 | Immediately usable after sign-up | I-02, I-03, I-17, M-01, M-11, M-17 |
| FR-ACC-03 | First-run profile | U-04, U-19, I-06, M-01 |
| FR-ACC-04 | Edit profile | U-04, I-03, I-06, M-16 |
| FR-ACC-05 | Timezone change | U-02, I-06, M-03, M-04 |
| FR-ACC-06 | Sign out | M-16 |
| FR-GRP-01 | Create a group | U-04, I-05, M-15 |
| FR-GRP-02 | Invite code and link | U-03, I-05, M-15 |
| FR-GRP-03 | Join a group | U-03, I-05, M-01, M-05, M-15 |
| FR-GRP-05 | Group switcher | I-05, M-15 |
| FR-GRP-07 | Member list and removal | U-06, I-05, M-05 |
| FR-GRP-11 | Membership is verified on every action | U-18, I-04, I-13, I-17, M-11 |
| FR-TGT-01 | Create a target from a template or from scratch | U-21, U-16, I-07, M-01, M-14 |
| FR-TGT-02 | Tracking modes and goals | U-21, U-16, I-07, M-14 |
| FR-TGT-03 | Frequency: daily or specific days | U-21, U-05, I-07, M-14 |
| FR-TGT-04 | Start date | I-07, M-14 |
| FR-TGT-06 | Edit a target | I-07, M-14 |
| FR-TGT-07 | Archive a target | U-05, I-07, M-14 |
| FR-TGT-08 | Delete a target with no history | I-01, I-07, M-14 |
| FR-VIS-01 | Choose which groups can see a target | U-21, I-07, I-08, M-14 |
| FR-VIS-02 | Unshared targets are private | U-06, I-08, I-16 |
| FR-VIS-03 | Group views count only shared targets | U-08, I-12, I-16, M-02 |
| FR-VIS-04 | Sharing is not retroactive | U-06, I-01, I-08, M-05 |
| FR-VIS-05 | Unsharing keeps the past | U-06, I-01, I-08, I-14, M-05 |
| FR-VIS-06 | Re-sharing | U-06, I-01, I-08, M-05 |
| FR-VIS-07 | Two streaks, clearly labelled | U-08, U-09, I-13 |
| FR-CHK-01 | Today's checklist | I-10, M-14 |
| FR-CHK-02 | Check off a checkbox task | U-14, I-09, M-07 |
| FR-CHK-03 | Count tasks | U-14, I-09, M-07 |
| FR-CHK-04 | Duration tasks | U-14, I-09, M-07 |
| FR-CHK-05 | Completion and over-achievement | U-21, U-05, I-09, M-07 |
| FR-CHK-06 | Edit window: today and yesterday | U-01, U-02, I-09, M-03, M-07 |
| FR-CHK-07 | Missed days need no action | U-05, I-10, M-03 |
| FR-CHK-08 | Reliable saving | U-14, I-09, M-06 |
| FR-CHK-09 | Same data on every device | U-14, I-09, M-06 |
| FR-CHK-10 | Empty and first-run states | I-10, M-14 |
| FR-CHK-11 | Daily summary | I-10, M-07 |
| FR-HM-01 | Personal heatmap layout | U-01, U-07, M-08 |
| FR-HM-02 | Cell states and levels | U-07, U-08, M-08 |
| FR-HM-03 | Tooltip | U-12, M-08 |
| FR-HM-04 | Accessibility and themes | U-12, M-08 |
| FR-HM-05 | Group comparison view | U-15, I-13, M-10 |
| FR-HM-06 | Default order | U-11, I-13, M-10 |
| FR-STK-01 | Current streak | U-09, I-11, I-13, M-10 |
| FR-STK-02 | Retroactive edits recalculate | U-09, I-11, M-10 |
| FR-STK-03 | Today never breaks a streak | U-09, I-10, I-11, M-03 |
| FR-ROL-01 | Roll call per group | U-10, I-12, M-02 |
| FR-ROL-02 | Cutoff | U-02, U-10, I-12, M-02 |
| FR-ROL-03 | Status changes as they act | U-13, U-15, I-12, I-13, I-18, M-02, M-09 |
| FR-FED-01 | Daily progress card | I-01, I-14, M-10 |
| FR-FED-03 | Order and paging | U-20, I-14, M-10 |
| FR-FED-04 | Feed respects sharing and membership | I-14, M-10 |
| FR-FED-05 | New activity appears without reloading | U-13, U-15, I-14, M-09, M-10 |
| FR-REA-01 | React to a card | I-01, I-15, M-10 |
| FR-REA-03 | Reactions follow their card | I-15, M-10 |
| FR-PWA-01 | Installable | U-17, M-12, M-17 |
| FR-PWA-03 | Installed behaviour | U-17, M-12 |
| FR-PWA-05 | Works without installing | M-13 |

