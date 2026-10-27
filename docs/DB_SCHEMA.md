# Pulse — Database Schema

| | |
|---|---|
| **Status** | Approved (rev 1) |
| **Date** | 2026-10-27 |
| **Inputs** | `docs/PRD.md` (rev 3), `docs/TAD.md`, `docs/adr/0001`–`0006` |
| **Companion** | `docs/API_SPEC.md` (surface IDs such as `A-12` refer to it) |
| **Next documents** | Tasks + test plan |

**Scope.** The complete draft `schema.prisma` (Prisma 7), the hand-written SQL that Prisma's schema language cannot express, and, for every model, its purpose, key fields, invariants, indexes (each tied to the query it serves), lifecycle behaviour, and which TAD algorithm reads and writes it. The schema does not depend on where the datasource URL is configured (spike S1): in Prisma 7 the URL lives in `prisma.config.ts`, so the `datasource` block below contains only the provider.

**Verified.** §10 records what was actually run against this schema (`prisma validate`, the generated DDL plus the hand-written SQL executed on an in-memory PostgreSQL, and constraint tests).

---

## 1. Conventions

| Topic | Convention |
|---|---|
| **Names** | Prisma models are `PascalCase` singular and map to `snake_case` plural tables with `@@map`; fields are `camelCase` and map to `snake_case` columns with `@map`. SQL below uses the table and column names. |
| **Ids** | `uuid(7)` (time-ordered), stored as native `uuid` (`@db.Uuid`). Join-style tables use composite primary keys instead of a surrogate id: `entries`, `target_revisions`, `reactions`, `streak_prefixes`. |
| **Local calendar days** | `DateTime @db.Date` (a `date` column). Prisma returns it as a JS `Date` at UTC midnight, so **only the repository layer converts** to and from the branded `LocalDate` string (`'YYYY-MM-DD'`) (ADR-0005). Never use these columns with timestamps. |
| **Instants** | `DateTime @db.Timestamptz(3)`. Used only for ordering and audit (`created_at`, `updated_at`, `last_activity_at`, `deleted_at`, `ended_at`), never for day logic. |
| **Enums** | Native Postgres enums. v1.0 declares exactly the values it needs plus the v1.1-ready ones the TAD names (`LEFT`, `PAUSED`). Adding a value later is an additive migration. |
| **Delete policy** | `Restrict` on every reference to `users`, `groups` and `entries`, so nothing in v1.0 can hard-delete history. `Cascade` only where the child is meaningless without the parent: a target's own rule/period/interval rows, a card's reactions, and derived caches. §6 explains every case. |
| **Hand-written SQL** | Partial unique indexes, exclusion constraints, `CHECK`s. They go in a second migration created with `--create-only` after the generated one (§3). Non-overlap has a tested fallback if `btree_gist` is ever unavailable (§3.1). |
| **Soft state** | `users.deleted_at` and `groups.deleted_at` exist from day one. Rows are retired, not erased (PRD §9.2 note 3). |

---

## 2. `schema.prisma`

