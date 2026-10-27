# Pulse — Product Requirements Document

| | |
|---|---|
| **Status** | Approved (rev 2) |
| **Date** | 2026-10-27 |
| **Revisions** | rev 1: initial approval. rev 2: heatmap keyboard model (FR-HM-03, FR-HM-04, NFR-A11Y-02); polling stops after 3 minutes without interaction (NFR-POLL-02, NFR-POLL-04); decisions OQ-12 and OQ-13. |
| **Input** | `docs/BRIEF.md` + discovery decisions (personal targets, v1.0/v1.1 cut, pause, in-app-only nudges) + the v1.0 scope pass |
| **Next document** | DB schema + API spec (the architecture is in `docs/TAD.md` and `docs/adr/`) |

**How to read this document.** This PRD says *what* Pulse does and *how well*, never *how it is built*. Every functional requirement has an ID (`FR-AREA-NN`) and a release tag, **[v1.0]** or **[v1.1]**. Exact values live in Business Rules (`BR-*`); quality targets live in Non-Functional Requirements (`NFR-*`). Scenarios are written Given / When / Then.

---

## 1. Overview

### 1.1 Problem

Developers who want to keep up DSA practice, side projects, courses, open source and reading docs usually drop off when they work alone, and the tools they have are either private (habit apps) or siloed to one platform (GitHub and LeetCode graphs). Pulse is a shared, honor-system daily checklist for a small friend group (2–15 people). Each person sets targets and checks them off daily, and the whole group can see who showed up today and how consistent everyone has been, on a heatmap, a roll call, streaks and a feed. The core feeling is: **"my friends can see if I showed up today."**

### 1.2 Users

| Role | Description |
|---|---|
| **Member** (primary) | A developer in a close friend group, using a phone, in their own timezone. Checks off 1–5 tasks a day in a few seconds and glances at the group to see who showed up. |
| **Owner** (secondary) | A member who created the group. Additionally invites people and removes members; from v1.1 also regenerates the invite code, renames the group, transfers ownership and deletes the group. Exactly one owner per group. |

### 1.3 Product principles

1. **Show up, be seen.** Everything is built around making a friend's consistency visible.
2. **Honor system.** No proof, links or screenshots. Checking a box is enough.
3. **Low friction.** A returning user can complete a simple task in two taps from opening the app.
4. **Mobile first.** Designed for a phone in one hand; desktop is the adaptation.
5. **Fair days.** A "day" is always the member's own local calendar day.
6. **Explicit sharing.** A target is visible to a group only if its owner shares it there; everything else stays private.

### 1.4 Key terms

| Term | Meaning |
|---|---|
| **Target** | A recurring commitment owned by one member (e.g. "Solve 2 LeetCode problems daily"). Has a tracking mode, a goal, a frequency and a start date. |
| **Task** | A target on one specific date for which it is scheduled. |
| **Progress** | The amount recorded against a task: done/not done, a count, or minutes. |
| **Complete** | A task is complete when its progress meets or exceeds its goal for that day. |
| **Scheduled** | A target is scheduled on a date if its frequency, start date, archive and pause state say so (BR-SCH). |
| **Local date** | The calendar date in a member's own timezone. |
| **Active day** | A day on which at least one scheduled task was completed. |
| **Share** | Making a target visible to, and counted by, a specific group. |
| **Personal view** | Screens about *me*, which count all my active targets (shared and private). |
| **Group view** | Screens about a *group* (heatmaps, roll call, streaks, feed), which count only targets shared to that group. |
| **Roll call** | Today's status of every member of a group, each judged on their own local date. |
| **Card** | One entry in a group's feed. |

### 1.5 Deviations from the brief

These are deliberate and were approved during discovery.

| # | Brief said | This PRD says | Why |
|---|---|---|---|
| D1 | A target belongs to a user *within a group*. | A target is **owned by the member** and **shared to zero or more groups**. One check-off counts in every group it is shared to. | Avoids re-entering the same habit per group; keeps one personal heatmap. |
| D2 | Tasks are generated when the user opens the app; missed days are backfilled. | **Days without progress count as incomplete automatically**, whether or not the member ever opened the app. | Otherwise members who stop showing up would be invisible, which is exactly who the group needs to see. |
| D3 | WEEKLY_QUOTA frequency in v1.0. | Moved to **v1.1** with an explicit credit rule (BR-QTA). | Its effect on heatmap, streak and roll call was undefined. |
| D4 | Roll call: Active / Slacking / Rest day. | Adds **Not yet** (before 18:00 local) and **No targets shared**. | Everyone would otherwise be "slacking" at 00:01. |
| D5 | Streak: a day counts if a task was done or nothing was scheduled. | Rest days **bridge** a streak but do not add to it; a streak cannot start on days with no tracked targets. | Prevents infinite streaks for members with no targets. |
| D6 | Feed batches rapid completions (undefined). | **One card per member per local day per group**, updated in place. | Stable card to react to; trivial undo. |
| D7 | Group lifecycle: remove member, delete group. | Adds **leave group**, **transfer ownership**, and group deletion that never deletes members' targets (all v1.1; v1.0 has member removal only). | Avoids orphaned groups and conflict with "never hard-delete targets with history". |
| D8 | Streaks are per member. | Streaks are **per member per scope**: a personal streak (all targets) and a group streak (shared targets only). | Follows from group-scoped visibility. |
| D9 | — | **Pause a target until a date** (v1.1). | Sick days and travel should not destroy streaks. |
| D10 | — | **PWA installability** (manifest + icons) in v1.0. | Friends use phones; install is free. |

---

## 2. Goals and non-goals

### 2.1 Goals

| ID | Goal | Measure |
|---|---|---|
| G1 | Anyone in a group can tell, at a glance, who has shown up today. | Roll call and comparison heatmap are on the first group screen; refresh within 45 s (NFR-POLL, NFR-FRESH). |
| G2 | Checking off a task is nearly effortless. | Checkbox task: ≤ 2 taps from app open; progress shows instantly (NFR-PERF-01). |
| G3 | A new friend is productive immediately. | From landing in a group, a user can create a target from a template and check it off in ≤ 60 s. |
| G4 | "A day" is fair to everyone in every timezone. | All day logic follows BR-DAY; no scenario in NFR-QUAL-01 fails. |
| G5 | Costs nothing to run. | Operates within free-tier hosting and database plans (NFR-COST-01). |
| G6 | The real friend group actually uses it. | *Proposed:* within 2 weeks of launch, every member of the first group has completed ≥ 1 task on ≥ 4 different days; at week 6, ≥ 70% of members are active on ≥ 3 days per week. (See OQ-6.) |

### 2.2 Non-goals

- Verifying that anyone really did the work (no proof, links, screenshots, or integrations).
- A public social network: no discovery, public profiles, or cross-group leaderboards.
- Points, levels, badges or other gamification beyond streaks and milestones.
- Push or email notifications (v1.0 and v1.1).
- Native mobile apps; Pulse is a web app that can be installed.
- Chat, comments or free-text posts. Reactions are the only social response.
- Instant (real-time) updates. Pulse refreshes on a schedule (NFR-POLL).
- Working offline. Installability does not imply offline use.

---

## 3. User journeys

### J1 — A new friend joins via invite  *(v1.0)*
1. Receives an invite link from the owner and opens it, signed out, and is prompted to sign in or sign up.
2. Signs up with Google or email and password, and is brought back to the invite automatically.
3. Confirms profile: display name, a suggested unique handle, auto-detected timezone, avatar from their account.
4. Is joined to the group and lands on its group view.
5. Is prompted to add a first target from templates, with "share with this group" pre-selected.
6. Checks it off. Their name moves to **Active** in the roll call.

