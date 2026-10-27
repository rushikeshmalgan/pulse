# Pulse — API Specification

| | |
|---|---|
| **Status** | Approved (rev 1) |
| **Date** | 2026-10-27 |
| **Inputs** | `docs/PRD.md` (rev 2), `docs/TAD.md`, `docs/adr/0001`–`0006`, `docs/DB_SCHEMA.md` |
| **Scope** | Every v1.0 Server Action, Route Handler, page-level data loader, the proxy and the manifest. v1.1 surfaces are listed by name only (§9). |

**How to read.** Surface IDs: `A-` Server Action, `R-` Route Handler, `L-` page-level data loader, `P-` proxy, `M-` metadata route. Each surface lists its type, file path, auth rule, input (a Zod schema where there is input), output type, error codes, side effects (tables written, feed card upserts, streak cache invalidation), rate limit, and the PRD v1.0 requirements it serves. Table names are those of `docs/DB_SCHEMA.md`; algorithm names are those of `docs/TAD.md`. Blocks starting `// @zod` are executable and were run against sample inputs (§10).

---

## 1. Conventions

### 1.1 Transports

| Kind | How it is called | Returns |
|---|---|---|
| **Server Action** | `useMutation` in a client component calls an exported `'use server'` function. Built with `defineAction` (TAD §9.2): parse input → `requireUser` → `run` → contain exceptions. There is no stable URL; Next.js posts to the page. | `Result<T>` (never throws for expected failures) |
| **Route Handler** | `fetch` from TanStack Query, or Clerk calling the webhook. | JSON payload with HTTP 200 on success; `{ "error": AppError }` with the mapped status on failure |
| **Page loader** | Called by a Server Component during render (`src/features/*/server/queries.ts`). No network surface. | A typed value; failures become `redirect()` or `notFound()` |

### 1.2 Result type and error codes

```ts
// src/lib/result.ts
export type Result<T, E extends AppError = AppError> = { ok: true; value: T } | { ok: false; error: E }
export const ok  = <T>(value: T): Result<T, never> => ({ ok: true, value })
export const err = <E extends AppError>(error: E): Result<never, E> => ({ ok: false, error })

// src/lib/errors.ts
export const ERROR_CODES = [
  'UNAUTHENTICATED', 'FORBIDDEN', 'NOT_FOUND', 'VALIDATION', 'CONFLICT',
  'STALE_DAY', 'NOT_SCHEDULED', 'OUT_OF_WINDOW', 'GROUP_FULL', 'RATE_LIMITED', 'INTERNAL',
] as const
export type ErrorCode = (typeof ERROR_CODES)[number]
export type AppError = {
  code: ErrorCode
  message: string                          // safe to show; never contains internals
  fieldErrors?: Record<string, string>     // keys are Zod paths, e.g. "rules.goal"
}
```

| Code | Meaning | HTTP (route handlers) | Client behaviour | Reachable in v1.0 |
|---|---|---|---|---|
| `UNAUTHENTICATED` | No session, or the user is tombstoned | 401 | Redirect to sign-in | yes |
| `FORBIDDEN` | A member attempted an owner-only action | 403 | Inline message | yes (`A-06`) |
| `NOT_FOUND` | Unknown id, deleted group, **not a member**, not the owner of the target, or an **invalid invite**. One message for all, so existence is never revealed (FR-GRP-11, FR-GRP-03) | 404 | "Not found" page or inline "invite isn't valid" | yes |
| `VALIDATION` | Zod failure or a rule violation; `fieldErrors` set | 422 | Field errors | yes |
| `CONFLICT` | Unique violation (handle taken) or a state conflict (delete a target that has history) | 409 | Inline message | yes |
| `STALE_DAY` | `expectedDate` differs from the server-resolved day (device open across midnight) | 409 | Refetch Today, short notice | yes (`A-12`) |
| `NOT_SCHEDULED` | Progress on a day the target is not scheduled | 422 | Refetch Today | yes (`A-12`) |
| `OUT_OF_WINDOW` | A write older than yesterday. Unreachable through the day token; raised only by the repository guard `assertWithinEditWindow` as defence in depth | 422 | Refetch Today | defensive only |
| `GROUP_FULL` | The 16th joiner | 409 | "Group is full" | yes (`A-04`) |
| `RATE_LIMITED` | Failed-code attempt limit (BR-GROUP-5) | 429 | Wait message | **no (v1.1)** |
| `INTERNAL` | Anything unexpected; logged to `error_events` | 500 | Generic message | yes |

### 1.3 Auth rules

| Rule | Meaning | Helper |
|---|---|---|
| `public` | No session required | none |
| `signature` | Verified Clerk webhook signature | `verifyWebhook` |
| `session` | A signed-in, non-tombstoned user; creates the `users` row on first use | `requireUser` |
| `onboarded` | `session` plus `onboarded_at` set; enforced by the `(app)` layout redirect, not by each surface | layout |
| `member` | `session` plus an `ACTIVE` membership in the group of a non-deleted group; otherwise `NOT_FOUND` | `requireGroupAccess(groupId)` |
| `owner` | `member` with role `OWNER`; a non-owner member gets `FORBIDDEN` | `requireGroupAccess(groupId, { role: 'OWNER' })` |
| `target-owner` | `session` and `targets.owner_id = user`; otherwise `NOT_FOUND` | `requireTargetOwner(targetId)` |

**Every authenticated surface** may insert the user's `users` row the first time it runs (`ensureUser`, TAD §4.3). That write is not repeated in the per-surface "side effects".

### 1.4 Shared Zod primitives

**Where they live.** The primitives below go in `src/lib/primitives.ts`. `targetRules` and the target action inputs go in `src/features/targets/schema.ts`; the account inputs and `clerkEvent` in `src/features/accounts/schema.ts`; group, sharing, checklist and reaction inputs in the `schema.ts` of their feature; `feedQuery` in `src/features/feed/schema.ts`. The same schema validates the form on the client and the action on the server (TAD §9.3). Numeric limits come from `core/limits.ts` (the literals appear here so the spec reads on its own).

```ts
// @zod
import { z } from 'zod'
import { isIanaZone } from '@/core/time/zone'       // true for 'UTC' or 'Area/Location' names the runtime accepts; rejects offsets like '+05:00'

export const uuid = z.uuid()
export const localDate = z.iso.date().brand<'LocalDate'>()      // 'YYYY-MM-DD', a real calendar date
export const zone = z.string().min(1).max(64).refine(isIanaZone, 'Unknown timezone')
export const handle = z
  .string().trim().toLowerCase()
  .regex(/^[a-z][a-z0-9_]{2,19}$/, '3–20 characters: letters, digits or underscore, starting with a letter')
export const displayName = z.string().trim().min(1).max(40)
export const groupName = z.string().trim().min(1).max(40)
export const targetName = z.string().trim().min(1).max(60)
export const targetUnit = z.string().trim().min(1).max(20)
export const mode = z.enum(['CHECKBOX', 'COUNT', 'DURATION'])
export const frequency = z.enum(['DAILY', 'SPECIFIC_DAYS'])
export const weekday = z.number().int().min(1).max(7)            // ISO: 1 = Monday … 7 = Sunday
export const dayToken = z.enum(['today', 'yesterday'])
export const reactionEmoji = z.enum(['FIRE', 'MUSCLE', 'CLAP', 'LAUGH', 'EYES'])
export const nextPath = z
  .string().max(256)
  .regex(/^\/(?!\/)[^\s\\]*$/, 'Must be a path on this site')    // blocks "//host" and absolute URLs (open redirect)
  .refine((p) => !p.startsWith('/api'), 'Must be a page')
export const groupParams = z.object({ groupId: uuid })
export const targetParams = z.object({ targetId: uuid })
```

The target rules are shared by create and update (BR-TGT-3, FR-TGT-02/03):

