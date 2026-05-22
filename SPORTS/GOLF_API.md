# Golf API Specification

Base URL: `{API_BASE_URI}` (configured via environment, e.g. `http://localhost:8081/api/`).

All endpoints expect/return `application/json`. Read endpoints are public; the admin editor uses the same `x-access-token` JWT contract as the rest of the API (see [`AUTH_API.md`](../AUTH_API.md)). Times are ISO 8601 unless explicitly marked as pre-formatted display strings.

This document is the single source of truth for golf. It covers two surfaces:

| Surface              | Section                                                | Used by                                                       |
|----------------------|--------------------------------------------------------|---------------------------------------------------------------|
| Public display       | [§1 Public display endpoints](#1-public-display-endpoints) | Golf explorer, leaderboard page, match-detail page, carousel, ticker |
| Admin score editor   | [§2 Admin score editor](#2-admin-score-editor)         | `GolfScoreEditorComponent` (`/scores/:slug`)                  |

The admin editor and the public display piggy-back on the generic `/matches` resource documented in [`MATCHES_API.md`](../MATCHES_API.md). The golf-specific shapes below extend that contract; they do not replace it.

Golf supports three competition formats. They share the same `Match` envelope but differ in what goes into `details`:

| Format             | `details.format`     | When used                                                      |
|--------------------|----------------------|----------------------------------------------------------------|
| Stroke play        | `STROKE_PLAY`        | Tournament leaderboard across N rounds (e.g. 72-hole event)    |
| Match play         | `MATCH_PLAY`         | Head-to-head between two players/teams, hole-by-hole           |
| Team competition   | `TEAM_COMPETITION`   | Ryder Cup–style multi-session team event with aggregate points |

For stroke-play tournaments the field is typically 50+ players and `homeTeamName` / `awayTeamName` are not meaningful in the team sense; see [§2.3](#23-field-rules-the-backend-must-enforce) for the convention.

---

## Display surfaces and the data each needs

| Component                      | Endpoint                                  | Purpose                                              |
|--------------------------------|-------------------------------------------|------------------------------------------------------|
| `featured-match`               | `GET /match/past-featured`                | Carousel of recent featured golf events              |
| `top-knock-ticker`             | `GET /match/past-featured`                | Top scorers ticker (same source)                     |
| `match-summary-carousel`       | `GET /match/golf/summaries`               | Mixed past/upcoming/live summary card strip          |
| `match-schedule`               | `GET /match/list-match`                   | Upcoming + recent events + countdown                 |
| `match-page` (event detail)    | `GET /match/golf/:slug`                   | Full leaderboard / scorecard / hole-by-hole          |
| Live in-page banner (optional) | `GET /match/golf/:slug/live`              | Lightweight live leaderboard / hole polling          |
| Golf score editor              | `GET /matches/:slug` + `PUT /matches/:slug` | Admin edits via the generic `/matches` resource    |

The frontend caches `past-featured` and per-slug event detail with `shareReplay(1)`, so these endpoints should be cheap and cacheable.

---

## Field conventions (shared across all sections)

- **Slugs**: every event is addressed by `slug` (kebab-case, URL-safe). Carousel and ticker may use a separate `embedSlug` alias; both MUST resolve to the same event detail.
- **Scores**:
  - Stroke play surfaces strokes vs. par. Encode `total` and `today` as integers where `0` is even-par (`E`), negative is under par. The display string (e.g. `"-5"`, `"E"`, `"+3"`) is derived client-side.
  - Match play surfaces "holes up" with holes remaining, and a result label such as `"3 & 2"`, `"1 UP"`, `"AS"`.
  - Team competition surfaces aggregate points (halves count as `0.5`) and per-pairing match-play status.
- **Times**:
  - Schedule and live endpoints — ISO 8601 with timezone (`2024-10-06T07:15:00+04:00`).
  - Match detail `startTime`, `eventDate`, `lastUpdated` are pre-formatted display strings and are rendered verbatim.
- **Unknowns**: prefer omitting the field or returning `null`. Do not use sentinel values like `-256`.
- **Empty strings**: `videoEmbedUrl`, `imageUrl`, and `result` may be empty strings when not yet known; the frontend treats them the same as missing.
- **`sport`**: golf endpoints always return `"GOLF"`, but the field is required on summary / featured shapes so a future cross-sport endpoint can reuse the same payload.
- **`thru`**: number of holes completed in the current round (`0..18`). The string `"F"` MAY be sent for a finished round; the client also accepts the integer `18` as equivalent.
- **Par**: course par (e.g. `72`) is per round. Stroke-play `total` and `today` are vs par.

---

# 1. Public display endpoints

Read endpoints consumed by the public golf pages. All are public (no auth).

## 1.1 List events (schedule)

Golf events appear in the same `/match/list-match` feed as other sports. Filter by `sport=GOLF`.

### `GET /match/list-match`

```
Query params (optional):
  sport?:      "GOLF"               // when omitted, returns all sports
  tournament?: "string"             // tournament slug
  from?:       "ISO8601"
  to?:         "ISO8601"
  status?:     "SCHEDULED | LIVE | COMPLETED | CANCELLED | POSTPONED"

Response 200: MatchScheduleItem[]
```

For golf events the existing `MatchScheduleItem` shape applies (see [`CRICKET_API.md` §1.1](./CRICKET_API.md)) with these conventions:

- `homeTeam` / `awayTeam`:
  - **Match play** — the two opponents (player or team names).
  - **Team competition** — the two team names (e.g. `"USA"`, `"EUROPE"`).
  - **Stroke play** — set both to the event name (e.g. `"Doha Open"` / `"Doha Open"`) or `homeTeam: "<event name>"`, `awayTeam: ""`; the client treats empty `awayTeam` as a stroke-play tournament card.
- `venue` is the course name (e.g. `"Doha Golf Club"`).
- `ground` MAY be a course label (e.g. `"Championship Course"`).

---

## 1.2 Past featured events (carousel + ticker)

Recent completed golf events with a compact summary. Drives the `featured-match` carousel and `top-knock-ticker`.

### `GET /match/past-featured`

```
Query params (optional):
  limit?: number   // default 10
  sport?: "GOLF"   // when omitted, mixes sports

Response 200: FeaturedMatchSummary[]
```

```jsonc
// FeaturedGolfSummary — the GOLF variant of FeaturedMatchSummary
{
  "embedSlug":  "doha-open-2026",
  "slug":       "doha-open-2026",
  "sport":      "GOLF",                          // drives the sport icon on the home carousel & ticker
  "tournament": "Doha Open 2026",
  "homeTeam":   "Doha Open 2026",                // see §1.1 conventions
  "awayTeam":   "",
  "startTime":  "2026-03-12T07:00:00+03:00",
  "venue":      "Doha Golf Club",
  "derived": {
    "wonBy":   "by 3 strokes",                   // human-readable result
    "comment": "Round 4 of 4"
  },
  "scoreboard": {
    "format":  "STROKE_PLAY",                    // STROKE_PLAY | MATCH_PLAY | TEAM_COMPETITION
    "winner":  "Tiger Woods",                    // player name, team name, or "" for tied team events
    "strokePlay": {                              // present when format = STROKE_PLAY
      "par":          72,
      "rounds":       4,
      "currentRound": 4,
      "leaders": [                               // top 3, ordered
        { "rank": 1, "name": "Tiger Woods",  "country": "USA", "total": -18, "today": -6, "thru": "F", "strokes": 270 },
        { "rank": 2, "name": "Rory McIlroy", "country": "NIR", "total": -15, "today": -4, "thru": "F", "strokes": 273 },
        { "rank": 3, "name": "Jon Rahm",     "country": "ESP", "total": -12, "today": -2, "thru": "F", "strokes": 276 }
      ]
    },
    "matchPlay": null,                           // present when format = MATCH_PLAY
    "teamCompetition": null                      // present when format = TEAM_COMPETITION
  }
}
```

Match play / team competition variants of `scoreboard`:

```jsonc
// scoreboard for format = MATCH_PLAY
"scoreboard": {
  "format": "MATCH_PLAY",
  "winner": "Tiger Woods",
  "strokePlay": null,
  "matchPlay": {
    "homePlayer":   "Tiger Woods",
    "awayPlayer":   "Phil Mickelson",
    "homeUp":       3,                           // negative when away is up; 0 when AS
    "holesRemaining": 0,
    "holesPlayed":  16,                          // useful for "3 & 2" labelling
    "resultLabel":  "3 & 2"
  },
  "teamCompetition": null
}

// scoreboard for format = TEAM_COMPETITION
"scoreboard": {
  "format": "TEAM_COMPETITION",
  "winner": "USA",                               // "" when halved
  "strokePlay": null,
  "matchPlay": null,
  "teamCompetition": {
    "homeTeam":       "USA",
    "awayTeam":       "EUROPE",
    "homeTeamPoints": 14.5,                      // halves count as 0.5
    "awayTeamPoints": 13.5,
    "totalPoints":    28,
    "resultLabel":    "USA WIN 14½ – 13½"
  }
}
```

Field semantics:
- `format` MUST be one of the three values; the client switches the card body based on it.
- `leaders` MAY be truncated server-side. The carousel shows the top three; the detail page (§1.4) carries the full leaderboard.
- `total`, `today` are integers vs. par (`0` for even). Negative is under par.
- `thru` is `0..18` or the literal string `"F"`. The client accepts either `"F"` or `18` for a finished round.

---

## 1.3 Event summaries (carousel)

Compact summary cards shown at the top of the golf explorer page. Drives the `match-summary-carousel` component. Mixes past results, in-progress events, and the next few upcoming events so the same card shape can render any of them.

### `GET /match/golf/summaries`

```
Query params (optional):
  limit?:      number   // default 10, hard cap 20
  tournament?: "string" // tournament slug; when omitted, mixes tournaments
  include?:    "PAST | LIVE | UPCOMING" (comma-separated; default "PAST,LIVE,UPCOMING")

Response 200: GolfSummary[]   // newest/next-up first
```

```jsonc
// GolfSummary
{
  "slug":            "doha-open-2026",
  "embedSlug":       "doha-open-2026",
  "sport":           "GOLF",
  "format":          "STROKE_PLAY",                    // drives the card body
  "tournament":      "Doha Open 2026",
  "tournamentLabel": "DOHA OPEN",                      // short code shown top-right
  "status":          "UPCOMING | LIVE | RESULT",
  "startTime":       "2026-03-12T07:00:00+03:00",
  "venue":           "Doha Golf Club",
  "venueLabel":      "DOHA GOLF CLUB",                 // optional pre-uppercased display

  // Stroke play card body — present when format = STROKE_PLAY
  "strokePlay": {
    "par":          72,
    "rounds":       4,
    "currentRound": 3,                                 // 0 for upcoming
    "leader": {                                        // null for upcoming
      "name":    "Tiger Woods",
      "country": "USA",
      "total":   -15,
      "today":   -4,
      "thru":    "F"
    },
    "fieldSize":    132                                // optional
  },

  // Match play card body — present when format = MATCH_PLAY
  "matchPlay": null,
  //   { "homePlayer": "...", "awayPlayer": "...", "homeUp": 2, "holesRemaining": 4, "resultLabel": "2 UP (thru 14)" }

  // Team competition card body — present when format = TEAM_COMPETITION
  "teamCompetition": null,
  //   { "homeTeam": "USA", "awayTeam": "EUROPE", "homeTeamPoints": 8.5, "awayTeamPoints": 7.5, "totalPoints": 28 }

  "result": {                                          // omit / null when status !== "RESULT"
    "kind":  "WIN | TIE | NO_RESULT",
    "label": "TIGER WOODS WON BY 3 STROKES"
  }
}
```

Field semantics:

- Exactly one of `strokePlay` / `matchPlay` / `teamCompetition` is non-null, matching `format`.
- `status` MUST agree with `result`: `UPCOMING` / `LIVE` carry no `result`, `RESULT` MUST carry one.
- For `UPCOMING` events `leader` / `homeUp` / point totals MUST be null (or absent) — the card shows tee time only.
- `tournamentLabel`, `venueLabel`, and `embedSlug` follow the same rules as in [`CRICKET_API.md` §1.3](./CRICKET_API.md).

Caching: same expectations as `match/past-featured` — `shareReplay(1)` on the client, so `Cache-Control: public, max-age=60` is fine.

---

## 1.4 Event detail (full leaderboard / scorecard page)

Powers `match-page` for golf — tabs: Overview, Leaderboard (or Scorecard for match play), Hole-by-Hole, Players, Gallery. The frontend interface is `GolfEventDetail` in `@core/services/golf.service.ts`.

### `GET /match/golf/:slug`

```
Path params:
  slug: string   // canonical event slug

Response 200: GolfEventDetail
Response 404: { "error": "Event not found" }
```

```jsonc
// GolfEventDetail (top-level)
{
  "slug":         "doha-open-2026",
  "tournament":   "Doha Open 2026",
  "seriesName":   "PGA Tour 2026",
  "status":       "Past | Live | Upcoming",
  "format":       "STROKE_PLAY | MATCH_PLAY | TEAM_COMPETITION",
  "stage":        "Round 3 | Quarter Final | Singles | null",
  "venue":        "Doha Golf Club",
  "city":         "Doha",
  "location":     "Doha Golf Club, Doha",            // display string
  "courseName":   "Championship Course",
  "coursePar":    72,
  "courseYards":  7374,                              // optional
  "rounds":       4,                                 // total scheduled rounds
  "currentRound": 4,                                 // 0 when upcoming, equals rounds when complete
  "cutLine":      -1,                                // optional, stroke play only; integer vs par or null
  "startTime":    "12-Mar-26 07:00 AM",              // display string (already formatted)
  "eventDate":    "12/03/2026 – 15/03/2026",         // display string

  "result":       "Tiger Woods won by 3 strokes",    // free text; empty string when no result

  "leaderboard":      [ LeaderboardEntry, ... ],     // stroke play (and team competition by team)
  "matchPlayCard":    MatchPlayCard,                 // match play only
  "teamCompetition":  TeamCompetitionDetail,         // team competition only

  "highlights":   [ Highlight, ... ],                // optional shot-of-the-day reel
  "videoEmbedUrl":"",                                // empty string when no video
  "streamTitle":  "Final Round — Doha Open 2026",
  "rightsHolder": "Rangers Media",
  "totalViews":   1240,
  "liveViewers":  8,

  "lastUpdated":   "2026-03-15 at 18:21",
  "lastUpdatedBy": "Jins Manager"
}
```

### Sub-shapes

```jsonc
// LeaderboardEntry — stroke play (one row per player)
{
  "rank":       1,                              // T-rank shown as "T1" by the client; the integer here is the numeric rank
  "tied":       false,                          // true when one or more entries share this rank
  "name":       "Tiger Woods",
  "playerId":   "tiger-woods",                  // optional FK to Player.slug
  "country":    "USA",                          // ISO 3-letter code
  "imageUrl":   "",                             // empty string allowed
  "total":      -18,                            // strokes vs par across all completed rounds
  "today":      -6,                             // strokes vs par for the current round
  "thru":       "F",                            // 0..18 or "F"
  "strokes":    270,                            // total strokes
  "status":     "ACTIVE | CUT | WD | DQ",       // WD = withdrawn, DQ = disqualified, CUT = missed the cut
  "rounds": [
    {
      "round":      1,
      "strokes":    68,
      "vsPar":      -4,
      "tee":        "07:42",                    // optional, scheduled tee time as display string
      "holes": [
        { "hole": 1,  "par": 4, "yards": 412, "strokes": 4 },
        { "hole": 2,  "par": 5, "yards": 552, "strokes": 4 }
        // … 18 entries when the round is complete; shorter while in progress
      ]
    }
    // … one entry per round actually started
  ]
}
```

```jsonc
// MatchPlayCard — present when format = MATCH_PLAY
{
  "homePlayer":    "Tiger Woods",
  "awayPlayer":    "Phil Mickelson",
  "homePlayerId":  "tiger-woods",               // optional FK
  "awayPlayerId":  "phil-mickelson",            // optional FK
  "homeCountry":   "USA",
  "awayCountry":   "USA",
  "holesPlayed":   16,
  "holesRemaining": 0,                          // 18 - holesPlayed (or fewer for closed-out matches)
  "homeUp":        3,                           // negative when away is up; 0 when AS
  "resultLabel":   "3 & 2",                     // empty string while in progress; e.g. "1 UP", "AS", "2 & 1"
  "holes": [
    { "hole": 1, "par": 4, "homeStrokes": 4, "awayStrokes": 5, "status": "HOME" },
    { "hole": 2, "par": 5, "homeStrokes": 5, "awayStrokes": 5, "status": "HALVED" }
    // … one entry per hole played
  ]
}
```

`MatchPlayCard.holes[].status` is `HOME | AWAY | HALVED`. The hole counter rolls forward only as holes are completed; a tied 17th hole on a 2 UP match closes the match `2 & 1` and `holes[]` MUST stop there.

```jsonc
// TeamCompetitionDetail — present when format = TEAM_COMPETITION
{
  "homeTeam":       "USA",
  "awayTeam":       "EUROPE",
  "homeTeamPoints": 14.5,                       // halves count as 0.5
  "awayTeamPoints": 13.5,
  "totalPoints":    28,
  "winningPoints":  14,                         // points needed to win (28-team Ryder Cup → 14½)
  "resultLabel":    "USA RETAIN 14½ – 13½",     // empty string while in progress
  "sessions": [
    {
      "name":       "Day 1 Foursomes",          // display label
      "sessionFormat": "FOURSOMES | FOURBALLS | SINGLES",
      "order":      1,                          // 1-based, for client sorting
      "status":     "SCHEDULED | LIVE | COMPLETED",
      "pairings": [
        {
          "order":         1,
          "homePlayers":   ["Scottie Scheffler", "Justin Thomas"],
          "awayPlayers":   ["Rory McIlroy", "Jon Rahm"],
          "homeUp":        2,                   // negative when away is up; 0 when AS
          "holesPlayed":   16,
          "holesRemaining": 0,
          "status":        "IN_PROGRESS | HOME_WIN | AWAY_WIN | HALVED",
          "resultLabel":   "2 & 1",             // empty string while in progress
          "pointsHome":    1,                   // 0, 0.5, or 1 once complete
          "pointsAway":    0
        }
      ]
    }
  ]
}
```

```jsonc
// Highlight (optional)
{
  "title":       "Tiger eagles the 18th",
  "playerName":  "Tiger Woods",
  "round":       4,
  "hole":        18,
  "videoEmbedUrl": "https://www.youtube.com/embed/...",
  "thumbnailUrl":  ""
}
```

`videoEmbedUrl` (top-level and per-highlight) MUST be either an empty string or a full URL whose host is one of: `www.youtube.com`, `youtube.com`, `player.vimeo.com`. Any other host is rejected by the frontend sanitizer.

Notes:
- The `leaderboard` array MUST be sorted by `rank` ascending; the client trusts that order.
- For team competition events, `leaderboard` MAY also be present and carry per-team totals (one entry per team with `name` = team name, `total` = points, `status` = `ACTIVE`); the per-pairing detail lives in `teamCompetition.sessions[]`.
- For stroke-play events, `matchPlayCard` and `teamCompetition` MUST be `null` (or omitted), and vice versa.

---

## 1.5 Live event polling (optional, for in-progress events)

Lightweight payload for polling/SSE while an event is `LIVE`. Avoid returning the full `GolfEventDetail` on every poll.

### `GET /match/golf/:slug/live`

```jsonc
Response 200:
{
  "slug":      "doha-open-2026",
  "status":    "LIVE",
  "format":    "STROKE_PLAY",                        // STROKE_PLAY | MATCH_PLAY | TEAM_COMPETITION
  "currentRound": 3,

  // Stroke play — present when format = STROKE_PLAY
  "leaderboard": [                                   // top N only (typically 10), already sorted
    { "rank": 1, "name": "Tiger Woods",  "total": -15, "today": -4, "thru": 14, "country": "USA" },
    { "rank": 2, "name": "Rory McIlroy", "total": -13, "today": -3, "thru": 15, "country": "NIR" }
  ],

  // Match play — present when format = MATCH_PLAY
  "matchPlay": null,
  //   { "homeUp": 2, "holesRemaining": 4, "currentHole": 15, "resultLabel": "2 UP (thru 14)" }

  // Team competition — present when format = TEAM_COMPETITION
  "teamCompetition": null,
  //   { "homeTeamPoints": 8.5, "awayTeamPoints": 7.5, "liveSession": "Day 2 Fourballs" }

  "lastUpdated":    "2026-03-15T13:11:00Z"
}
```

---

# 2. Admin score editor

This section covers the endpoints the **Golf Score Editor** (`/scores/:slug` → `GolfScoreEditorComponent`) calls when an admin updates a golf event. Golf scores piggy-back on the generic `/matches` resource (documented in [`MATCHES_API.md`](../MATCHES_API.md)); the editor never calls a golf-specific endpoint. All golf-specific state lives inside the `details` object on a `Match`.

## 2.1 Endpoints the editor calls

### `GET /matches/:slug` — load event into the editor

Called once when the editor opens. Response hydrates the editor for whichever format the event uses.

```json
Response 200: Match  // shape defined in MATCHES_API.md, with golf details described in §2.2 below
```

### `PUT /matches/:slug` — save full event OR save live update

The editor sends the **same payload shape** for both tabs:

- **Full Event save:** sends the form's `status` as-is.
- **Live save:** forces `status: "LIVE"` and only patches the in-progress fields (leaderboard entry for the player who just finished a hole, or the active pairing's `homeUp` / `holesPlayed`); all other `details.*` keys are echoed back unchanged from the previously loaded event.

The backend MUST treat this as a full replacement of the match document — the client always sends the complete state it wants persisted, including the full `leaderboard` / `matchPlayCard` / `teamCompetition` block. Do not attempt to diff or merge server-side.

```json
Response 200: Match  // the updated match, used to re-hydrate the editor
```

There is no separate "live" endpoint. The backend does not need to expose one.

---

## 2.2 Match payload (golf)

The full `PUT /matches/:slug` request body the editor sends:

```json
{
  "sport":          "GOLF",
  "tournamentSlug": "string | null",
  "matchType":      "LEAGUE | KNOCKOUT | FRIENDLY | INDEPENDENT | null",
  "homeTeamName":   "string",
  "awayTeamName":   "string",
  "venue":          "string?",
  "scheduledAt":    "ISO8601?",
  "referee":        "string?",
  "status":         "SCHEDULED | LIVE | COMPLETED | CANCELLED | POSTPONED",
  "notes":          "string?",
  "homeScore":      "number | null",
  "awayScore":      "number | null",
  "details": {
    "format":       "STROKE_PLAY | MATCH_PLAY | TEAM_COMPETITION",
    "coursePar":    "number?",
    "courseYards":  "number?",
    "rounds":       "number?",          // total scheduled rounds (stroke play / team comp)
    "currentRound": "number?",          // 0 when upcoming
    "cutLine":      "number | null",    // stroke play only; integer vs par
    "winner":       "string?",          // player or team name; empty string when no winner yet

    // ---- STROKE_PLAY: leaderboard
    "leaderboard": [
      {
        "rank":     "number",           // 1-based
        "tied":     "boolean?",
        "name":     "string",
        "playerId": "string | null",
        "country":  "string?",          // ISO 3-letter
        "total":    "number?",          // vs par
        "today":    "number?",          // vs par for current round
        "thru":     "number | \"F\" | null", // 0..18 or "F"
        "strokes":  "number?",
        "status":   "ACTIVE | CUT | WD | DQ",
        "rounds": [
          {
            "round":   "number",
            "strokes": "number?",
            "vsPar":   "number?",
            "tee":     "string?",       // display string
            "holes": [
              { "hole": "number (1..18)", "par": "number?", "yards": "number?", "strokes": "number?" }
            ]
          }
        ]
      }
    ],

    // ---- MATCH_PLAY: head-to-head card
    "matchPlay": {
      "homePlayer":     "string",
      "awayPlayer":     "string",
      "homePlayerId":   "string | null",
      "awayPlayerId":   "string | null",
      "homeCountry":    "string?",
      "awayCountry":    "string?",
      "holesPlayed":    "number? (0..18)",
      "holesRemaining": "number? (0..18)",
      "homeUp":         "number?",      // negative when away is up
      "resultLabel":    "string?",
      "holes": [
        {
          "hole":         "number (1..18)",
          "par":          "number?",
          "homeStrokes":  "number?",
          "awayStrokes":  "number?",
          "status":       "HOME | AWAY | HALVED"
        }
      ]
    },

    // ---- TEAM_COMPETITION: aggregate + sessions
    "teamCompetition": {
      "homeTeam":       "string",
      "awayTeam":       "string",
      "homeTeamPoints": "number? (0.5 increments)",
      "awayTeamPoints": "number? (0.5 increments)",
      "totalPoints":    "number?",
      "winningPoints":  "number?",
      "resultLabel":    "string?",
      "sessions": [
        {
          "name":          "string",
          "sessionFormat": "FOURSOMES | FOURBALLS | SINGLES",
          "order":         "number",
          "status":        "SCHEDULED | LIVE | COMPLETED",
          "pairings": [
            {
              "order":           "number",
              "homePlayers":     ["string", ...],
              "awayPlayers":     ["string", ...],
              "homeUp":          "number?",
              "holesPlayed":     "number? (0..18)",
              "holesRemaining":  "number? (0..18)",
              "status":          "IN_PROGRESS | HOME_WIN | AWAY_WIN | HALVED",
              "resultLabel":     "string?",
              "pointsHome":      "number? (0 | 0.5 | 1)",
              "pointsAway":      "number? (0 | 0.5 | 1)"
            }
          ]
        }
      ]
    }

    // Other sports' detail keys may co-exist on the same document and MUST be
    // preserved untouched on golf-only writes.
  },
  "homeTeamPlayers": [
    { "name": "string", "jerseyNumber": "number?", "position": "string?", "playerId": "string | null" }
  ],
  "awayTeamPlayers": [
    { "name": "string", "jerseyNumber": "number?", "position": "string?", "playerId": "string | null" }
  ]
}
```

The response is the same shape, plus server-managed fields the editor reads back (`slug`, `featured`, `createdAt`, `updatedAt`, etc. — see [`MATCHES_API.md`](../MATCHES_API.md)).

---

## 2.3 Field rules the backend MUST enforce

| Field | Rule |
|---|---|
| `sport` | Must equal `"GOLF"` for this editor. Reject other values with `400`. |
| `status` | One of `SCHEDULED \| LIVE \| COMPLETED \| CANCELLED \| POSTPONED`. The editor sends `LIVE` automatically on every live save. |
| `details.format` | Required. One of `STROKE_PLAY \| MATCH_PLAY \| TEAM_COMPETITION`. Reject other values with `400`. |
| `details.coursePar` | Integer `> 0` or null. Typical values 70–72. |
| `details.rounds` | Integer `≥ 1` or null. Stroke play / team competition only. |
| `details.currentRound` | Integer in `[0, details.rounds]` or null. `0` means the event has not started. |
| `details.cutLine` | Integer or null. Only meaningful for stroke play. |
| `details.leaderboard[].rank` | Integer `≥ 1`. Multiple entries MAY share a rank when `tied: true`. |
| `details.leaderboard[].thru` | Integer in `[0, 18]` **or** the literal string `"F"`. Reject other strings. |
| `details.leaderboard[].status` | One of `ACTIVE \| CUT \| WD \| DQ`. |
| `details.leaderboard[].rounds[].holes[].hole` | Integer in `[1, 18]`. |
| `details.matchPlay.holesPlayed` + `holesRemaining` | Each in `[0, 18]`. Sum SHOULD equal 18 unless the match has closed out early; do not enforce. |
| `details.matchPlay.holes[].status` | One of `HOME \| AWAY \| HALVED`. |
| `details.teamCompetition.homeTeamPoints` / `awayTeamPoints` | Number `≥ 0`, multiples of `0.5`. |
| `details.teamCompetition.sessions[].sessionFormat` | One of `FOURSOMES \| FOURBALLS \| SINGLES`. |
| `details.teamCompetition.sessions[].pairings[].pointsHome` / `pointsAway` | One of `0 \| 0.5 \| 1` (or null while the pairing is still `IN_PROGRESS`). Their sum SHOULD be `1` when both are non-null. |
| `homeScore` / `awayScore` | Optional match-total overrides (e.g. team-competition points). Treat as opaque numbers. |
| `notes` | Optional free text. Used for the result line. |
| `details` (unknown keys) | Preserve unknown keys verbatim. Golf writes must not drop cricket / football / etc. keys that may already be on the document. |

Cross-field validation beyond the table above is the editor's responsibility — the backend should not reject "leaderboard sorted wrong" or "homeUp inconsistent with holes[]"; admins do edit partial state during a live event.

When `details.format` switches (e.g. an admin re-types a draft event from stroke play to match play), the backend MUST still preserve the non-active blocks (`leaderboard` / `matchPlay` / `teamCompetition`) the client sent — it doesn't need to clear them.

---

## 2.4 Status transitions

The editor sets `status` in two ways:

1. **Full Event tab:** whatever the admin picks in the dropdown.
2. **Live tab:** always sends `LIVE`, regardless of previous state.

The backend MUST accept any of the five statuses on any update — there is no state machine. Going `COMPLETED → LIVE` (e.g. correcting a premature completion) is a legal admin action.

---

## 2.5 Example: live stroke-play update

Scenario: round 3 of the Doha Open. Tiger Woods finishes hole 14 of round 3 with a birdie, moving to `-15` total / `-4` today / `thru 14`. Admin clicks "Save Live Score".

Editor will `PUT /matches/{slug}` with:

```json
{
  "sport": "GOLF",
  "tournamentSlug": "doha-open-2026",
  "matchType": "INDEPENDENT",
  "homeTeamName": "Doha Open 2026",
  "awayTeamName": "",
  "venue": "Doha Golf Club",
  "scheduledAt": "2026-03-12T07:00:00.000Z",
  "referee": null,
  "status": "LIVE",
  "notes": null,
  "homeScore": null,
  "awayScore": null,
  "details": {
    "format": "STROKE_PLAY",
    "coursePar": 72,
    "rounds": 4,
    "currentRound": 3,
    "cutLine": -1,
    "winner": "",
    "leaderboard": [
      {
        "rank": 1, "tied": false, "name": "Tiger Woods", "playerId": "tiger-woods", "country": "USA",
        "total": -15, "today": -4, "thru": 14, "strokes": 197, "status": "ACTIVE",
        "rounds": [
          { "round": 1, "strokes": 68, "vsPar": -4, "holes": [ /* …18 entries… */ ] },
          { "round": 2, "strokes": 65, "vsPar": -7, "holes": [ /* …18 entries… */ ] },
          { "round": 3, "strokes": 64, "vsPar": -4, "holes": [ /* …14 entries so far… */ ] }
        ]
      }
      // …the rest of the field, echoed back from the previously loaded match…
    ]
    // matchPlay / teamCompetition omitted (format = STROKE_PLAY)
  },
  "homeTeamPlayers": [],
  "awayTeamPlayers": []
}
```

Expected response is the persisted `Match` with the same `details` echoed back and `status: "LIVE"`.

---

## 2.6 Example: live match-play update

Scenario: Tiger Woods is `2 UP` on Phil Mickelson after 14 holes. Mickelson wins the 15th. Admin saves.

Editor will `PUT /matches/{slug}` with:

```json
{
  "sport": "GOLF",
  "tournamentSlug": "presidents-cup-2026",
  "matchType": "KNOCKOUT",
  "homeTeamName": "Tiger Woods",
  "awayTeamName": "Phil Mickelson",
  "venue": "Royal Doha Golf Club",
  "scheduledAt": "2026-03-12T07:42:00.000Z",
  "referee": null,
  "status": "LIVE",
  "notes": null,
  "homeScore": null,
  "awayScore": null,
  "details": {
    "format": "MATCH_PLAY",
    "coursePar": 72,
    "winner": "",
    "matchPlay": {
      "homePlayer": "Tiger Woods", "awayPlayer": "Phil Mickelson",
      "homePlayerId": "tiger-woods", "awayPlayerId": "phil-mickelson",
      "homeCountry": "USA", "awayCountry": "USA",
      "holesPlayed": 15, "holesRemaining": 3,
      "homeUp": 1,
      "resultLabel": "1 UP (thru 15)",
      "holes": [
        { "hole": 1,  "par": 4, "homeStrokes": 4, "awayStrokes": 5, "status": "HOME" },
        // …
        { "hole": 15, "par": 3, "homeStrokes": 4, "awayStrokes": 3, "status": "AWAY" }
      ]
    }
    // leaderboard / teamCompetition omitted (format = MATCH_PLAY)
  },
  "homeTeamPlayers": [],
  "awayTeamPlayers": []
}
```

---

## 2.7 Things the backend does NOT need (yet)

The editor does **not** currently call, and the backend does not need to implement, any of:

- A shot-by-shot event log endpoint. Per-shot data lives only in the client when editing; each "Save Live Score" overwrites the leaderboard / card, not a delta.
- Per-player partial updates (`PATCH /matches/:slug/leaderboard/:playerId`). Whole-document `PUT` is sufficient.
- Server-side ranking recomputation. The editor sends the already-ranked `leaderboard`; the backend stores it verbatim.
- Server-side computation of `resultLabel` strings. The editor formats them client-side.

If/when shot-by-shot persistence is needed (e.g. for a public live feed), add a new endpoint rather than changing this contract.

---

## Caching

- `GET /match/past-featured` and `GET /match/golf/:slug` are cached on the client with `shareReplay(1)` per session, so a `Cache-Control: public, max-age=60` (or similar) on the server is safe.
- `GET /match/golf/:slug/live` should set `Cache-Control: no-store`.
- All other read endpoints retry up to 3 times with a 1s backoff on the client.
- The admin editor endpoints (§2) MUST NOT be cached.
