# Pulse — Tasks

| | |
|---|---|
| **Status** | Approved rev 0 (merges applied) |
| **Date** | 2026-10-28 |
| **Inputs** | `docs/PRD.md` (rev 3), `docs/TAD.md` (rev 2, §15 spikes, §16 test-plan items), `docs/adr/`, `docs/DB_SCHEMA.md`, `docs/API_SPEC.md` |
| **Companion** | `docs/TEST_PLAN.md` (suite IDs such as `U-05`, `I-09`, `M-03` refer to it) |
| **Size** | 72 tasks in 7 milestones, 22,450 estimated changed lines |

**How to read.** Each task fits one working session (about 400 hand-written changed lines or fewer, tests included). Tasks are ordered so that every dependency comes first, and pure logic with its tests comes before any screen that uses it. The coverage check at the end is generated from this data and verified by a script.

---

## 1. Conventions

**Task fields.** *Serves* lists the requirements (FR, BR, NFR) or spike (S1 to S6) a task delivers or evidences; an **enabler** task (tooling, foundations, schema) lists the first requirements that cannot be delivered without it. *Touches* names the API_SPEC surfaces and DB_SCHEMA models. *Size* is an estimate of **hand-written changed lines including tests**; generated files (the migration SQL, lockfiles, scaffold output) and files copied verbatim from an approved spec (`schema.prisma`, the constraints SQL, Zod blocks) are not counted. *Type* is `spike`, `infra`, `pure` (no I/O), `db` (touches the database) or `ui` (browser screens).

**Definition of done (every task).** The acceptance criteria hold; `npm run typecheck` and `npm run lint` pass; the tests named under Verification pass; the manual check, if any, was done in a browser; any document named in the task is updated in the same change; and the status below is set to `done`.

**Status values:** `todo`, `doing`, `blocked`, `done`. All tasks start as `todo`.

**Standard commands.** Defined by the tooling and schema tasks; later tasks use them by name.

| Command | What it does |
|---|---|
| `npm run typecheck` | `tsc --noEmit` |
| `npm run lint` | `eslint .` |
| `npm run test` | Vitest `unit` project (no database); append a path to filter, for example `npm run test -- core/streak` |
| `npm run test:tz` | the `unit` project once per time zone (eight runs) |
| `npm run test:db` | Vitest `integration` project against the `pulse-dev` Neon project (needs `.env.test`); append a path to filter |
| `npm run build` | `next build` |
| `npm run dev` | local server at http://localhost:3000 |
| `npm run db:dev` | `prisma migrate dev` (dev project only) |
| `npm run db:deploy` | `prisma migrate deploy` |
| `npm run db:seed` | development seed (needs `ALLOW_SEED=1`) |

**Setup checklist (you), before T-001.** Secrets never go into chat, commits or documents: values go only into the git-ignored files `.env.local` and `.env.test`. The environment check (T-004) reports the *names* of missing variables only, so you can tell me “T-004 says `DIRECT_URL` is missing”.

**Region pairing (decided).** Neon offers no Indian region. The nearest is **AWS Asia Pacific 1 (Singapore), `aws-ap-southeast-1`**. Production must use the same Neon region and the Vercel function region **`sin1` (Singapore)**; the walking-skeleton task (T-007) sets and measures it. Choose the Neon region when creating `pulse-dev`; it cannot be changed later.

*Neon, project `pulse-dev`.*

- Free plan, Postgres 17, region `aws-ap-southeast-1`. Keep the default branch `main` for dev data and create one branch named `test` (integration tests create and drop their own schemas there).
- Leave scale-to-zero at its Free default (5 minutes). Leave the IP allow list, Neon Auth, the Data API, Functions and Storage off.
- From the Connect dialog, get a pooled string (host contains `-pooler`) and a direct string for each branch. The default owner role must be able to create `btree_gist` (spike S1 confirms it).

| File | Variable | Value |
|---|---|---|
| `.env.local` | `DATABASE_URL` | `main`, pooled |
| `.env.local` | `DIRECT_URL` | `main`, direct |
| `.env.test` | `TEST_DATABASE_URL` | `test`, pooled |
| `.env.test` | `TEST_DIRECT_URL` | `test`, direct |

*Clerk, a Development application named “Pulse (dev)”.*

- Sign-in options: email address with password, plus Google (the dev instance uses Clerk’s shared Google credentials, nothing to configure). Keep email verification at its default. Leave Username disabled (Pulse has its own handle); first and last name optional. Do not enable Organizations, Billing, MFA or the Waitlist. Optional: allow users to delete their own account.
- For spike S5: add one temporary webhook endpoint pointing at a webhook inspector page (example events only, never real users), subscribed only to `user.created`, `user.updated` and `user.deleted`. Delete it after S5; the permanent endpoint comes with the webhook task.

| File | Variable | Source |
|---|---|---|
| `.env.local` | `NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY` | Clerk dashboard, API keys |
| `.env.local` | `CLERK_SECRET_KEY` | Clerk dashboard, API keys |
| `.env.local` | `NEXT_PUBLIC_CLERK_SIGN_IN_URL`, `NEXT_PUBLIC_CLERK_SIGN_UP_URL` | optional: `/sign-in`, `/sign-up` |
| `.env.local` (later) | `CLERK_WEBHOOK_SIGNING_SECRET` | the webhook task (T-017) |
| `.env.local` | `NEXT_PUBLIC_APP_URL`, `PULSE_POLL_MODE`, `ALLOW_SEED` | plain values: `http://localhost:3000`, `L0`, `1` (dev only) |

*Notes.* Next.js’s default `.gitignore` ignores every `.env*` file, which would also hide `.env.example` and `.env.test.example`; the guardrails task (T-004) adds exceptions for those two. **Before T-007:** a Vercel Hobby account connected to the GitHub repo. **Before M6:** an Android phone and an iPhone. The production Neon project (same region) and the Clerk production instance are created in the production task (a separate Neon project, so test runs never spend production compute hours). Development works on your local Node 22.23.2; Vercel runs Node 24.

---

## 2. Milestones at a glance

| Milestone | Tasks | Goal | Exit criteria |
|---|---|---|---|
| **M0 Spikes and setup** | T-001 to T-008 (8) | Answer the open technical questions (S1, S5), scaffold the app with its tooling and CI, build the pure day and time logic, and deploy a walking skeleton on the database with the region pairing measured. | S1 and S5 are recorded in `docs/spikes/`; the app builds in CI and deploys to Vercel in the region of the Neon project and shows “DB ok” with a poll-sized round trip within 150 ms; `npm run test`, `npm run test:tz` and `npm run test:db` pass; the database schema and constraints are in the dev project. |
| **M1 Auth and groups** | T-009 to T-026 (18) | Sign in, onboard, create and join groups, switch groups, edit the profile and time zone, remove and rejoin members, and survive deleted Clerk users. | Two real accounts can sign up, onboard, create a group, join by link with one tap, switch groups, change time zone and have a member removed and rejoin; a Clerk deletion tombstones a user; TP-05 passes. |
| **M2 Targets and checklist** | T-027 to T-040 (14) | Create targets from templates, share them to groups, edit, archive and delete them, and check off today and yesterday reliably. | A user can manage targets and tick tasks on a phone with instant, reliable saving; the edit window, scheduling and sharing rules hold; TP-07 passes. |
| **M3 Heatmap** | T-041 to T-047 (7) | The personal heatmap with its levels, states, accessible summary and day-by-day list. | The seeded account shows 52 or 13 weeks on Today with the correct states, one Tab stop and a working day list. |
| **M4 Roll call and streaks** | T-048 to T-059 (12) | The group page: roll call, comparison rows, streaks with the cache, and live updates through the pulse with the 3-minute stop. | The seeded group renders correctly; two browsers show the same progress within a poll; polling stops after 3 minutes; TP-01 and TP-02 pass. |
| **M5 Feed and reactions** | T-060 to T-066 (7) | One card per member per day, paging, reactions, and the live feed in the pulse. | Cards and reactions work across two accounts; TP-03 and TP-04 pass. |
| **M6 PWA and launch prep** | T-067 to T-072 (6) | Manifest and icons, security headers, accessibility pass, the measurement and device spikes (S3, S4, S6), the production environment and the launch rehearsal. | Pulse installs on Android and iOS, is measured within budget, is deployed to production, and every launch criterion of PRD §7.1 is met. |

---

## 3. Tasks

### M0 — Spikes and setup

*Exit criteria:* S1 and S5 are recorded in `docs/spikes/`; the app builds in CI and deploys to Vercel in the region of the Neon project and shows “DB ok” with a poll-sized round trip within 150 ms; `npm run test`, `npm run test:tz` and `npm run test:db` pass; the database schema and constraints are in the dev project.

#### T-001 · Spike S1: Prisma 7 + Neon adapter, btree_gist on the real project, migrate drift

- **Type:** spike · **Risk:** high · **Size:** about 120 lines · **Status:** todo
- **Serves:** S1
- **Depends on:** nothing
- **Touches:** Surfaces: none (design check). Models: all 12, applied to a throwaway Neon dev branch.
- **Files:** `docs/spikes/S1.md` (new, at most 60 lines); a throwaway folder outside the repo for the scripts
- **Questions:**
  1. Do Prisma 7.10.1 and `@prisma/adapter-neon` 7.10.1, on the **pooled** URL, run `findMany`, an **interactive `$transaction`**, `$queryRaw` with `SELECT … FOR UPDATE`, and `@default(uuid(7))` at runtime? (The join and `recordProgress` designs depend on interactive transactions.)
  2. Does the schema of DB_SCHEMA §2 (datasource block with only `provider`) work with a `prisma.config.ts` that points the CLI at the **direct** URL, for `migrate dev` and `migrate deploy`, and does the generated client (`prisma-client` generator with `output`) import under Node ESM and under Vitest?
  3. Does `CREATE EXTENSION btree_gist` succeed on the real Neon project (Free plan), and does every statement of DB_SCHEMA §3 apply?
  4. After DB_SCHEMA §3 is committed as a hand-written migration, does `prisma migrate dev` report drift on the partial unique indexes, exclusion constraints or `CHECK`s, or try to drop them?
  5. Is Node's global `WebSocket` enough for `PrismaNeon` (no `ws` package)?
  6. How long does the first query take after the compute has been idle for more than 5 minutes, with default settings and with raised `connect_timeout` and `pool_timeout`?
- **How to answer:**
  - Create a throwaway folder outside the repo; install exactly `prisma@7.10.1`, `@prisma/client@7.10.1`, `@prisma/adapter-neon@7.10.1`. Copy DB_SCHEMA §2 and §3 verbatim. Use a **new branch of the `pulse-dev` Neon project** (never production).
  - Questions 2 to 4: `npx prisma migrate dev --name init`, then `npx prisma migrate dev --create-only --name constraints`, paste §3 into it, `npx prisma migrate dev`, and run `npx prisma migrate dev` once more. The second run must say the database is in sync.
  - Question 1: a script that runs a transaction doing `SELECT id FROM groups WHERE id = $1 FOR UPDATE`, a count, and an insert, and fires 20 of them with `Promise.all` against one group seeded with 14 members; exactly one must win the 15th place.
  - Question 3: `SELECT * FROM pg_extension WHERE extname = 'btree_gist'` after the migration, plus the overlap tests from DB_SCHEMA §10.
  - Questions 5 and 6: run on the local Node 22.23.2 and note it; wait 6 minutes, then time the first query (`performance.now()`), three times.
- **If the answer is surprising:**
  - Q1 interactive transactions fail: **stop**. Amend ADR-0001 and redesign TAD §4.2 and §6.7 around single-statement writes (CTEs); ask you before proceeding.
  - Q2 configuration differs: fix DB_SCHEMA §2 comment, TAD §11 and §12.2.
  - Q3 `btree_gist` cannot be created: DB_SCHEMA §3.1 (trigger plus advisory lock) becomes the primary design; edit §3, TAD §14.3; TP-09 becomes mandatory and moves to the schema task.
  - Q4 drift is reported: move the hand-written SQL to `prisma/sql/` applied by `prisma db execute` after `migrate deploy`, or keep it and add a CI drift check; amend DB_SCHEMA §3 and TAD §12.2.
  - Q5 `ws` is required: **stop and ask you**, because it is a library outside the approved list.
  - Q6 above 4 seconds: NFR-PERF-03 is missed on first load; add a warm-up request to the sign-in page (TAD §12.1) and report it for a PRD note.
- **Acceptance:**
  - `docs/spikes/S1.md` answers questions 1 to 6 above, each with the command and its output as evidence.
  - Every surprise is turned into the doc change listed under “If the answer is surprising” and raised with you before any feature task starts.
  - No spike code enters the repo; the throwaway folder is deleted at the end.