```ts
// @zod
export const targetRules = z
  .object({
    mode,
    goal: z.number().int().optional(),                   // omitted for CHECKBOX
    frequency,
    weekdays: z.array(weekday).max(7).default([]),
  })
  .superRefine((v, ctx) => {
    const bad = (key: string, message: string) => ctx.addIssue({ code: 'custom', path: [key], message })
    if (v.mode === 'CHECKBOX' && v.goal !== undefined) bad('goal', 'A checkbox target has no goal')
    if (v.mode === 'COUNT' && !(v.goal !== undefined && v.goal >= 1 && v.goal <= 50)) bad('goal', 'Goal must be 1–50')
    if (v.mode === 'DURATION' && !(v.goal !== undefined && v.goal >= 5 && v.goal <= 480 && v.goal % 5 === 0))
      bad('goal', 'Goal must be 5–480 minutes, in steps of 5')
    const days = new Set(v.weekdays)
    if (v.frequency === 'DAILY' && days.size > 0) bad('weekdays', 'Daily targets have no weekdays')
    if (v.frequency === 'SPECIFIC_DAYS' && days.size === 0) bad('weekdays', 'Choose at least one weekday')
  })
  .transform((v) => {
    const days = [...new Set(v.weekdays)].sort((a, b) => a - b)
    // All seven weekdays is stored as DAILY (FR-TGT-03).
    return v.frequency === 'SPECIFIC_DAYS' && days.length === 7
      ? { ...v, frequency: 'DAILY' as const, weekdays: [] as number[] }
      : { ...v, weekdays: days }
  })
```

### 1.5 Shared DTO types

```ts
type LocalDate = string                       // 'YYYY-MM-DD'
type IsoInstant = string                      // ISO 8601 instant
type UserRef  = { userId: string; displayName: string; handle: string; imageUrl: string | null }
type GroupRef = { groupId: string; name: string }
type MyGroup  = GroupRef & { role: 'OWNER' | 'MEMBER' }
type RollCallState = 'ACTIVE' | 'NOT_YET' | 'SLACKING' | 'REST_DAY' | 'NO_TARGETS_SHARED'
type DayState = 'L0' | 'L1' | 'L2' | 'L3' | 'L4' | 'REST' | 'UNTRACKED' | 'TODAY_PENDING' | 'FUTURE'
type DayCell  = { date: LocalDate; state: DayState; done: number; scheduled: number }
type ReactionEmoji = 'FIRE' | 'MUSCLE' | 'CLAP' | 'LAUGH' | 'EYES'

type TaskDTO = {
  targetId: string; name: string
  mode: 'CHECKBOX' | 'COUNT' | 'DURATION'; unit: string | null
  goal: number | null                         // null for CHECKBOX
  value: number                               // 0 when there is no entry
  complete: boolean
  sharedWith: GroupRef[]                      // groups that see this target on this date (the share indicator, FR-CHK-01)
}

type FeedCardDTO = {
  id: string; groupId: string
  user: UserRef
  localDate: LocalDate
  kind: 'DAILY_PROGRESS'
  lastActivityAt: IsoInstant
  summary: { done: number; scheduled: number }            // "3 of 5": completed vs scheduled shared tasks that day
  tasks: Array<{ targetId: string; name: string; mode: 'CHECKBOX' | 'COUNT' | 'DURATION'
                 unit: string | null; goal: number | null; value: number; complete: boolean }>   // shared tasks with progress > 0
  reactions: Array<{ emoji: ReactionEmoji; count: number; mine: boolean }>   // only emojis with count > 0, in the fixed emoji order
}

type PollHint = {
  pollHintMs: number | null     // ≥ 20_000; null = do not poll in the background (level L2)
  idleStopMin: number           // 3 at L0, 1 at L1, 0 at L2
}
```

### 1.6 Cross-cutting rules

- **Cookies.** `pulse_group` holds the selected group id (FR-GRP-05): `httpOnly`, `SameSite=Lax`, `Path=/`, one-year `Max-Age`. Set by `A-03`, `A-04`, `A-05`; read and validated (must be an active membership, else ignored) by `L-01`.
- **Headers.** Every Route Handler response carries `Cache-Control: private, no-store`.
- **Rate limits.** None in v1.0 on any surface. The client controller caps polling at 180 requests per user per hour (NFR-POLL-02); the server's lever is `pollHint` (`R-02`). The invite-attempt limit (BR-GROUP-5) is v1.1 and will add `RATE_LIMITED` to `A-04`.
- **Idempotency.** Every action is safe to retry: writes set absolute values (`A-12`), insert-or-reopen (`A-11`, `A-04`), or are no-ops when the state already holds (`A-09`, `A-13` toggles by design).
- **Query-cache keys** the client invalidates after a mutation are listed in TAD §7.5; each action below returns the ids it needs (`affectedGroupIds`).
- **Day tokens.** Actions that depend on the day never accept a date from the client as authoritative (BR-DAY-1). `A-12` accepts `expectedDate` only to detect staleness.

---

## 2. Catalog

| ID | Name | Type | Path |
|---|---|---|---|
| A-01 | `completeOnboarding` | Server Action | `src/features/accounts/server/actions.ts` |
| A-02 | `updateProfile` | Server Action | `src/features/accounts/server/actions.ts` |
| A-03 | `createGroup` | Server Action | `src/features/groups/server/actions.ts` |
| A-04 | `joinGroup` | Server Action | `src/features/groups/server/actions.ts` |
| A-05 | `selectGroup` | Server Action | `src/features/groups/server/actions.ts` |
| A-06 | `removeMember` | Server Action | `src/features/groups/server/actions.ts` |
| A-07 | `createTarget` | Server Action | `src/features/targets/server/actions.ts` |
| A-08 | `updateTarget` | Server Action | `src/features/targets/server/actions.ts` |
| A-09 | `archiveTarget` | Server Action | `src/features/targets/server/actions.ts` |
| A-10 | `deleteTarget` | Server Action | `src/features/targets/server/actions.ts` |
| A-11 | `setShares` | Server Action | `src/features/sharing/server/actions.ts` |
| A-12 | `recordProgress` | Server Action | `src/features/checklist/server/actions.ts` |
| A-13 | `toggleReaction` | Server Action | `src/features/reactions/server/actions.ts` |
| R-01 | `GET /api/today` | Route Handler | `src/app/api/today/route.ts` |
| R-02 | `GET /api/groups/[groupId]/pulse` | Route Handler | `src/app/api/groups/[groupId]/pulse/route.ts` |
| R-03 | `GET /api/groups/[groupId]/feed` | Route Handler | `src/app/api/groups/[groupId]/feed/route.ts` |
| R-04 | `POST /api/webhooks/clerk` | Route Handler | `src/app/api/webhooks/clerk/route.ts` |
| L-01 | `loadAppShell` | Page loader | `src/app/(app)/layout.tsx` → `src/features/accounts/server/queries.ts` |
| L-02 | `loadOnboarding` | Page loader | `src/app/(app)/onboarding/page.tsx` |
| L-03 | `loadToday` | Page loader | `src/app/(app)/today/page.tsx` → `src/features/checklist/server/queries.ts` |
| L-04 | `loadPersonalHeatmap` | Page loader | `src/app/(app)/today/page.tsx` → `src/features/heatmap/server/queries.ts` |
| L-05 | `loadTargets` | Page loader | `src/app/(app)/targets/page.tsx` |
| L-06 | `loadTargetForm` | Page loader | `src/app/(app)/targets/new/page.tsx`, `src/app/(app)/targets/[targetId]/page.tsx` |
| L-07 | `loadGroupOverview` | Page loader | `src/app/(app)/groups/[groupId]/page.tsx` → `src/features/heatmap/server/queries.ts` |
| L-08 | `loadMembers` | Page loader | `src/app/(app)/groups/[groupId]/members/page.tsx` |
| L-09 | `loadJoin` | Page loader | `src/app/(app)/join/[code]/page.tsx` |
| L-10 | `loadProfile` | Page loader | `src/app/(app)/settings/profile/page.tsx` |
| P-01 | `proxy` | Proxy | `src/proxy.ts` |
| M-01 | `manifest` and icons | Metadata route | `src/app/manifest.ts`, `src/app/apple-icon.png`, `public/icons/*` |

Pages with no data of their own: `/groups/new` (a create-group form that calls `A-03`, and a "Join with a code" form that calls `A-04`), `/sign-in` and `/sign-up` (Clerk components).

---

## 3. Server Actions

### A-01 `completeOnboarding`

| | |
|---|---|
| **Type / file** | Server Action · `src/features/accounts/server/actions.ts` |
| **Auth** | `session` (the user may not be onboarded yet) |
| **Output** | `{ redirectTo: string }`: the validated `next` path, else `/today` |
| **Errors** | `UNAUTHENTICATED`, `VALIDATION` (`fieldErrors`), `CONFLICT` (handle taken), `INTERNAL` |
| **Side effects** | Writes `users`: `display_name`, `handle`, `timezone`, `onboarded_at = now`. If already onboarded: no write, same output (idempotent). No feed cards. No cache invalidation (a new user has no history, and `date_floor` stays null). |
| **Rate limit** | none |
| **FRs served** | FR-ACC-03 |

