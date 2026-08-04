# Wild Ice Scheduler

A replacement for the ice-allocation spreadsheet. The weekly grid the scheduler already
works in — one column group per day, a sub-column per rink, blocks coloured by level —
except a double-booking is **impossible to save** rather than something you catch by eye.

Single self-contained HTML file. No build step, no install, no server. Runs on any device.

**Live site:** <https://kp-coding-co.github.io/ice-scheduler/>

## What it does

- **Schedule** — the grid. Time down the side, rink columns grouped under their day,
  coloured blocks sized by duration, closed hours hatched. Click a block to edit it, click
  open ice to add a team. Swap two teams, undo/redo, publish to coaches. Also a **By team**
  view — one row per team, one column per day — which is what a coach opens the link for.
- **Ice** — the weekly rink windows: which sheet, which day, what hours. Date caveats live
  as notes on the window (`*Oct 13`), the same way they're written on the sheet today.
- **Fill gaps** — say who still needs ice and it works out where the remaining ice times
  fit *around everything already placed*. It doesn't move your work unless you ask it to.
  Anything it can't fit comes back with the reason.
- **Teams & Rules** — teams by level, the rink list, the age-group rule matrix, and a
  weekly usage report showing hours per team and how the prime ice got divided.

## Conflict detection

The point of the whole thing. Two teams can't be put on the same sheet at the same time:

- Every edit checks live as you pick times, **names the booking in the way**, and disables
  Save until it's resolved.
- Swapping two teams is checked too — it's refused if it would put a team in two places.
- Anything already in stored data (from an import or a restore) is caught by a whole-schedule
  scan and surfaced in a banner with links straight to each clash. Overlapping blocks are
  drawn side by side rather than on top of each other, so a conflict is always visible.

## The rules it enforces

Per age group, editable on the **Teams & Rules** tab:

- **Ice length** — how long one ice time runs
- **Earliest and latest start time** — separately for weeknights and weekends

The window is about when a team may *take* the ice, not when it has to be off: a U9 team
with a 6:15p latest start can skate until 7:15p, it just can't be handed a 6:30p slot.

Balanced as preferences rather than rules: prime-time and off-peak fairness across teams,
not stranding gaps too short for anyone to use, keeping teams off back-to-back days, home
rink. House-league teams can share one sheet across age groups (`U13+U18 HL`), and the
youngest bands can run cross-ice.

> **The default rules are placeholders.** They need the association's real numbers before
> any allocation should be trusted. The app says so on the rules tab.

## The week it opens with

Transcribed from the association's spreadsheet. The six rinks, their weekly windows and the
team list should be right; the individual bookings were read off a screenshot and will be
wrong in places. The app shows a banner saying so until it's dismissed.

## Season model

A repeating weekly template — Monday to Sunday — matching how ice is actually held and
negotiated. There are no calendar dates in the app. The `.ics` export turns each ice time
into a weekly recurring event through the end of the season, so a coach subscribes once and
their calendar stays right.

The trade-off: a holiday week, bye week or tournament that displaces the pattern can't be
expressed as data, only as a note on the window. If that starts to bite, the fix is additive
— keep the template as the planning artifact and let it generate dated instances.

## Run locally

Open `index.html` in any modern browser.

## Deploy

**Settings → Pages**, source `main`, root folder. Every push redeploys in ~30 seconds.

## Data

Everything lives in the browser's `localStorage` by default — one scheduler, one machine, no
setup. The ⚙ menu exports and restores a JSON snapshot.

To share the schedule so everyone with the link sees the same grid, create your own Supabase
project and fill in `SUPABASE_URL` and `SUPABASE_ANON_KEY` at the top of `index.html`.
`HANDOFF.md` has the SQL and the trade-offs — chiefly that there's no login, and the
passphrase on publishing is a speed bump against accidents rather than real security.

## Files

- `index.html` — the app
- `HANDOFF.md` — engineering notes: data model, the allocator, code locations, open work
