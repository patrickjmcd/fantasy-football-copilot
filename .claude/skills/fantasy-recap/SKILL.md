---
name: fantasy-recap
description: >
  Generate the "Roster Malpractice" weekly recap artifact for a completed
  week, for all three of the user's fantasy football leagues (one ESPN via
  MCP tools, two Yahoo via claude-in-chrome browser scraping). Use when the
  user asks for a weekly recap, results writeup, "who screwed up this week",
  or to update/refresh Roster Malpractice for any or all of their leagues.
---

# Fantasy Football Weekly Recap ("Roster Malpractice")

Looks backward at the week that just finished: final scores, lineup
mismanagement (ESPN only — see scope note below), and graded transactions.
Companion piece to `fantasy-preview` (forward-looking, same leagues).

## League registry

Same three slots as `fantasy-preview` (`espn`, `yahoo1`, `yahoo2`) — see that
skill's League registry section for the full explanation. Real IDs, team
IDs, display names, tone, and canonical artifact URLs live in
`.claude/skills/fantasy-leagues.local.md` (gitignored, never committed).
Repeated here because it's the thing most likely to get missed: **check each
slot's tone before writing a word** — `reserved` slots (dial it back, no
profanity, nothing personal) vs. `uncensored` (full roast voice, profanity OK
when grounded in real numbers).

## Canonical artifact URLs — redeploy in place, do not create new artifacts

Use the "Recap" column of the URL table in `fantasy-leagues.local.md`. If a
URL there is stale (deleted, or `read` fails), ask the user before forking a
new artifact, and update the local file with any new URL immediately.

## Scope: Yahoo CAN do full league-wide lineup grading — use the right URL

Earlier drafts of this skill claimed Yahoo only exposes bench data for your
own matchup. **That was wrong, twice, and cost real rework to fix — don't
repeat either mistake:**

1. `https://football.fantasysports.yahoo.com/f1/<league_id>/<team_id>/matchup?week=<N>`
   (a team_id in the path) is a dead end — it always resolves to *your own*
   matchup no matter what team_id you put there. Do not use this pattern for
   anyone but yourself.
2. The real, working, per-matchup route is:
   `https://football.fantasysports.yahoo.com/f1/<league_id>/matchup?week=<N>&mid1=<teamA>&mid2=<teamB>`
   — **no team_id path segment**, just `mid1`/`mid2` query params naming
   *either* two teams in the league. This works for any matchup, not just
   your own, and was verified across every game in both Yahoo leagues in one
   session. If a `mid1`/`mid2` request ever bounces back to the
   matchups list, it's very likely a transient browser/tab issue (a stuck
   tab was the actual cause once) — open a fresh tab and retry before
   concluding it's blocked.

Get team IDs from the league home page: `navigate` to
`https://football.fantasysports.yahoo.com/f1/<league_id>`, then
`read_page` with `filter: "interactive"` (scroll down first if the list gets
cut off) — every team name link's `href` ends in `/f1/<league_id>/<team_id>`.
Map every team once per league; team IDs don't change week to week.

For each matchup: navigate to the `mid1`/`mid2` URL, `find` the "Show Bench
Players" link, click it (a single click regularly fails to register — if
`get_page_text` still shows "Show Bench Players" instead of "Hide Bench
Players", `find` a fresh ref and click again, sometimes twice), then
`get_page_text`. That one page gives starters + bench, actual and projected
points, for **both** teams.

**Column order is easy to transpose — verify every single one.** Both the
starters and bench tables read left-to-right as `[leftProj, leftActual, POS,
rightActual, rightProj]`. Reading this quickly from a wall of text produces
exactly the actual/proj swap bug that bit this skill twice in one session
(right-side numbers silently swapped, only caught because the final report
step happened to checksum against Yahoo's own displayed total). **Do this
for real, don't skip it**: transcribe directly from the tool's *current*
output (never from memory of an earlier turn — that's how both errors
happened), sum each side's starters, and assert the sum equals the page's
own displayed total for that side before trusting any of it. A Python
script with a `report()` function that prints "starters sum check: X (given:
Y)" for every team, run once for the whole league, surfaces every
transcription error in one pass — far more reliable than eyeballing 10+
matchups of ~20 players each by hand.

Compute the optimal lineup once data is verified: for each required
single-position slot, take the best scorer at that position from the
combined starters+bench pool (excluding true IR/empty slots — but a player
merely IR-*eligible* who actually started and scored real points is a normal
rostered player for this purpose, not an exclusion); fill FLEX with the best
remaining eligible player after required slots are assigned. "Would have won"
= your optimal total beats the opponent's *actual* score (matches
`fantasy_core`'s own convention) — check both directions per matchup, since
sometimes both sides misplayed and the result doesn't flip either way, which
is worth saying plainly.

