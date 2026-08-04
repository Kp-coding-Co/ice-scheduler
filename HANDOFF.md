# Wild Ice Scheduler — Handoff

Reference doc for the next coding session. Last updated: 2026-08-04.

---

## What this is

A replacement for a minor hockey association's ice-allocation spreadsheet. One
self-contained HTML file, React + Babel from CDN, no build step, four tabs, dark mode.

The interface began as a clone of a baseball lineup builder
(`Kp-coding-Co/lineupbuilder2026`) and kept its shape — tab structure, the solver behind one
button, the warn-don't-block validation panel. It is otherwise a completely separate app:
own repo, own Pages site, own storage keys, own database. Neither app can reach the other's
data.

- **File**: `index.html` — the entire app
- **URL**: <https://kp-coding-co.github.io/ice-scheduler/>

### What it's for

The stated pain: conflicts are only caught manually, and finding one means rejigging the
week by hand. So the app does two things the spreadsheet can't —

1. **Makes a double-booking impossible to save.** Not a warning; a disabled Save button.
2. **Re-jigs for you.** The "Fill gaps" tab places what's still needed around what's already
   there, under the age-group rules.

Commercial tools in this space (RinkBook, Swift, EZFacility, SportsKey) do (1). None do (2),
and none do "ice requested vs received" — see Open Work.

---

## Season model: weekly template, no dates

**The most important structural decision in the app.** The schedule is a repeating
Monday–Sunday template, matching how ice is actually held and negotiated. There are no
calendar dates in the data model. `slot.day` and `assignment.day` are `0`–`6` (0 = Sunday).

Date caveats from the source sheet (`*Oct 13`, `*Sept 15`) live as **free text** in
`slot.note`. They're deliberately not modelled.

The only two dates in the codebase are `SEASON_FIRST_MONDAY` and `SEASON_LAST_DATE`, used
solely by the `.ics` export to emit `RRULE:FREQ=WEEKLY;UNTIL=…`. A coach subscribes once and
their calendar tracks the template.

**Known limitation:** a holiday week, bye week or tournament that displaces the pattern can't
be expressed. The upgrade is additive — keep the template as the planning artifact and let it
generate dated instances — but don't build it until someone actually asks.

---

## Data model

All arrays, stored under `ice_*` keys in localStorage (mirrored to Supabase when configured).

### `rinks` — `ice_rinks`
```js
{ id, name, short }
```

### `teams` — `ice_teams`
```js
{
  id,
  ageGroup: "u7"|"u9"|"u11"|"u13"|"u15"|"u18"|"senior",
  level: "Rep"|"AA"|"A"|"BB"|"B"|"HL"|"C",
  label: string,               // suffix for split squads: "Blue", "White"
  sessionsPerWeek: number,
  homeRinkId: string | "",     // preference, not a constraint
  blackoutDows: number[],
  notes: string,
}
```
**There is no `name` field.** A team's name is derived by `teamName(t)` as
`"<age> <level>"` plus the suffix — `U11 BB`, `U9 HL - Blue`. That's what the block says on
the sheet, so it's the single source of truth.

### `slots` — `ice_slots` — a weekly rink window
```js
{ id, rinkId, day: 0-6, start: "HH:MM", end: "HH:MM", note, blocked, blockLabel }
```

### `assignments` — `ice_assignments` — one scheduled ice time
```js
{ id, slotId, rinkId, day, start, end, teamIds: string[], kind, shared }
```
`teamIds` holds **two** entries for shared ice. Modelling it as one booking with two teams
(rather than two bookings) is what keeps "this sheet served two teams" true in every report
and export.

### `rules` — `ice_rules` — keyed by age group
```js
{
  durationMin: 60,
  weekday: { earliest: "17:00", latest: "19:15" },   // earliest/latest START
  weekend: { earliest: "07:00", latest: "19:00" },
  allowShared: false,          // cross-ice within the band
  maxPerDay: 1, maxPerWeek: 4,
}
```
The window governs when a team may **take** the ice, not when it must be off.

### `meta` — `ice_meta`
```js
{ published: bool, seedBanner: bool, version }
```
Bumping `DATA_VERSION` re-seeds a device onto the new shape.

---

## Levels drive the colour

`LEVEL_COLORS` (light and dark variants), keyed by level, is the visual language — the source
spreadsheet colours by level, not age, because a scheduler scanning the grid asks "did the
Rep teams get prime ice?". Age rides along as a small grey chip.

`SHARE_LEVELS` (currently `HL`) is the set of levels whose teams may share a sheet **across
age groups** — that's how a `U13+U18 HL` block works. `canShare(rules, a, b)` returns true if
both age groups allow cross-ice **or** both teams are the same shareable level.

---

## Conflict enforcement

Three layers:

- `findRinkConflict(assignments, {rinkId, day, start, end}, ignoreId)` — the hard one. Called
  live from `AssignModal` and `IceTimeModal`; a hit names the booking and disables Save.
- `findTeamConflict(...)` — same, for a team being in two places. Also gates the swap path in
  `ScheduleTab.handleBlockClick`.
- `scanConflicts(assignments)` — whole-schedule sweep for anything that arrived by import or
  restore. Feeds the banner and the red hatched blocks in the grid.

Overlapping blocks are laid out in **side-by-side lanes** (`assignLanes`) rather than stacked,
so a conflict is always visible. The original React prototype explicitly couldn't do this.

---

## The grid

`ScheduleGrid` — CSS Grid. Columns are `(day, rink)` pairs that have ice; day headers span
their rink sub-columns. Rows are **15 minutes** (`STEP`), not 30.

