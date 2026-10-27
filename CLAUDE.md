# Pulse

Friend-group upskilling tracker (2–15 people): an honor-system daily checklist of personal targets, a heatmap, a group roll call with streaks, and a feed. Next.js 16 + Prisma 7 on Neon, Clerk auth, free tiers only.

## Status

Planning complete (BRIEF, PRD rev 3, TAD rev 2, ADRs 0001–0006, DB_SCHEMA, API_SPEC, TASKS, TEST_PLAN committed). No app code exists yet. **Next task: T-001** (spike S1). Do not start a task until the user says so.

## Doc map and priority

| Doc | Role |
|---|---|
| `docs/PRD.md` | **What** to build (FR/BR/NFR IDs) |
| `docs/adr/`, `docs/TAD.md` | **How** (architecture, conventions, pinned versions) |
| `docs/DB_SCHEMA.md`, `docs/API_SPEC.md` | **Contracts** (schema, constraints, actions, routes, Zod) |
| `docs/TASKS.md` | **Order** (72 tasks, M0–M6, status per task, setup checklist) |
| `docs/TEST_PLAN.md` | **Verification** (suites, TP items, milestone gates) |

**If code and docs conflict, or docs conflict with each other, stop and ask the user.** Do not pick a winner silently.

## Commands

Defined by T-003 (tooling) and T-007 (db). **None exist until those tasks are done**; check `package.json` before using one.

| Command | Purpose |
|---|---|
| `npm run typecheck` / `lint` / `build` / `dev` | tsc, eslint, next build, local server (T-003) |
| `npm run test` | Vitest `unit` project, no DB (T-003) |
| `npm run test:tz` | unit project once per time zone, 8 runs (T-003) |
| `npm run test:db` | Vitest `integration` project on the `pulse-dev` Neon project (T-008; needs `.env.test`) |
| `npm run db:dev` / `db:deploy` / `db:seed` | prisma migrate dev / deploy, dev seed (T-007+; seed needs `ALLOW_SEED=1`) |

Local Node is 22.23.2; Vercel runs Node 24. Pins are exact (TAD §2); never use `prisma@latest`.

## Conventions (TAD)

- **Folders:** `src/app` routing only; `src/features/<name>/{components,server,schema.ts,index.ts}`; `src/core` pure logic (no I/O, no Next, no Prisma, no process time zone); `src/lib` shared infra; `src/components/ui` shadcn. Import rules are ESLint-enforced (TAD §3): `app` imports only feature `index.ts`; features never import each other's internals; `core` imports nothing outside `core`; Prisma only from `server/`, `lib/auth/`, webhook; every `server/` and `lib/auth/` file starts with `import 'server-only'`.
- **Errors:** expected failures return `Result` (`ok`/`err`, codes in API_SPEC §1.2), never throw across the action boundary. Server Actions are wrapped in `defineAction` (Zod parse, `requireUser`, `INTERNAL` without leaking details).
- **Zod:** every action/route input is parsed with the shared primitives (`lib/primitives.ts`, API_SPEC §1.4); types are inferred from schemas.
- **Authorization:** every group read/write goes through `requireGroupAccess` (branded `GroupAccess`); target ownership through `requireTargetOwner`. Never query group data without it.
- **Dates:** calendar days are `LocalDate` (branded string), stored as Postgres `DATE`. Never use `new Date()` or the process time zone in domain code; time comes from `lib/clock`. The server resolves "today" via `effectiveToday` (respects `users.date_floor`). Scheduled days are derived on read; no per-day task rows.
- **Tests:** pure logic first, then UI. Time is injected in every test. Vitest only.

## Rules

1. **One task at a time.** Work only on the task the user names.
2. **Plan first:** show a short plan (files, approach, risks) and **wait for approval** before implementing.
3. **Branch per task:** `feat/T-0XX-slug`. Never commit to `main` directly during implementation.
4. **Ask before** adding a dependency, changing the schema, or changing an API contract. If approved, update `DB_SCHEMA.md`/`API_SPEC.md` (and TAD/ADR if affected) in the same commit.
5. **Secrets:** never read, print, log or commit values from `.env*` files. Refer to variable names only. `.env.example` lists names only.
6. **Git safety:** never run `reset --hard`, `clean`, `checkout -- .`, force push, or any destructive command without explicit user instruction.
7. **Sizing:** a task is about 400 **hand-written** lines including tests; generated files and files copied verbatim from an approved spec are not counted. If a task will exceed that, stop and propose a split.
8. **Reports** at the end of a task list every acceptance criterion as met / not met with evidence (command + output), and state plainly what was **not** verified (e.g. manual checks not done).
9. **Same commit:** update the task's status in `docs/TASKS.md` and any spec that changed.
10. Do not commit unless asked. Show drafts of docs and wait for approval.