*Success:* steps 5–6 in ≤ 60 seconds.

### J2 — An owner creates a group  *(v1.0)*
1. Signs up, creates a group with a name.
2. Sees an 8-character invite code and link; shares them with friends from the phone's share sheet or by copying.
3. Watches members appear in the member list and roll call.

### J3 — The daily check-in  *(v1.0)*
1. Opens the installed app on the phone. Sees **Today**: the checklist of everything scheduled today and a "3 of 5 done" summary.
2. Taps a checkbox (instant), adds +1 to a count, or adds minutes to a duration.
3. Spots an item forgotten yesterday in the **Yesterday** section and completes it.
4. Sees their current streak.

### J4 — Checking on friends  *(v1.0)*
1. Opens a group: sees the roll call (Active / Not yet / Slacking / Rest day / No targets shared).
2. Scrolls the comparison heatmap, one row per member with streak and heatmap strip, ordered by current streak.
3. Reads the feed ("Priya finished 3 of 5") and reacts 🔥.
4. *(v1.1)* Nudges a friend who is slacking.

### J5 — Managing targets and sharing  *(v1.0)*
1. Creates a target from a template; chooses which groups can see it.
2. Later unshares it from one group: that group keeps the past, and stops counting it after today.
3. Changes a goal: effective tomorrow. Archives a target they no longer pursue: history stays.

### J6 — Travelling  *(v1.0, with v1.1 pause)*
1. Changes timezone in the profile (or accepts the auto-detected one). Today and yesterday are recalculated; nothing already recorded is re-dated.
2. *(v1.1)* Before a trip or illness, pauses a target until a date; paused days count as rest and bridge the streak.

### J7 — Being removed, and leaving  *(removal v1.0; leaving, transfer and deletion v1.1)*
1. The owner removes a member (from v1.1 a member can also leave). They vanish from the group's views; their targets and history remain theirs.
2. *(v1.1)* An owner who wants to leave must first transfer ownership, or delete the group.
3. A removed member can rejoin with the group's invite; their earlier shared history in that group reappears and they re-share what they want. Removal is not a ban until invites can be regenerated (v1.1).

### J8 — Weekly rhythm  *(v1.1)*
1. A friend is slacking at 7 p.m.; another member nudges them; they get an in-app notification with an unread badge, open the app and check off.
2. On Monday, the group sees a recap card: each member's weekly completion % and longest streak.

---

## 4. Functional requirements

### 4.1 Account and profile

**FR-ACC-01 [v1.0] Sign up and sign in**
- Given a visitor, when they choose Google or email-and-password, then they are signed in (creating an account if new) and land in the app.
- Given a signed-out visitor opened an invite link, when they finish signing in or up, then they return to that invite and are joined without re-entering the code.

**FR-ACC-02 [v1.0] Immediately usable after sign-up**
- Given a user has just completed sign-up, when they reach the app, then their profile exists and every feature works with no waiting, retry or manual refresh.

**FR-ACC-03 [v1.0] First-run profile**
- Given a user's first sign-in, when the profile step is shown, then display name is prefilled from their account, a unique handle is suggested, their timezone is auto-detected, and their avatar is the image from their account; the user can edit name, handle and timezone before continuing.

**FR-ACC-04 [v1.0] Edit profile**
- Given a signed-in user, when they change display name, handle or timezone and save, then the change applies everywhere immediately.
- Given the chosen handle belongs to someone else (case-insensitive), when they save, then it is rejected with a clear message and nothing changes.
- Given a member changes the image on their account, when they next open Pulse, then their avatar shows the new image.

**FR-ACC-05 [v1.0] Timezone change**
- Given a member changes timezone from A to B, when the change is saved, then today, yesterday, weekdays and the roll-call cutoff are computed in B immediately; progress already recorded keeps its original dates; and the member's current date never moves backwards (BR-DAY-5).

**FR-ACC-06 [v1.0] Sign out**
- Given a signed-in user, when they sign out, then their session ends on that device and no personal or group data remains visible there.

**FR-ACC-07 [v1.1] Deleted accounts**
- Given an account is deleted from the sign-in account settings (Pulse has no delete-account screen in v1.0), when deletion completes, then the person disappears from every group view (including past feed cards and heatmap rows), their personal data is erased, and any group they owned follows BR-OWN-3.

### 4.2 Groups

**FR-GRP-01 [v1.0] Create a group**
- Given a signed-in user, when they enter a group name (BR-GROUP-2) and confirm, then the group exists, they are its owner and a member, and its invite code and link are shown.

**FR-GRP-02 [v1.0] Invite code and link**
- Given a group, when its owner views invitation details, then they see an 8-character code (BR-GROUP-4) and a shareable link containing it, with copy and share-sheet actions.

**FR-GRP-03 [v1.0] Join a group**
- Given a valid code or link and a group with fewer than 15 members, when a signed-in user joins, then they become a MEMBER and see the group view.
- Given the code is typed by hand, when it contains lowercase letters, spaces or hyphens, then it still works.
- Given the user is already a member, when they open the link, then they are taken to the group with no duplicate membership.
- Given the group has 15 members, when someone tries to join, then they are told the group is full and are not added, including when two people try at the same moment (exactly one gets the last place).
- Given an invalid code, when someone tries it, then they see one generic "invite not valid" message that doesn't reveal whether the group exists.
- Given a member who was removed earlier, when they use the invite again, then they rejoin like anyone else, their earlier shared history reappears, and they re-share the targets they want (BR-SHARE-6, BR-GROUP-6).

**FR-GRP-04 [v1.1] Regenerate invite code**
- Given the owner, when they regenerate the code and confirm, then a new code and link are shown, the old code and link stop working immediately, and existing members are unaffected.
- Given a non-owner, when they try, then it is denied.

**FR-GRP-05 [v1.0] Group switcher**
- Given a member of two or more groups, when they open the switcher, then they see all their groups by name and can select one.
- Given a member previously selected a group, when they next open the app, then it shows that group.

**FR-GRP-06 [v1.1] Rename group**
- Given the owner, when they save a new name (BR-GROUP-2), then every member sees it.

**FR-GRP-07 [v1.0] Member list and removal**
- Given any member, when they open the member list, then they see each member's avatar, name, handle and role.
- Given the owner, when they remove a member and confirm, then that member loses access immediately and the removal follows BR-SHARE-5.
- Given the owner, when they try to remove themselves, then it is refused.
- Given a member was removed, when they use the group's invite again, then they can rejoin (FR-GRP-03); removal is not a ban, and keeping someone out requires regenerating the invite (FR-GRP-04, v1.1).

**FR-GRP-08 [v1.1] Leave a group**
- Given a MEMBER, when they leave and confirm, then the effects are the same as removal (BR-SHARE-5).
- Given the owner, when they try to leave, then they are told to transfer ownership or delete the group first.
- Given a member who left, when they use the invite again, then they can rejoin as in FR-GRP-03.

**FR-GRP-09 [v1.1] Transfer ownership**
- Given the owner, when they choose another active member and confirm, then that member becomes OWNER and the former owner becomes MEMBER; the group always has exactly one owner.

**FR-GRP-10 [v1.1] Delete a group**
- Given the owner, when they type the group's name to confirm, then the group, its feed, reactions and shared-visibility disappear for every member immediately and the invite code stops working. Members' own targets and history are untouched (BR-SHARE-7).