```prisma
// prisma/schema.prisma  (Prisma 7.10.1)

generator client {
  provider = "prisma-client"
  output   = "../src/generated/prisma"
}

datasource db {
  provider = "postgresql"
  // Connection URLs are configured in prisma.config.ts, not here (Prisma 7; spike S1).
}

// ───────────────────────────── enums ─────────────────────────────

enum Role {
  OWNER
  MEMBER
}

enum MembershipStatus {
  ACTIVE
  REMOVED
  LEFT // defined now, written from v1.1 (leave group)
}

enum TargetMode {
  CHECKBOX
  COUNT
  DURATION
}

enum Frequency {
  DAILY
  SPECIFIC_DAYS
  // WEEKLY_QUOTA is added in v1.1 (additive enum value)
}

enum InactiveKind {
  ARCHIVED // not tracked: the days do not exist for this target
  PAUSED // tracked, nothing scheduled; defined now, written from v1.1
}

enum FeedCardKind {
  DAILY_PROGRESS
  // MILESTONE and JOINED are added in v1.1 (additive enum values)
}

enum ReactionEmoji {
  FIRE // 🔥
  MUSCLE // 💪
  CLAP // 👏
  LAUGH // 😂
  EYES // 👀
}

// ───────────────────────────── identity and groups ─────────────────────────────

model User {
  id          String    @id @default(uuid(7)) @db.Uuid
  clerkId     String    @unique @map("clerk_id") @db.VarChar(64)
  handle      String    @unique @db.VarChar(20)
  displayName String    @map("display_name") @db.VarChar(40)
  imageUrl    String?   @map("image_url")
  timezone    String    @default("UTC") @db.VarChar(64)
  dateFloor   DateTime? @map("date_floor") @db.Date
  onboardedAt DateTime? @map("onboarded_at") @db.Timestamptz(3)
  deletedAt   DateTime? @map("deleted_at") @db.Timestamptz(3)
  createdAt   DateTime  @default(now()) @map("created_at") @db.Timestamptz(3)
  updatedAt   DateTime  @default(now()) @updatedAt @map("updated_at") @db.Timestamptz(3)

  memberships    Membership[]
  targets        Target[]
  feedCards      FeedCard[]
  reactions      Reaction[]
  streakPrefixes StreakPrefix[]

  @@map("users")
}

model Group {
  id         String    @id @default(uuid(7)) @db.Uuid
  name       String    @db.VarChar(40)
  inviteCode String    @unique @map("invite_code") @db.Char(8)
  createdAt  DateTime  @default(now()) @map("created_at") @db.Timestamptz(3)
  updatedAt  DateTime  @default(now()) @updatedAt @map("updated_at") @db.Timestamptz(3)
  deletedAt  DateTime? @map("deleted_at") @db.Timestamptz(3)

  memberships    Membership[]
  shareIntervals ShareInterval[]
  feedCards      FeedCard[]
  streakPrefixes StreakPrefix[]

  @@map("groups")
}

model Membership {
  id        String           @id @default(uuid(7)) @db.Uuid
  userId    String           @map("user_id") @db.Uuid
  groupId   String           @map("group_id") @db.Uuid
  role      Role             @default(MEMBER)
  status    MembershipStatus @default(ACTIVE)
  joinedAt  DateTime         @default(now()) @map("joined_at") @db.Timestamptz(3)
  endedAt   DateTime?        @map("ended_at") @db.Timestamptz(3)

  user  User  @relation(fields: [userId], references: [id], onDelete: Restrict)
  group Group @relation(fields: [groupId], references: [id], onDelete: Restrict)

  @@unique([userId, groupId])
  @@index([groupId, status])
  @@index([userId, status])
  @@map("memberships")
}

// ───────────────────────────── targets and their history ─────────────────────────────

model Target {
  id        String     @id @default(uuid(7)) @db.Uuid
  ownerId   String     @map("owner_id") @db.Uuid
  name      String     @db.VarChar(60)
  mode      TargetMode
  unit      String?    @db.VarChar(20)
  startDate DateTime   @map("start_date") @db.Date
  createdAt DateTime   @default(now()) @map("created_at") @db.Timestamptz(3)
  updatedAt DateTime   @default(now()) @updatedAt @map("updated_at") @db.Timestamptz(3)

  owner           User                   @relation(fields: [ownerId], references: [id], onDelete: Restrict)
  revisions       TargetRevision[]
  inactivePeriods TargetInactivePeriod[]
  shareIntervals  ShareInterval[]
  entries         Entry[]

  @@index([ownerId, createdAt])
  @@map("targets")
}

// The rules in force from a date. The first row has effectiveFrom = target.startDate.
model TargetRevision {
  targetId      String    @map("target_id") @db.Uuid
  effectiveFrom DateTime  @map("effective_from") @db.Date
  goal          Int? // null for CHECKBOX; COUNT 1–50; DURATION 5–480 in steps of 5 (validated in the app)
  frequency     Frequency
  weekdays      Int[]     @default([]) // ISO weekdays 1 = Monday … 7 = Sunday; empty for DAILY
  createdAt     DateTime  @default(now()) @map("created_at") @db.Timestamptz(3)

  target Target @relation(fields: [targetId], references: [id], onDelete: Cascade)

  @@id([targetId, effectiveFrom])
  @@map("target_revisions")
}

// Periods when a target is not scheduled. v1.0 writes ARCHIVED only; v1.1 adds PAUSED (and restore) with no migration.
model TargetInactivePeriod {
  id        String       @id @default(uuid(7)) @db.Uuid
  targetId  String       @map("target_id") @db.Uuid
  kind      InactiveKind
  fromDate  DateTime     @map("from_date") @db.Date
  toDate    DateTime?    @map("to_date") @db.Date // inclusive; null = open-ended
  createdAt DateTime     @default(now()) @map("created_at") @db.Timestamptz(3)

  target Target @relation(fields: [targetId], references: [id], onDelete: Cascade)

  @@index([targetId, kind, fromDate])
  @@map("target_inactive_periods")
}

// When a target was visible to a group. Both ends inclusive; toDate null = still shared.
model ShareInterval {
  id        String    @id @default(uuid(7)) @db.Uuid
  targetId  String    @map("target_id") @db.Uuid
  groupId   String    @map("group_id") @db.Uuid
  fromDate  DateTime  @map("from_date") @db.Date
  toDate    DateTime? @map("to_date") @db.Date
  createdAt DateTime  @default(now()) @map("created_at") @db.Timestamptz(3)

  target Target @relation(fields: [targetId], references: [id], onDelete: Cascade)
  group  Group  @relation(fields: [groupId], references: [id], onDelete: Restrict)

  @@index([groupId, fromDate])
  @@index([targetId, groupId, fromDate])
  @@map("share_intervals")
}

// Progress: one row per target and local date. No row means no progress.
model Entry {
  targetId  String   @map("target_id") @db.Uuid
  localDate DateTime @map("local_date") @db.Date
  value     Int // 1–1440; COUNT cap 999 and DURATION cap 1440 are enforced by the action
  updatedAt DateTime @default(now()) @updatedAt @map("updated_at") @db.Timestamptz(3)

  target Target @relation(fields: [targetId], references: [id], onDelete: Restrict)

  @@id([targetId, localDate])
  @@map("entries")
}

// ───────────────────────────── feed ─────────────────────────────

// One card per member, per group, per local day. It stores no content; the body is derived on read.
model FeedCard {
  id             String       @id @default(uuid(7)) @db.Uuid
  groupId        String       @map("group_id") @db.Uuid
  userId         String       @map("user_id") @db.Uuid
  localDate      DateTime     @map("local_date") @db.Date
  kind           FeedCardKind @default(DAILY_PROGRESS)
  lastActivityAt DateTime     @map("last_activity_at") @db.Timestamptz(3)
  createdAt      DateTime     @default(now()) @map("created_at") @db.Timestamptz(3)

  group     Group      @relation(fields: [groupId], references: [id], onDelete: Restrict)
  user      User       @relation(fields: [userId], references: [id], onDelete: Restrict)
  reactions Reaction[]

  @@unique([groupId, userId, localDate, kind])
  @@index([groupId, lastActivityAt(sort: Desc), id(sort: Desc)])
  @@map("feed_cards")
}

model Reaction {
  cardId    String        @map("card_id") @db.Uuid
  userId    String        @map("user_id") @db.Uuid
  emoji     ReactionEmoji
  createdAt DateTime      @default(now()) @map("created_at") @db.Timestamptz(3)

  card FeedCard @relation(fields: [cardId], references: [id], onDelete: Cascade)
  user User     @relation(fields: [userId], references: [id], onDelete: Restrict)

  @@id([cardId, userId, emoji])
  @@map("reactions")
}

// ───────────────────────────── derived data and operations ─────────────────────────────

// Derived cache: the group-scoped streak run through the last immutable day. Safe to delete at any time (TAD §6.4).
model StreakPrefix {
  userId      String   @map("user_id") @db.Uuid
  groupId     String   @map("group_id") @db.Uuid
  throughDate DateTime @map("through_date") @db.Date
  run         Int
  algoVersion Int      @map("algo_version")
  computedAt  DateTime @default(now()) @map("computed_at") @db.Timestamptz(3)

  user  User  @relation(fields: [userId], references: [id], onDelete: Cascade)
  group Group @relation(fields: [groupId], references: [id], onDelete: Cascade)

  @@id([userId, groupId])
  @@index([groupId])
  @@map("streak_prefixes")
}

// Application error record (NFR-OPS-01). Deliberately no relations: it must survive any change to users or groups.
model ErrorEvent {
  id          String   @id @default(uuid(7)) @db.Uuid
  at          DateTime @default(now()) @db.Timestamptz(3)
  source      String   @db.VarChar(64) // e.g. "action:recordProgress", "route:pulse", "webhook:clerk"
  code        String   @db.VarChar(32)
  userId      String?  @map("user_id") @db.Uuid
  message     String   @db.VarChar(500)
  fingerprint String   @db.VarChar(64)

  @@index([at])
  @@index([fingerprint, at])
  @@map("error_events")
}
```

