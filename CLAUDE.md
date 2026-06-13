# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Running the app

No build step. Open `index.html` directly in a browser — all CSS and JS are inline.

Deployed via GitHub Pages at `https://liameaglestone.github.io/world-cup-sweepstakes/`. Every push to `main` triggers a redeploy (takes ~1 minute).

## Architecture

Everything lives in a single `index.html`. The JS is structured in labelled sections (look for `/* ═══ ... ═══ */` banners):

1. **SWEEPSTAKES** — static config array. Each entry defines `id`, `name`, `color`, `buyIn`, `participants`, `teams[]` (each with `espn` display name for API matching), and `prizes[]` (each with a `stat` key that maps to a resolver).

2. **TEAM NAME NORMALISATION** — `ESPN_NAME_MAP` maps ESPN display name variants → canonical names. `normEspn()` is called everywhere a team name crosses the API boundary. If ESPN returns an unexpected variant and a team shows as "Checking…", add the variant here.

3. **APP STATE** — single `state` object. `standings` holds group table data keyed by normalised team name. `matchCache` is the central store for per-match stats, keyed by ESPN event ID.

4. **ESPN API** — three endpoints used:
   - `standings` → group table (GF, GA, position, note)
   - `scoreboard?dates=YYYYMMDD` — single-date only (ESPN does not support date ranges); `wcDates()` generates the full list from `WC_START` to today
   - `summary?event=ID` → match detail; key events (red cards, penalties) live in `data.header.competitions[0].details` with boolean flags `redCard`, `yellowCard`, `penaltyKick`. Do **not** use `data.plays` or `data.keyEvents` — they are unreliable.

5. **DATA PROCESSING** — `seedMatchCache` populates `matchCache` from scoreboards; `updateLiveMatches` refreshes `state.liveMatches` from today's board; `processSummary` enriches a cached match with yellows (from `boxscore.teams[].statistics`), corners, red cards, and penalties.

6. **COMPUTED STATS** — pure functions (`aggGroupYellows`, `aggCleanSheets`, etc.) that derive stats from `state.matchCache` on every render. No stored derived state.

7. **STAT RESOLVERS** — `resolveStat(stat, sweepstake)` maps a prize's `stat` string to `{ label, sub, yours }`. Adding a new prize category means adding a case here and a `stat` key in the config.

8. **REFRESH LOOP** — on boot: load localStorage cache → `refreshData()`. Each refresh: fetch standings + today's scoreboard (always), fetch all historical scoreboards once per session (`_historicalDone` flag), then fetch summaries for any unprocessed completed matches. Processed match summaries are persisted to `localStorage` (key `wc26_matchCache`, 12h TTL) to avoid redundant API calls.

## Adding / updating sweepstakes

- **New sweepstake**: add an entry to the `SWEEPSTAKES` array. Pick a `color` (`gold` | `blue` | `red`) — each has a matching `.card--{color}` CSS class.
- **TBD teams** (Bridgemere has two): set `espn` to the ESPN display name once known, replacing `null`. If ESPN uses an unusual name, add it to `ESPN_NAME_MAP`.
- **New prize stat**: add a `case` to `resolveStat()` and an aggregation function if needed.

## Key constraints

- The penalty `conceder` is the team **opposite** the one in `d.team` (ESPN records the taker, not the conceder).
- `processStandings` handles two ESPN response shapes: `data.standings` as array, or `data.standings.groups` as array.
- Historical scoreboard fetches happen once per session. To force a re-fetch (e.g., after fixing `seedMatchCache`), the manual refresh button resets `_historicalDone = false`.
- The ESPN outbound calls happen in the **browser**, not from this repo's server environment. The ESPN API is blocked from the Claude Code remote environment — don't try to curl/test it from here.