**FR-GRP-11 [v1.0] Membership is verified on every action**
- Given a person who is not an active member of a group, when they try to read or change anything about that group by any route (including old links), then access is denied and the response does not reveal whether the group exists or anything about it.

**FR-GRP-12 [v1.1] Invite preview**
- Given a visitor (signed in or not) opens a valid invite link and is not yet a member, when the page loads, then they see only the group name and member count, with a button to sign in or join.

**FR-GRP-13 [v1.1] Group and invite-attempt limits**
- Given a signed-in user who already belongs to 10 groups, when they try to create or join another, then they are told the limit has been reached (BR-GROUP-1).
- Given a user has made 10 failed code attempts within an hour, when they try again, then they are asked to wait (BR-GROUP-5).

### 4.3 Targets

**FR-TGT-01 [v1.0] Create a target from a template or from scratch**
- Given a member creating a target, when they pick one of the six templates (BR-TPL) or "Custom", then the form is pre-filled with that template's mode, goal and unit, and the member can edit every field before saving.

**FR-TGT-02 [v1.0] Tracking modes and goals**
- Given a member creating a target, when they choose a tracking mode, then they can pick CHECKBOX (done / not done), COUNT (a number toward a goal, e.g. 2 of 3) or DURATION (minutes toward a goal), with goal ranges per BR-TGT-3.
- Given a goal outside its range, when the member saves, then it is rejected with the allowed range.

**FR-TGT-03 [v1.0] Frequency: daily or specific days**
- Given a DAILY target, when any date on or after its start date is evaluated, then the target is scheduled that day.
- Given a SPECIFIC_DAYS target with 1–6 weekdays chosen, when a date is evaluated, then the target is scheduled only if that date's weekday is chosen.
- Given a member selects all 7 weekdays, when they save, then the target is stored as DAILY.

**FR-TGT-04 [v1.0] Start date**
- Given a new target, when the member saves it, then its start date is today and its first task is today.
- Given a start date in the past, when it is attempted by any route, then it is refused, so history cannot be invented (BR-TGT-4).

**FR-TGT-05 [v1.1] Active-target limit**
- Given a member has 10 active targets, when they try to create an 11th, then they are told to archive one first (BR-TGT-1).

**FR-TGT-06 [v1.0] Edit a target**
- Given any target, when the member edits its name, then the new name applies everywhere, including past days.
- Given a target with history, when the member edits goal or weekdays, then the change takes effect from the next local day and today's task keeps the old rule (BR-TGT-5).
- Given a target with no recorded progress whose start date is today or later, when the member edits any rule, then the change applies immediately from the start date.
- Given a target has recorded progress, when the member tries to change its tracking mode, then it is refused with a suggestion to archive and create a new target.

**FR-TGT-07 [v1.0] Archive a target**
- Given a target, when the owner archives it, then it stops being scheduled from the next local day, and its past days stay exactly as they were in every personal and group view.

**FR-TGT-08 [v1.0] Delete a target with no history**
- Given a target with no recorded progress that has not yet had a fully elapsed scheduled day, when the owner deletes it, then it is permanently removed.
- Given a target with history, when the owner attempts to delete it, then only Archive is offered (BR-TGT-6).

**FR-TGT-09 [v1.1] Restore an archived target**
- Given an archived target, when the owner restores it, then it is scheduled again from today (if within the 10-target limit); the days it was archived show as "not tracked".

**FR-TGT-10 [v1.1] Future start date**
- Given a new target, when the member picks a start date up to 30 days ahead, then the target is first scheduled on that date and does not appear before it.
- Given a start date later than 30 days from today, when the member saves, then it is rejected (BR-TGT-4).

### 4.4 Sharing and group-scoped visibility

**FR-VIS-01 [v1.0] Choose which groups can see a target**
- Given a member creating or editing a target, when they choose where it is visible, then they can pick any subset of the groups they actively belong to, or none.
- Given the member is viewing a group when they create a target, then that group is pre-selected; otherwise none is.
- Given a member who belongs to no group, when they create a target, then it is private and they can still use the personal view.

**FR-VIS-02 [v1.0] Unshared targets are private**
- Given a target not shared with group X, when any other member of X looks at any screen of X, then they see no trace of it: not its name, its progress, its day count, nor any total that includes it.

**FR-VIS-03 [v1.0] Group views count only shared targets**
- Given a member with shared and private targets, when anyone views group X's heatmap, roll call, streaks or feed, then every number and status for that member is computed only from targets shared with X on the relevant day. "X of Y tasks" in a group tooltip excludes private targets from both X and Y.
- Given a member views their own personal view, when it is shown, then it counts all their active targets.

**FR-VIS-04 [v1.0] Sharing is not retroactive**
- Given a target created on Oct 1 and shared with group X on Oct 10, when X's members view history, then the target counts from Oct 10 onward; Oct 1–9 are not tracked in X.