---

## 3. Hand-written SQL (second migration, `prisma migrate dev --create-only`)

Everything here is invisible to Prisma's schema language. It is applied after the generated migration and committed with it.

```sql
-- prisma/migrations/<timestamp>_constraints/migration.sql

CREATE EXTENSION IF NOT EXISTS btree_gist;   -- needed for the exclusion constraints; Neon documents it as supported (§3.1)

-- users ---------------------------------------------------------------------------------------
ALTER TABLE users ADD CONSTRAINT users_handle_format
  CHECK (handle ~ '^[a-z][a-z0-9_]{2,19}$');                      -- BR-ID; stored lowercase, so plain UNIQUE is case-insensitive
ALTER TABLE users ADD CONSTRAINT users_display_name_not_blank
  CHECK (length(btrim(display_name)) > 0);
ALTER TABLE users ADD CONSTRAINT users_timezone_not_blank
  CHECK (length(btrim(timezone)) > 0);

-- groups --------------------------------------------------------------------------------------
ALTER TABLE groups ADD CONSTRAINT groups_invite_code_format
  CHECK (invite_code ~ '^[A-HJKMNP-Z2-9]{8}$');                   -- BR-GROUP-4: A–Z without I, L, O, plus 2–9
ALTER TABLE groups ADD CONSTRAINT groups_name_not_blank
  CHECK (length(btrim(name)) > 0);

-- memberships ---------------------------------------------------------------------------------
ALTER TABLE memberships ADD CONSTRAINT memberships_ended_consistent
  CHECK ((status = 'ACTIVE' AND ended_at IS NULL) OR (status <> 'ACTIVE' AND ended_at IS NOT NULL));
CREATE UNIQUE INDEX memberships_one_active_owner
  ON memberships (group_id) WHERE role = 'OWNER' AND status = 'ACTIVE';   -- zero is allowed (ownerless group, NFR-REL-02)

-- targets and rules ---------------------------------------------------------------------------
ALTER TABLE targets ADD CONSTRAINT targets_name_not_blank
  CHECK (length(btrim(name)) > 0);
ALTER TABLE target_revisions ADD CONSTRAINT target_revisions_goal_range
  CHECK (goal IS NULL OR goal BETWEEN 1 AND 480);
ALTER TABLE target_revisions ADD CONSTRAINT target_revisions_weekdays_match_frequency
  CHECK ((frequency = 'DAILY' AND cardinality(weekdays) = 0)
      OR (frequency = 'SPECIFIC_DAYS' AND cardinality(weekdays) BETWEEN 1 AND 6
          AND weekdays <@ ARRAY[1,2,3,4,5,6,7]));

-- inactive periods ----------------------------------------------------------------------------
ALTER TABLE target_inactive_periods ADD CONSTRAINT target_inactive_periods_range
  CHECK (to_date IS NULL OR from_date <= to_date);
ALTER TABLE target_inactive_periods ADD CONSTRAINT target_inactive_periods_no_overlap
  EXCLUDE USING gist (target_id WITH =, kind WITH =, daterange(from_date, to_date, '[]') WITH &&);

-- share intervals -----------------------------------------------------------------------------
ALTER TABLE share_intervals ADD CONSTRAINT share_intervals_range
  CHECK (to_date IS NULL OR from_date <= to_date);
ALTER TABLE share_intervals ADD CONSTRAINT share_intervals_no_overlap
  EXCLUDE USING gist (target_id WITH =, group_id WITH =, daterange(from_date, to_date, '[]') WITH &&);
CREATE UNIQUE INDEX share_intervals_one_open
  ON share_intervals (target_id, group_id) WHERE to_date IS NULL;           -- implied by the exclusion; kept for clear errors and lookups
CREATE INDEX share_intervals_open_by_group
  ON share_intervals (group_id) WHERE to_date IS NULL;

-- entries -------------------------------------------------------------------------------------
ALTER TABLE entries ADD CONSTRAINT entries_value_range
  CHECK (value BETWEEN 1 AND 1440);                               -- zero is stored as a deleted row
```

The group cap of 15 is **not** a constraint: it is enforced by the join transaction, which locks the group row (`SELECT … FOR UPDATE`, issued as raw SQL because Prisma has no row-lock API) and then counts `ACTIVE` memberships (TAD §4.2, API `A-04`).

### 3.1 If `btree_gist` is unavailable: the fallback

**Finding.** Neon's [supported extensions page](https://neon.com/docs/extensions/pg-extensions) (checked 2026-10-27, read through a summarising fetch tool) lists `btree_gist` as supported: version 1.6 on Postgres 14, 1.7 on 15 to 17, 1.8 on 18. It makes no Free-plan distinction for `btree_gist` (it flags only `pg_repack` as paid-only) and lists no special enabling step beyond `CREATE EXTENSION`. **So the exclusion constraints in §3 remain the primary design.** Spike S1 still runs `CREATE EXTENSION btree_gist` on the real project to confirm.

**Fallback.** If the extension were ever unavailable (or the database moved to a host without it), non-overlap is enforced by a **trigger that serialises writers with a transaction-scoped advisory lock** and then checks for an overlapping row. Apply §3 with the `CREATE EXTENSION` line and the two `EXCLUDE` constraints removed, plus this:

