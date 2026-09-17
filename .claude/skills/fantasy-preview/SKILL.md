---
name: fantasy-preview
description: >
  Generate this week's "The Morning Line" fantasy football preview artifact for
  all three of the user's leagues (one ESPN via MCP tools, two Yahoo via
  claude-in-chrome browser scraping). Use when the user asks for a weekly
  preview, matchup forecast, "the line", or to update/refresh the Morning Line
  for any or all of their leagues.
---

# Fantasy Football Weekly Preview ("The Morning Line")

Produces a forward-looking preview artifact per league: projected matchup
spreads, per-matchup breakdowns, current waiver-wire activity, and a "Waiver
Watch" of top available players. Companion piece to the `fantasy-recap` skill
(that one looks backward at the completed week; this one looks forward at the
upcoming one).

## League registry

Real league IDs, team IDs, display names, tone, and canonical artifact URLs
live in `.claude/skills/fantasy-leagues.local.md` — **gitignored, never
committed**, so this skill file stays shareable without leaking anyone's
private league data. Read that file first; it defines three slots this skill
always operates on:

- **`espn`** — the ESPN league, via MCP tools (`.env` `ESPN_LEAGUE_ID`/`ESPN_TEAM_ID`).
- **`yahoo1`** and **`yahoo2`** — two Yahoo leagues, via claude-in-chrome
  browser scraping (no API access).

Each slot has a **tone**: `uncensored` (full roast voice, profanity OK when
grounded in real numbers) or `reserved` (dial it back — wit and light snark
fine, no profanity, nothing personal). Always check the local file's tone
value per slot before writing a word of prose for that league.

If `.claude/skills/fantasy-leagues.local.md` doesn't exist (fresh clone, new
machine), ask the user for each slot's platform, league ID, their team ID,
league display name, team count/ruleset notes, and tone, then create the file
from their answers using the template at its bottom — don't guess or invent
IDs.

## Canonical artifact URLs — redeploy in place, do not create new artifacts

Publish with `url:` set to the value from `fantasy-leagues.local.md` for that
slot, so the link stays stable week over week. Read the artifact first
(`action: "read"`) if you need to see the current content before editing,
then republish the same file path with the same `url`.

If a URL there is stale (deleted, or `read` fails), ask the user before
creating a fresh one — don't silently fork a new artifact and lose the old
link. Update `fantasy-leagues.local.md` with any new URL immediately.

## Step 1 — ESPN league (MCP tools + direct client script)

1. Load the deferred `mcp__espn-draft__*` tools if needed: `ToolSearch`
   `select:mcp__espn-draft__league_lineup_report,mcp__espn-draft__week_transactions,mcp__espn-draft__waiver_targets,mcp__espn-draft__best_available,mcp__espn-draft__my_roster`.
2. For matchup projections with full lineup + injury/bye detail (richer than
   the MCP tools alone expose), run a python one-off against the repo's own
   client, e.g.:
   ```
   cd <repo> && .venv/bin/python -c "
   from mcp_espn import client
   league = client._connect()
   week = league.current_week
   target_week, recaps = client.get_league_weekly_recaps(week=week)
   # sum projected_points of started results per team, pull top starters,
   # injury_status, bye_week — see prior session's script for the exact shape
   "
   ```
3. Pull this week's transactions + grades via `client.get_week_transactions`,
   `client.get_transaction_value_levels`, `client.get_all_team_rosters`, and
   `fantasy_core.grade_transaction` (mirrors what `week_transactions` MCP tool
   does, but grab the raw `TransactionGrade.reasons` for prose).
4. Pull Waiver Watch via `client.get_available_players(position=<pos>, limit=6)`
   for QB/RB/WR/TE.
5. Rank all matchups by projected margin ascending; band spreads as: `<3`
   TOSS-UP, `3–6` LEAN, `6–10` COMFORTABLE, `>10` LOPSIDED (use a 4th band
   only if margins actually exceed 10; a 3-band system is fine for a tighter
   week).

## Step 2 — Yahoo leagues (claude-in-chrome, repeat per league)

Load browser tools if deferred: `ToolSearch`
`select:mcp__claude-in-chrome__tabs_context_mcp,mcp__claude-in-chrome__navigate,mcp__claude-in-chrome__computer,mcp__claude-in-chrome__read_page,mcp__claude-in-chrome__tabs_create_mcp,mcp__claude-in-chrome__tabs_close_mcp,mcp__claude-in-chrome__get_page_text,mcp__claude-in-chrome__find,mcp__claude-in-chrome__browser_batch`.