**FR-VIS-05 [v1.0] Unsharing keeps the past**
- Given a target shared with group X from Oct 1, when its owner unshares it on Oct 20, then the target stops counting in X from Oct 21; Oct 1–20 (including Oct 20's progress) remain in X's heatmap, streaks and feed exactly as they were.
- Given feed cards that list the target, when it is unshared, then those past cards are kept.

**FR-VIS-06 [v1.0] Re-sharing**
- Given a target was unshared from group X and later shared with X again, when X's history is viewed, then the days in between show as not tracked (or reflect only other shared targets) and history on either side is unchanged.

**FR-VIS-07 [v1.0] Two streaks, clearly labelled**
- Given a member who belongs to a group, when the personal view and that group's view are each shown, then the personal view shows their *personal* streak (all targets) and the group view shows their *group* streak (targets shared to that group), each labelled so they are never confused.

### 4.5 Daily checklist

**FR-CHK-01 [v1.0] Today's checklist**
- Given a member opens the app, when Today loads, then it lists every target scheduled today in their local date (shared and private, marked with a share indicator), in creation order, with each task's progress and goal. Completed tasks stay in place.

**FR-CHK-02 [v1.0] Check off a checkbox task**
- Given a CHECKBOX task, when the member taps it, then it shows complete instantly; tapping again undoes it.

**FR-CHK-03 [v1.0] Count tasks**
- Given a COUNT task showing 1 of 3, when the member taps +, then it shows 2 of 3 instantly. The member can also decrease by one or set an exact value.

**FR-CHK-04 [v1.0] Duration tasks**
- Given a DURATION task, when the member taps a quick-add (+15, +30 or +60 minutes), then the minutes are added instantly. The member can also enter an exact total or reduce it.

**FR-CHK-05 [v1.0] Completion and over-achievement**
- Given progress, when it meets or exceeds the goal, then the task is complete. Over-achievement is stored and shown but does not change heatmap level (BR-PRG).

**FR-CHK-06 [v1.0] Edit window: today and yesterday**
- Given the Yesterday section, when the member updates a task there, then it is saved against yesterday's local date. Yesterday remains editable until their local midnight at the end of today.
- Given any day older than yesterday, when a member tries to change a task on it, then the change is rejected, even if the device clock is wrong or altered.

**FR-CHK-07 [v1.0] Missed days need no action**
- Given a member did not open the app on some days, when they open it later, then those days already count as incomplete in all views, and nothing prompts them to "catch up" beyond yesterday.

**FR-CHK-08 [v1.0] Reliable saving**
- Given a member taps a task, when the screen updates instantly but the save then fails (e.g. offline), then the change is reversed with a visible message and a retry option; changes are never silently lost.
- Given a single tap, when the app retries the save, then the tap is applied exactly once.

**FR-CHK-09 [v1.0] Same data on every device**
- Given the same member on two devices, when both change the same task, then the later save wins, and each device shows the saved value after its next refresh.

**FR-CHK-10 [v1.0] Empty and first-run states**
- Given a member with no active targets, when they open Today, then it prompts them to add a target from the templates.
- Given a member with active targets but none scheduled today, when they open Today, then it shows "Nothing scheduled today".

**FR-CHK-11 [v1.0] Daily summary**
- Given a member has tasks scheduled today, when Today is shown, then it displays "N of M done" and the member's personal streak.

### 4.6 Contribution heatmap

**FR-HM-01 [v1.0] Personal heatmap layout**
- Given the personal view, when the screen is at least 640 px wide, then the heatmap shows the last 52 weeks; when it is narrower, the last 13 weeks (about 3 months).
- Given a heatmap is shown, when it is drawn, then weeks start on Monday, month labels run along the top, weekday labels (Mon, Wed, Fri) run down the side, and the current week is the last column with days after today blank.

**FR-HM-02 [v1.0] Cell states and levels**
- Given any day in a heatmap, when it is drawn, then it follows BR-LEVEL: levels 0–4 from the completed/scheduled ratio, plus visually distinct *nothing scheduled*, *not tracked* and *today in progress* states.
- Given a day with nothing scheduled and a day that was missed, when both are shown, then they are visibly different.

**FR-HM-03 [v1.0] Tooltip**
- Given a heatmap cell, when a pointer hovers it or it is tapped, then a tooltip shows the full date, "X of Y tasks", the percentage, and for non-numeric states a label ("Nothing scheduled", "Not tracked").
- Given a keyboard or screen-reader user, when they open the day-by-day list (FR-HM-04), then every day shown gives the same information: full date, "X of Y tasks", percentage and state label.

**FR-HM-04 [v1.0] Accessibility and themes**
- Given a keyboard or screen-reader user, when they reach a heatmap, then it is announced as one item with a text summary of the period, it needs a single Tab stop, and it offers a day-by-day list giving each day's date, "X of Y tasks", percentage and state.
- Given any theme, when a cell or state is shown, then its meaning never depends on colour alone (states also differ by pattern or outline) and contrast meets WCAG 2.1 AA in both light and dark themes.

**FR-HM-05 [v1.0] Group comparison view**
- Given a group, when its comparison view is opened, then it shows one row per active member with avatar, name, group streak, and a heatmap strip over the same dates for every member; the strip covers 52 weeks on screens ≥ 640 px and 8 weeks on narrower screens.
- Given the comparison view, when cells are drawn, then they use the states of FR-HM-02, computed only from targets shared with that group (FR-VIS-03), with the viewer's current week at the right edge; where a member's own local date is behind the viewer's, their later cells are blank.

**FR-HM-06 [v1.0] Default order**
- Given the comparison view, when it is shown, then rows are ordered by current group streak descending, then last-7-day completion %, then name.

**FR-HM-07 [v1.1] Arrow-key navigation**
- Given a keyboard user has focus in a heatmap, when they press an arrow key, then focus moves to the neighbouring day (left and right move one week, up and down move one day) and the whole heatmap needs only one Tab stop.

**FR-HM-08 [v1.1] Sort controls**
- Given the comparison view, when the viewer chooses a sort (current group streak, last-7-day completion %, last-30-day completion %, or name), then rows reorder accordingly.

### 4.7 Streaks

**FR-STK-01 [v1.0] Current streak**
- Given a member, when their streak is displayed, then the current streak follows BR-STREAK, counting all targets in the personal view and only shared targets in each group view.

**FR-STK-02 [v1.0] Retroactive edits recalculate**
- Given a member completes a task for yesterday, when it makes yesterday an active day, then the current streak is recalculated everywhere immediately, including the comparison view.

**FR-STK-03 [v1.0] Today never breaks a streak**
- Given today has scheduled tasks and none completed yet, when the streak is calculated, then today neither adds to nor breaks the streak; it counts as missed only once the day has ended (BR-STREAK-4).

**FR-STK-04 [v1.1] Milestones**
- Given a member's group streak is below a milestone, when a completion brings it to 7, 30, 50 or 100 active days, then a milestone card appears once for that streak run (BR-MILE).

**FR-STK-05 [v1.1] Longest streak**
- Given a member, when their streaks are displayed, then the longest streak also appears next to the current one, computed in the same scope (BR-STREAK-3).

### 4.8 Daily roll call

**FR-ROL-01 [v1.0] Roll call per group**
- Given a group, when its roll call is shown, then every active member is listed under one of five states (BR-ROLL), judged on that member's own local date using only targets shared with the group, with done/scheduled counts (e.g. "2 / 3").
- Given a roll call with members in several states, when it is shown, then sections appear in this order: Active, Not yet, Slacking, Rest day, No targets shared.

**FR-ROL-02 [v1.0] Cutoff**
- Given a member has scheduled shared tasks and no progress today, when the roll call is shown, then they appear as **Not yet** if their local time is before 18:00 and as **Slacking** at or after 18:00.

**FR-ROL-03 [v1.0] Status changes as they act**
- Given a member in Not yet or Slacking records any progress on a shared task, when other members' views next refresh, then they move to **Active**.

### 4.9 Activity feed

**FR-FED-01 [v1.0] Daily progress card**
- Given a member has no card yet for a local day, when they record their first progress on a shared task, then one card is created in each group that task is shared with, showing the member, the date, each shared task with progress, and a summary ("3 of 5").
- Given a card already exists for that day, when the member records further progress, then the same card updates in place and no new card is created.
- Given a card exists, when the member removes all progress for that day, then the card is deleted along with its reactions.

**FR-FED-02 [v1.1] Milestone and joined cards**
- Given a member's group streak reaches a milestone, when it does, then a milestone card is posted to that group (BR-MILE).
- Given a member joins a group for the first time, when the join completes, then a "joined" card appears in that group's feed.

**FR-FED-03 [v1.0] Order and paging**
- Given a feed with more than 20 cards, when the viewer scrolls to the end of what is loaded, then the next 20 load, ordered by most recent activity, newest first.
- Given new cards arrive while the viewer is paging, when the next page loads, then no card appears twice and none is skipped.

**FR-FED-04 [v1.0] Feed respects sharing and membership**
- Given a card in group X, when anyone views it, then it lists only tasks whose targets are shared with X (FR-VIS-03).
- Given a member has been removed, when the feed is viewed, then their cards are hidden from that group (BR-SHARE-5).
- Given a target was later unshared, when old cards are viewed, then they keep their content (FR-VIS-05).

**FR-FED-05 [v1.0] New activity appears without reloading**
- Given a viewer is on the group view, when another member adds or updates a card, then it appears on the viewer's next refresh cycle (NFR-POLL) with no manual reload.

### 4.10 Reactions

**FR-REA-01 [v1.0] React to a card**
- Given a card, when a member taps one of 🔥 💪 👏 😂 👀, then their reaction is added and the count updates; tapping the same emoji again removes it. A member can add any combination of the five, at most one of each, per card (including on their own card).

**FR-REA-02 [v1.1] See who reacted**
- Given a card with reactions, when it is shown, then counts appear per emoji; when the viewer long-presses or hovers an emoji, then the names of who reacted are revealed.

**FR-REA-03 [v1.0] Reactions follow their card**
- Given a card with reactions, when the card is deleted, then its reactions are deleted.
- Given a member who reacted, when they are removed from the group, then their reactions are hidden from that group.

### 4.11 Installable web app (PWA)

**FR-PWA-01 [v1.0] Installable**
- Given the app is opened on a supporting browser, when the browser checks installability, then it meets the install criteria: a manifest with name "Pulse", short name, standalone display, theme and background colours for light and dark, and a start page of Today; icons at 192×192, 512×512, a 512×512 maskable icon, and a 180×180 Apple touch icon.

**FR-PWA-02 [v1.1] Install hint**
- Given a user has completed their first check-off and the app is not installed, when they next see Today, then a dismissible hint explains how to install (the native prompt where available; "Add to Home Screen" steps on iOS). Once dismissed it does not reappear for 30 days.

**FR-PWA-03 [v1.0] Installed behaviour**
- Given the installed app, when it is launched from the home screen, then it opens full-screen with Pulse's icon and name, opens on Today, and keeps the user signed in as in the browser.

**FR-PWA-04 [v1.1] Offline behaviour**
- Given the device is offline, when the app opens, then a friendly "You're offline" screen explains that Pulse needs a connection; no changes are queued or silently lost (FR-CHK-08).

**FR-PWA-05 [v1.0] Works without installing**
- Given a user in an ordinary browser tab, when they use any feature, then it works without installing the app.

### 4.12 Nudges and notifications

**FR-NUD-01 [v1.1] Nudge a slacking member**
- Given a viewer sees a member in **Slacking**, when the viewer taps Nudge, then the member receives an in-app notification naming the sender (BR-NUDGE).
- Given a member who is not Slacking, when a viewer looks at the roll call, then Nudge is unavailable for that member.
- Given a member views their own roll-call row, when it is shown, then Nudge is unavailable.

**FR-NUD-02 [v1.1] One nudge per day**
- Given a viewer already nudged a member today (the *recipient's* local date), when they try again, then it is refused and Nudge shows as "Nudged"; it resets at the recipient's next local day.

**FR-NOT-01 [v1.1] Notification list and unread count**
- Given a member has unread notifications, when they view any screen, then a badge shows the unread count in the main navigation and updates on the next refresh cycle (NFR-POLL-03).

**FR-NOT-02 [v1.1] Read state**
- Given a member with unread notifications, when they open the notification list, then all listed notifications are marked read and the badge clears.
- Given a member already has 50 notifications, when a new one arrives, then only the most recent 50 are kept.

**FR-NOT-03 [v1.1] Notifications are private and group-safe**
- Given a notification, when anyone other than its recipient looks for it, then it is not visible to them.
- Given the sender has since left the group, when the recipient views the notification, then it remains and the sender shows as "a former member".

### 4.13 Pause

**FR-PAU-01 [v1.1] Pause a target**
- Given a target, when the owner chooses a pause start (no earlier than yesterday) and an end date (BR-PAUSE), then every paused day shows as *nothing scheduled* for that target: not a miss, bridging the streak, and not counted in group totals.

**FR-PAU-02 [v1.1] Pause is visible**
- Given a paused target, when its owner opens Today, then it shows "Paused until <date>".
- Given a member whose shared targets are all paused today, when a group's roll call is shown, then the member appears as **Rest day**; otherwise their status comes from their remaining shared targets.

**FR-PAU-03 [v1.1] Change or end a pause**
- Given an active or upcoming pause, when the owner shortens, extends within BR-PAUSE limits, or ends it, then days from the next local day on follow the new range. A pause cannot be edited to cover days older than yesterday.

### 4.14 Weekly quota target

**FR-QTA-01 [v1.1] Weekly quota frequency**
- Given a member creates a target with a weekly quota of Q (1–6) times per week, when they open Today on any day, then the target appears with its weekly progress ("2 of 3 this week").

**FR-QTA-02 [v1.1] Quota credit rule**
- Given a quota target, when it is completed on a day, then that day gets credit of 1 scheduled and 1 completed task from the target (BR-QTA-1).
- Given a quota target is not completed on a day, when that day is evaluated, then it contributes nothing (not scheduled, not missed), and any weekly shortfall appears only in the weekly recap.

### 4.15 Shared group goals

**FR-SGL-01 [v1.1] Create a shared goal**
- Given an active member of a group, when they create a goal (name, mode, goal, DAILY or SPECIFIC_DAYS), then it appears for members to join (BR-SGL-1).

**FR-SGL-02 [v1.1] Opt in**
- Given a shared goal, when a member opts in, then they get their own personal target with the goal's rules, shared automatically to that group; they cannot edit its mode, goal or frequency. It counts toward their 10-target limit.

**FR-SGL-03 [v1.1] Collective progress**
- Given a shared goal with opted-in members, when the group view is shown, then the group sees "x / y done today", where y is opted-in members for whom it is scheduled today (on their own local date, not paused) and x is how many have completed it.

**FR-SGL-04 [v1.1] Opt out**
- Given an opted-in member, when they opt out, then their linked target is archived from the next local day; their history stays.

**FR-SGL-05 [v1.1] End a goal**
- Given the goal's creator or the group owner, when they end it, then all linked targets are archived from the next local day.

### 4.16 Weekly recap

**FR-REC-01 [v1.1] Recap card**
- Given a calendar week (Monday–Sunday) has ended in every active member's timezone, when the group view is opened, then it offers a recap card listing each member's weekly completion % and longest active streak (BR-REC).

**FR-REC-02 [v1.1] Browse recent recaps**
- Given recaps exist, when the viewer browses them, then the last 8 are available.
- Given a member joined mid-week, when the recap is built, then they are judged only from their join date; a member with no tracked days shows "—".

---

## 5. Business rules (exact values)

### BR-DAY — Days and windows
| ID | Rule |
|---|---|
| BR-DAY-1 | A member's **local date** is the calendar date in their profile timezone at the current instant. The server's clock is the only source of "now"; device clocks and dates supplied by a device are never trusted. |
| BR-DAY-2 | Weeks run **Monday to Sunday**. |
| BR-DAY-3 | The **edit window** is the member's local *today* and local *yesterday*. Yesterday stays editable until their local midnight at the end of today. |
| BR-DAY-4 | Every piece of progress belongs to the local date it was recorded for; changing timezone never re-dates it. |
| BR-DAY-5 | After a timezone change the new timezone is used immediately, but a member's current date **never moves backwards**: if the new timezone's date is earlier, they stay on their established date until the new timezone reaches it. |
| BR-DAY-6 | Days are calendar dates, so daylight-saving days of 23 or 25 hours behave like any other day. |

### BR-SCH — When is a target scheduled?
A target is **scheduled** for a member on date D when all of these hold:
1. D is on or after the target's start date.
2. D is before the target's archive effective date (if archived).
3. D is not inside a pause *(v1.1)*.
4. The frequency matches: DAILY → every date; SPECIFIC_DAYS → D's weekday is selected; WEEKLY_QUOTA → see BR-QTA *(v1.1)*.

Rule changes (goal, weekdays) apply only from their effective date. Earlier days are judged on the rules in force on those days.

### BR-PRG — Progress and completion
| ID | Rule |
|---|---|
| BR-PRG-1 | CHECKBOX progress is 0 or 1. COUNT is a whole number 0–999. DURATION is whole minutes 0–1,440 per day per target. |
| BR-PRG-2 | A task is **complete** when progress ≥ goal. Partial progress is *not* complete and does not raise heatmap level. |
| BR-PRG-3 | Over-achievement is stored and shown but never changes levels, streaks or percentages. |
| BR-PRG-4 | DURATION quick-adds are **+15, +30, +60** minutes; any exact value is also allowed. |
| BR-PRG-5 | For the roll call, **any progress above 0** on a scheduled shared task counts as Active. For streaks, at least one task must be *complete*. |

### BR-TGT — Target limits and edits
| ID | Rule |
|---|---|
| BR-TGT-1 | At most **10 active** targets per member, enforced from v1.1. Archived targets don't count. |
| BR-TGT-2 | Target name: 1–60 characters. |
| BR-TGT-3 | Goals: COUNT 1–50 per day; DURATION 5–480 minutes in steps of 5; CHECKBOX has no goal value. SPECIFIC_DAYS: 1–6 weekdays. WEEKLY_QUOTA: 1–6 per week *(v1.1)*. |
| BR-TGT-4 | Start date: **today** in v1.0. From v1.1 a member may also pick any date up to **30 days ahead**. Never in the past. |
| BR-TGT-5 | Edits to goal or weekdays on a target **with history** take effect from the **next local day**. A target with **no recorded progress** whose start date is today or later may be edited freely, effective from its start date. Name edits apply to all days. Tracking mode is locked once any progress exists. |
| BR-TGT-6 | **History** means any recorded progress, or at least one fully elapsed scheduled day. Targets with history are only archived, never deleted. |
| BR-TGT-7 | Archiving takes effect the **next local day**; the same applies to unsharing (BR-SHARE-3). This prevents dodging a miss by hiding a task at 11 p.m. |

### BR-TPL — Templates (defaults, all editable)
| Template | Mode | Default goal | Unit | Default frequency |
|---|---|---|---|---|
| Solve N LeetCode problems | COUNT | 2 | problems | DAILY |
| Work N minutes on side project | DURATION | 60 | minutes | DAILY |
| Study a course for N minutes | DURATION | 45 | minutes | DAILY |
| Read docs for N minutes | DURATION | 30 | minutes | DAILY |
| Make an open-source contribution | CHECKBOX | — | — | SPECIFIC_DAYS (member picks) |
| Revise a DSA topic | CHECKBOX | — | — | DAILY |
| Custom | any | — | — | DAILY |

### BR-SHARE — Sharing and membership effects
| ID | Rule |
|---|---|
| BR-SHARE-1 | A target can only be shared with groups where the owner is an active member. |
| BR-SHARE-2 | **Sharing** counts from the day it is saved, today included. It is never retroactive. |
| BR-SHARE-3 | **Unsharing** takes effect the **next local day** of the target's owner. All days up to and including the day it was unshared remain in that group's views. |
| BR-SHARE-4 | A group's view of a member on date D is computed from exactly the targets that were shared with that group on D. |
| BR-SHARE-5 | When a member **leaves or is removed**, they vanish from that group's views, past ones included (rows, roll call, feed cards, reactions). Their data is kept, and their targets stop being shared with that group immediately. |
| BR-SHARE-6 | On **rejoin**, their earlier shared history in that group reappears and the absence days show as *not tracked*. Nothing is shared again automatically; the member re-shares the targets they want. |
| BR-SHARE-7 | **Deleting a group** *(v1.1)* ends all sharing to it. It never deletes or archives anyone's targets or progress. |

**Worked example (unsharing).** Alex shares *LeetCode* with Group A on Oct 1 and unshares it on Oct 20. Group A's heatmap keeps Alex's cells for Oct 1–20, including Oct 20's progress, unchanged. From Oct 21 the target no longer counts in A. Alex's personal heatmap is unaffected, and Group B (still shared) is unaffected.

### BR-VIS — What each screen counts

| Screen | Counts | Streak shown |
|---|---|---|
| Today, personal heatmap, personal streak | **All** of the member's active targets | Personal |
| Group heatmap, comparison row, roll call, feed; milestone and recap cards *(v1.1)* | **Only** targets shared with *that* group on that day | Group |
| Another member's private target | Never shown to anyone but its owner | — |

### BR-LEVEL — Heatmap day states
For a member and scope (personal, or one group) on date D, let *scheduled* = tasks scheduled and visible in that scope, and *completed* = how many of them are complete. Percentage `p = completed ÷ scheduled × 100` (exact, not rounded first).

| State | When | Look |
|---|---|---|
| **Level 0 — missed** | Tracked, scheduled ≥ 1, `p = 0`, and D is before today | Lowest grey |
| **Level 1** | `0 < p ≤ 25` | Light green |
| **Level 2** | `25 < p ≤ 50` | |
| **Level 3** | `50 < p ≤ 75` | |
| **Level 4** | `75 < p ≤ 100` | Deep emerald |
| **Nothing scheduled** | Tracked, scheduled = 0 | Distinct, neutral style (e.g. dotted outline), never the Level 0 colour |
| **Not tracked** | No target was active (and, for a group, shared) on D | Faint/blank |
| **Today (in progress)** | D is today. Shows the level reached so far, with a "today" marker; with no progress yet, shows the marker only (never "missed") | Outlined |
| **Future** | D is after today | Not drawn |

*Examples:* 1 of 4 = 25% → Level 1. 1 of 3 = 33.3% → Level 2. 3 of 4 = 75% → Level 3. 4 of 5 = 80% → Level 4.

### BR-STREAK — Streaks
| ID | Rule |
|---|---|
| BR-STREAK-1 | Scan days in date order within a scope. An **active day** (≥ 1 scheduled task complete) adds 1. A **rest day** (tracked, nothing scheduled) and a **not-tracked day** add 0 and **do not break** the streak. A **missed day** (tracked, ≥ 1 scheduled, 0 complete) **breaks** it. |
| BR-STREAK-2 | A streak can only *start* on an active day. A member with no tracked targets has a streak of 0. |
| BR-STREAK-3 | **Current streak** = active days in the unbroken run ending today or yesterday. **Longest streak** = the largest such run across all history in that scope (displayed from v1.1, FR-STK-05). |
| BR-STREAK-4 | **Today** with scheduled tasks and no completed task adds 0 and breaks nothing. **Yesterday** counts as missed once today starts, but is restored if it is completed within the edit window. |
| BR-STREAK-5 | Streaks are always recalculated from recorded progress, so retroactive edits (BR-DAY-3) are always reflected. |

**Worked example.** Mon active, Tue active, Wed rest, Thu active, Fri missed, Sat active, Sun (today) active → current streak **2** (Sat, Sun); longest **3** (Mon, Tue, Thu — the rest day bridges).

### BR-MILE — Milestones *(v1.1)*
- Milestones are **7, 30, 50 and 100** active days in a group-scoped current streak.
- Each is posted **once per streak run** per member per group (a run that breaks and rebuilds can post it again).
- A milestone card is removed only if, on the same local day, the progress that triggered it is removed and the streak falls back below the milestone. Earlier cards are never removed.

### BR-ROLL — Roll call states
Evaluated per member, per group, on **that member's own local date**, from targets shared with that group.

| State | Condition |
|---|---|
| **Active** | ≥ 1 scheduled shared task has progress > 0 |
| **Not yet** | Scheduled shared tasks, no progress, local time **before 18:00** |
| **Slacking** | Scheduled shared tasks, no progress, local time **18:00 or later** |
| **Rest day** | The member has shared targets but none is scheduled today |
| **No targets shared** | The member has no shared (active) targets in this group |

The 18:00 cutoff is the same for everyone and not configurable in v1.0 / v1.1.

### BR-FEED — Feed and reactions
| ID | Rule |
|---|---|
| BR-FEED-1 | A daily progress card is identified by (member, local date, group). |
| BR-FEED-2 | Page size is **20**. Order: most recent activity first. |
| BR-FEED-3 | Reactions: exactly 🔥 💪 👏 😂 👀. One of each type per member per card. |

### BR-GROUP — Groups, ownership and invites
| ID | Rule |
|---|---|
| BR-GROUP-1 | A group has **1–15** active members (owner alone is allowed; the 16th joiner is refused). A member may belong to at most **10** groups, enforced from v1.1. |
| BR-GROUP-2 | Group name: 1–40 characters. |
| BR-GROUP-3 | Roles: exactly one OWNER; everyone else MEMBER. |
| BR-GROUP-4 | Invite code: **8 characters** from A–Z excluding I, L, O, plus digits 2–9 (31 symbols), entered case-insensitively. |
| BR-GROUP-5 | *(v1.1)* After **10 failed** code attempts in an hour, a user must wait until the hour has passed. |
| BR-GROUP-6 | Removal is not a ban: a removed member can rejoin with the group's invite. Until the invite can be regenerated (v1.1), the owner cannot prevent this. |
| BR-OWN-1 | *(v1.1)* Owner leaves only after transferring ownership or deleting the group. |
| BR-OWN-2 | Removal and leaving never delete the member's targets or progress (BR-SHARE-5). |
| BR-OWN-3 | *(v1.1)* If the owner's account is deleted, ownership passes to the active member who joined earliest; if there is none, the group is deleted. |

### BR-ID — Identity
- Display name: 1–40 characters.
- Handle: 3–20 characters, lowercase letters, digits and underscore, starting with a letter; unique ignoring case; changeable at any time (the old handle is released).

### Rules for v1.1 features
| ID | Rule |
|---|---|
| BR-NUDGE-1 | A nudge may be sent only to a member in **Slacking**; at most **one per sender → recipient per recipient's local day**, across all groups. No custom text; a fixed friendly message. |
| BR-PAUSE-1 | A pause starts no earlier than **yesterday** and lasts at most **60 days**; pauses on a target may not overlap. |
| BR-PAUSE-2 | Paused days count as *nothing scheduled* for that target. |
| BR-QTA-1 | A quota target is credited on any day it is completed (progress ≥ goal): +1 scheduled, +1 completed. On other days it contributes nothing. Weekly result = credited days ≥ quota; used only by the recap. A week's result is not evaluated for the week in which the target starts if it starts after Monday. |
| BR-SGL-1 | Any active member can create a shared goal (DAILY or SPECIFIC_DAYS only). At most **5** active shared goals per group. |
| BR-REC-1 | Weekly completion % = completed ÷ scheduled tasks for the week in the group's scope. "Longest active streak" = the member's longest group streak as of the end of that week. |
| BR-NOTIF-1 | Each member keeps their most recent **50** notifications. |

---

## 6. Non-functional requirements

### 6.1 Performance
| ID | Requirement |
|---|---|
| NFR-PERF-01 | A check-off is reflected on screen in ≤ **100 ms** of the tap. |
| NFR-PERF-02 | The save is confirmed by the server within **1 s** (95th percentile), when the app is warm. |
| NFR-PERF-03 | Today renders in ≤ **2.5 s** on a mid-range phone on 4G; ≤ **4 s** for the first load after the service has been idle. |
| NFR-PERF-04 | A 52-week heatmap for a 15-member group renders in ≤ **1 s** after data arrives. |

### 6.2 Refresh budget (polling) and freshness
| ID | Requirement |
|---|---|
| NFR-POLL-01 | Only the **group view** refreshes on a schedule, and **only while the app is visible in the foreground**. A hidden or background tab makes **0** requests. Other screens refresh on open and on regaining focus (at most once per 10 s). |
| NFR-POLL-02 | **Cadence and cap.** While the group view is open and visible and the user interacted in the last **3 minutes**: one refresh every **20 s**, i.e. at most **180 requests per active user per hour**. After 3 minutes without interaction, refreshing **stops**. A hidden or background tab never refreshes. When the tab becomes visible again or the user interacts, refreshing **resumes immediately** (an immediate refresh if the data is more than 10 s old, then every 20 s). |
| NFR-POLL-03 | **One request per cycle.** A refresh cycle fetches everything the group view needs (roll call, new feed activity and reactions, and in v1.1 the unread-notification count) in a single request. |
| NFR-POLL-04 | **Planning figures.** Expected use (15 members × 2 visits a day, about 3 minutes each) is 9 to 18 refreshes per visit, so ≈ 8,000 to 16,000 refresh requests a month. Stress ceiling at the cap (15 members interacting continuously for 2 hours a day) ≈ 162,000 a month. The architecture phase must verify that the expected figure plus normal traffic fits the free-tier request quota and database compute hours in force at build time. If it doesn't, then before relaxing the cadence (30 s = 120 per hour, 60 s = 60 per hour) the architecture may shorten the 3-minute idle stop to 1 minute, and as a last resort remove background refreshing (refresh on focus and on pull only). No feature is cut for budget reasons before these steps. |
| NFR-FRESH-01 | Another member's progress appears on a viewer's open group view within **45 s** (95th percentile). A member's own changes appear instantly on their own screen. |

### 6.3 Correctness and consistency
| ID | Requirement |
|---|---|
| NFR-QUAL-01 | Day logic is verified against scenarios covering timezones from UTC−12 to UTC+14, daylight-saving transitions, timezone changes mid-day (forward and backward), week boundaries, the edit window at local midnight, and the streak and level examples in this document. |
| NFR-QUAL-02 | The same member-day shows the same percentage, level and streak contribution on every screen that displays it. |
| NFR-QUAL-03 | No data is lost silently: every change either saves or visibly fails (FR-CHK-08). |

### 6.4 Security and privacy
| ID | Requirement |
|---|---|
| NFR-SEC-01 | Every read and write of group data checks that the actor is an active member (FR-GRP-11). Non-members cannot tell whether a group exists. |
| NFR-SEC-02 | Private targets never leak into any group-visible number, label, total, ordering or card (FR-VIS-02/03). |
| NFR-SEC-03 | Invite codes are unguessable in practice (31⁸ ≈ 8.5 × 10¹¹ possibilities); from v1.1 guessing is also rate-limited (BR-GROUP-5). |
| NFR-SEC-04 | Clock and date values from a device are never trusted (BR-DAY-1). |
| NFR-PRIV-01 | Pulse collects only what it needs: name, handle, avatar, timezone, sign-in identity, and activity the member records. From v1.1, deleted accounts have their personal data erased (FR-ACC-07). |

### 6.5 Accessibility and usability
| ID | Requirement |
|---|---|
| NFR-A11Y-01 | WCAG 2.1 AA for all screens, in light and dark themes. |
| NFR-A11Y-02 | Fully keyboard-operable; sensible screen-reader labels; colour never the only signal; reduced-motion preference respected (animation is used sparingly). In v1.0 each heatmap is a single keyboard stop with a text summary and a day-by-day list; arrow-key navigation arrives in v1.1 (FR-HM-07). |
| NFR-A11Y-03 | Touch targets ≥ **44 × 44 px**. |
| NFR-UX-01 | Mobile-first: all features usable from **360 px** wide with no horizontal page scroll (the heatmap may scroll within itself). |
| NFR-UX-02 | Light and dark themes follow the device setting; the user can override. |

### 6.6 Compatibility, cost and operations
| ID | Requirement |
|---|---|
| NFR-COMPAT-01 | Latest two versions of Chrome, Edge, Firefox and Safari; Safari on iOS 16.4+ for installed use. |
| NFR-COST-01 | Recurring cost is **$0**: free-tier plans only, no paid domain required. |
| NFR-REL-01 | Best-effort availability; no formal SLA. Limits: ≤ 15 members per group from v1.0; ≤ 10 groups and ≤ 10 active targets per member from v1.1. |
| NFR-REL-02 | A member whose account is deleted outside Pulse (in the sign-in account settings) must never cause an error or a blank page for other members, even though account deletion itself is only supported from v1.1 (FR-ACC-07). Until then their earlier entries may remain visible or appear as "Former member", but every group page keeps working, including when that member was the group's owner. |
| NFR-OPS-01 | Errors that affect users are recorded so they can be diagnosed; no third-party analytics or advertising trackers. |

---

## 7. Release scope

### 7.1 v1.0 — "My friends can use it" (58 requirements)

| Area | Requirements |
|---|---|
| Account & profile | FR-ACC-01 … FR-ACC-06 |
| Groups | FR-GRP-01, FR-GRP-02, FR-GRP-03, FR-GRP-05, FR-GRP-07, FR-GRP-11 |
| Targets | FR-TGT-01 … FR-TGT-04, FR-TGT-06 … FR-TGT-08 |
| Sharing & visibility | FR-VIS-01 … FR-VIS-07 |
| Daily checklist | FR-CHK-01 … FR-CHK-11 |
| Heatmap | FR-HM-01 … FR-HM-06 |
| Streaks | FR-STK-01 … FR-STK-03 |
| Roll call | FR-ROL-01 … FR-ROL-03 |
| Feed | FR-FED-01, FR-FED-03 … FR-FED-05 |
| Reactions | FR-REA-01, FR-REA-03 |
| PWA | FR-PWA-01, FR-PWA-03, FR-PWA-05 |

**Accepted v1.0 limits.** In v1.0 the owner cannot rename the group, regenerate the invite, transfer ownership or delete the group; members cannot leave; removal is not a ban (BR-GROUP-6); there is no 10-group, 10-target or code-attempt limit; milestone and "joined" cards, the longest-streak display, sort controls, future start dates, the install hint and the offline screen are absent. All of these ship in v1.1.

**Launch criteria for v1.0:** all v1.0 scenarios pass; NFR-QUAL-01 scenarios pass; the polling cap of NFR-POLL-02 is verified in use; the app installs on a current Android phone and on iOS Home Screen; and the real friend group has signed up and used it for a full day.

### 7.2 v1.1 — "Group management, the social layer and flexibility" (36 requirements)

The order is indicative; the task plan will slice v1.1 further.

| Order | Feature | Requirements |
|---|---|---|
| 1 | Group management & guardrails *(deferred from v1.0)* | FR-GRP-04, FR-GRP-06, FR-GRP-08, FR-GRP-09, FR-GRP-10, FR-GRP-12, FR-GRP-13, FR-ACC-07, FR-TGT-05 |
| 2 | Nudges & in-app notifications | FR-NUD-01, FR-NUD-02, FR-NOT-01 … FR-NOT-03 |
| 3 | Pause a target | FR-PAU-01 … FR-PAU-03 |
| 4 | Weekly quota target | FR-QTA-01, FR-QTA-02 |
| 5 | Shared group goals | FR-SGL-01 … FR-SGL-05 |
| 6 | Weekly recap | FR-REC-01, FR-REC-02 |
| 7 | Feed, heatmap, PWA and target polish *(deferred from v1.0, except FR-TGT-09)* | FR-FED-02, FR-STK-04, FR-STK-05, FR-REA-02, FR-HM-07, FR-HM-08, FR-PWA-02, FR-PWA-04, FR-TGT-09, FR-TGT-10 |

---

## 8. Out of scope

Not in v1.0 or v1.1; may be reconsidered later:

- Push or email notifications (nudges are in-app only; installing the app does not change that).
- Any form of proof, verification or integration with LeetCode, GitHub or learning platforms.
- Offline use and queued changes.
- Instant (real-time) updates.
- Native iOS/Android apps.
- Streak-freeze tokens or other forfeits and rewards.
- Cross-group leaderboards, public profiles, group discovery.
- Chat, comments, free-text notes on check-ins, photo or file uploads.
- Reminders or scheduled prompts.
- Target categories/tags, custom roll-call cutoffs, per-member notification settings.
- More roles than OWNER and MEMBER; multiple owners.
- Data export or import.
- Languages other than English.
- A member detail screen with a person's task-by-task history for other members.
- Payments, ads, or analytics.

---

## 9. Open questions and decisions

No product questions remain open. Everything below was resolved on 2026-10-27; one dependency is handed to the architecture phase.

### 9.1 Resolved

| ID | Question | Decision |
|---|---|---|
| OQ-1 | Unshare and archive timing | Take effect the **next local day**; today still counts (BR-SHARE-3, BR-TGT-7). |
| OQ-2 | Roll-call cutoff | Fixed **18:00** local for everyone, not configurable. |
| OQ-3 | Group strip range | **52 weeks** on wide screens, **8 weeks** on narrow ones; the personal heatmap shows 13 weeks on narrow screens. |
| OQ-4 | Recap "longest active streak" | The member's longest group streak **as of the end of that week**. |
| OQ-5 | Recap timing | Appears once the week has ended in **every** active member's timezone. |
| OQ-6 | Success measures (G6) | As stated in G6. |
| OQ-7 | Who creates shared goals | **Any** active member (v1.1). |
| OQ-8 | Notification content | **Nudges only** (v1.1). |
| OQ-9 | Limits | 15 members per group is enforced from v1.0. The limits of 10 groups per member and 10 active targets per member are enforced from v1.1; the 60-day pause is v1.1. |
| OQ-10 | Free-tier headroom | Handed to the architecture phase (see 9.2). |
| OQ-11 | Rejoining after removal | A removed member **can rejoin** with the same invite; removal is not a ban in v1.0 (BR-GROUP-6). The rejoin scenario in FR-GRP-03 and BR-SHARE-5/6 therefore stay in v1.0. Keeping someone out requires regenerating the invite (FR-GRP-04, v1.1). |
| OQ-12 | Heatmap keyboard model *(rev 2)* | Each heatmap is a **single keyboard stop** with a text summary and a day-by-day list; no arrow-key navigation in v1.0. The list replaces "tooltip on keyboard focus". |
| OQ-13 | Polling idle stop *(rev 2)* | Refreshing stops after **3 minutes** without interaction (was 15); there is no 60-second tier; hidden tabs never refresh; refreshing resumes immediately on focus or interaction. |

### 9.2 Carry-forward notes for the Technical Architecture Document

1. **Free-tier headroom.** Check the polling plan (NFR-POLL-04) against the free-tier request and database-hours quotas current at build time; sources disagreed during discovery.
2. **Deleted users.** Even though FR-ACC-07 is v1.1, a user deleted in the sign-in provider must not break group pages for other members (NFR-REL-02). The design needs a defined behaviour for entries, memberships and ownership that point to a user who no longer exists. Owner succession (BR-OWN-3) is v1.1, so v1.0 only has to keep such a group working.
3. **Cut screens, not concepts.** The v1.0 scope pass removed screens, not ideas. The design should still keep a single owner role, a membership state that can be active, removed or left, a history of when each target was shared with each group (BR-SHARE-4), target rule changes with effective dates (BR-TGT-5), and groups that are retired rather than erased, so v1.1 adds screens without reworking existing data.
4. **No invite-attempt limit in v1.0.** Until BR-GROUP-5 ships in v1.1, protection against guessing rests on the size of the code space (NFR-SEC-03). Revisit if the app is ever opened to the public.
5. **Keyboard focus.** *Resolved in rev 2:* each heatmap is a single keyboard stop with a text summary and a day-by-day list (FR-HM-04, NFR-A11Y-02); arrow-key navigation remains FR-HM-07 (v1.1).