```sql
-- FALLBACK: non-overlap without btree_gist (replaces the two EXCLUDE constraints of §3)

CREATE FUNCTION share_intervals_no_overlap() RETURNS trigger LANGUAGE plpgsql AS $$
BEGIN
  -- One writer at a time per (target, group); the lock is released at commit or rollback.
  PERFORM pg_advisory_xact_lock(hashtextextended('share_intervals:' || NEW.target_id::text || ':' || NEW.group_id::text, 0));
  IF EXISTS (
    SELECT 1 FROM share_intervals s
    WHERE s.target_id = NEW.target_id AND s.group_id = NEW.group_id AND s.id <> NEW.id
      AND daterange(s.from_date, s.to_date, '[]') && daterange(NEW.from_date, NEW.to_date, '[]')
  ) THEN
    RAISE EXCEPTION 'share_intervals_no_overlap: overlapping interval for target % and group %', NEW.target_id, NEW.group_id
      USING ERRCODE = '23P01';                                   -- exclusion_violation, the code the constraint raises
  END IF;
  RETURN NEW;
END
$$;

CREATE TRIGGER share_intervals_no_overlap
  BEFORE INSERT OR UPDATE OF from_date, to_date ON share_intervals
  FOR EACH ROW EXECUTE FUNCTION share_intervals_no_overlap();

CREATE FUNCTION target_inactive_periods_no_overlap() RETURNS trigger LANGUAGE plpgsql AS $$
BEGIN
  PERFORM pg_advisory_xact_lock(hashtextextended('target_inactive_periods:' || NEW.target_id::text || ':' || NEW.kind::text, 0));
  IF EXISTS (
    SELECT 1 FROM target_inactive_periods p
    WHERE p.target_id = NEW.target_id AND p.kind = NEW.kind AND p.id <> NEW.id
      AND daterange(p.from_date, p.to_date, '[]') && daterange(NEW.from_date, NEW.to_date, '[]')
  ) THEN
    RAISE EXCEPTION 'target_inactive_periods_no_overlap: overlapping % period for target %', NEW.kind, NEW.target_id
      USING ERRCODE = '23P01';
  END IF;
  RETURN NEW;
END
$$;

CREATE TRIGGER target_inactive_periods_no_overlap
  BEFORE INSERT OR UPDATE OF from_date, to_date, kind ON target_inactive_periods
  FOR EACH ROW EXECUTE FUNCTION target_inactive_periods_no_overlap();
```

How it behaves:

- **Same key, same rule.** The lock key is the pair the constraint covered: `(target, group)` for share intervals, `(target, kind)` for inactive periods. A concurrent writer for the same key waits for the first transaction to finish, and its overlap query then runs after the wait, so it sees the committed row. This relies on **READ COMMITTED**, which is Prisma's default on PostgreSQL; these writes must never run under REPEATABLE READ or SERIALIZABLE.
- **Same error.** It raises SQLSTATE `23P01` (`exclusion_violation`), the code the constraint raises, so the application's error mapping is identical in both designs.
- **Everything else in §3 stays in force**: the partial unique index `share_intervals_one_open`, the `CHECK`s, the owner index. The application discipline is unchanged: every write goes through `share`, `unshare`, `onMemberRemoved` and archive, inside a transaction (TAD §6.2).
- **A weaker last resort**, if triggers are also unwelcome: those functions first take a row lock on the target (`SELECT … FROM targets WHERE id = $1 FOR UPDATE`) and then run the same overlap query before writing. It protects only code that follows the discipline, not a stray script, so the trigger is preferred.
- **What was tested** (§10): the sequential behaviour on PostgreSQL with no extension loaded, with the same SQLSTATE as the primary design. A two-connection race cannot be exercised in the in-memory database used for the checks, so it is requirement **TP-09** in TAD §16, needed only if this fallback is ever adopted.

---

## 4. Models

IDs in **Written by** and **Read by** refer to `docs/API_SPEC.md` (`A-` actions, `R-` route handlers, `L-` loaders, `P-` proxy). Algorithm names refer to TAD sections.

### 4.1 `users`

- **Purpose.** The application profile of every Clerk user. The only place profile data lives; no email is stored (NFR-PRIV-01).
- **Key fields.** `clerk_id` (identity link), `handle`, `display_name`, `image_url`, `timezone` (IANA name), `date_floor` (BR-DAY-5), `onboarded_at`, `deleted_at`.
- **Invariants.**
  1. `handle` is always lowercase and matches `^[a-z][a-z0-9_]{2,19}$`, and is unique. (Storing lowercase makes the plain unique index case-insensitive, FR-ACC-04.)
  2. `clerk_id` is unique and never reused. A tombstoned row **keeps** its `clerk_id`, so a late webhook cannot recreate the user.
  3. If `deleted_at` is set: `display_name = 'Former member'`, `image_url` is null, `handle = 'deleted_' + first 12 hex digits of id`, and the user has no `ACTIVE` membership. Written only by the tombstone transaction (§6).
  4. `date_floor` is written only when the timezone changes. The member's effective today is `max(date_floor, calendar date in timezone)`.
  5. `onboarded_at` is null until the first profile confirmation; the app shell redirects to onboarding until it is set.
  6. Rows are never hard-deleted in v1.0 (every reference is `Restrict`).

### 4.2 `groups`

- **Purpose.** A friend group and its invite code.
- **Key fields.** `name`, `invite_code` (8 characters, upper case), `deleted_at`.
- **Invariants.**
  1. `invite_code` is unique and matches `^[A-HJKMNP-Z2-9]{8}$`.
  2. At most 15 `ACTIVE` memberships, enforced by the join transaction under a row lock (§3).
  3. `deleted_at` stays null in v1.0 (delete group is v1.1). When set, `requireGroupAccess` treats the group as nonexistent.
  4. Never hard-deleted.

### 4.3 `memberships`

- **Purpose.** Who belongs to which group, in which role and state. The root of every authorization check.
- **Key fields.** `role` (`OWNER`, `MEMBER`), `status` (`ACTIVE`, `REMOVED`, `LEFT`), `joined_at` (first join; unchanged on rejoin), `ended_at`.
- **Invariants.**
  1. One row per `(user, group)`. Rejoining flips the same row back to `ACTIVE` and clears `ended_at`.
  2. `status = ACTIVE` ⇔ `ended_at IS NULL`.
  3. **At most one** `ACTIVE` `OWNER` per group. **Zero is allowed**: an ownerless group after the owner's account is deleted (NFR-REL-02). v1.0 only ever writes `OWNER` when creating the group.
  4. Every read of group data joins this table with `status = 'ACTIVE'`; that is the single filter that makes removed and deleted members vanish (BR-SHARE-5).
  5. `LEFT` is unused in v1.0.

### 4.4 `targets`

- **Purpose.** A member's personal commitment (ADR-0006). Immutable identity; rules live in `target_revisions`.
- **Key fields.** `owner_id`, `name`, `mode`, `unit`, `start_date`.
- **Invariants.**
  1. `owner_id` never changes.
  2. In v1.0 `start_date` is the member's effective today at creation and is never in the past (BR-TGT-4).
  3. `mode` is locked once any `entries` row exists (BR-TGT-5; enforced by the action).
  4. A target is hard-deleted only without history (BR-TGT-6): the action checks "no entry and no fully elapsed scheduled day", and the `Restrict` reference from `entries` stops the delete at the database even if the check is bypassed.

