# Pulse — Project Brief
This is the input brief for Pulse. It is not an instruction to start building.

## Vision
Pulse keeps a small group of developer friends (2–15 people) consistent with upskilling: DSA practice, side projects, courses, open source, reading docs. Each person sets targets and checks off daily tasks, and everyone can see everyone else's consistency on a GitHub/LeetCode-style heatmap. The core feeling: "my friends can see if I showed up today." It will be used by my real friend group.

It runs on the honor system: no proof, links, or screenshots. Checking a box is enough. Check-offs should be low-friction.

## Tech Stack (decided)
- Next.js, latest stable (App Router, Server Components, Server Actions, TypeScript strict)
- Tailwind CSS, shadcn/ui, Lucide icons, Framer Motion (sparingly)
- PostgreSQL on Neon, via Prisma ORM
- Clerk auth (Google OAuth + email/password), with users synced to our DB via a Clerk webhook
- TanStack Query for mutations, optimistic updates, and polling (~20s); no realtime service in v1
- Zod for validation; date-fns + date-fns-tz for dates
- Custom SVG heatmap (no calendar library)
- Deploy on Vercel + Neon, using free tiers

## Features

### A. Auth, Profiles & Groups
- Profile: avatar, display name, unique handle, timezone (auto-detected, editable).
- Users can belong to multiple groups; group switcher.
- Create group → 8-char invite code (no ambiguous characters) and link /join/[code]; owner can regenerate it.
- Roles: OWNER, MEMBER. Owner can remove members and delete the group.
- Every action must verify group membership.

### B. Targets
- A target belongs to a user within a group and is visible to that group.
- Quick-start templates: "Solve N LeetCode problems", "Work N minutes on side project", "Study a course for N minutes", "Read docs for N minutes", "Make an open-source contribution", "Revise a DSA topic", plus custom.
- Tracking modes: CHECKBOX, COUNT (e.g. 2/3), DURATION (minutes).
- Frequency: DAILY; SPECIFIC_DAYS (chosen weekdays); WEEKLY_QUOTA (e.g. 3x/week — met when weekly completions reach the quota; days without completion aren't counted as misses).
- Shared group goals: members opt in; each member gets their own daily task; the group sees collective progress ("4/6 done today").
- Targets with history are archived, never hard-deleted. Each target has a start date.

### C. Daily Checklist
- Tasks generated lazily when the user opens the app (no cron), with missed past days backfilled as incomplete.
- "Today" is always the user's local calendar day from their timezone, never server time.
- One-tap check-off with optimistic UI; increments for COUNT/DURATION.
- Edit window: today and yesterday only.

### D. Contribution Heatmap
- Last 52 weeks on desktop, ~3 months on mobile; Monday-start weeks; month and weekday labels.
- Levels by completed/scheduled: 0 = 0%, 1 = 1–25%, 2 = 26–50%, 3 = 51–75%, 4 = 76–100% (gray → deep emerald). "Nothing scheduled" looks different from "missed."
- Tooltip on hover and keyboard focus (date, X of Y tasks, %). Accessible, light/dark mode.
- Group comparison view: one row per member with avatar, streak, and aligned heatmap strip; sortable.

### E. Consistency & Accountability
- Streaks (current and longest): a day counts if ≥1 scheduled task was completed or nothing was scheduled.
- Daily roll call using each member's own local date: Active today / Slacking today / Rest day.
- Activity feed: completions, streak milestones (7/30/50/100), member joined; cursor-paginated; batch rapid completions.
- Emoji reactions (🔥 💪 👏 😂 👀).
- Nudge slacking members: max one per sender → recipient per day; in-app notifications with unread count.
- Weekly recap card: each member's weekly completion % and longest active streak.

## Constraints
- Mobile-first: friends will check off tasks from their phones.
- Free tiers only.
- I'm building it with Claude Code in VS Code, step by step, spec-first: PRD → TAD + ADRs → DB schema + API spec → tasks + test plan → implementation.