1. `tabs_context_mcp{createIfEmpty:true}`, then `navigate` to
   `https://football.fantasysports.yahoo.com/f1/<league_id>` and
   `get_page_text`. If it doesn't show the user's team name/league (login
   expired), tell the user to log into Yahoo Fantasy in Chrome and stop —
   don't attempt Yahoo credential entry yourself.
2. This single page gives you, in one shot:
   - **All matchups for the current week** with projected totals per side
     (pre-kickoff — actual points show as 0.00 until games start).
   - **Full standings**: W-L-T, PF, PA, Streak, season waiver-move count.
   - **Recent Transactions** (last ~5 — thin; see step 3 for the full log).
3. Navigate to `https://football.fantasysports.yahoo.com/f1/<league_id>/transactions`
   for the complete recent transaction list (typically the last ~9). Yahoo
   doesn't expose a VOR/replacement-level grade like the ESPN pipeline does —
   write qualitative "take" commentary per move instead, grounded in name
   recognition and roster fit you can infer (e.g. "cut a clear starter for a
   questionable bench piece"). Never invent a numeric grade for Yahoo moves.
4. Navigate to `https://football.fantasysports.yahoo.com/f1/<league_id>/players?status=A&sort=AR`
   for the available-player pool sorted by Yahoo's overall rank. Pull the
   top few per position (QB/RB/WR/TE — RB pool in the 12-team IDP league runs
   very thin, worth flagging explicitly if so) for Waiver Watch. Rows show
   rank, weekly proj points, and `% Ros`; a `W (date)` roster-status tag means
   the player is on post-drop waivers until that date, not yet a true free
   agent — mention that explicitly for anyone flagged this way.
5. For your own team's matchup with full starter-by-starter detail, navigate
   to `https://football.fantasysports.yahoo.com/f1/<league_id>/<my_team_id>/matchup`.
   Other matchups only need the league-home totals — don't burn extra page
   loads pulling every team's full lineup; a team-level narrative built from
   standings (record, PF/PA, streak) plus any transactions found in step 3 is
   the right level of depth for a weekly preview.
6. Close any tab you opened with `tabs_close_mcp` when done with that league.

## Step 3 — Build the artifact

Reuse the established "Morning Line" visual system across all three editions
for series identity (same fonts: Special Elite display, Source Serif 4 body,
JetBrains Mono data/labels; same dark-first noir-newsprint palette structure)
but give each league its own accent color so they're visually distinct at a
glance — accent colors per slot are listed in `fantasy-leagues.local.md`
(espn: rust-red, yahoo1: teal-green, yahoo2: purple as of this writing). Keep
each league's accent consistent week to week.

Sections, in order:
1. Masthead (title + league name/week/team-count meta).
2. "Game of the Week" banner — closest projected spread, with a one-line hook
   from real data (a transaction, a bench situation, a streak).
3. A short "flip-note"/wire-watch callout — one or two sentences flagging
   something worth tracking (a repeated past mistake, a hot roster).
4. "The Line" table — all matchups ranked tightest-spread-first, with a
   TOSS-UP/LEAN/COMFORTABLE(/LOPSIDED) read chip.
5. "Matchup Files" — one card per game. Two-column sides where you have
   player-level detail (ESPN: every game; Yahoo: your own game only, others
   get a prose note per side instead of a stat list).
6. Wire/transactions section — ESPN gets real letter grades; Yahoo gets
   qualitative takes.
7. "Waiver Watch" grid by position.
8. Footer with a data-source/scope caveat (Yahoo editions: no VOR grading,
   scraped from the site not the API) and the spread-band legend.

Before writing code, run `Skill("artifact-design")` if it hasn't loaded this
session, and load `artifact-capabilities` only if you're adding a capability
(not needed for this static weekly report).

**Mobile sizing, learned the hard way**: don't give `table.line` / `table.wire`
a `min-width` above ~420–480px — anything higher (600–640px, which is what
these tables shipped with originally) forces horizontal scrolling on every
real phone, since actual phone viewports run 360–430px. Keep `min-width` in
that 420–480px range, add `word-break:break-word` on the `td`/`th` rule (long
unspaced team/manager names otherwise blow out the cell), and add a
`@media (max-width:480px){ table.line th, table.line td, table.wire th,
table.wire td{padding:7px 8px; font-size:.78rem} }` block so the table itself
shrinks rather than relying purely on the horizontal-scroll wrapper.

Publish each file with `Artifact` using its canonical `url` from the table
above so the link stays the same every week.

## Tone calibration reminder

Ground every line in a real number from this week's data — never fabricate a
stat, a grade, or a "started X over Y" claim you didn't actually verify.
Check each slot's tone in `fantasy-leagues.local.md` before writing: a
`reserved` slot stays clean-ish and playful over profane; only an
`uncensored` slot can run hot.