```ts
// @zod
export const completeOnboardingInput = z.object({
  displayName,
  handle,
  timezone: zone,                // detected on the client with Intl.DateTimeFormat().resolvedOptions().timeZone
  next: nextPath.optional(),     // where the user was going (for example an invite link)
})
```

### A-02 `updateProfile`

| | |
|---|---|
| **Type / file** | Server Action · `src/features/accounts/server/actions.ts` |
| **Auth** | `onboarded` `session` |
| **Output** | `{ displayName: string; handle: string; timezone: string; effectiveToday: LocalDate; timezoneChanged: boolean }` |
| **Errors** | `UNAUTHENTICATED`, `VALIDATION`, `CONFLICT` (handle taken), `INTERNAL` |
| **Side effects** | Writes `users`. **If the timezone changed**, in one transaction: `date_floor = effectiveToday(old zone, old floor, now)` (BR-DAY-5), `timezone = new`, and **all of the user's `streak_prefixes` are deleted**. Entries and share intervals are never re-dated. The client calls `queryClient.clear()` and `router.refresh()` when `timezoneChanged`. No feed cards. |
| **Rate limit** | none |
| **FRs served** | FR-ACC-04, FR-ACC-05 |

TAD §5.4 names this logic `changeTimezone`; here it is one action so a profile form saves atomically.

```ts
// @zod
export const updateProfileInput = z
  .object({ displayName: displayName.optional(), handle: handle.optional(), timezone: zone.optional() })
  .refine((v) => v.displayName !== undefined || v.handle !== undefined || v.timezone !== undefined, {
    message: 'Nothing to change',
  })
```

### A-03 `createGroup`

| | |
|---|---|
| **Type / file** | Server Action · `src/features/groups/server/actions.ts` |
| **Auth** | `onboarded` `session` |
| **Output** | `{ groupId: string; name: string; inviteCode: string; inviteUrl: string }` where `inviteUrl = NEXT_PUBLIC_APP_URL + '/join/' + inviteCode` |
| **Errors** | `UNAUTHENTICATED`, `VALIDATION`, `INTERNAL` (code generation exhausted its 5 retries) |
| **Side effects** | One transaction: inserts `groups` (8-character code from the 31-symbol alphabet with `crypto.randomInt`; retries on a unique violation) and a `memberships` row with `role = OWNER`, `status = ACTIVE`. Sets the `pulse_group` cookie. No group-count limit in v1.0 (FR-GRP-13 is v1.1). |
| **Rate limit** | none |
| **FRs served** | FR-GRP-01, FR-GRP-02 |

```ts
// @zod
export const createGroupInput = z.object({ name: groupName })
```

### A-04 `joinGroup`

| | |
|---|---|
| **Type / file** | Server Action · `src/features/groups/server/actions.ts` |
| **Auth** | `onboarded` `session` |
| **Output** | `{ groupId: string; name: string; alreadyMember: boolean; rejoined: boolean }` |
| **Errors** | `UNAUTHENTICATED`; `NOT_FOUND` with the single message "This invite isn't valid." for a malformed, unknown or deleted-group code (never reveals which); `GROUP_FULL`; `INTERNAL` |
| **Side effects** | Normalises the code (upper-case, spaces and hyphens removed). One transaction: `SELECT … FROM groups WHERE invite_code = $1 AND deleted_at IS NULL FOR UPDATE` (serialises concurrent joins) → if the user's membership is `ACTIVE`, return `alreadyMember` → count `ACTIVE` memberships, `GROUP_FULL` at 15 → reactivate the existing row (`status = ACTIVE`, `ended_at = NULL`, `rejoined`) **and delete the user's `streak_prefixes` row for the group**, or insert a new `MEMBER` row. **No share interval is reopened** (BR-SHARE-6). Sets the `pulse_group` cookie. No feed card in v1.0 (the "joined" card is v1.1). |
| **Rate limit** | none in v1.0; `RATE_LIMITED` after 10 failed attempts in an hour arrives in v1.1 (BR-GROUP-5) |
| **FRs served** | FR-GRP-03, FR-ACC-01 (return-to-invite), FR-GRP-11 |

**Called only from a user action.** Either the one-tap **Join group** button on the invite page (`L-09`) or the **Join with a code** form on `/groups/new` (a code typed by hand, FR-GRP-03). Nothing joins on a GET, and nobody joins without tapping.

```ts
// @zod
export const joinGroupInput = z.object({ code: z.string().max(64) })    // validity is decided server-side so every failure looks the same
```

### A-05 `selectGroup`

| | |
|---|---|
| **Type / file** | Server Action · `src/features/groups/server/actions.ts` |
| **Auth** | `member` of the group |
| **Output** | `{ groupId: string }` |
| **Errors** | `UNAUTHENTICATED`, `NOT_FOUND`, `INTERNAL` |
| **Side effects** | Sets the `pulse_group` cookie. No table is written. |
| **Rate limit** | none |
| **FRs served** | FR-GRP-05 |

```ts
// @zod
export const selectGroupInput = z.object({ groupId: uuid })
```

### A-06 `removeMember`

| | |
|---|---|
| **Type / file** | Server Action · `src/features/groups/server/actions.ts` |
| **Auth** | `owner` of the group |
| **Output** | `{ removedUserId: string }` |
| **Errors** | `UNAUTHENTICATED`; `NOT_FOUND` (group not visible, or the target user is not an `ACTIVE` member); `FORBIDDEN` (caller is not the owner); `VALIDATION` (`fieldErrors.userId`: the owner cannot remove themselves); `INTERNAL` |
| **Side effects** | One transaction: `memberships.status = REMOVED`, `ended_at = now`; the removed user's open `share_intervals` for this group are **closed at their `yesterday`, or deleted if they began today** (TAD §6.2 `onMemberRemoved`, refined in DB_SCHEMA §9); **their `streak_prefixes` row for the group is deleted**. Entries, targets, feed cards and reactions are untouched (they vanish through the `ACTIVE` filter). Removal is not a ban: the member can rejoin with the invite (BR-GROUP-6). |
| **Rate limit** | none |
| **FRs served** | FR-GRP-07 |

```ts
// @zod
export const removeMemberInput = z.object({ groupId: uuid, userId: uuid })
```

### A-07 `createTarget`