> **Don't change `STEP` to 30.** This association's ice starts at 6:15, 5:15, 3:15, 8:30. A
> 30-minute grid rounds those to the wrong slot and silently corrupts the schedule.

The visible time range comes from `gridBounds(slots)` — computed from the ice that actually
exists, so a rink running to 11:15pm is shown rather than clipped.

Phones default to the **By team** view; the grid is a scheduler's tool and needs width.

---

## The allocator

`solveSchedule()` — greedy with restarts, best of 60.

1. Expand requests: one per team per ice time needed.
2. Order **hardest-first** — narrowest start window picks first. Ties broken randomly, which
   is what makes restarts explore.
3. Enumerate legal placements: inside a window, inside the band's start range, long enough,
   no rink or team collision, not a blackout. Plus **join** placements for shared ice.
4. Score and take the cheapest, with jitter. Cost terms by weight: stranded ice (priced per
   wasted minute), prime/off-peak fairness, same-day and back-to-back spacing, window
   position, home rink.
5. Score the week — unplaced dominates, then stranded ice, then variance in prime/off-peak.

**Additive by default.** `FillTab` passes everything already on the board as `locked`, so a
run fills gaps without moving existing work. "Start the week over" is an explicit opt-in.
This framing is deliberate: the allocator is there to answer "I found a conflict and now I
have to rejig everything", not to write the season.

Each click passes a fresh random `seed`, so "try again" explores a different arrangement.

Performance: ~1s for 18 teams and a week of ice at 60 runs, in the browser, no worker.

---

## Persistence

localStorage is the working store. `SUPABASE_URL`/`SUPABASE_ANON_KEY` are **deliberately
blank** — the app runs local-only and the header reads *Local only*. This is not a degraded
state.

To share, create a **new** Supabase project (never point this at another app's project),
paste the URL and anon key into those constants, and run once:

```sql
create table if not exists public.ice_data (
  id int primary key,
  rinks jsonb, teams jsonb, slots jsonb,
  assignments jsonb, age_rules jsonb, meta jsonb
);
alter table public.ice_data enable row level security;
drop policy if exists "ice_data_anon_select" on public.ice_data;
create policy "ice_data_anon_select" on public.ice_data for select to anon using (true);
drop policy if exists "ice_data_anon_update" on public.ice_data;
create policy "ice_data_anon_update" on public.ice_data for update to anon using (true) with check (true);
insert into public.ice_data (id) values (1) on conflict (id) do nothing;
```

Once on: the anon key is public by design, there's no login, and `EDIT_PASSPHRASE`
(`coldsteel`) is client-side only. It gates publishing and restoring a backup — a speed bump
against accidents, not security. The intended access model is "the scheduler edits, coaches
read", enforced socially by who knows the passphrase.

---

## Code locations

| Thing | Search for |
|---|---|
| Palette / theme | `C_LIGHT`, `C_DARK`, `applyTheme` |
| Level colours | `LEVEL_COLORS_LIGHT`, `levelColor`, `SHARE_LEVELS` |
| Team naming | `function teamName` |
| Age rules | `DEFAULT_RULES`, `ruleFor`, `windowFor`, `canShare` |
| Seed data | `DEFAULT_RINKS`, `T`, `DEFAULT_SLOTS`, `SEED_BOOKINGS` |
| Free-space math | `freeSegments`, `slotUtilization` |
| Allocator | `candidatePlacements`, `placementCost`, `solveOnce`, `solveSchedule` |
| Conflicts | `findRinkConflict`, `findTeamConflict`, `scanConflicts`, `assignLanes` |
| Validation | `validateSchedule` |
| The grid | `ScheduleGrid`, `gridBounds`, `ROW_H` |
| Exports | `exportCSV`, `exportICS`, `exportPrintable` |
| Tabs | `ScheduleTab`, `IceTab`, `FillTab`, `TeamsTab` |
| Passphrase gate | `EDIT_PASSPHRASE`, `EditGate` |

---

## Testing

No test runner. The app is driven in headless Chromium via Playwright scripts kept in the
working scratchpad (not committed): render, open a block, force a conflict and assert Save is
disabled, add into open ice, swap, fill gaps, all three exports, publish gate, the other tabs,
dark mode, iPhone viewport.

**CDNs are blocked in the dev sandbox**, so the harness rewrites the CDN `<script>` tags to
npm-installed copies of the same React/Babel versions before loading. The committed file's
CDN tags are never modified.

---

## Open work

- **The rules are placeholders.** Ice lengths and start windows are guesses. Nothing else
  matters until they're real — every allocation depends on them.
- **The seeded bookings are a screenshot reading** and will be wrong in places. The in-app
  banner says so; clear it once they're corrected.
- **"Ice received vs requested"** — the association tracks allocated-vs-confirmed on a
  separate spreadsheet tab. None of the four commercial tools cover it. The most defensible
  thing on this list, and the natural next build.
- **No real auth.** The passphrase is shared and client-side. Per-user identity would need a
  login and would make an audit log meaningful; neither exists.
- **No audit log.** No record of who changed what.
- **Games vs practices.** `kind` exists and shows in exports, but the allocator treats all
  requests alike — games realistically want longer ice, an opponent and home/away.
- **Travel time between rinks.** Nothing stops a team getting ice at two arenas an hour apart
  on the same day beyond the same-day penalty.
- **Rink column order** within a day follows the `rinks` array, not the source sheet's order.
  Cosmetic, but it's a recognition detail if someone's comparing side by side.