### 4.5 `target_revisions`

- **Purpose.** The rule in force from a date: goal, frequency, weekdays. History is judged on the revision in force that day (ADR-0003).
- **Key fields.** `(target_id, effective_from)` primary key, `goal`, `frequency`, `weekdays`.
- **Invariants.**
  1. The first row has `effective_from = targets.start_date`.
  2. At most one revision per date (primary key). An edit with history writes `effective_from = today + 1`, replacing a row that already has that date; an edit without history updates the first row in place (FR-TGT-06).
  3. `frequency = DAILY` ⇔ `weekdays = {}`; `SPECIFIC_DAYS` has 1–6 distinct ISO weekdays (all seven is stored as `DAILY`).
  4. `goal` is null for `CHECKBOX`; ranges by mode (BR-TGT-3) are validated in Zod, with broad `1–480` bounds in the database.
  5. Deleted only with the target (cascade), which is only possible without history.

### 4.6 `target_inactive_periods`

- **Purpose.** Days when a target is not scheduled: `ARCHIVED` (days are *not tracked*) and, from v1.1, `PAUSED` (days are *tracked* but nothing is scheduled). One table, so pause and restore need no migration (TAD §14.2).
- **Key fields.** `target_id`, `kind`, `from_date`, `to_date` (inclusive, null = open).
- **Invariants.**
  1. `from_date <= to_date`.
  2. Periods of the same kind never overlap for a target (exclusion constraint).
  3. If an `ARCHIVED` and a `PAUSED` period cover the same date, **`ARCHIVED` wins** (the day is untracked). (Clarification of TAD §6.1.)
  4. Archiving inserts an open `ARCHIVED` period with `from_date = today + 1` (BR-TGT-7). Archive is idempotent.
  5. v1.0 writes only `ARCHIVED`. Restore (v1.1) closes the period at `today − 1`, or deletes it if `from_date > today − 1`.

### 4.7 `share_intervals`

- **Purpose.** When each target was visible to each group. The basis of every group view (ADR-0006).
- **Key fields.** `target_id`, `group_id`, `from_date`, `to_date` (both inclusive, `to_date` null = still shared).
- **Invariants.**
  1. At most one open interval per `(target, group)`; no two intervals of the same pair overlap.
  2. **Never retroactive:** a new interval starts at the owner's effective today (BR-SHARE-2).
  3. **Unshare** sets `to_date = today`, so today still counts and tomorrow does not (BR-SHARE-3).
  4. **Removal or account deletion** closes at `yesterday`, because the effect is immediate (BR-SHARE-5). If `from_date > yesterday` (the interval began today) the row is **deleted** instead, since it never counted. (Clarification of TAD §6.2.)
  5. Only the target's owner, while an `ACTIVE` member of the group, can open an interval (BR-SHARE-1; enforced by `A-07` and `A-11`).
  6. **Every group read joins `memberships` (`ACTIVE`) on the owner and group.** Closed intervals of removed members therefore stay stored but stay invisible, and reappear on rejoin (BR-SHARE-6).

### 4.8 `entries`

- **Purpose.** Progress. The only per-day fact the app records.
- **Key fields.** `(target_id, local_date)` primary key, `value`.
- **Invariants.**
  1. No row means no progress. Setting a value to 0 **deletes** the row (so `value` is always 1–1440).
  2. Written only for the owner's effective today or yesterday (the edit window); the action API cannot express any other date.
  3. `COUNT` ≤ 999 and `DURATION` ≤ 1440 per day (BR-PRG-1), enforced by the action.
  4. Never deleted except by setting the value to 0 (`Restrict` from the target).

### 4.9 `feed_cards`

- **Purpose.** The activity feed: one card per member, per group, per local day, updated in place (FR-FED-01, BR-FEED-1). A card is a marker with a sort key; **its content is derived on read** from entries and share intervals, so it can never disagree with them.
- **Key fields.** `(group_id, user_id, local_date, kind)` unique, `last_activity_at`.
- **Invariants.**
  1. **One feed card per member per group per local day** (per kind; v1.0 has only `DAILY_PROGRESS`).
  2. A card exists ⇔ at least one task of that member that is shared with that group on that date and scheduled that day has progress above 0. Maintained inside `recordProgress` and `setShares` (TAD §6.6).
  3. `last_activity_at` is set on every write that leaves the card in existence.
  4. Cards of non-`ACTIVE` members are never returned.
  5. Deleting a card deletes its reactions (cascade).

### 4.10 `reactions`

- **Purpose.** Emoji reactions on cards (FR-REA-01).
- **Key fields.** `(card_id, user_id, emoji)` primary key.
- **Invariants.**
  1. A member has at most one reaction of each emoji on a card; toggling inserts or deletes.
  2. Reactions are counted and shown **only for `ACTIVE` members** (FR-REA-03).
  3. Deleted with their card (cascade).

### 4.11 `streak_prefixes`

- **Purpose.** Derived cache: the group streak run through the last immutable day, so polling never scans history (TAD §6.4).
- **Key fields.** `(user_id, group_id)` primary key, `through_date`, `run`, `algo_version`.
- **Invariants.**
  1. `through_date` is never later than `effective_today − 2` at the time it was computed.
  2. Correctness never depends on this table: the value must always equal a full recomputation (**TP-01**, `verifyStreakCache`).
  3. Deleted on: rejoin (`A-04`), timezone change (`A-02`), removal (`A-06`), account tombstone (`R-04`), `algo_version` bump, and by any seed or repair script that writes entries older than yesterday.

### 4.12 `error_events`

- **Purpose.** Our own error record, because Vercel Hobby keeps runtime logs for 1 hour (NFR-OPS-01, TAD §9.4).
- **Key fields.** `at`, `source`, `code`, `user_id?`, `message` (≤ 500 characters), `fingerprint`.
- **Invariants.**
  1. Append-only; **no foreign keys**, so it survives any change to users or groups.
  2. Rows older than 30 days are deleted opportunistically on write.
  3. No stack traces, no request bodies, no personal data beyond `user_id`.

---

## 5. Constraints and indexes, with the query each one serves