- **Verification:**
  - Run the throwaway scripts described under “How to answer” and paste their output into `docs/spikes/S1.md`.
  - manual: Neon console, Tables tab: 12 tables and 7 enums exist on the throwaway branch; delete the branch afterwards.

#### T-002 · Spike S5: Clerk webhook payloads and local testing path

- **Type:** spike · **Risk:** med · **Size:** about 100 lines · **Status:** todo
- **Serves:** S5, FR-ACC-02
- **Depends on:** nothing
- **Touches:** Surfaces: R-04 (design check only). Models: none.
- **Files:** `docs/spikes/S5.md` (new, at most 40 lines); `tests/fixtures/clerk/*.json` (sanitised example payloads)
- **Questions:**
  1. What are the exact fields of `user.created`, `user.updated` and `user.deleted`? API_SPEC R-04 assumes `id`, `first_name`, `last_name`, `username`, `image_url`, and `{ id }` for `user.deleted`.
  2. What is the signing-secret environment variable, and what does `verifyWebhook` do with a bad signature (throw, which error)?
  3. Does an OAuth (Google) sign-up carry names and an image, and an email sign-up carry neither?
  4. Which events must be subscribed in the Clerk dashboard so nothing else arrives?
  5. Can we develop locally without a tunnel?
- **How to answer:**
  - In a Clerk **development** instance, open Webhooks, add an endpoint, and use the Testing tab to send example events. Read the bodies from the message attempts. For an inspector URL use only example data (never real users).
  - Real payloads are captured later, in the webhook task, against the Vercel preview URL from the walking-skeleton task: one Google sign-up, one email sign-up, one profile edit, one deletion.
  - Decide the local workflow: `ensureUser` makes webhooks unnecessary for day-to-day work, so no tunnel is needed.
- **If the answer is surprising:**
  - Field names differ: update the `clerkEvent` block and prose of API_SPEC R-04, TAD §4.3, and the scope of the webhook task.
  - The secret variable has another name: update TAD §11 and `.env.example`.
  - `user.deleted` carries no usable id: update TAD §4.5 and DB_SCHEMA §6.2 (the tombstone key).
  - Email sign-ups have no `image_url`: nothing changes (initials are already the fallback).
  - Bad signatures do not throw as assumed: update the 400/500 mapping of R-04.
- **Acceptance:**
  - `docs/spikes/S5.md` records the exact JSON shape of `user.created`, `user.updated` and `user.deleted` and the signing-secret variable name.
  - A development workflow is chosen and written down: rely on `ensureUser` locally and on a Vercel preview URL for real deliveries.
  - Sanitised example payloads are saved as fixtures for the webhook tests.
- **Verification:**
  - Compare the saved payloads against the `clerkEvent` Zod block of API_SPEC R-04 with a one-off script and paste the result.
  - manual: Clerk dashboard, Webhooks, Testing tab: send an example of each of the three events and read the delivered body.

#### T-003 · Scaffold, test tooling and CI (spike S2)

- **Type:** infra, enabler · **Risk:** med · **Size:** about 250 lines · **Status:** todo
- **Serves:** S2, NFR-COMPAT-01, NFR-COST-01, NFR-QUAL-01
- **Depends on:** nothing
- **Touches:** Surfaces: none. Models: none.
- **Files:** `package.json`, `.npmrc`, `.nvmrc`, `tsconfig.json`, `next.config.ts`, `eslint.config.mjs`, `postcss.config.mjs`, `src/app/layout.tsx`, `src/app/page.tsx`, `src/app/globals.css`, `vitest.config.ts`, `scripts/test-tz.mjs`, `tests/helpers/clock.ts`, `tests/smoke.test.ts`, `.github/workflows/ci.yml`
- **Acceptance:**
  - `create-next-app` 16.4.3 (App Router, `src/`, TypeScript, Tailwind 4.3.3, ESLint) with every package pinned **exactly** as in TAD §2 (TypeScript 6.0.3, ESLint 9.39.5, eslint-config-next 16.4.3, @types/node 24.19.2); `save-exact=true`; `engines.node` is `>=24 <25`; TypeScript `strict` is on; `/` renders the word Pulse.
  - If a pin breaks the build, the exact error is recorded in TAD §0.2 and you are asked before any pin changes (that is the S2 outcome).
  - Vitest 5.0.5 with two projects: `unit` (no database) and `integration` (database, long timeouts). `npm run test:tz` runs the `unit` project once per zone (Pacific/Kiritimati, Etc/GMT+12, Asia/Kolkata, Australia/Lord_Howe, America/New_York, Pacific/Apia, Africa/Casablanca, UTC) by setting `TZ` in a child process (no new library; works on Windows) and stops at the first failure. The smoke test asserts the process time zone equals the requested one.
  - CI on every push and pull request: Node 24, `npm ci`, `typecheck`, `lint`, `test`, `test:tz`, and `npm audit --omit=dev` (report-only). Integration tests are **not** run in CI (they need the Neon dev project); they stay a local command.
- **Verification:**
  - `npm ci` · `npm run typecheck` · `npm run lint` · `npm run build` · `npm ls typescript eslint next` (must show the exact pins)
  - `npm run test` · `npm run test:tz` (eight runs, all green)
  - manual: `npm run dev`, open http://localhost:3000, see the page; push a branch and watch the GitHub Actions run (five steps) turn green.

#### T-004 · Folder skeleton, import-boundary lint rules and environment validation