| | |
|---|---|
| **Type / file** | Server Action · `src/features/targets/server/actions.ts` |
| **Auth** | `onboarded` `session`; each `groupId` must be an `ACTIVE` membership of the caller (BR-SHARE-1) |
| **Output** | `{ targetId: string; startDate: LocalDate; sharedGroupIds: string[] }` |
| **Errors** | `UNAUTHENTICATED`; `VALIDATION` (`fieldErrors` such as `rules.goal`, `rules.weekdays`, `name`); `NOT_FOUND` (a `groupId` the caller is not an active member of); `INTERNAL` |
| **Side effects** | One transaction: inserts `targets` (`start_date` = the caller's `effectiveToday`; never a past or future date in v1.0), the first `target_revisions` row (`effective_from = start_date`), and one `share_intervals` row per group (`from_date = today`, open). No `entries`, so **no feed cards**. No streak invalidation (nothing in the past changes). The client invalidates `['today']` and the pulse of each shared group. No target-count limit in v1.0 (FR-TGT-05 is v1.1). |
| **Rate limit** | none |
| **FRs served** | FR-TGT-01, FR-TGT-02, FR-TGT-03, FR-TGT-04, FR-VIS-01, FR-VIS-04 |

```ts
// @zod
export const createTargetInput = z.object({
  name: targetName,
  unit: targetUnit.optional(),
  rules: targetRules,
  groupIds: z.array(uuid).max(50).default([]),     // empty = private (FR-VIS-01)
})
```

Templates (`core/templates`) pre-fill the form on the client; the server validates only the submitted fields.

### A-08 `updateTarget`

| | |
|---|---|
| **Type / file** | Server Action · `src/features/targets/server/actions.ts` |
| **Auth** | `target-owner` |
| **Output** | `{ targetId: string; effectiveFrom: LocalDate \| null }`: `null` when only labels (name, unit) changed; the target's start date when rules were edited in place; tomorrow's date when a new revision was written |
| **Errors** | `UNAUTHENTICATED`, `NOT_FOUND`, `VALIDATION` (including `fieldErrors.mode`: "Tracking mode can't change once progress exists. Archive this target and create a new one."), `INTERNAL` |
| **Side effects** | **Name and unit** update `targets` immediately and apply to all days (FR-TGT-06 first scenario). **Rules** (`mode`, `goal`, `frequency`, `weekdays`) are merged with the latest revision (including a pending one) and validated with `targetRules`. Without history and with `start_date >= today`: the first `target_revisions` row is updated in place (and `targets.mode` if changed). With history: a revision with `effective_from = today + 1` is upserted, replacing a pending one, and a changed `mode` is refused. "History" = any `entries` row or a fully elapsed scheduled day (TAD §6.1). No feed cards and no streak invalidation (only today-or-later changes). |
| **Rate limit** | none |
| **FRs served** | FR-TGT-06, FR-TGT-02, FR-TGT-03 |

```ts
// @zod
export const updateTargetInput = z
  .object({
    targetId: uuid,
    name: targetName.optional(),
    unit: targetUnit.nullable().optional(),          // null clears the unit
    mode: mode.optional(),
    goal: z.number().int().nullable().optional(),    // null clears the goal
    frequency: frequency.optional(),
    weekdays: z.array(weekday).max(7).optional(),
  })
  .refine((v) => Object.entries(v).some(([k, x]) => k !== 'targetId' && x !== undefined), { message: 'Nothing to change' })
```

### A-09 `archiveTarget`

| | |
|---|---|
| **Type / file** | Server Action · `src/features/targets/server/actions.ts` |
| **Auth** | `target-owner` |
| **Output** | `{ effectiveFrom: LocalDate; affectedGroupIds: string[] }`; `effectiveFrom` is tomorrow; `affectedGroupIds` are the groups sharing the target today |
| **Errors** | `UNAUTHENTICATED`, `NOT_FOUND`, `INTERNAL` |
| **Side effects** | Inserts `target_inactive_periods` (`kind = ARCHIVED`, `from_date = today + 1`, open) unless one is already open (idempotent). Past days are untouched in every view. No feed cards and no streak invalidation. |
| **Rate limit** | none |
| **FRs served** | FR-TGT-07 |

```ts
// @zod
export const archiveTargetInput = z.object({ targetId: uuid })
```

### A-10 `deleteTarget`

| | |
|---|---|
| **Type / file** | Server Action · `src/features/targets/server/actions.ts` |
| **Auth** | `target-owner` |
| **Output** | `{ affectedGroupIds: string[] }` (captured before the delete) |
| **Errors** | `UNAUTHENTICATED`, `NOT_FOUND`, `CONFLICT` ("This target has history. Archive it instead."), `INTERNAL` |
| **Side effects** | Allowed only without history (no `entries` row and no fully elapsed scheduled day). Deletes `targets`; `target_revisions`, `target_inactive_periods` and `share_intervals` cascade. The `Restrict` reference from `entries` blocks the delete at the database as a second guard. No feed cards exist (no entries). |
| **Rate limit** | none |
| **FRs served** | FR-TGT-08 |

```ts
// @zod
export const deleteTargetInput = z.object({ targetId: uuid })
```

### A-11 `setShares`

| | |
|---|---|
| **Type / file** | Server Action · `src/features/sharing/server/actions.ts` |
| **Auth** | `target-owner`; every group added must be an `ACTIVE` membership of the caller |
| **Output** | `{ added: string[]; removed: string[]; affectedGroupIds: string[] }` |
| **Errors** | `UNAUTHENTICATED`, `NOT_FOUND` (target, or a group the caller is not an active member of), `VALIDATION`, `INTERNAL` |
| **Side effects** | Computes the difference between the desired group set and the currently open intervals, then in one transaction: for each **added** group, `share` (TAD §6.2): insert `from_date = today`, or reopen an interval closed today; for each **removed** group, `unshare`: `to_date = today` (today still counts; tomorrow does not; past days stay, FR-VIS-05). **For each added group, `syncFeedCards` runs for today** (an upsert into `feed_cards`), so a target that already has progress today appears on that group's card immediately (refinement, DB_SCHEMA §9). Cards of removed groups are not touched (FR-VIS-05). No streak invalidation (only today-or-later changes). |
| **Rate limit** | none |
| **FRs served** | FR-VIS-01, FR-VIS-02, FR-VIS-04, FR-VIS-05, FR-VIS-06, FR-FED-04 |

```ts
// @zod
export const setSharesInput = z
  .object({ targetId: uuid, groupIds: z.array(uuid).max(50) })
  .transform((v) => ({ ...v, groupIds: [...new Set(v.groupIds)] }))
```

### A-12 `recordProgress`

| | |
|---|---|
| **Type / file** | Server Action · `src/features/checklist/server/actions.ts` |
| **Auth** | `target-owner` |
| **Output** | `{ targetId: string; date: LocalDate; value: number; complete: boolean; affectedGroupIds: string[] }` |
| **Errors** | `UNAUTHENTICATED`; `NOT_FOUND`; `STALE_DAY` (`expectedDate` ≠ server-resolved date); `NOT_SCHEDULED`; `VALIDATION` (`value` above the mode's cap: COUNT 999, DURATION 1440, CHECKBOX 1; BR-PRG-1); `INTERNAL` |
| **Side effects** | Resolves `date = resolveDay(user, now, day)` from the user's stored zone and `date_floor`; never trusts the client. One transaction (TAD §6.7): `entries` upsert, or **delete when `value = 0`**; then `syncFeedCards` for every group that shares the target on `date` (interval covers the date, owner `ACTIVE`): **delete the member's `feed_cards` row** for that group and date when no shared scheduled task has progress, otherwise **upsert it** (`last_activity_at = now`; its `reactions` cascade on delete). No streak-cache write: the edit window (today and yesterday) is outside the cached prefix (TAD §6.4). The value is **absolute**, so retries are idempotent; the client serialises taps per entry with a mutation scope (TAD §7.4). |
| **Rate limit** | none |
| **FRs served** | FR-CHK-02, FR-CHK-03, FR-CHK-04, FR-CHK-05, FR-CHK-06, FR-CHK-08, FR-CHK-09, FR-FED-01, FR-STK-02 |

```ts
// @zod
export const recordProgressInput = z.object({
  targetId: uuid,
  day: dayToken,                       // "today" or "yesterday" in the member's own timezone (BR-DAY-3)
  expectedDate: localDate,             // the date the screen believes it is showing; used only to detect staleness
  value: z.number().int().min(0).max(1440),
})
```

### A-13 `toggleReaction`

| | |
|---|---|
| **Type / file** | Server Action · `src/features/reactions/server/actions.ts` |
| **Auth** | `member` of the card's group, resolved from the card; the card's author must be an `ACTIVE` member too, else `NOT_FOUND` |
| **Output** | `{ cardId: string; emoji: ReactionEmoji; active: boolean; reactions: FeedCardDTO['reactions'] }` |
| **Errors** | `UNAUTHENTICATED`, `NOT_FOUND`, `VALIDATION`, `INTERNAL` |
| **Side effects** | Reads `feed_cards` to find the card's group, then inserts the `reactions` row `(card_id, user_id, emoji)` or deletes it if present (a toggle). Reacting to your own card is allowed (FR-REA-01). No `feed_cards` or streak writes. |
| **Rate limit** | none |
| **FRs served** | FR-REA-01, FR-REA-03 |

```ts
// @zod
export const toggleReactionInput = z.object({ cardId: uuid, emoji: reactionEmoji })
```

---

## 4. Route Handlers

All Route Handlers are `GET` except the webhook, run on the Node.js runtime, and answer errors with `{ "error": AppError }` and the status of §1.2.

### R-01 `GET /api/today`

| | |
|---|---|
| **Type / file** | Route Handler · `src/app/api/today/route.ts` (same function as `L-03`) |
| **Auth** | `onboarded` `session` |
| **Input** | none |
| **Output** | `TodayData` (below) |
| **Errors** | `UNAUTHENTICATED` (401), `INTERNAL` (500) |
| **Side effects** | Read-only. |
| **Rate limit** | none; the client refetches on focus at most once per 10 s and after mutations |
| **FRs served** | FR-CHK-01, FR-CHK-06, FR-CHK-07, FR-CHK-09, FR-CHK-10, FR-CHK-11, FR-STK-01, FR-STK-03, FR-VIS-07 |

```ts
type TodayData = {
  serverTime: IsoInstant
  zone: string
  today: LocalDate                  // the member's effective today (BR-DAY-5)
  yesterday: LocalDate
  hasTargets: boolean               // at least one target that is tracked today; false → "add a target from the templates" (FR-CHK-10)
  tasksToday: TaskDTO[]             // scheduled today, in target creation order (FR-CHK-01)
  tasksYesterday: TaskDTO[]         // the Yesterday section; editable until local midnight (FR-CHK-06)
  summary: { done: number; scheduled: number }      // "N of M done" for today (FR-CHK-11)
  streak: { personal: number }      // all targets; today pending neither adds nor breaks (FR-STK-03)
}
```

Reads: `targets` (+ `target_revisions`, `target_inactive_periods`), `entries` for yesterday and today, `share_intervals` and `memberships` for the share indicators, and the member's whole `entries` history for `fullStreak`. Days with no entry are incomplete by derivation, so nothing needs "catching up" (FR-CHK-07).

### R-02 `GET /api/groups/[groupId]/pulse`

| | |
|---|---|
| **Type / file** | Route Handler · `src/app/api/groups/[groupId]/pulse/route.ts` |
| **Auth** | `member` |
| **Input** | path `groupId` validated with `groupParams`; no query string |
| **Output** | `Pulse` (below) |
| **Errors** | `UNAUTHENTICATED` (401), `NOT_FOUND` (404; also for an invalid uuid or a non-member), `INTERNAL` (500) |
| **Side effects** | Reads only, **except** `savePrefix` in `groupStreak`: a member's `streak_prefixes` row is upserted when it is missing, when its `through_date` is earlier than the member's `today − 2`, or when it comes from an older `algo_version` (TAD §6.4). It is derived data, so this write never changes any visible result. |
| **Rate limit** | none server-side. The client caps itself at 180 requests per user per hour and stops after `idleStopMin` minutes without interaction (NFR-POLL-02). |
| **FRs served** | FR-ROL-01, FR-ROL-02, FR-ROL-03, FR-HM-05, FR-STK-01, FR-STK-02, FR-STK-03, FR-VIS-03, FR-VIS-07, FR-FED-01, FR-FED-04, FR-FED-05, FR-REA-01, FR-REA-03, FR-GRP-11 |

```ts
type PulseMember = {
  userId: string
  todayLocal: LocalDate              // that member's effective today
  state: RollCallState               // evaluated on the member's own date and clock (TAD §5.6)
  done: number; scheduled: number    // the member's today, group scope
  streak: number                     // group-scoped current streak (TAD §6.4)
  recent: DayCell[]                  // exactly two cells: the member's yesterday, then today
}

type Pulse = PollHint & {
  serverTime: IsoInstant
  viewerToday: LocalDate
  members: PulseMember[]             // every ACTIVE member; order is unspecified, match by userId
  feed: { cards: FeedCardDTO[]; nextCursor: string | null }      // page 1, 20 cards, newest activity first
}
```

**Polling hint (server-sent).** `PULSE_POLL_MODE` (`L0` default, `L1`, `L2`) selects the values returned in every `Pulse` and in `GroupOverview`:

| Mode | `pollHintMs` | `idleStopMin` | Meaning |
|---|---|---|---|
| `L0` | 20000 | 3 | PRD default (NFR-POLL-02) |
| `L1` | 20000 | 1 | stop sooner (NFR-POLL-04 fallback) |
| `L2` | `null` | 0 | no background refreshing: refresh on focus, pull and after the viewer's own actions |

The client uses `max(20000, pollHintMs)` and never polls faster than 20 s whatever the server says.

**Client merge rules.** (1) Patch each member's `state`, `done`, `scheduled`, `streak` and replace the last two cells of the strip with `recent`. (2) Replace page 1 of the feed, de-duplicating by `id` and keeping older loaded pages. (3) The poll never reorders rows; the overview order (FR-HM-06) is kept. (4) If the set of `userId`s differs from the overview's (someone joined or was removed), call `router.refresh()`.

**Database reads (budget ≤ 20 KB, ≤ 40 ms CPU).** Active `memberships` with `users` (≤ 15 rows) → shared structure for the group over the 4-date window (`share_intervals`, `targets`, `target_revisions`, `target_inactive_periods`) → `entries` for those targets and dates → `streak_prefixes` (≤ 15 rows) → `feed_cards` page 1 with `reactions` counts. No history scans.

### R-03 `GET /api/groups/[groupId]/feed`

| | |
|---|---|
| **Type / file** | Route Handler · `src/app/api/groups/[groupId]/feed/route.ts` |
| **Auth** | `member` |
| **Input** | path `groupParams`; query `feedQuery` |
| **Output** | `{ cards: FeedCardDTO[]; nextCursor: string \| null }` (20 per page) |
| **Errors** | `UNAUTHENTICATED`, `NOT_FOUND`, `VALIDATION` (malformed cursor), `INTERNAL` |
| **Side effects** | Read-only. |
| **Rate limit** | none |
| **FRs served** | FR-FED-03, FR-FED-04, FR-REA-03, FR-GRP-11 |

```ts
// @zod
export const feedQuery = z.object({
  cursor: z.string().max(128).regex(/^[A-Za-z0-9_-]+$/).optional(),   // opaque: base64url of "<lastActivityAt ISO>|<card id>"
})
```

Keyset pagination on `feed_cards` by `(last_activity_at DESC, id DESC)` (TAD §6.6). Cards of non-`ACTIVE` members and reactions of non-`ACTIVE` members are excluded. Card bodies are derived from `entries` and `share_intervals` as they stood on the card's date, so a later unshare does not change an old card (FR-FED-04).

### R-04 `POST /api/webhooks/clerk`

| | |
|---|---|
| **Type / file** | Route Handler · `src/app/api/webhooks/clerk/route.ts` |
| **Auth** | `signature` (`verifyWebhook` with `CLERK_WEBHOOK_SIGNING_SECRET`); public in `P-01` |
| **Input** | the signed Clerk event; fields we read are validated by `clerkEvent` |
| **Output** | empty body |
| **Errors** | HTTP `400` invalid signature or malformed payload (no processing); HTTP `500` on any database failure so Clerk retries. Unknown or unsubscribed event types answer `200` and do nothing. |
| **Side effects** | `user.created` / `user.updated` → upsert the profile fields of `users` (display name from `first_name` and `last_name`, else `username`, else "New member"; `image_url`), **ignored if the row is tombstoned**. `user.deleted` → `tombstoneUser` (DB_SCHEMA §6.2): in one transaction `users` is anonymised, **all `memberships` become `REMOVED`**, **open `share_intervals` are closed at their yesterday or deleted if they began today**, **`streak_prefixes` rows of the user are deleted**. Nothing else is deleted. All handlers are idempotent and order-tolerant. |
| **Rate limit** | none; Clerk controls delivery and retries |
| **FRs served** | FR-ACC-02, FR-ACC-04 |

```ts
// @zod
const clerkUserData = z.object({
  id: z.string().min(1).max(64),
  first_name: z.string().nullish(),
  last_name: z.string().nullish(),
  username: z.string().nullish(),
  image_url: z.url().nullish(),
})
export const clerkEvent = z.discriminatedUnion('type', [
  z.object({ type: z.literal('user.created'), data: clerkUserData }),
  z.object({ type: z.literal('user.updated'), data: clerkUserData }),
  z.object({ type: z.literal('user.deleted'), data: z.object({ id: z.string().min(1).max(64) }) }),
])
```

The payload field names (`first_name`, `last_name`, `username`, `image_url`) are what the handler expects from Clerk's user object; confirm them against a real event in spike S5. NFR-REL-02 is satisfied here: a user deleted in Clerk can never break another member's page.

---

## 5. Page-level data loaders

A loader runs on the server during render. `UNAUTHENTICATED` becomes a redirect to sign-in (normally already done by `P-01`); `NOT_FOUND` becomes `notFound()`; an invalid route parameter is treated as `NOT_FOUND`, never as a validation page.

### L-01 `loadAppShell`

| | |
|---|---|
| **Type / file** | Page loader · `src/app/(app)/layout.tsx` → `src/features/accounts/server/queries.ts` |
| **Auth** | `session`; redirects to `/onboarding?next=<current path>` while `onboarded_at` is null |
| **Input** | the `pulse_group` cookie |
| **Output** | `{ user: { userId; displayName; handle; imageUrl; timezone }; groups: MyGroup[]; selectedGroupId: string \| null }` |
| **Errors** | `UNAUTHENTICATED` (redirect) |
| **Side effects** | `ensureUser` may insert the `users` row on the first request (FR-ACC-02). Otherwise read-only. A stale or non-member cookie value is ignored (falls back to the first group). |
| **Rate limit** | none |
| **FRs served** | FR-ACC-01, FR-ACC-02, FR-ACC-06, FR-GRP-05 |

`groups` come from `memberships` (`ACTIVE`) joined with non-deleted `groups`. The shell also hosts the Clerk user menu, which provides sign-out; on sign-out the client clears the query cache so nothing remains on the device (FR-ACC-06).

### L-02 `loadOnboarding`

| | |
|---|---|
| **Type / file** | Page loader · `src/app/(app)/onboarding/page.tsx` |
| **Auth** | `session` (works before onboarding) |
| **Input** | search params `{ next?: nextPath }` |
| **Output** | `{ displayName: string; suggestedHandle: string; imageUrl: string \| null; next: string \| null }` |
| **Errors** | `UNAUTHENTICATED` (redirect). An invalid `next` is dropped silently. |
| **Side effects** | Read-only. If the user is already onboarded, redirect to `next` or `/today`. |
| **Rate limit** | none |
| **FRs served** | FR-ACC-03 |

The detected timezone is read in the browser (`Intl.DateTimeFormat().resolvedOptions().timeZone`) and submitted with `A-01`.

### L-03 `loadToday`

| | |
|---|---|
| **Type / file** | Page loader · `src/app/(app)/today/page.tsx` → `src/features/checklist/server/queries.ts` (`getTodayData`, shared with `R-01`) |
| **Auth** | `onboarded` `session` |
| **Input** | none |
| **Output** | `TodayData` (see `R-01`), passed as `initialData` for the `['today']` query |
| **Errors** | `UNAUTHENTICATED` (redirect) |
| **Side effects** | Read-only. |
| **Rate limit** | none |
| **FRs served** | FR-CHK-01, FR-CHK-07, FR-CHK-10, FR-CHK-11, FR-STK-01, FR-VIS-07 |

### L-04 `loadPersonalHeatmap`

| | |
|---|---|
| **Type / file** | Page loader · `src/app/(app)/today/page.tsx` (separate Suspense boundary) → `src/features/heatmap/server/queries.ts` |
| **Auth** | `onboarded` `session` |
| **Input** | none |
| **Output** | `{ viewerToday: LocalDate; from: LocalDate; cells: DayCell[] }`: 52 weeks, from the Monday 51 weeks before the current week's Monday through `viewerToday` (the client pads the rest of the week and slices to 13 weeks on narrow screens) |
| **Errors** | `UNAUTHENTICATED` (redirect) |
| **Side effects** | Read-only. Personal scope: all of the member's targets (BR-VIS). |
| **Rate limit** | none |
| **FRs served** | FR-HM-01, FR-HM-02, FR-HM-03, FR-HM-04 |

The tooltip, the text summary and the day-by-day list (TAD §8) are generated on the client from `cells`.

### L-05 `loadTargets`

| | |
|---|---|
| **Type / file** | Page loader · `src/app/(app)/targets/page.tsx` |
| **Auth** | `onboarded` `session` |
| **Input** | none |
| **Output** | `{ active: TargetRow[]; archived: TargetRow[] }` |
| **Errors** | `UNAUTHENTICATED` (redirect) |
| **Side effects** | Read-only. |
| **Rate limit** | none |
| **FRs served** | FR-TGT-06, FR-TGT-07, FR-TGT-08, FR-VIS-01 |

```ts
type TargetRow = {
  targetId: string; name: string
  mode: 'CHECKBOX' | 'COUNT' | 'DURATION'; unit: string | null
  startDate: LocalDate
  rule: { goal: number | null; frequency: 'DAILY' | 'SPECIFIC_DAYS'; weekdays: number[] }          // in force today
  pendingRule: { effectiveFrom: LocalDate; goal: number | null; frequency: 'DAILY' | 'SPECIFIC_DAYS'; weekdays: number[] } | null   // a revision effective tomorrow
  archivedFrom: LocalDate | null
  hasHistory: boolean                // drives "Delete" vs "Archive only" (FR-TGT-08)
  shares: Array<GroupRef & { since: LocalDate; through: LocalDate | null }>   // through = null while still shared
}
```

### L-06 `loadTargetForm`

| | |
|---|---|
| **Type / file** | Page loader · `src/app/(app)/targets/new/page.tsx` and `src/app/(app)/targets/[targetId]/page.tsx` |
| **Auth** | `onboarded` `session`; for an existing target, `target-owner` |
| **Input** | `/targets/new?group=<uuid>` (search params `{ group?: uuid }`) or path `targetParams` |
| **Output** | `{ templates: Template[]; groups: MyGroup[]; preselectedGroupId: string \| null; target: TargetRow \| null }` |
| **Errors** | `UNAUTHENTICATED` (redirect); `NOT_FOUND` for another member's target |
| **Side effects** | Read-only. `preselectedGroupId` is the `group` search param, but only if it is an `ACTIVE` membership (FR-VIS-01: the form is opened from a group view; otherwise none is preselected). |
| **Rate limit** | none |
| **FRs served** | FR-TGT-01, FR-TGT-02, FR-TGT-03, FR-TGT-04, FR-TGT-06, FR-VIS-01 |

`Template` is the static list in `core/templates` (BR-TPL): `{ id; label; mode; goal: number | null; unit: string | null; frequency; weekdays }`.

### L-07 `loadGroupOverview`

| | |
|---|---|
| **Type / file** | Page loader · `src/app/(app)/groups/[groupId]/page.tsx` → `src/features/heatmap/server/queries.ts` (with `src/features/rollcall` and `src/features/feed`) |
| **Auth** | `member` |
| **Input** | path `groupParams` |
| **Output** | `GroupOverview` (below), passed as `initialData` for `['group', id, 'overview']`; the `pulse` query is seeded from the same data |
| **Errors** | `UNAUTHENTICATED` (redirect), `NOT_FOUND` |
| **Side effects** | Reads like `R-02`, plus 52 weeks of entries for the group's shared targets. May upsert `streak_prefixes` (derived). |
| **Rate limit** | none |
| **FRs served** | FR-HM-02, FR-HM-03, FR-HM-04, FR-HM-05, FR-HM-06, FR-ROL-01, FR-ROL-02, FR-STK-01, FR-STK-03, FR-VIS-02, FR-VIS-03, FR-VIS-07, FR-FED-01, FR-FED-03, FR-FED-04, FR-REA-01, FR-REA-03, FR-GRP-11 |

```ts
type OverviewMember = UserRef & {
  role: 'OWNER' | 'MEMBER'
  todayLocal: LocalDate
  state: RollCallState
  done: number; scheduled: number
  streak: number
  cells: DayCell[]                   // same range and rules as L-04, group scope; dates after this member's own today are 'FUTURE'
}
type GroupOverview = PollHint & {
  group: { groupId: string; name: string; memberCount: number; viewerRole: 'OWNER' | 'MEMBER' }
  viewerToday: LocalDate
  members: OverviewMember[]          // default order (FR-HM-06)
  feed: { cards: FeedCardDTO[]; nextCursor: string | null }
}
```

**Default order (FR-HM-06):** current group streak descending, then last-7-day completion percentage descending, then display name ascending. The 7-day percentage is `done ÷ scheduled` over the member's last seven local days in group scope; a member with nothing scheduled in those days sorts after members who have a percentage. Only the overview computes the order; the poll never reorders.

**Roll call sections** (FR-ROL-01) are formed by the client from `state`, in the order Active, Not yet, Slacking, Rest day, No targets shared.

### L-08 `loadMembers`

| | |
|---|---|
| **Type / file** | Page loader · `src/app/(app)/groups/[groupId]/members/page.tsx` |
| **Auth** | `member` |
| **Input** | path `groupParams` |
| **Output** | `MembersPage` (below) |
| **Errors** | `UNAUTHENTICATED` (redirect), `NOT_FOUND` |
| **Side effects** | Read-only. |
| **Rate limit** | none |
| **FRs served** | FR-GRP-02, FR-GRP-07, FR-GRP-11 |

```ts
type MembersPage = {
  group: GroupRef
  viewerRole: 'OWNER' | 'MEMBER'
  invite: { code: string; url: string } | null     // owner only (FR-GRP-02)
  members: Array<UserRef & { role: 'OWNER' | 'MEMBER'; joinedAt: IsoInstant }>   // ACTIVE members only
  ownerless: boolean                                // no ACTIVE owner (the owner's account was deleted): owner controls are hidden for everyone (NFR-REL-02)
}
```

### L-09 `loadJoin`

| | |
|---|---|
| **Type / file** | Page loader · `src/app/(app)/join/[code]/page.tsx` |
| **Auth** | `onboarded` `session` (`P-01` redirects signed-out visitors to sign-in with this URL as the return path; the layout sends un-onboarded users through onboarding with `next`) |
| **Input** | path `{ code: string (max 64) }` |
| **Output** | `{ code: string }` (the raw path segment; normalisation and validity are decided by `A-04`) |
| **Errors** | `UNAUTHENTICATED` (redirect) |
| **Side effects** | **None.** A GET never writes, and nobody joins without a user action. The page shows a short invitation ("You've been invited to join a group on Pulse") and a single **Join group** button. The button is a client component that submits `A-04` when tapped, shows a pending state, then navigates to the group; an invalid code is reported by `A-04` as "This invite isn't valid." beside the button. Visitors and link-preview bots that are not signed in never reach this page. The page does not show the group's name: that preview is FR-GRP-12 (v1.1). |
| **Rate limit** | none |
| **FRs served** | FR-GRP-03, FR-ACC-01 |

### L-10 `loadProfile`

| | |
|---|---|
| **Type / file** | Page loader · `src/app/(app)/settings/profile/page.tsx` |
| **Auth** | `onboarded` `session` |
| **Input** | none |
| **Output** | `{ displayName: string; handle: string; timezone: string; imageUrl: string \| null }` |
| **Errors** | `UNAUTHENTICATED` (redirect) |
| **Side effects** | Read-only. The avatar is the Clerk-hosted image copied into `users.image_url`; changing it happens in Clerk's account settings and arrives through `R-04` (FR-ACC-04 third scenario). |
| **Rate limit** | none |
| **FRs served** | FR-ACC-04 |

---

## 6. Proxy and metadata

### P-01 `proxy`

| | |
|---|---|
| **Type / file** | Proxy (Next.js 16's name for middleware) · `src/proxy.ts` |
| **Auth** | `clerkMiddleware`; an optimistic check only (every server entry point re-verifies) |
| **Output** | pass-through, or redirect to sign-in with the return URL |
| **Side effects** | none |
| **Rate limit** | none |
| **FRs served** | FR-ACC-01, FR-GRP-03 (invite link survives sign-in) |

**Public routes** (no session): `/sign-in(.*)`, `/sign-up(.*)`, `/api/webhooks/clerk`, **`/manifest.webmanifest`, `/icons/*`, `/apple-icon*` and `/favicon.ico`**. The manifest and icons must be public: browsers fetch the manifest without credentials, so a protected manifest would break installability (FR-PWA-01); this is an addition to TAD §4.1 (DB_SCHEMA §9). Everything else calls `auth.protect()`. The matcher excludes `_next` and static files.

### M-01 `manifest` and icons

| | |
|---|---|
| **Type / file** | Metadata routes · `src/app/manifest.ts` (served at `/manifest.webmanifest`), `src/app/apple-icon.png` (180×180), `public/icons/icon-192.png`, `icon-512.png`, `icon-512-maskable.png` |
| **Auth** | `public` (see `P-01`) |
| **Output** | a Web App Manifest |
| **Side effects** | none |
| **Rate limit** | none |
| **FRs served** | FR-PWA-01, FR-PWA-03 |

```ts
// src/app/manifest.ts
export default function manifest(): MetadataRoute.Manifest {
  return {
    name: 'Pulse',
    short_name: 'Pulse',
    start_url: '/today',
    display: 'standalone',
    background_color: '#ffffff',
    theme_color: '#10b981',
    icons: [
      { src: '/icons/icon-192.png', sizes: '192x192', type: 'image/png' },
      { src: '/icons/icon-512.png', sizes: '512x512', type: 'image/png' },
      { src: '/icons/icon-512-maskable.png', sizes: '512x512', type: 'image/png', purpose: 'maskable' },
    ],
  }
}
```

The dark-mode theme colour is supplied by `<meta name="theme-color" media="(prefers-color-scheme: dark)">` from the root layout's `viewport` export, because a manifest holds a single colour. No service worker in v1.0 (TAD §8.5).

---

## 7. Coverage check

The matrix below is generated from the "FRs served" rows of §3–§6 by a script and compared with the 58 v1.0 requirements of PRD rev 2.

<!-- BEGIN COVERAGE MATRIX -->
| FR | Title | Served by |
|---|---|---|
| FR-ACC-01 | Sign up and sign in | A-04, L-01, L-09, P-01 |
| FR-ACC-02 | Immediately usable after sign-up | R-04, L-01 |
| FR-ACC-03 | First-run profile | A-01, L-02 |
| FR-ACC-04 | Edit profile | A-02, R-04, L-10 |
| FR-ACC-05 | Timezone change | A-02 |
| FR-ACC-06 | Sign out | L-01 |
| FR-GRP-01 | Create a group | A-03 |
| FR-GRP-02 | Invite code and link | A-03, L-08 |
| FR-GRP-03 | Join a group | A-04, L-09, P-01 |
| FR-GRP-05 | Group switcher | A-05, L-01 |
| FR-GRP-07 | Member list and removal | A-06, L-08 |
| FR-GRP-11 | Membership is verified on every action | A-04, R-02, R-03, L-07, L-08 |
| FR-TGT-01 | Create a target from a template or from scratch | A-07, L-06 |
| FR-TGT-02 | Tracking modes and goals | A-07, A-08, L-06 |
| FR-TGT-03 | Frequency: daily or specific days | A-07, A-08, L-06 |
| FR-TGT-04 | Start date | A-07, L-06 |
| FR-TGT-06 | Edit a target | A-08, L-05, L-06 |
| FR-TGT-07 | Archive a target | A-09, L-05 |
| FR-TGT-08 | Delete a target with no history | A-10, L-05 |
| FR-VIS-01 | Choose which groups can see a target | A-07, A-11, L-05, L-06 |
| FR-VIS-02 | Unshared targets are private | A-11, L-07 |
| FR-VIS-03 | Group views count only shared targets | R-02, L-07 |
| FR-VIS-04 | Sharing is not retroactive | A-07, A-11 |
| FR-VIS-05 | Unsharing keeps the past | A-11 |
| FR-VIS-06 | Re-sharing | A-11 |
| FR-VIS-07 | Two streaks, clearly labelled | R-01, R-02, L-03, L-07 |
| FR-CHK-01 | Today's checklist | R-01, L-03 |
| FR-CHK-02 | Check off a checkbox task | A-12 |
| FR-CHK-03 | Count tasks | A-12 |
| FR-CHK-04 | Duration tasks | A-12 |
| FR-CHK-05 | Completion and over-achievement | A-12 |
| FR-CHK-06 | Edit window: today and yesterday | A-12, R-01 |
| FR-CHK-07 | Missed days need no action | R-01, L-03 |
| FR-CHK-08 | Reliable saving | A-12 |
| FR-CHK-09 | Same data on every device | A-12, R-01 |
| FR-CHK-10 | Empty and first-run states | R-01, L-03 |
| FR-CHK-11 | Daily summary | R-01, L-03 |
| FR-HM-01 | Personal heatmap layout | L-04 |
| FR-HM-02 | Cell states and levels | L-04, L-07 |
| FR-HM-03 | Tooltip | L-04, L-07 |
| FR-HM-04 | Accessibility and themes | L-04, L-07 |
| FR-HM-05 | Group comparison view | R-02, L-07 |
| FR-HM-06 | Default order | L-07 |
| FR-STK-01 | Current streak | R-01, R-02, L-03, L-07 |
| FR-STK-02 | Retroactive edits recalculate | A-12, R-02 |
| FR-STK-03 | Today never breaks a streak | R-01, R-02, L-07 |
| FR-ROL-01 | Roll call per group | R-02, L-07 |
| FR-ROL-02 | Cutoff | R-02, L-07 |
| FR-ROL-03 | Status changes as they act | R-02 |
| FR-FED-01 | Daily progress card | A-12, R-02, L-07 |
| FR-FED-03 | Order and paging | R-03, L-07 |
| FR-FED-04 | Feed respects sharing and membership | A-11, R-02, R-03, L-07 |
| FR-FED-05 | New activity appears without reloading | R-02 |
| FR-REA-01 | React to a card | A-13, R-02, L-07 |
| FR-REA-03 | Reactions follow their card | A-13, R-02, R-03, L-07 |
| FR-PWA-01 | Installable | M-01 |
| FR-PWA-03 | Installed behaviour | M-01 |
| FR-PWA-05 | Works without installing | **no surface** (whole-app property, see 7.1) |
<!-- END COVERAGE MATRIX -->

### 7.1 Results

<!-- BEGIN COVERAGE RESULTS -->
- **v1.0 requirements:** 58. **Surfaces:** 29 (13 actions, 4 route handlers, 10 page loaders, 1 proxy, 1 metadata route).
- **v1.0 requirements not served by any endpoint, action or page:** **FR-PWA-05**: a property of every page, with no endpoint, action or loader behind it. It is satisfied by never requiring installation and by not depending on a service worker (TAD §1.4), and it is verified by test-plan item TP-08 (TAD §16), which runs the whole flow in an ordinary browser tab, not by a surface.
- **Surfaces that serve no v1.0 requirement:** **none**.
- **Surfaces citing an unknown or v1.1 requirement:** **none**.
- **Served, but the behaviour lives partly outside the surface** (in client components or shared code; the surfaces in the matrix only carry the data or the guard):
  - FR-ACC-06 (sign-out): the Clerk user menu in the shell (`L-01`); the client clears the query cache.
  - FR-CHK-05 (completion): the rule `value >= goal` lives in `core` and is applied by `A-12`, `R-01`, `R-02` and the loaders.
  - FR-CHK-08 (reliable saving): `A-12` returns a `Result`; the rollback, retry and toast are client behaviour (TAD §7.4).
  - FR-HM-03, FR-HM-04 (tooltip, accessibility): client components fed by `L-04` and `L-07` (TAD §8).
  - FR-GRP-11 (membership on every action): cross-cutting, enforced by `requireGroupAccess` in every `member` or `owner` surface; listed on the main ones.
<!-- END COVERAGE RESULTS -->

---

## 8. Refinements to the TAD, and notes for PRD rev 3

### 8.1 Refinements to the TAD (approved 2026-10-27; applied in TAD rev 2)

Found while specifying the surfaces. None changes a PRD requirement. The schema-level refinements are in `DB_SCHEMA.md` §9.

1. **`changeTimezone` is part of `updateProfile`** (`A-02`). TAD §5.4 and §7.5 name it as a separate mutation. One action saves the profile form atomically and keeps the BR-DAY-5 floor update in the same transaction.
2. **The join page performs no write, and nobody joins without a user action.** TAD §4.2 showed the join happening during the `GET /join/…` request. Now `L-09` shows a one-tap **Join group** button that submits `A-04` (a POST): the visitor never re-enters the code (FR-ACC-01) but must tap once. A "Join with a code" form on `/groups/new` serves codes typed by hand (FR-GRP-03).
3. **`Pulse` shape** (TAD §7.2). Each member's `recent` holds `DayCell`s (with `state`) instead of a `tracked` flag, so the client never re-derives states (NFR-QUAL-02). `pollHintMs` may be `null` (level L2). The poll never reorders members; the overview owns the default order (FR-HM-06).
4. **Fixed hint values per `PULSE_POLL_MODE`:** L0 = 20 s and 3 minutes, L1 = 20 s and 1 minute, L2 = no background refreshing (`R-02`).
5. **`OUT_OF_WINDOW` is defensive only.** The day token cannot express a date older than yesterday, so only a repository guard raises it; TAD §9.1 listed it as an ordinary user-facing code.
6. **Two loaders on `/today`.** The personal heatmap (`L-04`) is its own loader and Suspense boundary so the checklist never waits for the 52-week read. TAD §3 listed only a Today page.
7. **New shared module** `src/lib/primitives.ts` and per-feature `schema.ts` files (§1.4) are additions to the TAD §3 tree.
8. **Onboarding `next` is validated** as a same-site path (`nextPath`): a security detail the TAD did not spell out.
9. **Webhook display name** is `first_name` plus `last_name`, else `username`, else "New member" (`R-04`); the webhook never overwrites the member's handle.

### 8.2 Notes for PRD rev 3 (recorded, not applied; the PRD is not edited)

1. **FR-HM-06, ranking.** The PRD orders comparison rows by current group streak, then last-7-day completion percentage, then name, but does not say where a member with nothing scheduled in the last seven local days belongs, because their percentage is undefined. **Decision (approved 2026-10-27):** they sort after every member who has a percentage, then by name. The poll never reorders rows, so the order is fixed by the overview (`L-07`). Add this sentence to FR-HM-06.
2. **FR-ACC-01 (second scenario) and journey J1 step 4** say the visitor returns to the invite and "is joined". The approved flow puts one tap on **Join group** between returning and being a member, and nothing joins without a user action. Reword both to "returns to the invite page and joins with one tap". The 60-second measure of goal G3 is unaffected.
3. **FR-GRP-03, hand-typed codes.** They are served by a "Join with a code" form on `/groups/new` (`A-04`). The PRD does not say where that form lives; name the screen in rev 3.

---

## 9. Reserved for v1.1 (names only; not specified here)

| Surface (proposed name) | v1.1 requirement(s) | Schema support already present |
|---|---|---|
| `renameGroup`, `regenerateInvite`, `deleteGroup`, `transferOwnership`, `leaveGroup` | FR-GRP-04, 06, 08, 09, 10 | `groups.name`, `groups.invite_code`, `groups.deleted_at`, `memberships.role`, `MembershipStatus.LEFT` |
| `loadInvitePreview` (public) | FR-GRP-12 | none needed |
| `A-04` gains `RATE_LIMITED`; group and target count limits | FR-GRP-13, FR-TGT-05 | counters need one new table |
| `deleteAccount` completion (erasure, owner succession) | FR-ACC-07 | `users.deleted_at`, tombstone already written |
| `restoreTarget`, `pauseTarget`, `updatePause`, future start date in `A-07` | FR-TGT-09/10, FR-PAU-* | `target_inactive_periods` (`PAUSED`), `targets.start_date` |
| `sendNudge`, `markNotificationsRead`, `GET` notifications; `unread` in `Pulse` | FR-NUD-*, FR-NOT-* | new tables |
| Weekly quota, shared goals, weekly recap | FR-QTA-*, FR-SGL-*, FR-REC-* | enum value + nullable columns; new tables; recap is derived |
| Milestone and joined cards, longest streak | FR-STK-04/05, FR-FED-02 | enum values, nullable columns |
| Who-reacted list, sort controls, arrow keys, install hint, offline screen | FR-REA-02, FR-HM-07/08, FR-PWA-02/04 | none needed or client-only |

---

## 10. Verification

Run on 2026-10-27 in an isolated scratch directory (nothing was added to the repository): Node 22.23.2, `zod` 4.6.7, and `typescript` 6.0.3 used only to strip types from the code blocks.

| Check | Result |
|---|---|
| The 17 `// @zod` blocks transpile and load under Zod 4.6.7 | pass |
| 83 sample-input assertions (valid inputs accepted, invalid rejected at the expected field path) | 83 of 83 pass |
| Coverage matrix generated from the "FRs served" rows; compared with the 58 v1.0 requirements of PRD rev 2 | 57 served by a surface; FR-PWA-05 is the one whole-app property (§7.1) |
| Every surface serves at least one v1.0 requirement; none cites a v1.1 or unknown requirement | pass (29 surfaces) |
| The catalog in §2 lists exactly the surfaces specified | pass (29 of 29) |

What the assertions covered: every shared primitive (including an impossible calendar date, offset-style and legacy timezone names, and open-redirect paths); the target rules at every boundary (COUNT 1 and 50, DURATION 5 to 480 in steps of 5, CHECKBOX with and without a goal, weekday duplicates, all seven weekdays becoming DAILY); the progress input boundaries (0, 1440, 1441, fractions, bad day token, bad date); the Clerk event union (valid events, extra fields stripped, unknown event types not parsed).

**Not verified** (the application does not exist yet): the behaviour of any action, route or loader; the Clerk payload field names (spike S5); the real `isIanaZone` (the check used an equivalent stub); and every file path, which is design. At implementation, the `// @zod` blocks should become the schema files verbatim, and the test plan should keep them in step with this document.