Query labels come from the TAD sections named in the last column. **PK** and **UQ** mean the primary key or a unique constraint; **IX** is a plain index; **Hand** means the SQL in §3.

| Table | Constraint / index | Kind | Serves |
|---|---|---|---|
| users | `clerk_id` | UQ | `requireUser` / `ensureUser`: find the user by Clerk id on every request (TAD §4.3, §4.4) |
| users | `handle` | UQ | Handle availability on onboarding and profile edit; collision retry in `ensureUser` (FR-ACC-03/04) |
| users | `handle` format, name and timezone not blank | Hand (CHECK) | BR-ID; guarantees the lowercase storage that makes the UQ case-insensitive |
| groups | `invite_code` | UQ | Join by code: `SELECT … WHERE invite_code = $1 FOR UPDATE` (TAD §4.2) |
| groups | `invite_code` format | Hand (CHECK) | BR-GROUP-4 |
| memberships | `(user_id, group_id)` | UQ | `requireGroupAccess` (TAD §4.4); rejoin finds the existing row (FR-GRP-03) |
| memberships | `(group_id, status)` | IX | Active members for roll call, member list, comparison rows (TAD §5.6); the cap count in the join transaction; the `ACTIVE` filter on feed and share reads |
| memberships | `(user_id, status)` | IX | "My groups" for the switcher and app shell (FR-GRP-05); tombstone and removal sweeps by user (TAD §4.5) |
| memberships | one `ACTIVE` `OWNER` per group | Hand (partial UQ) | Ownership invariant; allows zero (ownerless group) |
| memberships | `ended_at` consistent with `status` | Hand (CHECK) | Removal and rejoin bookkeeping |
| targets | `(owner_id, created_at)` | IX | Today's checklist in creation order (FR-CHK-01); targets list; personal streak and heatmap reads (TAD §6.3, §6.4) |
| target_revisions | `(target_id, effective_from)` | PK | `revisionOn` for every scheduled-day check (TAD §6.1); the in-place or next-day edit upsert (FR-TGT-06) |
| target_revisions | goal range; weekdays match frequency | Hand (CHECK) | BR-TGT-3 safety net |
| target_inactive_periods | `(target_id, kind, from_date)` | IX | `inactiveKindOn` and archive idempotency (TAD §6.1, FR-TGT-07) |
| target_inactive_periods | no overlap per `(target, kind)`; `from <= to` | Hand (EXCLUDE, CHECK) | Prevents double archive; keeps period lookups unambiguous |
| share_intervals | `(group_id, from_date)` | IX | Shared structure of a group over a date window: `group_id = $g AND from_date <= $hi AND (to_date IS NULL OR to_date >= $lo)` for roll call, streaks, poll, overview (TAD §5.6, §7.2) |
| share_intervals | `(target_id, group_id, from_date)` | IX | `groupsSharingOn(target, date)` inside `recordProgress` (TAD §6.6, §6.7); `share` / `unshare` lookups (TAD §6.2); Today's share indicators |
| share_intervals | open interval per group (`WHERE to_date IS NULL`) | Hand (partial IX) | "Currently shared" lookups and the member-removal close-out (TAD §6.2) |
| share_intervals | one open interval per `(target, group)` | Hand (partial UQ) | Invariant 1 with a clear error |
| share_intervals | no overlap per `(target, group)`; `from <= to` | Hand (EXCLUDE, CHECK) | Invariants 1 and 2; makes "reopen same-day" the only legal way to restore a closed interval |
| entries | `(target_id, local_date)` | PK | Entry upsert in `recordProgress` (TAD §6.7); entries for a set of targets over a date range (roll call, poll, streaks, heatmaps); "has history" check (FR-TGT-08) |
| entries | `value` 1–1440 | Hand (CHECK) | BR-PRG-1 safety net |
| feed_cards | `(group_id, user_id, local_date, kind)` | UQ | The upsert and delete in `syncFeedCards` (TAD §6.6); invariant "one card per member per group per local day" |
| feed_cards | `(group_id, last_activity_at DESC, id DESC)` | IX | Keyset paging of the feed: page 1 in every poll, older pages on scroll (TAD §6.6, §7.2, FR-FED-03) |
| reactions | `(card_id, user_id, emoji)` | PK | Toggle (insert or delete); reaction counts for the cards of a page (TAD §7.2) |
| streak_prefixes | `(user_id, group_id)` | PK | `loadPrefix` / `savePrefix` per member in each poll (TAD §6.4); invalidation by user |
| streak_prefixes | `(group_id)` | IX | Invalidation and cascade by group |
| error_events | `(at)` | IX | Pruning rows older than 30 days (TAD §9.4) |
| error_events | `(fingerprint, at)` | IX | Grouping repeated errors when reading the table by hand |

**Indexes deliberately not created in v1.0.** No index on `feed_cards.user_id`, `reactions.user_id` or `memberships.group_id` alone. Nothing in v1.0 queries by those columns, and nothing is ever hard-deleted, so the usual foreign-key lookup never runs. Erasure of a user's data (FR-ACC-07, v1.1) will add the indexes it needs; that is an additive migration.

---

## 6. Delete, tombstone and archive behaviour

### 6.1 What happens to what

| Event | Effect on stored data | Visible effect |
|---|---|---|
| **Archive a target** (FR-TGT-07) | Insert `ARCHIVED` period from `today + 1`, open-ended. Nothing else changes. | Stops being scheduled tomorrow; all past days stay in every personal and group view. |
| **Delete a target** (FR-TGT-08) | Allowed only without history. Rows in `target_revisions`, `target_inactive_periods`, `share_intervals` cascade. The `Restrict` from `entries` blocks the delete if any entry exists. | The target disappears. |
| **Unshare** (FR-VIS-05) | Set `to_date = today` on the open interval. | Group counts today, stops from tomorrow; past stays. |
| **Remove a member** (FR-GRP-07) | `memberships.status = REMOVED`, `ended_at = now`. All of the member's open intervals in that group close at `yesterday`, or are deleted when they began today. `streak_prefixes` row deleted. Entries, targets, cards and reactions untouched. | Member vanishes from roll call, rows, feed and reactions of that group, past included. Their own targets and history are untouched. |
| **Rejoin** (FR-GRP-03) | `status = ACTIVE`, `ended_at = NULL`, `streak_prefixes` row deleted. **No interval is reopened.** | Earlier shared history reappears; absence days are untracked; the member re-shares what they want. |
| **Progress set to 0** | The `entries` row is deleted; the feed card is deleted if no shared task has progress left; reactions cascade. | Card disappears. |
| **Timezone change** (FR-ACC-05) | `timezone` and `date_floor` updated; all of the user's `streak_prefixes` deleted. Entries and intervals are never re-dated. | Today and yesterday are recomputed in the new zone; the date never moves backwards. |
| **Account deleted in Clerk** (NFR-REL-02) | The tombstone transaction below. Nothing is hard-deleted. | The person vanishes from every group view; every group page keeps working. |
| **Delete group** (v1.1) | `groups.deleted_at` set. Intervals, cards and memberships stay but become unreachable. | Group is gone for everyone; members' own targets are untouched (BR-SHARE-7). |
| **Error rows** | Deleted after 30 days. | None. |

