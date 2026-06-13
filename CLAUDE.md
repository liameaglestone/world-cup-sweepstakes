# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Running the app

No build step. Open `index.html` directly in a browser — all CSS and JS are inline.

Deployed via GitHub Pages at `https://liameaglestone.github.io/world-cup-sweepstakes/`. Every push to `main` triggers a redeploy (takes ~1 minute).

## Architecture

Everything lives in a single `index.html`. The JS is structured in labelled sections (look for `/* ═══ ... ═══ */` banners):

1. **SWEEPSTAKES** — static config array (~line 584). Each entry defines `id`, `name`, `color`, `buyIn`, `participants`, `teams[]` (each with `espn` display name for API matching), and `prizes[]` (each with a `stat` key that maps to a resolver).

2. **TEAM NAME NORMALISATION** (~line 650) — `ESPN_NAME_MAP` maps ESPN display name variants → canonical names. `normEspn()` is called everywhere a team name crosses the API boundary. If ESPN returns an unexpected variant and a team shows as "Checking…", add the variant here.

3. **APP STATE** (~line 676) — single `state` object. `standings` holds team stats keyed by normalised team name. `matchCache` is the central store for per-match stats, keyed by ESPN event ID. `liveMatches` holds currently in-progress matches.

4. **ESPN API** (~line 691) — three endpoints used:
   - `scoreboard?dates=YYYYMMDD` — **single-date only** (ESPN does not support date ranges); `wcDates()` generates the full list from `WC_START` (`2026-06-11`) to today
   - `summary?event=ID` → match detail; key events (red cards, penalties) live in `data.header.competitions[0].details` with boolean flags `redCard`, `penaltyKick`. Do **not** use `data.plays` or `data.keyEvents` — they are unreliable.
   - `standings` — **returns `{}` for 2026 WC** (ESPN hasn't populated it). Standings are instead computed from match results via `computeStandingsFromMatches()`.
   - `apiFetch()` uses a manual `AbortController` + `setTimeout` for the request timeout (not `AbortSignal.timeout()` which isn't supported in Safari < 16).

5. **DATA PROCESSING** — `seedMatchCache` (~line 828) populates `matchCache` from scoreboards; `updateLiveMatches` refreshes `state.liveMatches` from today's board; `processSummary` (~line 927) enriches a cached match with yellows, corners, red cards, and penalties.

6. **COMPUTED STATS** — pure functions (`aggGroupYellows`, `aggCleanSheets`, etc.) that derive stats from `state.matchCache` on every render. No stored derived state.

7. **STAT RESOLVERS** — `resolveStat(stat, sweepstake)` (~line 1082) maps a prize's `stat` string to `{ label, sub, yours }`. Adding a new prize category means adding a case here and a `stat` key in the config.

8. **REFRESH LOOP** (~line 1361) — on boot: load localStorage cache → `refreshData()`. Each refresh: fetch standings + today's scoreboard (always), fetch all historical scoreboards once per session (`_historicalDone` flag), then fetch summaries for any unprocessed completed matches. Processed match summaries are persisted to `localStorage` (key `wc26_matchCache_v2`, 12h TTL) to avoid redundant API calls.

9. **DIAGNOSTICS** (~line 477) — `runDiagnostics()` runs a series of ESPN API calls from the browser and displays results on-screen. Triggered by the 🔍 Diagnose button. Shows scoreboard events with status/teams/IDs and match summaries with full boxscore stat names and detail event flags. Since ESPN is blocked from the Claude Code server environment, this is the primary way to inspect live API responses.

## Known ESPN quirks for 2026 WC

- **Match completion status**: ESPN uses `STATUS_FULL_TIME` (not `STATUS_FINAL`) for completed 2026 WC matches. `matchIsCompleted()` checks `comp.status.type.completed === true` OR status name patterns (FINAL/FULL_TIME/ENDED/COMPLETE).
- **Standings API empty**: `fetchStandings()` tries 4 URL variants; all return `{}`. Use `computeStandingsFromMatches()` instead.
- **`yellowCard` boolean absent**: The `yellowCard` boolean flag in `details[]` is undefined in 2026 WC data. Yellow card counts come from `data.boxscore.teams[].statistics` where `s.name === 'yellowCards'` (case-insensitive check: `key === 'yellowcards'`).
- **Corner stat name unknown**: The correct `s.name` for corners in boxscore statistics is not yet confirmed (`cornerKicks` returns undefined). Use the diagnostic panel to inspect all stat names from a match summary.
- **`type.text` undefined**: In `details[]` entries, `d.type?.text` is undefined. Use `d.type?.id` and `d.type?.description` instead.

## `processSummary` detail

Called once per completed match. Reads from two places in the ESPN summary response:

- **Yellow cards & corners**: `data.boxscore.teams[].statistics[]` — find by `s.name` (lowercased). Yellow: `yellowcards`. Corner: TBD (run diagnostic to find the name).
- **Red cards**: `data.header.competitions[0].details[]` — `d.redCard === true` or `d.yellowRedCard === true`. Records `{ team, min }` in `cached.reds`.
- **Penalties**: same `details[]` — `d.penaltyKick === true`. The **conceder** is the team *opposite* `d.team` (ESPN records the taker, not the conceder).
- Fallback: if `details` is empty, scans `data.commentary` / `data.plays` for red/penalty keywords.
- Sets `cached.processed = true` and calls `saveCache()` when done.

## `computeStandingsFromMatches`

Derives W/D/L/GF/GA/pts/GD from every completed match in `matchCache`. Sets `computed: true` on each entry. Only fills in teams not already in `state.standings` (so ESPN data takes precedence if it ever comes back). Called at the end of every `refreshData()` cycle.

## Adding / updating sweepstakes

- **New sweepstake**: add an entry to the `SWEEPSTAKES` array. Pick a `color` (`gold` | `blue` | `red`) — each has a matching `.card--{color}` CSS class.
- **TBD teams** (Bridgemere has two slots): set `espn` to the ESPN display name once known, replacing `null`. If ESPN uses an unusual name, add it to `ESPN_NAME_MAP`.
- **New prize stat**: add a `case` to `resolveStat()` and an aggregation function if needed.

## Key constraints

- The ESPN outbound calls happen in the **browser**, not from this repo's server environment. The ESPN API is blocked from the Claude Code remote environment — don't try to curl/test it from here. Use the on-screen 🔍 Diagnose button to inspect API responses.
- Historical scoreboard fetches happen once per session. To force a re-fetch (e.g. after fixing `seedMatchCache`), the manual refresh button resets `_historicalDone = false`.
- Cache key is `wc26_matchCache_v2` — bump the version suffix when match processing logic changes, to bust stale cached `processed:true` entries.
- `processStandings` handles two ESPN response shapes: `data.standings` as array, or `data.standings.groups` as array (currently moot since the API returns `{}`).