- **Type:** infra, enabler · **Risk:** med · **Size:** about 300 lines · **Status:** todo
- **Serves:** S2, NFR-SEC-02
- **Depends on:** T-003
- **Touches:** Surfaces: none. Models: none.
- **Files:** `src/{core,lib,features,components}/…` skeleton, `eslint.config.mjs`, `tests/lint/boundaries.test.ts`, `.env.example`, `src/lib/env.server.ts`, `src/lib/env.public.ts`
- **Acceptance:**
  - The tree of TAD §3 exists with placeholder `index.ts` files; `server-only` 0.0.1 is installed and imported by every future `server/` and `lib/auth/` module.
  - ESLint `no-restricted-imports` rules enforce the five dependency rules of TAD §3 (app imports only feature indexes; features do not import each other's internals; `core/` imports nothing outside `core/`; the Prisma client only from server modules; client code never imports `server/`).
  - `lib/env.server.ts` and `lib/env.public.ts` parse the variables of TAD §11 with Zod and fail at boot if one is missing; `.env.example` lists names only.
- **Verification:**
  - `npm run lint` · `npm run test -- tests/lint` (fixture files that break each rule must produce an ESLint error) · `npm run build`

#### T-005 · core/time: LocalDate and UTC-only date arithmetic

- **Type:** pure, enabler · **Risk:** low · **Size:** about 280 lines · **Status:** todo
- **Serves:** FR-CHK-06, FR-HM-01, BR-DAY-2, BR-DAY-6, NFR-QUAL-01
- **Depends on:** T-003
- **Touches:** Surfaces: none. Models: none.
- **Files:** `src/core/time/local-date.ts`, `src/core/time/iso-week.ts` (week start only), `src/lib/db-dates.ts` (Date to and from LocalDate), `tests/core/time/local-date.test.ts`
- **Acceptance:**
  - Branded `LocalDate`, `parseLocalDate`, `addDays`, `diffDays`, `weekdayISO`, `isoWeekStart`, and the repository-only converters for Prisma `DATE` values (UTC midnight).
  - No function reads the process time zone: the same results under all eight `TZ` values.
  - Cases: month and year boundaries, leap day, ISO weekday and Monday-start week for known dates, `addDays` across a local DST day.
- **Verification:**
  - `npm run test -- core/time/local-date` · `npm run test:tz`

#### T-006 · core/time: zones, effectiveToday (BR-DAY-5), resolveDay and the roll-call cutoff

- **Type:** pure, enabler · **Risk:** med · **Size:** about 380 lines · **Status:** todo
- **Serves:** FR-ACC-05, FR-ROL-02, FR-CHK-06, BR-DAY-1, BR-DAY-3, BR-DAY-5, NFR-QUAL-01
- **Depends on:** T-005
- **Touches:** Surfaces: none. Models: none (reads the shape of `users.timezone` and `users.date_floor`).
- **Files:** `src/core/time/zone.ts`, `src/core/time/effective-today.ts`, `tests/core/time/zone.test.ts`, `tests/core/time/effective-today.test.ts`
- **Acceptance:**
  - `isIanaZone` (accepts `UTC` and `Area/Location`, rejects offsets such as `+05:00`), `calendarDateIn`, `clockTimeIn`, `effectiveToday` with the date floor, `resolveDay('today'|'yesterday')`, `afterCutoff` (false during a hold), and the floor update used when the zone changes.
  - The TAD §5.4 worked example (Kolkata to Los Angeles) is a test that reproduces its three-row table.
  - Cases: DST days of America/New_York, the skipped day of Pacific/Apia (2011-12-30), Lord_Howe, Kiritimati, Casablanca, a forward jump, a backward jump, and `afterCutoff` at 17:59 and 18:00.
- **Verification:**
  - `npm run test -- core/time` · `npm run test:tz`

#### T-007 · Prisma schema, migrations, database client and walking-skeleton deploy to Vercel

- **Type:** db, enabler · **Risk:** med · **Size:** about 260 lines · **Status:** todo
- **Serves:** S1, FR-ACC-02, FR-GRP-03, FR-VIS-04, NFR-COST-01, NFR-COMPAT-01, NFR-PERF-02
- **Depends on:** T-001, T-003, T-004
- **Touches:** Surfaces: none (a temporary home page). Models: all 12.
- **Files:** `prisma/schema.prisma`, `prisma/migrations/<ts>_init/`, `prisma/migrations/<ts>_constraints/`, `prisma.config.ts`, `src/lib/db.ts`, `package.json` (db scripts, `vercel-build`), `vercel.json` (function region), `src/app/page.tsx` (temporary), Vercel project settings (outside the repo)
- **Acceptance:**
  - `schema.prisma` is DB_SCHEMA §2 verbatim; the constraints migration is DB_SCHEMA §3 verbatim, adjusted only by the S1 outcome.
  - `lib/db.ts` builds one `PrismaClient` with `PrismaNeon` on the pooled `DATABASE_URL`; `prisma.config.ts` points the CLI at `DIRECT_URL`. `npm run db:deploy` applies both migrations to the `pulse-dev` project.
  - A Vercel Hobby project is connected to the GitHub repo with Node 24. The **function region is set to `sin1` (Singapore) in `vercel.json`, matching the Neon region `aws-ap-southeast-1`**; if Hobby does not allow it, stop and ask you (answer recorded in `docs/spikes/S1.md`).
  - `vercel-build` runs `prisma migrate deploy` **only** when `VERCEL_ENV=production`, then `next build`; preview and production builds pass.
  - The temporary home page shows “DB ok”, the function region read from `process.env.VERCEL_REGION`, and the **round-trip time of a poll-sized query**: five sequential indexed `SELECT … LIMIT 20` statements (membership, target structure, rules, entries, shares) over the real tables, median of ten warm repetitions after one warm-up. The median for all five is at most 150 ms (the 200 ms poll wall-time assumption of TAD §12.3, less handler overhead). The measured number is recorded in `docs/spikes/S1.md`; if it is above 150 ms, the region pairing is reviewed and raised with you before any feature task starts. The page is removed in the shell task.
- **Verification:**
  - `npx prisma validate` · `npm run db:deploy` · `npm run typecheck` · `npm run build`
  - Push a branch (preview build) and to `main` (production build); open the **deployed preview URL**: “DB ok”, region `sin1` and the five-query round-trip median appear; repeat the measurement from the page three times and record the lowest and highest.
  - manual: Neon console, SQL editor: `SELECT count(*) FROM pg_tables WHERE schemaname='public'` returns 13 (12 plus the Prisma migrations table); `SELECT extname FROM pg_extension` lists `btree_gist`. Vercel dashboard, Functions: region equals `sin1`; Neon project region is Singapore.

#### T-008 · Integration-test harness and the constraint suite

- **Type:** db, enabler · **Risk:** high · **Size:** about 380 lines · **Status:** todo
- **Serves:** FR-GRP-03, FR-VIS-04, NFR-REL-01
- **Depends on:** T-003, T-007
- **Touches:** Surfaces: none. Models: all 12 (fixtures).
- **Files:** `tests/integration/setup.ts`, `tests/integration/fixtures.ts`, `tests/integration/constraints.test.ts`, `vitest.config.ts`, `.env.test.example`
- **Acceptance:**
  - Each test file gets its own Postgres **schema** in the `pulse-dev` Neon project, created from the migration SQL (with `btree_gist` available) and dropped afterwards; no test touches the `public` schema or production.
  - Fixture builders create users, groups, memberships, targets, revisions, shares and entries; helpers mock Clerk `auth`, `next/headers` cookies and `server-only`.
  - `constraints.test.ts` ports the 68 checks verified in DB_SCHEMA §10 as Vitest tests (the fallback checks are skipped unless the S1 outcome adopts the fallback).
- **Verification:**
  - `npm run test:db -- constraints` (all green)
  - manual: Neon console shows no leftover test schemas after the run.

### M1 — Auth and groups

*Exit criteria:* Two real accounts can sign up, onboard, create a group, join by link with one tap, switch groups, change time zone and have a member removed and rejoin; a Clerk deletion tombstones a user; TP-05 passes.

#### T-009 · Foundations: Result, errors, clock, limits, templates

- **Type:** pure, enabler · **Risk:** low · **Size:** about 220 lines · **Status:** todo
- **Serves:** FR-TGT-01, FR-TGT-02, BR-TGT-3, BR-GROUP-1
- **Depends on:** T-004
- **Touches:** Surfaces: none. Models: none.
- **Files:** `src/lib/result.ts`, `src/lib/errors.ts`, `src/lib/clock.ts`, `src/core/limits.ts`, `src/core/templates.ts`, `tests/lib/*.test.ts`, `tests/core/templates.test.ts`
- **Acceptance:**
  - `Result`, `ok`, `err` and the error codes of API_SPEC §1.2, with constructors and a Zod-to-`AppError` mapper that fills `fieldErrors`.
  - `core/limits.ts` holds every number of the PRD business rules once (names, goal ranges, caps, quick-add values, 18:00 cutoff, group cap 15, feed page 20, polling constants).
  - `core/templates.ts` holds the six templates and Custom with the BR-TPL defaults; a test compares them with the PRD table.
- **Verification:**
  - `npm run test -- lib core/templates core/limits` · `npm run typecheck` · `npm run lint`

#### T-010 · Input helpers: invite codes and shared Zod primitives

- **Type:** pure, enabler · **Risk:** low · **Size:** about 360 lines · **Status:** todo
- **Serves:** FR-GRP-02, FR-GRP-03, BR-GROUP-4, FR-ACC-03, FR-ACC-04, FR-GRP-01, FR-TGT-02
- **Depends on:** T-006, T-009
- **Touches:** Surfaces: none. Models: none.
- **Files:** `src/core/invite-code.ts`, `src/lib/primitives.ts`, `tests/core/invite-code.test.ts`, `tests/lib/primitives.test.ts`
- **Acceptance:**
  - `generateInviteCode(rng)` produces 8 characters from the 31-symbol alphabet (A–Z without I, L, O, plus 2–9) using an injected random function; `normalizeInviteCode` upper-cases and strips spaces and hyphens. 10,000 generated codes contain no ambiguous character and match `^[A-HJKMNP-Z2-9]{8}$`; normalisation accepts `abcd-2345` and `ABCD 2345`.
  - The primitives of API_SPEC §1.4 are implemented verbatim (uuid, localDate, zone, handle, names, mode, frequency, weekday, dayToken, reactionEmoji, nextPath, params), and the test file reproduces the executable assertions of API_SPEC §10 for them (impossible date, offset zone, open-redirect paths, handle normalisation).
- **Verification:**
  - `npm run test -- core/invite-code lib/primitives` · `npm run typecheck` · `npm run lint`

#### T-011 · defineAction and logError

- **Type:** db, enabler · **Risk:** low · **Size:** about 260 lines · **Status:** todo
- **Serves:** FR-ACC-03, NFR-OPS-01
- **Depends on:** T-007, T-008, T-009
- **Touches:** Surfaces: all actions (wrapper). Models: error_events.
- **Files:** `src/lib/define-action.ts`, `src/lib/log-error.ts`, `tests/integration/define-action.test.ts`
- **Acceptance:**
  - `defineAction` parses input with Zod (`VALIDATION` with `fieldErrors`), calls `requireUser`, runs the body, and turns any exception into `INTERNAL` without leaking details (TAD §9.2).
  - `logError` inserts an `error_events` row (message cut to 500 characters, fingerprint) and prunes rows older than 30 days on about one write in twenty.
- **Verification:**
  - `npm run test:db -- define-action` (one row per forced error; no stack trace stored; validation never writes a row)

#### T-012 · Clerk wiring, proxy and the sign-in and sign-up pages

- **Type:** infra · **Risk:** med · **Size:** about 250 lines · **Status:** todo
- **Serves:** FR-ACC-01, FR-ACC-06
- **Depends on:** T-003, T-004
- **Touches:** Surfaces: P-01. Models: none.
- **Files:** `src/proxy.ts`, `src/app/layout.tsx` (ClerkProvider), `src/app/(public)/sign-in/[[...sign-in]]/page.tsx`, `src/app/(public)/sign-up/[[...sign-up]]/page.tsx`, `.env.example`
- **Acceptance:**
  - `@clerk/nextjs` 7.9.21; `proxy.ts` protects everything except `/sign-in`, `/sign-up`, `/api/webhooks/clerk`, `/manifest.webmanifest`, `/icons/*`, `/apple-icon*` and `/favicon.ico`.
  - A signed-out visitor to a protected page is redirected to sign-in and returned to the page after signing in.
  - Google and email sign-up work against the Clerk development instance; the Clerk user menu offers sign-out.
- **Verification:**
  - `npm run typecheck` · `npm run build` · `curl -I http://localhost:3000/manifest.webmanifest` (must not redirect to sign-in; 404 is fine until the manifest task)
  - manual: private window, open /today, get sent to sign-in, sign up with Google, come back; sign out from the user menu.

#### T-013 · ensureUser and requireUser

- **Type:** db, enabler · **Risk:** med · **Size:** about 350 lines · **Status:** todo
- **Serves:** FR-ACC-02, NFR-REL-02
- **Depends on:** T-007, T-008, T-010, T-011, T-012
- **Touches:** Surfaces: every authenticated surface. Models: users.
- **Files:** `src/lib/auth/ensure-user.ts`, `src/lib/auth/require-user.ts`, `src/features/accounts/server/handle.ts`, `tests/integration/ensure-user.test.ts`, `tests/features/accounts/handle.test.ts`
- **Acceptance:**
  - The first authenticated request creates the `users` row (one query on the hot path afterwards); two concurrent first requests create exactly one row.
  - `suggestHandle` produces a valid handle from the name and retries up to five times on a collision; a tombstoned user is treated as signed out (`UNAUTHENTICATED`).
- **Verification:**
  - `npm run test:db -- ensure-user` · `npm run test -- features/accounts/handle`
  - manual: sign up with a new account; the user appears in the Neon `users` table with a handle and `onboarded_at` null.

#### T-014 · Theme provider and design tokens

- **Type:** infra, enabler · **Risk:** med · **Size:** about 380 lines · **Status:** todo
- **Serves:** NFR-UX-02, NFR-A11Y-02, FR-HM-04
- **Depends on:** T-003
- **Touches:** Surfaces: none. Models: none.
- **Files:** `src/lib/theme.tsx`, `src/app/globals.css` (tokens), `src/components/ui/*` (shadcn: button, card, input, label, select, dialog, sonner), `components.json`
- **Acceptance:**
  - Light and dark follow the device; the user can override, remembered in `localStorage` inside try/catch; no flash on load.
  - Colour tokens include the five heatmap levels and the distinct empty, untracked and today states for both themes; `prefers-reduced-motion` is respected.
  - shadcn 4.23.0 is initialised; only the listed components are added.
- **Verification:**
  - `npm run build` · `npm run lint`
  - manual: toggle the device theme; reload; switch the override; no white flash in dark mode.

#### T-015 · Onboarding: completeOnboarding (A-01) and the onboarding page (L-02)

- **Type:** ui · **Risk:** med · **Size:** about 380 lines · **Status:** todo
- **Serves:** FR-ACC-03
- **Depends on:** T-006, T-010, T-011, T-013, T-014
- **Touches:** Surfaces: A-01, L-02. Models: users.
- **Files:** `src/features/accounts/server/actions.ts`, `src/features/accounts/schema.ts`, `src/app/(app)/onboarding/page.tsx`, `src/features/accounts/components/OnboardingForm.tsx`, `tests/integration/onboarding.test.ts`
- **Acceptance:**
  - The form is prefilled with the Clerk name, a suggested handle and the detected time zone; saving sets `onboarded_at`, then redirects to the validated `next` path or `/today`.
  - A taken handle returns `CONFLICT` with a field error; calling the action twice is harmless; `next` values like `//evil.com` are dropped.
- **Verification:**
  - `npm run test:db -- onboarding` · `npm run test -- lib/primitives`
  - manual: new private-window sign-up lands on /onboarding with detected zone; take an existing handle, see the error; finish and land on /today.

#### T-016 · App shell (L-01): onboarding redirect, navigation, query provider, error pages

- **Type:** ui, enabler · **Risk:** med · **Size:** about 330 lines · **Status:** todo
- **Serves:** FR-ACC-01, FR-ACC-06, FR-GRP-11, NFR-UX-01
- **Depends on:** T-012, T-013, T-014, T-015
- **Touches:** Surfaces: L-01. Models: users, memberships (groups for the switcher data).
- **Files:** `src/app/(app)/layout.tsx`, `src/app/not-found.tsx`, `src/app/error.tsx`, `src/lib/query-client.tsx`, `src/features/accounts/server/queries.ts`, `src/app/page.tsx` (temporary page removed)
- **Acceptance:**
  - The layout requires a user, redirects un-onboarded users to `/onboarding?next=<path>`, and renders navigation (Today, Targets, Groups) and the Clerk user menu.
  - `QueryClientProvider` is set up; signing out clears the query cache so no data remains on the device.
  - One uniform not-found page and one error page (the same words for a missing group and a group you are not in).
  - The temporary home page is replaced: `/` sends a signed-in user to `/today` (the proxy already sends a signed-out visitor to sign-in).
- **Verification:**
  - `npm run build` · `npm run lint`
  - manual at 360 px wide: navigation fits, no horizontal scroll; sign out, press Back: no private data on screen.

#### T-017 · Clerk webhook: verification and profile sync (R-04)

- **Type:** db · **Risk:** med · **Size:** about 330 lines · **Status:** todo
- **Serves:** FR-ACC-02, FR-ACC-04
- **Depends on:** T-002, T-008, T-011, T-013
- **Touches:** Surfaces: R-04. Models: users.
- **Files:** `src/app/api/webhooks/clerk/route.ts`, `src/features/accounts/schema.ts` (`clerkEvent`), `src/features/accounts/server/webhook.ts`, `tests/integration/webhook.test.ts`, `tests/fixtures/clerk/`
- **Acceptance:**
  - `verifyWebhook` guards the route (invalid signature gives 400 and no processing); `user.created` and `user.updated` upsert display name and image, never the handle, and are ignored for tombstoned rows.
  - Unknown event types answer 200; a database failure answers 500; handlers are idempotent and tolerate reordering.
  - `user.deleted` is dispatched to `tombstoneUser`, which the next tasks fill (a logged no-op until then).
- **Verification:**
  - `npm run test:db -- webhook`
  - manual: deploy a preview, point the Clerk dev webhook at it, edit your name in Clerk, see it change in `users`; capture real payloads into the fixtures (S5).

#### T-018 · Sharing foundation: share-interval logic and close-out

- **Type:** pure, enabler · **Risk:** med · **Size:** about 390 lines · **Status:** todo
- **Serves:** FR-VIS-04, FR-VIS-05, FR-VIS-06, FR-VIS-02, FR-GRP-07, BR-SHARE-1, BR-SHARE-2, BR-SHARE-3, BR-SHARE-5
- **Depends on:** T-005, T-006, T-007, T-008
- **Touches:** Surfaces: none yet (used by A-06, A-07, A-11, R-04). Models: share_intervals, targets, memberships.
- **Files:** `src/core/schedule/share-interval.ts`, `src/features/sharing/server/repo.ts`, `tests/core/schedule/share-interval.test.ts`, `tests/integration/sharing-repo.test.ts`
- **Acceptance:**
  - Pure decisions: `isSharedOn`, and plans for share (insert, or reopen when closed today), unshare (close at today), and member close-out (close at yesterday, or delete the interval when it began today).
  - The repository applies a plan inside a transaction for one target and group, or for every target of a user in a group (and all groups for a tombstone).
  - Integration tests prove the plans respect the database constraints (no overlap, one open interval, `from <= to`).
- **Verification:**
  - `npm run test -- core/schedule/share-interval` · `npm run test:db -- sharing-repo` · `npm run test:tz`

#### T-019 · tombstoneUser (R-04: user.deleted)

- **Type:** db · **Risk:** med · **Size:** about 300 lines · **Status:** todo
- **Serves:** FR-ACC-02, FR-ACC-04, NFR-REL-02, BR-SHARE-5
- **Depends on:** T-006, T-017, T-018
- **Touches:** Surfaces: R-04. Models: users, memberships, share_intervals, streak_prefixes.
- **Files:** `src/features/accounts/server/tombstone.ts`, `tests/integration/tombstone.test.ts`
- **Acceptance:**
  - One transaction implements DB_SCHEMA §6.2: anonymise (name, image, handle `deleted_` plus 12 hex), keep the row and `clerk_id`, set every membership to `REMOVED`, close or delete open intervals, delete streak prefixes. Nothing else is deleted.
  - A deleted owner leaves a group with no active owner; a second delivery and a late `user.updated` change nothing; the tombstone handle passes the handle `CHECK`.
- **Verification:**
  - `npm run test:db -- tombstone`

#### T-020 · requireGroupAccess and requireTargetOwner

- **Type:** db, enabler · **Risk:** med · **Size:** about 300 lines · **Status:** todo
- **Serves:** FR-GRP-11, NFR-SEC-01
- **Depends on:** T-007, T-008, T-013
- **Touches:** Surfaces: every `member`, `owner` and `target-owner` surface. Models: memberships, groups, targets.
- **Files:** `src/lib/auth/group-access.ts`, `src/lib/auth/target-access.ts`, `tests/integration/access.test.ts`, `tests/types/access.typecheck.ts`
- **Acceptance:**
  - A member gets a branded `GroupAccess`; a non-member, a removed member, an unknown id and a deleted group all produce the **same** `NOT_FOUND` error value; a non-owner on an owner action gets `FORBIDDEN`.
  - `GroupAccess` cannot be constructed outside `lib/auth`: a type test with `@ts-expect-error` proves it compiles only through the helper.
- **Verification:**
  - `npm run test:db -- access` · `npm run typecheck` (the `@ts-expect-error` lines must be needed)

#### T-021 · createGroup (A-03), create form and invite panel

- **Type:** ui · **Risk:** low · **Size:** about 360 lines · **Status:** todo
- **Serves:** FR-GRP-01, FR-GRP-02
- **Depends on:** T-010, T-011, T-016, T-020
- **Touches:** Surfaces: A-03. Models: groups, memberships.
- **Files:** `src/features/groups/server/actions.ts`, `src/features/groups/schema.ts`, `src/app/(app)/groups/new/page.tsx`, `src/features/groups/components/CreateGroupForm.tsx`, `src/features/groups/components/InvitePanel.tsx`, `tests/integration/groups-create.test.ts`
- **Acceptance:**
  - One transaction creates the group (code retried on a collision) and an `OWNER` membership; the response carries the code and the link built from `NEXT_PUBLIC_APP_URL`; the `pulse_group` cookie is set.
  - The panel shows the code and link with Copy and, where available, the phone share sheet.
- **Verification:**
  - `npm run test:db -- groups-create` (injecting a colliding random function forces the retry)
  - manual on a phone: create a group, copy the link, open the share sheet.

#### T-022 · joinGroup (A-04)

- **Type:** db · **Risk:** high · **Size:** about 330 lines · **Status:** todo
- **Serves:** FR-GRP-03, FR-GRP-11, NFR-REL-01
- **Depends on:** T-007, T-008, T-010, T-011, T-013
- **Touches:** Surfaces: A-04. Models: groups, memberships, streak_prefixes.
- **Files:** `src/features/groups/server/actions.ts`, `src/features/groups/server/join.ts`, `tests/integration/groups-join.test.ts`
- **Acceptance:**
  - One transaction: lock the group row (`FOR UPDATE`), return `alreadyMember`, refuse the 16th with `GROUP_FULL`, reactivate a removed row and delete the user's prefix for that group (no interval reopened), or insert a `MEMBER`.
  - Every invalid, malformed or deleted-group code returns the same `NOT_FOUND` message.
  - **TP-05:** 20 concurrent joins into a group of 14 produce exactly one new member.
- **Verification:**
  - `npm run test:db -- groups-join`

#### T-023 · Invite page and Join-with-a-code form (L-09)

- **Type:** ui · **Risk:** med · **Size:** about 280 lines · **Status:** todo
- **Serves:** FR-GRP-03, FR-ACC-01
- **Depends on:** T-015, T-016, T-022
- **Touches:** Surfaces: L-09, A-04. Models: none.
- **Files:** `src/app/(app)/join/[code]/page.tsx`, `src/features/groups/components/JoinButton.tsx`, `src/features/groups/components/JoinWithCodeForm.tsx`, `src/app/(app)/groups/new/page.tsx`
- **Acceptance:**
  - The invite page writes nothing: it shows the invitation and one **Join group** button that calls `joinGroup`, with a pending state and an inline error.
  - A **Join with a code** form on the new-group screen accepts hand-typed codes (lowercase, spaces, hyphens).
  - After sign-up and onboarding the visitor returns to the invite page through `next`.
- **Verification:**
  - `npm run build` · `npm run lint`
  - manual (M-01): private window, open the invite link, sign up, finish onboarding, land back on the invite page, tap Join group, reach the group; then type a code as `abcd-2345` into the form.

#### T-024 · selectGroup (A-05) and the group switcher

- **Type:** ui · **Risk:** low · **Size:** about 220 lines · **Status:** todo
- **Serves:** FR-GRP-05
- **Depends on:** T-016, T-020, T-021
- **Touches:** Surfaces: A-05, L-01. Models: memberships.
- **Files:** `src/features/groups/server/actions.ts`, `src/features/groups/components/GroupSwitcher.tsx`, `src/features/accounts/server/queries.ts`, `tests/integration/groups-select.test.ts`
- **Acceptance:**
  - The switcher lists the user's active groups by name; choosing one sets the `pulse_group` cookie; the next visit opens that group; a stale cookie falls back to the first group.
  - A non-member group id returns `NOT_FOUND`.
- **Verification:**
  - `npm run test:db -- groups-select`
  - manual: belong to two groups, switch, reload, still on the chosen group.

#### T-025 · Members page and removeMember (L-08, A-06)

- **Type:** ui · **Risk:** med · **Size:** about 380 lines · **Status:** todo
- **Serves:** FR-GRP-07, FR-GRP-02, FR-GRP-11
- **Depends on:** T-016, T-018, T-020, T-022
- **Touches:** Surfaces: L-08, A-06. Models: memberships, share_intervals, streak_prefixes.
- **Files:** `src/app/(app)/groups/[groupId]/members/page.tsx`, `src/features/groups/server/actions.ts`, `src/features/groups/server/queries.ts`, `src/features/groups/components/MemberList.tsx`, `tests/integration/groups-remove.test.ts`
- **Acceptance:**
  - Any member sees avatar, name, handle and role of active members; only the owner sees the invite and the Remove action (with a confirmation).
  - Removal closes or deletes the removed user's intervals for that group, deletes their prefix, and leaves entries and targets alone; the owner cannot remove themselves; a removed user can rejoin.
  - `ownerless` hides owner controls for everyone.
- **Verification:**
  - `npm run test:db -- groups-remove`
  - manual (M-05, part): two accounts; owner removes the other; the removed account loses the group; it rejoins with the link.

#### T-026 · updateProfile (A-02) and the profile page (L-10)

- **Type:** ui · **Risk:** med · **Size:** about 360 lines · **Status:** todo
- **Serves:** FR-ACC-04, FR-ACC-05, BR-DAY-5
- **Depends on:** T-006, T-010, T-013, T-016
- **Touches:** Surfaces: A-02, L-10. Models: users, streak_prefixes.
- **Files:** `src/features/accounts/server/actions.ts`, `src/app/(app)/settings/profile/page.tsx`, `src/features/accounts/components/ProfileForm.tsx`, `tests/integration/profile.test.ts`
- **Acceptance:**
  - Name, handle and time zone can change. A zone change runs in one transaction: floor = effective today in the old zone, new zone saved, all of the user's prefixes deleted; the client clears its cache and refreshes.
  - A second change keeps the date monotonic; a taken handle returns `CONFLICT`; the avatar shows the Clerk image.
- **Verification:**
  - `npm run test:db -- profile`
  - manual (M-04): change the zone to one that is a day behind; Today keeps its date until the new zone catches up.

### M2 — Targets and checklist

*Exit criteria:* A user can manage targets and tick tasks on a phone with instant, reliable saving; the edit window, scheduling and sharing rules hold; TP-07 passes.

#### T-027 · core/schedule: scheduling and completion

- **Type:** pure, enabler · **Risk:** low · **Size:** about 380 lines · **Status:** todo
- **Serves:** FR-TGT-03, FR-CHK-05, FR-CHK-07, BR-SCH, BR-PRG-2
- **Depends on:** T-005, T-009
- **Touches:** Surfaces: none. Models: none (shapes of targets, revisions, inactive periods).
- **Files:** `src/core/schedule/revision.ts`, `src/core/schedule/is-scheduled.ts`, `tests/core/schedule/is-scheduled.test.ts`
- **Acceptance:**
  - `revisionOn`, `inactiveKindOn` (**ARCHIVED wins over PAUSED**), `isTracked`, `isScheduled`, `goalOn`, `isComplete`, and a pure `hasHistory` predicate, exactly as TAD §6.1.
  - Cases: DAILY and SPECIFIC_DAYS across a revision change, a revision effective tomorrow, archive from tomorrow, a PAUSED period (v1.1 shape, tested now), before the start date, CHECKBOX behaving as goal 1.
  - A sanity benchmark runs about 55,000 checks (15 members, 10 targets, 364 days) well under half a second.
- **Verification:**
  - `npm run test -- core/schedule/is-scheduled` · `npm run test:tz`

#### T-028 · Target rules and action input schemas

- **Type:** pure, enabler · **Risk:** low · **Size:** about 300 lines · **Status:** todo
- **Serves:** FR-TGT-01, FR-TGT-02, FR-TGT-03, FR-VIS-01, BR-TGT-3
- **Depends on:** T-009, T-010
- **Touches:** Surfaces: A-07, A-08, A-09, A-10, A-11, A-12 (input schemas). Models: none.
- **Files:** `src/features/targets/schema.ts`, `src/features/sharing/schema.ts`, `src/features/checklist/schema.ts`, `tests/features/targets/schema.test.ts`
- **Acceptance:**
  - `targetRules`, `createTargetInput`, `updateTargetInput`, `archiveTargetInput`, `deleteTargetInput`, `setSharesInput` and `recordProgressInput` are the `// @zod` blocks of API_SPEC, verbatim.
  - The tests replay every boundary assertion of API_SPEC §10: COUNT 1 and 50, DURATION 5 to 480 in steps of 5, CHECKBOX with and without a goal, duplicate weekdays, all seven weekdays becoming DAILY, value 0 and 1440 and 1441.
- **Verification:**
  - `npm run test -- features/targets/schema`

#### T-029 · Target repository and createTarget (A-07)

- **Type:** db · **Risk:** med · **Size:** about 360 lines · **Status:** todo
- **Serves:** FR-TGT-01, FR-TGT-02, FR-TGT-03, FR-TGT-04, FR-VIS-01, FR-VIS-04
- **Depends on:** T-006, T-008, T-011, T-018, T-020, T-027, T-028
- **Touches:** Surfaces: A-07. Models: targets, target_revisions, share_intervals.
- **Files:** `src/features/targets/server/repo.ts`, `src/features/targets/server/actions.ts`, `tests/integration/targets-create.test.ts`
- **Acceptance:**
  - One transaction creates the target with `start_date` = the caller's effective today, the first revision, and an open share interval from today for each chosen group.
  - A group the caller is not an active member of returns `NOT_FOUND`; seven weekdays are stored as DAILY; a caller in Pacific/Kiritimati gets that zone's today.
- **Verification:**
  - `npm run test:db -- targets-create`

#### T-030 · updateTarget (A-08)

- **Type:** db · **Risk:** med · **Size:** about 340 lines · **Status:** todo
- **Serves:** FR-TGT-06, BR-TGT-5
- **Depends on:** T-029
- **Touches:** Surfaces: A-08. Models: targets, target_revisions.
- **Files:** `src/features/targets/server/actions.ts`, `src/features/targets/server/repo.ts`, `tests/integration/targets-update.test.ts`
- **Acceptance:**
  - Name and unit change at once for all days. Rules merge with the latest revision: without history and starting today or later they are updated in place; with history a revision at tomorrow is written (replacing a pending one). Mode is refused once any entry exists, with the field error text of API_SPEC.
  - Two edits on the same day leave one pending revision; an edit at the local midnight boundary (fake clock) lands on the right day.
- **Verification:**
  - `npm run test:db -- targets-update`

#### T-031 · archiveTarget and deleteTarget (A-09, A-10)

- **Type:** db · **Risk:** low · **Size:** about 300 lines · **Status:** todo
- **Serves:** FR-TGT-07, FR-TGT-08, BR-TGT-6, BR-TGT-7
- **Depends on:** T-029
- **Touches:** Surfaces: A-09, A-10. Models: target_inactive_periods, targets, target_revisions, share_intervals.
- **Files:** `src/features/targets/server/actions.ts`, `tests/integration/targets-archive-delete.test.ts`
- **Acceptance:**
  - Archive inserts an open ARCHIVED period from tomorrow and is idempotent; past days are untouched.
  - Delete is allowed only without history (no entry and no elapsed scheduled day); with history it returns `CONFLICT`; the foreign key from entries is a second guard. Both return the groups to refresh.
- **Verification:**
  - `npm run test:db -- targets-archive-delete`

#### T-032 · setShares (A-11)

- **Type:** db · **Risk:** med · **Size:** about 300 lines · **Status:** todo
- **Serves:** FR-VIS-01, FR-VIS-04, FR-VIS-05, FR-VIS-06
- **Depends on:** T-018, T-020, T-029
- **Touches:** Surfaces: A-11. Models: share_intervals.
- **Files:** `src/features/sharing/server/actions.ts`, `tests/integration/sharing-set.test.ts`
- **Acceptance:**
  - The action diffs the desired groups against the open intervals: added groups open from today (or reopen an interval closed today), removed groups close at today.
  - A group the caller is not an active member of returns `NOT_FOUND`. The hook that syncs today's feed card for newly shared groups is added in the feed-sync task.
- **Verification:**
  - `npm run test:db -- sharing-set`

#### T-033 · Targets list page (L-05)

- **Type:** ui · **Risk:** low · **Size:** about 380 lines · **Status:** todo
- **Serves:** FR-TGT-06, FR-TGT-07, FR-TGT-08, FR-VIS-01
- **Depends on:** T-016, T-031, T-032
- **Touches:** Surfaces: L-05, A-09, A-10. Models: targets, target_revisions, target_inactive_periods, share_intervals.
- **Files:** `src/app/(app)/targets/page.tsx`, `src/features/targets/server/queries.ts`, `src/features/targets/components/TargetList.tsx`
- **Acceptance:**
  - Active and archived targets show name, mode, current rule, a pending rule “effective tomorrow”, and which groups see them.
  - Archive and Delete work from the list; Delete is offered only when the target has no history.
- **Verification:**
  - `npm run build` · `npm run lint`
  - manual (M-14): create two targets (next tasks), archive one, delete an unused one, see the archived section.

#### T-034 · Target form: fields and validation (L-06)

- **Type:** ui · **Risk:** med · **Size:** about 380 lines · **Status:** todo
- **Serves:** FR-TGT-02, FR-TGT-03, FR-TGT-06
- **Depends on:** T-014, T-016, T-028, T-029, T-030
- **Touches:** Surfaces: L-06, A-07, A-08. Models: targets, target_revisions.
- **Files:** `src/app/(app)/targets/new/page.tsx`, `src/app/(app)/targets/[targetId]/page.tsx`, `src/features/targets/components/TargetForm.tsx`
- **Acceptance:**
  - Name, mode, goal, unit, frequency and weekday picker, validated on the client with `targetRules`; server field errors are shown.
  - Editing shows “changes take effect tomorrow” when the target has history, and explains why the mode is locked.
- **Verification:**
  - `npm run build` · `npm run lint`
  - manual at 360 px: create a COUNT target, a DURATION target and a SPECIFIC_DAYS checkbox; try goal 7 minutes (rejected with the message).

#### T-035 · Target form: templates and share selector

- **Type:** ui · **Risk:** low · **Size:** about 300 lines · **Status:** todo
- **Serves:** FR-TGT-01, FR-VIS-01
- **Depends on:** T-032, T-034
- **Touches:** Surfaces: L-06, A-07, A-11. Models: share_intervals.
- **Files:** `src/features/targets/components/TemplatePicker.tsx`, `src/features/sharing/components/ShareSelector.tsx`, `src/app/(app)/targets/new/page.tsx`
- **Acceptance:**
  - The six templates and Custom pre-fill mode, goal, unit and frequency with the BR-TPL defaults, all editable.
  - Groups appear as checkboxes; `/targets/new?group=<id>` pre-selects that group only if the user is an active member; none is pre-selected otherwise; a user with no group can save a private target.
- **Verification:**
  - `npm run build` · `npm run lint` · `npm run test -- core/templates`
  - manual (M-14): open the form from a group page (group pre-ticked) and from /targets (nothing ticked).

#### T-036 · getTodayData (R-01 / L-03 data)

- **Type:** db · **Risk:** med · **Size:** about 380 lines · **Status:** todo
- **Serves:** FR-CHK-01, FR-CHK-06, FR-CHK-07, FR-CHK-10
- **Depends on:** T-006, T-008, T-027, T-029
- **Touches:** Surfaces: R-01, L-03. Models: targets, target_revisions, target_inactive_periods, entries, share_intervals, memberships.
- **Files:** `src/features/checklist/server/queries.ts`, `tests/integration/today.test.ts`
- **Acceptance:**
  - Returns tasks for the member's effective today and yesterday in creation order with value, completion, and the groups that see each task on that date; `hasTargets`, summary; the personal streak is 0 until the personal-streak task.
  - Days with no entry are incomplete without any “catch up”; archived and not-yet-started targets are absent; a Kiritimati user gets that zone's dates.
- **Verification:**
  - `npm run test:db -- today`

#### T-037 · recordProgress (A-12) without feed sync

- **Type:** db · **Risk:** med · **Size:** about 360 lines · **Status:** todo
- **Serves:** FR-CHK-02, FR-CHK-03, FR-CHK-04, FR-CHK-05, FR-CHK-06, FR-CHK-08, FR-CHK-09
- **Depends on:** T-006, T-011, T-020, T-036
- **Touches:** Surfaces: A-12. Models: entries.
- **Files:** `src/features/checklist/server/actions.ts`, `src/features/checklist/server/repo.ts`, `tests/integration/record-progress.test.ts`
- **Acceptance:**
  - The server resolves the day from the user's zone and floor; `expectedDate` mismatch returns `STALE_DAY`; an unscheduled day returns `NOT_SCHEDULED`; values above the mode cap return `VALIDATION`.
  - The value is absolute: repeating a call changes nothing; 0 deletes the row; the response lists the groups that share the target on that date.
  - `assertWithinEditWindow` guards the repository (defence in depth). The feed-card sync is added in M5.
- **Verification:**
  - `npm run test:db -- record-progress`

#### T-038 · R-01 route, Today page and empty states

- **Type:** ui · **Risk:** low · **Size:** about 280 lines · **Status:** todo
- **Serves:** FR-CHK-01, FR-CHK-07, FR-CHK-10
- **Depends on:** T-016, T-036
- **Touches:** Surfaces: R-01, L-03. Models: none (reads).
- **Files:** `src/app/api/today/route.ts`, `src/app/(app)/today/page.tsx`, `src/features/checklist/components/TodayClient.tsx`
- **Acceptance:**
  - The route handler and the page call the same `getTodayData`; the page seeds `['today']` as initial data and refetches on focus (at most every 10 s).
  - Empty states: no targets, with a prompt to use the templates; targets but none scheduled today, “Nothing scheduled today”; a skeleton while loading.
- **Verification:**
  - `npm run build` · `curl -i http://localhost:3000/api/today` (401 when signed out)
  - manual: a new account sees the templates prompt; after creating a weekday-only target on a day it is not scheduled, sees “Nothing scheduled today”.

#### T-039 · Progress mutation logic: serial per entry, optimistic, rollback

- **Type:** pure · **Risk:** med · **Size:** about 330 lines · **Status:** todo
- **Serves:** FR-CHK-02, FR-CHK-03, FR-CHK-04, FR-CHK-08, FR-CHK-09
- **Depends on:** T-037, T-038
- **Touches:** Surfaces: A-12, R-01. Models: none.
- **Files:** `src/features/checklist/mutations.ts`, `tests/features/checklist/mutations.test.ts`
- **Acceptance:**
  - A mutation factory gives each entry its own TanStack mutation scope, patches the `['today']` cache with the same core functions as the server, rolls back with a visible message and a retry on failure, and refetches on `STALE_DAY`.
  - **TP-07:** unit tests with a real `QueryClient` and a fake action prove rapid taps run in order, a retried tap is applied once, a failure restores the previous value, and `STALE_DAY` refetches without a rollback toast.
- **Verification:**
  - `npm run test -- features/checklist/mutations`

#### T-040 · Task rows, Yesterday section and daily summary

- **Type:** ui · **Risk:** med · **Size:** about 390 lines · **Status:** todo
- **Serves:** FR-CHK-02, FR-CHK-03, FR-CHK-04, FR-CHK-06, FR-CHK-11, NFR-A11Y-03
- **Depends on:** T-039
- **Touches:** Surfaces: A-12, R-01. Models: none.
- **Files:** `src/features/checklist/components/TaskRow.tsx`, `src/features/checklist/components/CountStepper.tsx`, `src/features/checklist/components/DurationInput.tsx`, `src/features/checklist/components/YesterdaySection.tsx`
- **Acceptance:**
  - A checkbox ticks in one tap; COUNT has plus, minus and an exact value; DURATION has +15, +30, +60 and an exact total; all update instantly.
  - The Yesterday section edits yesterday's tasks; the header shows “N of M done”; every control is at least 44 by 44 px.
- **Verification:**
  - `npm run build` · `npm run lint`
  - manual on a phone (M-07): tick a checkbox, add a count, add 30 minutes; each responds immediately; DevTools offline then tap: change is reversed with a retry message (M-06).

### M3 — Heatmap

*Exit criteria:* The seeded account shows 52 or 13 weeks on Today with the correct states, one Tab stop and a working day list.

#### T-041 · Heatmap helpers: level, week layout and range

- **Type:** pure, enabler · **Risk:** low · **Size:** about 300 lines · **Status:** todo
- **Serves:** FR-HM-01, FR-HM-02, BR-LEVEL
- **Depends on:** T-005
- **Touches:** Surfaces: none. Models: none.
- **Files:** `src/core/level.ts`, `src/core/time/iso-week.ts` (range), `tests/core/level.test.ts`, `tests/core/time/heatmap-range.test.ts`
- **Acceptance:**
  - `level(done, scheduled)` uses integer arithmetic only: 1 of 4 is level 1, 1 of 3 level 2, 3 of 4 level 3, 4 of 5 level 4, 0 level 0.
  - `heatmapRange(viewerToday, weeks)` returns Monday-start columns ending at the current week, with 52, 13 and 8 week variants.
- **Verification:**
  - `npm run test -- core/level core/time/heatmap-range` · `npm run test:tz`

#### T-042 · core/schedule/day-state

- **Type:** pure, enabler · **Risk:** med · **Size:** about 380 lines · **Status:** todo
- **Serves:** FR-HM-02, FR-VIS-03, FR-VIS-07, FR-VIS-04, FR-VIS-05
- **Depends on:** T-018, T-027, T-041
- **Touches:** Surfaces: none. Models: none.
- **Files:** `src/core/schedule/day-state.ts`, `tests/core/schedule/day-state.test.ts`
- **Acceptance:**
  - `dayState(scope, date, memberToday)` returns L0 to L4, REST, UNTRACKED, TODAY_PENDING or FUTURE for personal and group scope, using the targets visible in that scope **on that date**.
  - Cases: the same member-day in personal and group scope; sharing not retroactive; unshare keeping the past; archived days untracked; REST versus UNTRACKED; today pending versus a missed past day.
- **Verification:**
  - `npm run test -- core/schedule/day-state` · `npm run test:tz`

#### T-043 · Development seed script

- **Type:** db · **Risk:** low · **Size:** about 380 lines · **Status:** todo
- **Serves:** FR-HM-05, FR-ROL-01, FR-STK-01
- **Depends on:** T-019, T-022, T-032
- **Touches:** Surfaces: none. Models: all (dev data).
- **Files:** `prisma/seed.ts`, `package.json` (`db:seed`)
- **Acceptance:**
  - Seeds a group of six members in different zones (Asia/Kolkata, America/Los_Angeles, Pacific/Kiritimati, Europe/London, Australia/Lord_Howe, UTC) with 90 days of varied history, private and shared targets, an unshare in the middle of history, a removed member, a tombstoned user and a second, ownerless group.
  - It is idempotent and refuses to run unless `ALLOW_SEED=1` and the URL is not the production project.
- **Verification:**
  - `npm run db:seed` twice (second run changes nothing) · `npm run typecheck`
  - manual: Neon console shows the six users; later tasks use them for manual checks. If Node's built-in TypeScript stripping cannot run the seed, convert it to `.mjs` and say so.

#### T-044 · getPersonalHeatmap (L-04)

- **Type:** db · **Risk:** med · **Size:** about 260 lines · **Status:** todo
- **Serves:** FR-HM-01, FR-HM-02
- **Depends on:** T-036, T-041, T-042
- **Touches:** Surfaces: L-04. Models: targets, target_revisions, target_inactive_periods, entries.
- **Files:** `src/features/heatmap/server/queries.ts`, `tests/integration/personal-heatmap.test.ts`
- **Acceptance:**
  - Returns 52 weeks of `DayCell`s for the viewer in personal scope (private targets included) from one entries query, with the member's own effective today as the edge.
  - Matches the seeded history for known days (counts, REST and UNTRACKED days).
- **Verification:**
  - `npm run test:db -- personal-heatmap`

#### T-045 · Heatmap SVG component

- **Type:** ui · **Risk:** med · **Size:** about 360 lines · **Status:** todo
- **Serves:** FR-HM-01, FR-HM-02, FR-HM-04
- **Depends on:** T-014, T-044
- **Touches:** Surfaces: L-04. Models: none.
- **Files:** `src/features/heatmap/components/Heatmap.tsx`, `src/features/heatmap/components/legend.tsx`, `src/app/(app)/today/page.tsx`
- **Acceptance:**
  - Weeks start on Monday; month labels on top; Mon, Wed, Fri on the side; the current week is the last column with later days blank.
  - Levels use the theme tokens; missed is a filled grey, nothing scheduled a dotted outline, not tracked a faint empty cell, today a ring (never colour alone); renders on the Today page in its own Suspense boundary.
- **Verification:**
  - `npm run build` · `npm run lint`
  - manual (M-08): seeded account in light and dark; the two empty states look different from a missed day.

#### T-046 · Heatmap accessibility: summary, day-by-day list and tooltip

- **Type:** ui · **Risk:** med · **Size:** about 380 lines · **Status:** todo
- **Serves:** FR-HM-03, FR-HM-04, NFR-A11Y-01, NFR-A11Y-02
- **Depends on:** T-045
- **Touches:** Surfaces: L-04. Models: none.
- **Files:** `src/features/heatmap/summary.ts`, `src/features/heatmap/components/DayList.tsx`, `src/features/heatmap/components/Tooltip.tsx`, `tests/features/heatmap/summary.test.ts`
- **Acceptance:**
  - The grid is `role="img"` with a generated summary; individual cells are not focusable; the figure has **one** Tab stop, a “Day-by-day details” button that reveals a table grouped by month (date, X of Y tasks, percentage, state).
  - Hover and tap show the tooltip with the same four facts. The summary text and the month grouping are pure functions with tests.
- **Verification:**
  - `npm run test -- features/heatmap/summary` · `npm run build`
  - manual (M-08): Tab through the Today page: the heatmap takes one stop; Enter opens the list; a screen reader announces the summary once.

#### T-047 · Responsive range and Today integration

- **Type:** ui · **Risk:** low · **Size:** about 200 lines · **Status:** todo
- **Serves:** FR-HM-01, NFR-UX-01
- **Depends on:** T-046
- **Touches:** Surfaces: L-04. Models: none.
- **Files:** `src/features/heatmap/components/useWideScreen.ts`, `src/features/heatmap/components/Heatmap.tsx`
- **Acceptance:**
  - Below 640 px the heatmap shows 13 weeks, above it 52; the server markup is the narrow variant and upgrades after hydration; the heatmap scrolls inside itself, never the page.
- **Verification:**
  - `npm run build`
  - manual at 360 px and 1280 px: the heatmap shows 13 and 52 weeks; no horizontal page scroll.

### M4 — Roll call and streaks

*Exit criteria:* The seeded group renders correctly; two browsers show the same progress within a poll; polling stops after 3 minutes; TP-01 and TP-02 pass.

#### T-048 · core/streak

- **Type:** pure, enabler · **Risk:** low · **Size:** about 280 lines · **Status:** todo
- **Serves:** FR-STK-01, FR-STK-02, FR-STK-03, BR-STREAK
- **Depends on:** T-042
- **Touches:** Surfaces: none. Models: none.
- **Files:** `src/core/streak.ts`, `tests/core/streak.test.ts`
- **Acceptance:**
  - `toStreakDay`, `fold` and `fullStreak`: ACTIVE adds one, MISSED resets, REST and UNTRACKED bridge, TODAY_PENDING neither adds nor breaks, a streak starts only on an active day.
  - The PRD worked example (Mon A, Tue A, Wed rest, Thu A, Fri missed, Sat A, Sun A) gives current 2 and longest 3; yesterday missed then completed restores the run.
- **Verification:**
  - `npm run test -- core/streak` · `npm run test:tz`

#### T-049 · core/rollcall

- **Type:** pure, enabler · **Risk:** low · **Size:** about 260 lines · **Status:** todo
- **Serves:** FR-ROL-01, FR-ROL-02, BR-ROLL
- **Depends on:** T-006, T-042
- **Touches:** Surfaces: none. Models: none.
- **Files:** `src/core/rollcall.ts`, `tests/core/rollcall.test.ts`
- **Acceptance:**
  - `classify` returns ACTIVE, NOT_YET, SLACKING, REST_DAY or NO_TARGETS_SHARED from a member's own date, shared tasks and clock; partial progress counts as active.
  - Cases: 17:59 versus 18:00 in the member's zone, a BR-DAY-5 hold (never SLACKING), no shared targets, shared targets none scheduled, section order.
- **Verification:**
  - `npm run test -- core/rollcall` · `npm run test:tz`

#### T-050 · Streak prefix cache, groupStreak and verifyStreakCache

- **Type:** db · **Risk:** high · **Size:** about 380 lines · **Status:** todo
- **Serves:** FR-STK-01, FR-STK-02, FR-STK-03
- **Depends on:** T-008, T-018, T-042, T-048
- **Touches:** Surfaces: R-02, L-07 (callers). Models: streak_prefixes, entries, share_intervals.
- **Files:** `src/features/heatmap/server/streak-cache.ts`, `tests/integration/streak-cache.test.ts`
- **Acceptance:**
  - `groupStreak` loads or extends the closed-days prefix and folds yesterday and today; `fullGroupStreak` is the cache-free oracle; `verifyStreakCache` returns any difference; `invalidateStreakPrefixes` and `ALGO_VERSION` exist.
  - Basic cases: first call creates the prefix, the next reuses it, a later day extends it by one day, an algorithm bump recomputes, a direct write of an old entry is detected by `verifyStreakCache`. The full equivalence suite is TP-01 in the next two tasks.
- **Verification:**
  - `npm run test:db -- streak-cache`

#### T-051 · TP-01 equivalence suite (1 of 2): harness and scenarios 1 to 3

- **Type:** db · **Risk:** high · **Size:** about 390 lines · **Status:** todo
- **Serves:** FR-STK-01, FR-STK-02
- **Depends on:** T-050
- **Touches:** Surfaces: none (tests). Models: streak_prefixes.
- **Files:** `tests/helpers/prng.ts`, `tests/helpers/tp01.ts`, `tests/streak/tp01-rollover-unshare-tz.test.ts` (in memory, unit project), `tests/integration/tp01-fixed.test.ts` (real database)
- **Acceptance:**
  - A hand-written seeded random generator and an operation generator (record progress today or yesterday, share, unshare, archive, change a rule, advance the clock, change time zone, remove, rejoin).
  - After every operation the cached value must equal `fullGroupStreak` for every member. Randomised runs use an in-memory implementation of the store interface (at least 1,000 sequences); fixed regression cases run on the real database.
  - Scenarios covered here: day rollover (one day and several), unshare and re-share, time-zone change forward and backward including a hold.
- **Verification:**
  - `npm run test -- tests/streak/tp01-rollover` · `npm run test:db -- tp01-fixed`

#### T-052 · TP-01 equivalence suite (2 of 2): scenarios 4 to 7

- **Type:** db · **Risk:** med · **Size:** about 330 lines · **Status:** todo
- **Serves:** FR-STK-02, FR-STK-01
- **Depends on:** T-051
- **Touches:** Surfaces: none (tests). Models: streak_prefixes, entries.
- **Files:** `tests/streak/tp01-backfill-rejoin-rules.test.ts` (in memory), `tests/integration/tp01-fixed.test.ts` (extended)
- **Acceptance:**
  - Backfilled check-offs: yesterday after the prefix exists; an edit just before and after local midnight; older entries written directly by a script, with and without `invalidateStreakPrefixes`, where the cache **must** disagree until invalidated and `verifyStreakCache` must report it.
  - Removal and rejoin; rule changes effective tomorrow and archive; an `ALGO_VERSION` bump.
- **Verification:**
  - `npm run test -- tests/streak` · `npm run test:db -- tp01-fixed`

#### T-053 · Personal streak on Today

- **Type:** ui · **Risk:** low · **Size:** about 220 lines · **Status:** todo
- **Serves:** FR-CHK-11, FR-STK-01, FR-STK-03, FR-VIS-07
- **Depends on:** T-036, T-040, T-048
- **Touches:** Surfaces: R-01, L-03. Models: entries, targets (history).
- **Files:** `src/features/checklist/server/queries.ts`, `src/features/checklist/components/StreakBadge.tsx`, `tests/integration/today.test.ts`
- **Acceptance:**
  - `getTodayData` fills `streak.personal` from `fullStreak` over the member's history (all targets); the badge is labelled “Personal streak”.
  - Today pending neither adds nor breaks; completing yesterday restores the number immediately.
- **Verification:**
  - `npm run test:db -- today`
  - manual: seeded account shows the expected personal streak; tick yesterday's missed task and watch the number change.

#### T-054 · computeRollCall query (cross-timezone)

- **Type:** db · **Risk:** high · **Size:** about 380 lines · **Status:** todo
- **Serves:** FR-ROL-01, FR-ROL-02, FR-ROL-03, FR-VIS-03
- **Depends on:** T-006, T-008, T-018, T-020, T-049
- **Touches:** Surfaces: R-02, L-07 (caller). Models: memberships, users, targets, target_revisions, target_inactive_periods, share_intervals, entries.
- **Files:** `src/features/rollcall/server/queries.ts`, `tests/integration/rollcall.test.ts`
- **Acceptance:**
  - Each member is judged on their own effective today with one bounded window query (the dates that can be anyone's yesterday or today).
  - Members in Kiritimati, Kolkata, London and Los Angeles at the same instant get the right states; a hold shows NOT_YET; removed and tombstoned members, and private targets, never appear or count.
- **Verification:**
  - `npm run test:db -- rollcall`

#### T-055 · getGroupOverview (L-07) and the default order

- **Type:** db · **Risk:** med · **Size:** about 390 lines · **Status:** todo
- **Serves:** FR-HM-05, FR-HM-06, FR-VIS-03, FR-VIS-07, FR-STK-01
- **Depends on:** T-042, T-044, T-050, T-054
- **Touches:** Surfaces: L-07. Models: memberships, users, share_intervals, entries, streak_prefixes.
- **Files:** `src/features/heatmap/server/overview.ts`, `src/features/heatmap/order.ts`, `tests/features/heatmap/order.test.ts`, `tests/integration/overview.test.ts`
- **Acceptance:**
  - Returns the active members with role, roll-call state, group streak and 52 weeks of group-scope cells (dates after a member's own today are FUTURE), the poll hint from `PULSE_POLL_MODE`, and an empty feed until the feed tasks.
  - `compareMembers`: group streak descending, then last-7-day percentage descending, then name; a member with nothing scheduled in those seven days sorts after everyone who has a percentage (PRD rev 3).
- **Verification:**
  - `npm run test -- features/heatmap/order` · `npm run test:db -- overview`

#### T-056 · Group page UI: roll call and comparison rows

- **Type:** ui · **Risk:** med · **Size:** about 390 lines · **Status:** todo
- **Serves:** FR-ROL-01, FR-HM-05, FR-VIS-07
- **Depends on:** T-016, T-046, T-055
- **Touches:** Surfaces: L-07. Models: none.
- **Files:** `src/app/(app)/groups/[groupId]/page.tsx`, `src/features/rollcall/components/RollCall.tsx`, `src/features/heatmap/components/ComparisonRows.tsx`
- **Acceptance:**
  - Roll call sections in the order Active, Not yet, Slacking, Rest day, No targets shared with counts such as “2 / 3”; comparison rows with avatar (initials fallback), name, a “Group streak” label and a strip (52 weeks wide, 8 weeks narrow).
  - Each row has its own text summary and one details button; a group the user is not in shows the uniform not-found page.
- **Verification:**
  - `npm run build` · `npm run lint`
  - manual (M-02): seeded group: sections and rows look right in light, dark and at 360 px; open /groups/<someone else's group id>: uniform not-found.

#### T-057 · Pulse route (R-02) and the polling hint

- **Type:** db · **Risk:** med · **Size:** about 380 lines · **Status:** todo
- **Serves:** FR-ROL-03, FR-ROL-01, FR-STK-03, FR-HM-05, FR-GRP-11
- **Depends on:** T-050, T-054, T-055
- **Touches:** Surfaces: R-02. Models: as the roll call and streak cache; feed_cards later.
- **Files:** `src/app/api/groups/[groupId]/pulse/route.ts`, `src/features/rollcall/server/pulse.ts`, `tests/integration/pulse.test.ts`
- **Acceptance:**
  - Returns the exact `Pulse` shape of API_SPEC with `recent` cells, group streaks and an empty feed page until the feed tasks.
  - `PULSE_POLL_MODE` L0, L1, L2 gives `{20000, 3}`, `{20000, 1}`, `{null, 0}`; a non-member and an invalid id get the same 404; headers are `private, no-store`.
  - The handler logs its query count and response size so the budget task can measure them.
- **Verification:**
  - `npm run test:db -- pulse` · `curl -i http://localhost:3000/api/groups/<id>/pulse` (401 signed out)

#### T-058 · Poll controller: idle-stop state machine and hook

- **Type:** pure · **Risk:** med · **Size:** about 330 lines · **Status:** todo
- **Serves:** FR-ROL-03, NFR-POLL-01, NFR-POLL-02
- **Depends on:** T-057
- **Touches:** Surfaces: R-02. Models: none.
- **Files:** `src/features/rollcall/poll-machine.ts`, `src/features/rollcall/use-pulse.ts`, `tests/features/rollcall/poll-machine.test.ts`
- **Acceptance:**
  - A pure state machine plus a thin hook: ACTIVE polls every `max(20000, hint)` ms; one timeout at the last interaction plus the idle stop flips to STOPPED; interaction or becoming visible resumes at once (refetch if older than 10 s); a hidden tab makes no request; a `null` hint never polls.
  - Unit tests with fake timers simulate one hour of continuous interaction and assert at most 180 requests, the 3-minute and 1-minute stops, hidden tabs, and resume.
- **Verification:**
  - `npm run test -- features/rollcall/poll-machine`

#### T-059 · Wire the pulse into the group page

- **Type:** ui · **Risk:** med · **Size:** about 300 lines · **Status:** todo
- **Serves:** FR-ROL-03, FR-STK-02, FR-HM-05
- **Depends on:** T-056, T-058
- **Touches:** Surfaces: R-02, L-07. Models: none.
- **Files:** `src/features/rollcall/apply-pulse.ts`, `tests/features/rollcall/apply-pulse.test.ts`, `src/features/rollcall/components/GroupLive.tsx`
- **Acceptance:**
  - `applyPulse` patches each member's state, counts, streak and last two cells, never reorders rows, and asks for a page refresh when the set of members differs from the overview.
  - The group page polls only while visible and interacting.
- **Verification:**
  - `npm run test -- features/rollcall/apply-pulse`
  - manual (M-09, M-10 part): two browsers, two accounts: ticking a task in one moves that member to Active in the other within about 40 s; leave one tab idle: requests stop after 3 minutes (DevTools, Network); switching tabs makes none; returning resumes at once.

### M5 — Feed and reactions

*Exit criteria:* Cards and reactions work across two accounts; TP-03 and TP-04 pass.

#### T-060 · syncFeedCards and its wiring into A-12 and A-11

- **Type:** db · **Risk:** med · **Size:** about 390 lines · **Status:** todo
- **Serves:** FR-FED-01, FR-FED-04, FR-VIS-05
- **Depends on:** T-032, T-037
- **Touches:** Surfaces: A-12, A-11. Models: feed_cards, reactions (cascade), share_intervals, entries.
- **Files:** `src/features/feed/server/sync.ts`, `src/features/checklist/server/actions.ts`, `src/features/sharing/server/actions.ts`, `tests/integration/feed-sync.test.ts`
- **Acceptance:**
  - First progress of a member's day creates one card per group that sees the target; later progress updates the same card's `last_activity_at`; removing all shared progress deletes the card and its reactions.
  - Newly shared groups get today's card; unsharing keeps old cards; yesterday's edits touch yesterday's card; a removed member gets no card; a unique-violation race is retried once.
- **Verification:**
  - `npm run test:db -- feed-sync`

#### T-061 · Feed read model and R-03

- **Type:** db · **Risk:** med · **Size:** about 390 lines · **Status:** todo
- **Serves:** FR-FED-03, FR-FED-04, FR-REA-03, FR-FED-01
- **Depends on:** T-020, T-060
- **Touches:** Surfaces: R-03. Models: feed_cards, reactions, entries, share_intervals, memberships.
- **Files:** `src/features/feed/server/queries.ts`, `src/features/feed/cursor.ts`, `src/app/api/groups/[groupId]/feed/route.ts`, `tests/integration/feed-read.test.ts`
- **Acceptance:**
  - Keyset pages of 20 ordered by `(last_activity_at, id)` descending with an opaque cursor; card bodies list only tasks shared with that group on that date that have progress, with “done of scheduled”.
  - No duplicate or skipped card while cards update between pages; cards and reactions of removed members are hidden; an old card keeps its content after an unshare.
- **Verification:**
  - `npm run test:db -- feed-read`

#### T-062 · toggleReaction (A-13)

- **Type:** db · **Risk:** low · **Size:** about 220 lines · **Status:** todo
- **Serves:** FR-REA-01, FR-REA-03
- **Depends on:** T-061
- **Touches:** Surfaces: A-13. Models: reactions, feed_cards.
- **Files:** `src/features/reactions/server/actions.ts`, `tests/integration/reactions.test.ts`
- **Acceptance:**
  - Toggles one of the five emojis on a card of a group the caller belongs to; the card's author must be an active member (else `NOT_FOUND`); reacting to your own card is allowed; returns the new counts with `mine`.
- **Verification:**
  - `npm run test:db -- reactions`

#### T-063 · Feed UI

- **Type:** ui · **Risk:** low · **Size:** about 300 lines · **Status:** todo
- **Serves:** FR-FED-01, FR-FED-03
- **Depends on:** T-056, T-061
- **Touches:** Surfaces: R-03, L-07. Models: none.
- **Files:** `src/features/feed/components/FeedList.tsx`, `src/features/feed/components/FeedCard.tsx`, `src/features/feed/use-feed.ts`
- **Acceptance:**
  - The group page lists cards (avatar, name, date, tasks with progress, “3 of 5”) from the overview, with a Load more button driven by an infinite query on R-03 and an empty state.
- **Verification:**
  - `npm run build` · `npm run lint`
  - manual (M-10): seeded group shows cards; load more reaches older days.

#### T-064 · Reactions UI and the live feed in the pulse

- **Type:** ui · **Risk:** med · **Size:** about 330 lines · **Status:** todo
- **Serves:** FR-REA-01, FR-FED-05, FR-FED-01
- **Depends on:** T-059, T-062, T-063
- **Touches:** Surfaces: A-13, R-02. Models: none.
- **Files:** `src/features/reactions/components/ReactionBar.tsx`, `src/features/rollcall/apply-pulse.ts`, `src/features/rollcall/server/pulse.ts`
- **Acceptance:**
  - Five emoji buttons with counts and a pressed state; the toggle is optimistic with rollback.
  - The pulse now returns the real feed page 1; `applyPulse` merges it by card id and keeps older loaded pages.
- **Verification:**
  - `npm run test -- features/rollcall/apply-pulse` · `npm run build`
  - manual (M-10): account A ticks a task, account B sees the card appear or update within one poll; B reacts, A sees the count.

#### T-065 · TP-04 private-target leak suite

- **Type:** db · **Risk:** med · **Size:** about 360 lines · **Status:** todo
- **Serves:** FR-VIS-02, FR-VIS-03, NFR-SEC-02
- **Depends on:** T-055, T-057, T-061
- **Touches:** Surfaces: L-07, R-02, R-03, L-04. Models: all read models.
- **Files:** `tests/integration/leak.test.ts`
- **Acceptance:**
  - One fixture gives a member private targets and targets shared only with group A. For group B the suite asserts that no overview cell, roll-call state or count, streak, pulse field, feed card, task name, total or ordering changes when the private or other-group data changes.
- **Verification:**
  - `npm run test:db -- leak`

#### T-066 · TP-03 deleted-user suite

- **Type:** db · **Risk:** low · **Size:** about 300 lines · **Status:** todo
- **Serves:** FR-GRP-11, FR-ACC-02, NFR-REL-02
- **Depends on:** T-019, T-025, T-057, T-061
- **Touches:** Surfaces: L-07, L-08, R-02, R-03, A-13, R-04. Models: users, memberships, feed_cards, reactions.
- **Files:** `tests/integration/deleted-user.test.ts`
- **Acceptance:**
  - With a deleted member and a deleted owner in the fixture, every group surface returns without error and never shows them; their cards and reactions are hidden; reacting to their card gives `NOT_FOUND`; their slot is free for joining; the group is ownerless and `ownerless` is true.
- **Verification:**
  - `npm run test:db -- deleted-user`
  - manual (M-11): delete a test user in the Clerk dashboard and reload the group pages of a second account.

### M6 — PWA and launch prep

*Exit criteria:* Pulse installs on Android and iOS, is measured within budget, is deployed to production, and every launch criterion of PRD §7.1 is met.

#### T-067 · Manifest, icons and the install spike S4 (M-01, TP-08)

- **Type:** infra · **Risk:** med · **Size:** about 320 lines · **Status:** todo
- **Serves:** S4, FR-PWA-01, FR-PWA-03, FR-PWA-05
- **Depends on:** T-007, T-012, T-014
- **Touches:** Surfaces: M-01, P-01. Models: none.
- **Files:** `src/app/manifest.ts`, `src/app/icons/[name]/route.tsx`, `src/app/apple-icon.tsx`, `src/app/layout.tsx` (viewport theme colours), `tests/app/manifest.test.ts`, `docs/spikes/S4.md` (new, at most 40 lines)
- **Acceptance:**
  - The manifest has name and short name Pulse, `start_url` `/today`, `display` `standalone`, and 192, 512 and maskable 512 icons; the Apple touch icon is 180×180; light and dark theme colours come from the viewport export.
  - Icons are drawn with `next/og` (no image library); a unit test checks the manifest object; manifest and icons load while signed out.
  - Spike S4, on the HTTPS preview URL: `docs/spikes/S4.md` records, for Android Chrome and iOS Safari, whether the install option appears with no service worker, that the app opens standalone on /today, stays signed in, and that every v1.0 flow also works in a plain browser tab. If Android needs a service worker or fetch handler: **stop and ask you** (new capability; FR-PWA-04 is v1.1) and amend TAD §1.4 and §8.5. If iOS opens in a tab or loses the session: amend FR-PWA-03 in a PRD note and TAD §8.5.
- **Verification:**
  - `npm run test -- app/manifest` · `npm run build` · `curl -i http://localhost:3000/manifest.webmanifest` (200, JSON)
  - manual: Chrome DevTools, Application, Manifest: no errors, icons render.
  - manual (M-12, M-13): follow the steps in TEST_PLAN M-12 and M-13 on a real Android phone and a real iPhone using the HTTPS preview URL (Android: browser menu, Install app; iOS: Share, Add to Home Screen; launch from the icon; sign in; tick a task; kill and reopen).

#### T-068 · Security headers, CSP and audit

- **Type:** infra · **Risk:** med · **Size:** about 200 lines · **Status:** todo
- **Serves:** FR-ACC-01
- **Depends on:** T-007, T-012
- **Touches:** Surfaces: P-01. Models: none.
- **Files:** `next.config.ts`, `.github/workflows/ci.yml`
- **Acceptance:**
  - `nosniff`, `Referrer-Policy`, `Permissions-Policy`, `X-Frame-Options`, and a Content-Security-Policy that allows Clerk's documented origins, first in report-only mode then enforced.
  - Sign-in, sign-up, the user menu and the webhook still work with the policy enforced; `npm audit --omit=dev` is in CI.
- **Verification:**
  - `curl -I https://<preview-url>/` (headers present) · `npm audit --omit=dev`
  - manual: preview URL: sign up with Google, open the user menu, console shows no CSP violations.

#### T-069 · Accessibility and responsive pass

- **Type:** ui · **Risk:** med · **Size:** about 300 lines · **Status:** todo
- **Serves:** FR-HM-04, NFR-A11Y-01, NFR-A11Y-02, NFR-A11Y-03, NFR-UX-01
- **Depends on:** T-026, T-035, T-047, T-064
- **Touches:** Surfaces: all UI. Models: none.
- **Files:** Whatever the pass finds, bounded to about 300 changed lines; the findings are listed in the pull request
- **Acceptance:**
  - Every control is at least 44 by 44 px; focus is visible; the keyboard path reaches every action on Today, Targets, Group and Members; the heatmap is one stop with its list; contrast meets WCAG 2.1 AA in both themes; no horizontal page scroll at 360 px; reduced motion is respected.
  - A short screen-reader run (NVDA on Windows, VoiceOver on iPhone) reads the heatmap summary once and the roll call sensibly.
- **Verification:**
  - `npm run build` · `npm run lint`
  - manual (M-07, M-08): the full checklist; any failure is fixed or filed as a task.

#### T-070 · Spike S3: poll handler measurement (TP-06)

- **Type:** spike · **Risk:** med · **Size:** about 200 lines · **Status:** todo
- **Serves:** S3, FR-ROL-03, NFR-POLL-03
- **Depends on:** T-043, T-064
- **Touches:** Surfaces: R-02. Models: as the pulse.
- **Files:** `docs/spikes/S3.md` (new), `docs/TAD.md` §12.3 (measured numbers), `tests/integration/pulse-budget.test.ts`
- **Question:**
  - Is one poll within 40 ms of CPU (p95) and 20 KB of database reads for 15 members with 90 days of history?
- **How to answer:**
  - Seed 15 members in the dev project; call the handler 200 times in-process measuring `process.cpuUsage()` and `performance.now()`; count rows and approximate bytes from the Prisma query log; repeat once against a Vercel preview and compare invocation duration in the dashboard.
- **If the answer is surprising:**
  - CPU above 40 ms: optimise the shared-structure query first, then lift the structure cache (TAD §12.4 levers); if still above, raise it with you before changing NFR-POLL-04 or the ladder.
  - Bytes above 20 KB: trim selected columns; update TAD §12.3 transfer figures.
- **Acceptance:**
  - `docs/spikes/S3.md` reports CPU time per poll (p50, p95), query count and database bytes for a 15-member, 150-target, 90-day group, replacing the assumptions in TAD §12.3.
  - A test fails if the poll exceeds 40 ms CPU (p95) or 20 KB of database reads on the seeded group.
- **Verification:**
  - `npm run test:db -- pulse-budget`

#### T-071 · Production environment, first production deploy and the Neon compute check (spike S6)

- **Type:** infra · **Risk:** med · **Size:** about 120 lines · **Status:** todo
- **Serves:** S6, FR-ACC-01, FR-ACC-02, NFR-COST-01
- **Depends on:** T-007, T-017, T-068
- **Touches:** Surfaces: R-04, P-01. Models: all (production database).
- **Files:** Vercel and Neon and Clerk settings (outside the repo); `.env.example` if a name changed; `docs/spikes/S6.md` (new, at most 30 lines); `docs/TAD.md` §12.3 (corrected figures, only if the minimum compute is not 0.25 CU)
- **Acceptance:**
  - A separate `pulse-prod` Neon project in the **same region as `pulse-dev` (`aws-ap-southeast-1`, Singapore)** (so tests never spend its 100 compute hours), the Vercel Production function region `sin1`, a Clerk **production** instance with the real domain and Google OAuth credentials, the webhook endpoint registered with its signing secret, and all variables from TAD §11 set in Vercel Production.
  - The first production build runs `prisma migrate deploy`; a Neon snapshot is taken first; Node 24 and the function region match the Neon region.
  - Spike S6: the minimum autoscaling size of the `pulse-prod` compute is read from the console and recorded in `docs/spikes/S6.md`; the 0.25 CU assumption is confirmed or the TAD §12.3 figures are rescaled (minimum above 0.25 CU: rescale §12.3 and ADR-0001; if the expected case passes 60%, move to L1 at once; if it passes 80%, discuss reducing visits or ending free-tier-only). The poll-sized round-trip measurement of the walking skeleton is repeated against production and recorded.
- **Verification:**
  - Production deploy from `main` · `curl -i https://<prod-domain>/manifest.webmanifest`
  - manual (M-17): sign up on production, see the user in the production `users` table, edit the name in Clerk and watch it sync.
  - manual: Neon console, Settings, Compute: note the autoscaling range; Vercel dashboard: function region equals the Neon region.

#### T-072 · Launch rehearsal and friend onboarding

- **Type:** ui · **Risk:** med · **Size:** about 100 lines · **Status:** todo
- **Serves:** FR-PWA-05, FR-PWA-01, FR-GRP-03
- **Depends on:** T-052, T-065, T-066, T-067, T-069, T-070, T-071
- **Touches:** Surfaces: all. Models: all (production).
- **Files:** Fixes only, bounded to about 100 changed lines
- **Acceptance:**
  - The whole manual QA checklist of TEST_PLAN §6 passes on production with two or three real accounts; every item of the PRD §7.1 launch criteria is ticked (all v1.0 scenarios pass; NFR-QUAL-01 scenarios pass; the polling cap verified in use; installs on Android and on the iOS Home Screen; the real group has used it for a full day).
  - The invite link is sent to the friend group; the weekly Neon and Vercel usage check is in the calendar. After week 1 the real Neon compute hours and visits per day are added to `docs/spikes/S6.md`, with the thresholds that trigger ladder level L1 or L2 (compare `hours used ÷ days elapsed` with the 3.3 CU-hours per day pace).
- **Verification:**
  - `npm run typecheck && npm run lint && npm run test && npm run test:tz && npm run test:db` (all green) · the manual checklist
  - manual: TEST_PLAN M-01 to M-17 on production, ticked off in the pull request.

---

## 4. v1.1 backlog (titles and requirement IDs only)

Sliced into tasks when v1.0 ships. Order follows PRD §7.2.

| Milestone | Title | Requirements |
|---|---|---|
| V1.1-A | Group management and guardrails | FR-GRP-04, FR-GRP-06, FR-GRP-08, FR-GRP-09, FR-GRP-10, FR-GRP-12, FR-GRP-13, FR-ACC-07, FR-TGT-05 |
| V1.1-B | Nudges and in-app notifications | FR-NUD-01, FR-NUD-02, FR-NOT-01, FR-NOT-02, FR-NOT-03 |
| V1.1-C | Pause a target | FR-PAU-01, FR-PAU-02, FR-PAU-03 |
| V1.1-D | Weekly quota target | FR-QTA-01, FR-QTA-02 |
| V1.1-E | Shared group goals | FR-SGL-01, FR-SGL-02, FR-SGL-03, FR-SGL-04, FR-SGL-05 |
| V1.1-F | Weekly recap | FR-REC-01, FR-REC-02 |
| V1.1-G | Feed, heatmap, PWA and target polish | FR-FED-02, FR-STK-04, FR-STK-05, FR-REA-02, FR-HM-07, FR-HM-08, FR-PWA-02, FR-PWA-04, FR-TGT-09, FR-TGT-10 |

---

## 5. Coverage check

Generated from the task data and verified by a script against PRD rev 3.

- **Tasks:** 72; **v1.0 requirements:** 58; **v1.1 requirements in the backlog:** 36.
- **Every v1.0 FR maps to at least one task:** yes (matrix below).
- **Every task maps to at least one FR or spike:** yes.
- **Spikes:** S1 is T-001, S2 is T-003, S3 is T-070, S4 is T-067, S5 is T-002, S6 is T-071.
- **Dependencies:** every task depends only on earlier tasks (checked); the first two tasks, T-001 and T-002, are the spikes S1 and S5, as requested.
- **Sizes:** the largest estimates are T-018 (390), T-040 (390), T-051 (390), T-055 (390), T-056 (390); none exceeds 400.
- **Pure logic before UI:** the pure day, scheduling, share-interval, level, streak and roll-call tasks T-005, T-006, T-018, T-027, T-041, T-042, T-048, T-049 all come before the screens that use them.
- **Test-plan items:** TP-01 to TP-09 each have an owning task (TEST_PLAN §3).

| FR | Title | Tasks |
|---|---|---|
| FR-ACC-01 | Sign up and sign in | T-012, T-016, T-023, T-068, T-071 |
| FR-ACC-02 | Immediately usable after sign-up | T-002, T-007, T-013, T-017, T-019, T-066, T-071 |
| FR-ACC-03 | First-run profile | T-010, T-011, T-015 |
| FR-ACC-04 | Edit profile | T-010, T-017, T-019, T-026 |
| FR-ACC-05 | Timezone change | T-006, T-026 |
| FR-ACC-06 | Sign out | T-012, T-016 |
| FR-GRP-01 | Create a group | T-010, T-021 |
| FR-GRP-02 | Invite code and link | T-010, T-021, T-025 |
| FR-GRP-03 | Join a group | T-007, T-008, T-010, T-022, T-023, T-072 |
| FR-GRP-05 | Group switcher | T-024 |
| FR-GRP-07 | Member list and removal | T-018, T-025 |
| FR-GRP-11 | Membership is verified on every action | T-016, T-020, T-022, T-025, T-057, T-066 |
| FR-TGT-01 | Create a target from a template or from scratch | T-009, T-028, T-029, T-035 |
| FR-TGT-02 | Tracking modes and goals | T-009, T-010, T-028, T-029, T-034 |
| FR-TGT-03 | Frequency: daily or specific days | T-027, T-028, T-029, T-034 |
| FR-TGT-04 | Start date | T-029 |
| FR-TGT-06 | Edit a target | T-030, T-033, T-034 |
| FR-TGT-07 | Archive a target | T-031, T-033 |
| FR-TGT-08 | Delete a target with no history | T-031, T-033 |
| FR-VIS-01 | Choose which groups can see a target | T-028, T-029, T-032, T-033, T-035 |
| FR-VIS-02 | Unshared targets are private | T-018, T-065 |
| FR-VIS-03 | Group views count only shared targets | T-042, T-054, T-055, T-065 |
| FR-VIS-04 | Sharing is not retroactive | T-007, T-008, T-018, T-029, T-032, T-042 |
| FR-VIS-05 | Unsharing keeps the past | T-018, T-032, T-042, T-060 |
| FR-VIS-06 | Re-sharing | T-018, T-032 |
| FR-VIS-07 | Two streaks, clearly labelled | T-042, T-053, T-055, T-056 |
| FR-CHK-01 | Today's checklist | T-036, T-038 |
| FR-CHK-02 | Check off a checkbox task | T-037, T-039, T-040 |
| FR-CHK-03 | Count tasks | T-037, T-039, T-040 |
| FR-CHK-04 | Duration tasks | T-037, T-039, T-040 |
| FR-CHK-05 | Completion and over-achievement | T-027, T-037 |
| FR-CHK-06 | Edit window: today and yesterday | T-005, T-006, T-036, T-037, T-040 |
| FR-CHK-07 | Missed days need no action | T-027, T-036, T-038 |
| FR-CHK-08 | Reliable saving | T-037, T-039 |
| FR-CHK-09 | Same data on every device | T-037, T-039 |
| FR-CHK-10 | Empty and first-run states | T-036, T-038 |
| FR-CHK-11 | Daily summary | T-040, T-053 |
| FR-HM-01 | Personal heatmap layout | T-005, T-041, T-044, T-045, T-047 |
| FR-HM-02 | Cell states and levels | T-041, T-042, T-044, T-045 |
| FR-HM-03 | Tooltip | T-046 |
| FR-HM-04 | Accessibility and themes | T-014, T-045, T-046, T-069 |
| FR-HM-05 | Group comparison view | T-043, T-055, T-056, T-057, T-059 |
| FR-HM-06 | Default order | T-055 |
| FR-STK-01 | Current streak | T-043, T-048, T-050, T-051, T-052, T-053, T-055 |
| FR-STK-02 | Retroactive edits recalculate | T-048, T-050, T-051, T-052, T-059 |
| FR-STK-03 | Today never breaks a streak | T-048, T-050, T-053, T-057 |
| FR-ROL-01 | Roll call per group | T-043, T-049, T-054, T-056, T-057 |
| FR-ROL-02 | Cutoff | T-006, T-049, T-054 |
| FR-ROL-03 | Status changes as they act | T-054, T-057, T-058, T-059, T-070 |
| FR-FED-01 | Daily progress card | T-060, T-061, T-063, T-064 |
| FR-FED-03 | Order and paging | T-061, T-063 |
| FR-FED-04 | Feed respects sharing and membership | T-060, T-061 |
| FR-FED-05 | New activity appears without reloading | T-064 |
| FR-REA-01 | React to a card | T-062, T-064 |
| FR-REA-03 | Reactions follow their card | T-061, T-062 |
| FR-PWA-01 | Installable | T-067, T-072 |
| FR-PWA-03 | Installed behaviour | T-067 |
| FR-PWA-05 | Works without installing | T-067, T-072 |