### 6.2 The tombstone transaction (NFR-REL-02)

Run by `R-04` on `user.deleted`; idempotent; one transaction.

```ts
async function tombstoneUser(tx, clerkId: string, now: Date) {
  const u = await tx.user.findUnique({ where: { clerkId } })
  if (!u || u.deletedAt) return                                      // unknown or already done: success
  const today = effectiveToday(u, now)                               // their own effective today (TAD §5.4)
  const closeAt = addDays(today, -1)

  await tx.user.update({ where: { id: u.id }, data: {
    deletedAt: now, displayName: 'Former member', imageUrl: null,
    handle: 'deleted_' + u.id.replaceAll('-', '').slice(0, 12),      // 20 characters, frees the real handle
  } })
  await tx.membership.updateMany({ where: { userId: u.id, status: 'ACTIVE' }, data: { status: 'REMOVED', endedAt: now } })
  await tx.$executeRaw`DELETE FROM share_intervals si USING targets t
                       WHERE si.target_id = t.id AND t.owner_id = ${u.id}::uuid AND si.to_date IS NULL AND si.from_date > ${closeAt}`
  await tx.$executeRaw`UPDATE share_intervals si SET to_date = ${closeAt} FROM targets t
                       WHERE si.target_id = t.id AND t.owner_id = ${u.id}::uuid AND si.to_date IS NULL`
  await tx.streakPrefix.deleteMany({ where: { userId: u.id } })
}
```

| Case | Result |
|---|---|
| Deleted user was a member | Removed from all views; their slot is free for the 15-member cap. |
| Deleted user was the **owner** | Their membership becomes `REMOVED` and keeps `role = OWNER`. The group has **no active owner**; the partial unique index allows this. Member pages, roll call, feed, joining and the invite all work. Owner-only actions are unavailable to everyone until v1.1 succession (BR-OWN-3). `loadMembers` exposes `ownerless: true`. |
| Their entries, cards, reactions, targets | Kept, so no foreign key dangles. They are unreachable because every read requires an `ACTIVE` membership. |
| Late `user.updated` for the tombstoned user | `applyProfile` skips rows with `deleted_at` set. |
| Duplicate or reordered `user.deleted` | No-op after the first. |
| A lingering session | `requireUser` returns `UNAUTHENTICATED` for tombstoned users. |
| Avatar is null | Components render initials; no query or component may assume `image_url` is non-null. |

The required tests are **TP-03** (a deleted member and a deleted owner leave the group page, roll call, comparison, feed, reactions and member list working and never shown) and **TP-04** (a private target never influences any group number).

---

## 7. Which algorithm reads and writes each model

| Model | Written by | Read by |
|---|---|---|
| `users` | `ensureUser` / `upsertFromClerk` (TAD §4.3; every authenticated surface, first request only); `A-01` onboarding; `A-02` profile and timezone change (`changeTimezone`, TAD §5.4); `R-04` webhook: `applyProfile`, `tombstoneUser` | `requireUser` (every surface); `effectiveToday` (TAD §5.4) in every day-dependent surface; `computeRollCall` for each member's zone (TAD §5.6) |
| `groups` | `A-03` create (invite code generation, TAD §4.2); v1.1: rename, regenerate, delete | `A-04` join by code (row lock); `requireGroupAccess` (`deleted_at`); `L-01`, `L-07`, `L-08` |
| `memberships` | `A-03` (OWNER row); `A-04` join / rejoin (`FOR UPDATE` on the group, cap count); `A-06` removal; `R-04` tombstone | `requireGroupAccess` (TAD §4.4); `computeRollCall` (TAD §5.6); `L-01` switcher; `L-08`; every group read as the `ACTIVE` filter |
| `targets` | `A-07` create; `A-08` update (name, unit, mode before history); `A-10` delete | `R-01` / `L-03` Today; `L-05`, `L-06`; `requireTargetOwner`; `dayState` and streak scans (TAD §6.3, §6.4); `computeRollCall` |
| `target_revisions` | `A-07` (first row); `A-08` (in-place or next-day revision) | `isScheduled`, `goalOn`, `isComplete` (TAD §6.1); every scan that derives scheduled days |
| `target_inactive_periods` | `A-09` archive (`ARCHIVED` from tomorrow); v1.1: pause, restore | `isTracked`, `isScheduled` (TAD §6.1) |
| `share_intervals` | `A-07` (initial shares); `A-11` (`share`, `unshare`, TAD §6.2); `A-06` and `R-04` (`onMemberRemoved`) | `isSharedOn` / `targetsVisibleIn` (TAD §6.3); `groupsSharingOn` (TAD §6.6); `computeRollCall`; `groupStreak`; `R-02`, `R-03`, `L-07` |
| `entries` | `A-12` `recordProgress` (TAD §6.7) | `R-01` / `L-03` Today; `isComplete`; `dayState` and streaks (TAD §6.3, §6.4); `computeRollCall` (TAD §5.6); feed bodies (TAD §6.6); the has-history check in `A-08`, `A-10` |
| `feed_cards` | `A-12` `syncFeedCards` (TAD §6.6); `A-11` `syncFeedCards` for newly shared groups | `R-02` (page 1), `R-03` (older pages), `L-07`; `A-13` (card lookup for authorization) |
| `reactions` | `A-13` toggle; cascade from `feed_cards` | `R-02`, `R-03`, `L-07` (counts and "mine") |
| `streak_prefixes` | `groupStreak` (`savePrefix`, TAD §6.4); deleted by `A-02`, `A-04` (rejoin), `A-06`, `R-04`, `invalidateStreakPrefixes` (repair scripts), version bump | `groupStreak` in `R-02` and `L-07`; `verifyStreakCache` (tests and dev script only) |
| `error_events` | `logError` (TAD §9.4) from `defineAction` and the route handlers | Read by hand in the Neon SQL editor; never by the app |

