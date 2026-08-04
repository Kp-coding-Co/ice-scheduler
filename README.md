# Ice Scheduler

Browser app for allocating **multiple teams into ice-time openings across multiple rinks**.
Give it the blocks of ice you've bought, the teams that need ice, and each age group's
rules — how long their ice time runs, and the earliest and latest they're allowed to start —
and it fills the week.

Single self-contained HTML file, no build step, no server. Runs on this device by default;
optional shared sync if you wire up a Supabase project.

**Live site:** <https://kp-coding-co.github.io/ice-scheduler/>

## What it does

- **Ice** — the calendar of every block of ice the association has purchased, per rink,
  with a fill bar showing how much of each block is allocated. Add blocks one at a time or
  as a recurring weekly pattern across a date range, which is how ice actually gets bought.
  Mark a block **blocked** for a tournament, public skate, or maintenance and the allocator
  leaves it alone.
- **Allocate** — pick a week, check supply (ice per day per rink) against demand (per-team
  session counts, defaulting to each team's weekly target, overridable for the week), and
  allocate. Anything it can't place comes back with the reason.
- **Schedule** — the week board, day by day and rink by rink, with open gaps shown inline
  so you can see what's left. Swap two teams, drop a team into a gap, nudge a start time,
  change length or session type, undo/redo/reset, publish. Conflict detection is live.
  Exports: printable sheet, CSV, and `.ics` calendar files for the whole association or
  one team.
- **Teams & Rules** — teams with weekly targets, home rinks and blackout dates; the rink
  list; the editable age-group rule matrix; and a season usage report showing hours per
  team and how the prime ice got divided.

## The rules it enforces

Per age group, and editable:

- **Ice length** — how long one ice time runs (50m for the youngest bands up through 75m
  for the oldest, by default)
- **Earliest and latest start time** — separately for weeknights and weekends

The window is about when a team may *take* the ice, not when it has to be off. A U9 team
with a 6:15p latest start can skate until 7:05p; it just can't be handed a 6:30p slot.

These are hard: the allocator leaves a request unplaced rather than break one, and tells
you why. Rink collisions, team collisions and blackout dates are hard too.

Balanced as soft preferences: prime-time and off-peak fairness across teams, not stranding
gaps too short for anyone to use, keeping teams off back-to-back days and off two ice times
in a day, and home-rink preference. The youngest bands can share a sheet where you allow it.

Each run uses a fresh random seed, so if you don't like a week, allocate again and you'll
get a different — equally well-scored — arrangement.

## Run locally

Open `index.html` in any modern browser. No build step, no install.

## Deploy via GitHub Pages

1. **Settings → Pages**, source `main` branch, root folder.
2. Site is served at `https://<username>.github.io/ice-scheduler/`.

Every push to `main` redeploys within ~30 seconds.

## Data

Everything lives in the browser's `localStorage` by default — one scheduler, one machine,
no setup. The ⚙ menu exports a JSON snapshot for backup and restores one.

To share the schedule so everyone with the link sees the same board, create your own
Supabase project and fill in `SUPABASE_URL` and `SUPABASE_ANON_KEY` at the top of
`index.html`. `HANDOFF.md` has the one-time SQL and explains the trade-offs — chiefly that
there's no login, and the passphrase gate on publishing is a speed bump against accidents
rather than real security.

## Files

- `index.html` — the app
- `HANDOFF.md` — engineering notes: data model, the allocator, code locations, open work