**Bottom line**: full lineup-efficiency grading (actual vs. optimal, points
left on the bench, specific start/sit mistakes, an A–F grade, "would have
won" flips) is achievable for **every team in a Yahoo league**, matching the
ESPN edition's depth. The only real gap left vs. ESPN is transaction VOR
grading (no replacement-level baseline available for Yahoo) — keep
transaction commentary qualitative there, as in `fantasy-preview`.

## Step 1 — ESPN league (MCP tools + direct client script)

1. Load `mcp__espn-draft__league_lineup_report` and
   `mcp__espn-draft__week_transactions` (`ToolSearch` if deferred).
2. Call `league_lineup_report(week=<completed week>, vulgar=<per tone>)` —
   this alone gives every team's actual vs. optimal score, points left on
   bench, start/sit mistakes, and manager grade.
3. Call `week_transactions(week=<completed week>)` for graded moves.
4. Write it up in the "Roster Malpractice" voice: a "Dipshit of the Week"
   banner (worst points-left-on-bench, ideally in a loss), a docket table of
   all teams ranked worst-lineup-management-first with grade chips, and a
   "Case Files" card per team with 2–3 sentence roast-grounded-in-numbers
   verdicts plus "exhibit" chips for notable transactions.

## Step 2 — Yahoo leagues (claude-in-chrome, repeat per league)

Load browser tools if deferred (same set as `fantasy-preview`).

1. `tabs_context_mcp{createIfEmpty:true}`, `navigate` to
   `https://football.fantasysports.yahoo.com/f1/<league_id>`, `get_page_text`.
   This gives standings (PF/PA/record) and a short recent-transactions list.
2. Get every team's id from the league home page (see the scope section
   above), then pull full starters+bench box scores for **every matchup**
   via the `mid1`/`mid2` URL — this is now the default, not a fallback.
   Sanity-check the closest/biggest-margin pairings against the "Biggest
   Blowout of the Week" callout the page itself shows.
   (The old PF/PA-pairing trick for reconstructing scores without opening a
   page per game still works as a quick cross-check, but the real box scores
   are cheap enough now that there's no need to rely on it alone.)
3. Navigate to `https://football.fantasysports.yahoo.com/f1/<league_id>/transactions`
   for the transaction log. Filter mentally to moves dated before/during the
   week being recapped (the feed isn't week-scoped, so a move from days
   before kickoff is season-prep, not an in-week panic move — don't
   overstate its urgency). Moves dated the Tuesday morning after the week
   closed are actually prep for the *next* week — that's `fantasy-preview`'s
   material, not this week's; skip or only briefly forward-reference them
   ("...and it got worse — see this week's Morning Line").
4. No VOR grading available for Yahoo — write qualitative takes per move, as
   in `fantasy-preview`.
5. Close tabs when done with that league.

## Step 3 — Build the artifact

Same "Roster Malpractice" visual system as the existing ESPN edition (dark
newsprint/docket aesthetic, Special Elite + Source Serif 4 + JetBrains Mono),
same per-league accent colors as `fantasy-preview` uses for that league, for
visual continuity between a league's preview and recap editions.

Sections — **same structure for Yahoo and ESPN now**, matching the existing
"Roster Malpractice" artifact:
1. Masthead.
2. "Dipshit of the Week" banner (worst points-left-on-bench, ideally in a
   loss) — for a `reserved`-tone slot, rename to something cleaner like
   "Lineup of the Week (the bad kind)" per the tone rule, same
   content.
3. Flip-note: every "would have won with the optimal lineup" case in the
   league.
4. Docket table — every team ranked worst-lineup-management-first: result,
   grade chip, optimal total, points left on bench.
5. Case Files — one card per team, 2–4 sentences, real numbers (who started
   over whom, points lost), grade stamp.
6. Standings snapshot.
7. Footer noting the actual scope gap (transaction VOR grading, ESPN only)
   and crediting the data source (Yahoo: pulled per-matchup, no API).

If time/turns don't allow pulling every Yahoo matchup's bench data in a
session, it's fine to grade only some teams — but say so plainly in the
docket/footer (e.g. "ungraded — bench data not pulled this round") rather
than inventing numbers for a team you didn't actually fetch.

Run `Skill("artifact-design")` first if not already loaded this session.
Publish with the canonical `url` so each league's recap link stays stable
week over week; after the first Yahoo publish, update this file's URL table
above with the new links.

**Mobile sizing, learned the hard way**: keep `table.docket`'s `min-width`
around 480px, not 560–640px — real phones run 360–430px wide, and anything
higher forces horizontal scrolling on the docket/standings tables that are
the core of this report. Shorten the "Left on bench" header to "Bench" in
the `<th>`. Add `@media (max-width:480px){ table.docket th, table.docket
td{padding:7px 8px; font-size:.78rem} .stat-chip{font-size:.66rem} }`, and
make sure the `@media (max-width:520px){ .case-top{flex-direction:
column-reverse} .stamp{align-self:flex-end} }` block is present (it's easy
to drop when spinning up a new league's file from an existing one).

## Tone calibration reminder

Ground every line in a real number from that week's data. Check each slot's
tone in `fantasy-leagues.local.md` before writing: `reserved` stays
clean-ish, no profanity; `uncensored` can run hot, profanity fine, as long as
it's earned by the numbers.