`fullStreak` (the personal streak and the oracle for TP-01) reads `targets`, `target_revisions`, `target_inactive_periods` and `entries`, and `share_intervals` only for group scope; it writes nothing.

---

## 8. v1.1 readiness

Everything the TAD (§14.2) promises as **zero-migration** is already present:

| v1.1 concern | Already in this schema |
|---|---|
| Owner role: transfer, succession | `memberships.role` (`OWNER`/`MEMBER`) and the one-active-owner partial index |
| Membership status: leave, rejoin | `MembershipStatus.LEFT`, `ended_at`, and the unique `(user, group)` row |
| Share intervals | the whole `share_intervals` model and its constraints |
| Target pause and restore | `target_inactive_periods` with `InactiveKind.PAUSED`; `isScheduled` and the tracked/untracked rule already distinguish `PAUSED` (tracked) from `ARCHIVED` (untracked) |
| Delete group, deleted accounts | `groups.deleted_at`, `users.deleted_at` |
| Future start dates | `targets.start_date` and the first revision already accept a future date |

Additive-only for the rest, none rewriting data: `WEEKLY_QUOTA` (enum value plus a nullable `quota` on `target_revisions`); `MILESTONE` and `JOINED` (enum values plus a nullable `payload` on `feed_cards`); `nudges`, `notifications`, `shared_goals`, `shared_goal_members` and invite-attempt counters (new tables); `targets.shared_goal_id` (nullable column); `streak_prefixes.best` (nullable column).

---

## 9. Refinements to the TAD (approved 2026-10-27; applied in TAD rev 2)

Each is a clarification discovered while designing the schema and API; none changes a PRD requirement.

1. **Removal closes at `yesterday`, or deletes the interval** if it began today (TAD §6.2 closed at yesterday unconditionally, which violates `from <= to` for an interval opened today). Same rule in the tombstone.
2. **`ARCHIVED` takes precedence over `PAUSED`** when both cover a date (TAD §6.1 used `find`, which depends on array order).
3. **`error_events.where` is renamed `source`** (`WHERE` is an SQL keyword).
4. **Handle uniqueness** uses lowercase storage, a format `CHECK` and a plain unique index, instead of a functional `lower(handle)` index (TAD §14.3). Same guarantee, simpler.
5. **`setShares` syncs today's feed card** for each newly shared group (TAD §6.2 and §6.6 did not say so). Without it, sharing a target that already has progress today would leave that group's view without the card until the next progress change.
6. **`target_revisions` and `entries` use composite primary keys** rather than a surrogate id plus a unique constraint (TAD §14.1 wrote "unique").
7. **Manifest and icons must be public routes** in `proxy.ts` (TAD §4.1 lists only sign-in, sign-up and the webhook). Browsers fetch the manifest without credentials, so a protected manifest would break installability (FR-PWA-01). See API `P-01`.
8. **`btree_gist` is documented as supported on Neon, and a tested fallback exists** (§3.1). TAD §14.3 said "exclusion constraint or a transactional check" without choosing; the exclusion constraint is primary and the trigger with an advisory lock is the fallback.

---

## 10. Verification

Run on 2026-10-27 in an isolated scratch directory (nothing was added to the repository): Prisma CLI 7.10.1, PGlite 0.4.3 (PostgreSQL 17.5 compiled to WebAssembly) with the `btree_gist` extension, Node 22.23.2. The Prisma and SQL blocks above were extracted from this file and used unchanged.

| Check | Result |
|---|---|
| `prisma validate` on the schema in §2 | valid |
| `prisma migrate diff --from-empty --to-schema` generates the DDL | 12 tables, 7 enums, 15 indexes, 14 foreign keys (8 `RESTRICT`, 6 `CASCADE`) |
| The generated DDL executes on PostgreSQL, then the §3 SQL executes | both succeed |
| 68 constraint and behaviour tests on the primary design | 68 of 68 pass |
| The two non-overlap rules raise SQLSTATE `23P01` | 2 of 2 pass |
| **Fallback** (§3.1): a second database with **no extension loaded**, running the §3 SQL minus `CREATE EXTENSION` and the two `EXCLUDE` constraints, plus the fallback triggers | 22 of 22 pass (92 checks in all) |

The tests cover: handle format (upper case, too short, leading digit, the 20-character tombstone handle) and uniqueness of handle and Clerk id; invite-code format (I, O, 0, 1, lower case) and uniqueness; one active owner per group, zero active owners after the owner is removed, and a new owner afterwards; the `status`/`ended_at` consistency rule; revision rules (daily with weekdays, specific days empty, seven weekdays, weekday 8, goal 0, two revisions on one date); inactive periods (overlap rejected, `ARCHIVED` plus overlapping `PAUSED` allowed, from after to); share intervals (overlap rejected, adjacent allowed, second open interval rejected, from after to, closing and reopening the latest); entry value bounds and the composite key; `Restrict` blocking deletion of a target with entries and of a user with memberships; one feed card per member, group and day; one reaction per emoji; cascade from card to reactions; `error_events` carrying no foreign keys; and a feed-page query that the planner serves from the keyset index without a sort (checked on 300 synthetic rows).

**Not verified** (needs Neon or the application):

1. That `btree_gist` can be created on the real Neon project. Neon's documentation lists it as supported with no Free-plan restriction (§3.1), and the fallback is tested, but spike S1 still runs `CREATE EXTENSION btree_gist` once to confirm.
2. That `prisma migrate dev` reports no drift after the hand-written migration. Partial indexes and exclusion constraints are outside Prisma's model and may be reported as drift. Fallback: keep them in a separate append-only migration and verify with `prisma migrate diff` in CI (spike S1).
3. Runtime generation of `uuid(7)` defaults and interactive transactions through `PrismaNeon` (spike S1).
4. The `FOR UPDATE` row lock issued with `$queryRaw`.
5. Query performance at real data volumes.
6. Two-connection races on the non-overlap rules (TP-09 in the TAD). The in-memory database has a single connection; the primary design relies on PostgreSQL's own exclusion constraint, and the fallback's advisory-lock behaviour is argued in §3.1 but would need this test if it were ever adopted.
