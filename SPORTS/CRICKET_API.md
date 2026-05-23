# Cricket API Specification

Base URL: `{API_BASE_URI}` (configured via environment, e.g. `http://localhost:8081/api/`).

All endpoints expect/return `application/json`. Read endpoints are public; the admin editor uses the same `x-access-token` JWT contract as the rest of the API (see [`AUTH_API.md`](./AUTH_API.md)). Times are ISO 8601 unless explicitly marked as pre-formatted display strings.

This document is the single source of truth for cricket. It covers three surfaces:

| Surface                | Section                                              | Used by                                           |
|------------------------|------------------------------------------------------|---------------------------------------------------|
| Public display         | [§1 Public display endpoints](#1-public-display-endpoints) | Cricket explorer, match-detail page, carousels, ticker, live banner |
| Admin score editor     | [§2 Admin score editor](#2-admin-score-editor)       | `CricketScoreEditorComponent` (`/scores/:slug`)   |
| Legacy `/api/cric-scores` | [§3 Legacy `/api/cric-scores` (deprecated)](#3-legacy-apicric-scores-deprecated) | Old read paths; **do not build new code against this** |

The admin editor and the public display already piggy-back on the generic `/matches` resource documented in [`MATCHES_API.md`](./MATCHES_API.md). The cricket-specific shapes below extend that contract; they do not replace it.

---

## Display surfaces and the data each needs

| Component                      | Endpoint                                  | Purpose                                       |
|--------------------------------|-------------------------------------------|-----------------------------------------------|
| `featured-match`               | `GET /match/past-featured`                | Carousel of recent featured match scorecards  |
| `top-knock-ticker`             | `GET /match/past-featured`                | Top knocks ticker (same source)               |
| `match-summary-carousel`       | `GET /match/cricket/summaries`            | Mixed past/upcoming/live summary card strip   |
| `match-schedule`               | `GET /match/list-match`                   | Upcoming + recent matches + countdown         |
| `match-page` (match detail)    | `GET /match/cricket/:slug`                | Full scorecard, commentary, analysis, MVP     |
| Live in-page banner (optional) | `GET /match/cricket/:slug/live`           | Lightweight live score polling                |
| Cricket score editor           | `GET /matches/:slug` + `PUT /matches/:slug` | Admin edits via the generic `/matches` resource |

The frontend caches `past-featured` and per-slug match detail with `shareReplay(1)`, so these endpoints should be cheap and cacheable.

---

## Field conventions (shared across all sections)

- **Slugs**: every match is addressed by `slug` (kebab-case, URL-safe). Carousel and ticker may use a separate `embedSlug` alias; both MUST resolve to the same match detail.
- **Scores**: surface both the structured form (`final`, `finalWick`, `finalOver`, `finalOverBall`) and the display string (`score: "138/5"`, `overs: "12.0 Ov"`). The carousel uses the structured form; the detail page uses the strings as-is.
- **Times**:
  - Schedule and live endpoints — ISO 8601 with timezone (`2024-10-06T17:15:00+04:00`).
  - Match detail `startTime`, `matchDate`, `lastUpdated`, and the `innings[].from/to` fields are pre-formatted display strings and are rendered verbatim.
- **Unknowns**: prefer omitting the field or returning `null` over the `-256` sentinel that appears in legacy mocks.
- **Empty strings**: `videoEmbedUrl`, `imageUrl`, and `result` may be empty strings when not yet known; the frontend treats them the same as missing.
- **`sport`**: cricket endpoints always return `"CRICKET"`, but the field is required on summary / featured shapes so a future cross-sport endpoint can reuse the same payload. Recognised values (case-insensitive on the client): `"CRICKET"`, `"SOCCER"`, `"RUGBY"`, `"BASKETBALL"`, `"VOLLEYBALL"`, `"NETBALL"`, `"BADMINTON"`, `"BOWLING"`, `"CYCLING"`, `"MOTOR-SPORTS"`. Unknown values fall back to a generic icon.

---

# 1. Public display endpoints

Read endpoints consumed by the public cricket pages. All are public (no auth).

## 1.1 List matches (schedule)

Returns all matches the schedule view should know about (past + upcoming). The client filters upcoming vs. past by `startTime` and selects the next one for the countdown.

### `GET /match/list-match`

```
Query params (optional):
  sport?:      "CRICKET"            // when omitted, returns all sports
  tournament?: "string"             // tournament slug
  from?:       "ISO8601"
  to?:         "ISO8601"
  status?:     "SCHEDULED | LIVE | COMPLETED | CANCELLED | POSTPONED"

Response 200: MatchScheduleItem[]
```

```jsonc
// MatchScheduleItem
{
  "slug":        "sl-united-vs-sri-lankan-cc",   // URL-friendly id used by /match/cricket/:slug
  "embedSlug":   "sl-united-cc-vs-sri-lankan-cc",// optional alias used by carousel
  "tournament":  "SLCL Season VII",
  "stage":       "Group | Quarter Final | Semi Final | Final | null",
  "match":       "PR QF 1",                      // optional bracket label
  "matchNumber": 41,                             // optional sequence number
  "homeTeam":    "SL United",
  "awayTeam":    "Sri Lankan CC",
  "startTime":   "2024-10-06T17:15:00+04:00",    // ISO8601 with timezone
  "group":       "C",                            // optional, group-stage only
  "venue":       "Seven District Oval 1",
  "ground":      "G1",                           // optional short label
  "status":      "SCHEDULED | LIVE | COMPLETED | CANCELLED | POSTPONED",
  "featured":    false
}
```

Notes:
- `startTime` MUST include a timezone offset; the countdown widget parses it with `new Date(startTime)`.
- The schedule component groups matches by date in the user's local timezone.

---

## 1.2 Past featured matches (carousel + ticker)

Recent completed matches with a compact scorecard summary. Drives the `featured-match` carousel and `top-knock-ticker`.

### `GET /match/past-featured`

```
Query params (optional):
  limit?: number   // default 10

Response 200: FeaturedMatchSummary[]
```

```jsonc
// FeaturedMatchSummary
{
  "embedSlug":  "sl-united-cc-vs-sri-lankan-cc",
  "slug":       "sl-united-vs-sri-lankan-cc",    // canonical slug for match detail
  "sport":      "CRICKET",                       // sport code; drives the sport icon on the home carousel & ticker
  "tournament": "SLCL Season VII",
  "homeTeam":   "SL United",
  "awayTeam":   "Sri Lankan CC",
  "startTime":  "2024-10-06T17:15:00+04:00",
  "group":      "C",
  "venue":      "Seven District Oval 1",
  "derived": {
    "wonBy":   "by 2 runs",                      // human-readable result
    "comment": "All Out"                         // e.g. "10 balls remaining"
  },
  "scoreboard": {
    "winner": "SL United",                       // team name; empty string if tied/no result
    "homeTeam": {
      "batFirst":     true,
      "final":        184,                       // runs
      "finalWick":    7,                         // wickets
      "finalOver":    20,                        // completed overs
      "finalOverBall":0,                         // balls in the current over (0-5)
      "batting": [
        {
          "order": 1,                            // batting order; -256 means "unknown"
          "name":  "Mohamed Suraij",
          "runs":  63,
          "out":   { "out": true }
        }
      ]
    },
    "awayTeam": {
      "batFirst":     false,
      "final":        182,
      "finalWick":    10,
      "finalOver":    19,
      "finalOverBall":5,
      "batting": [
        { "order": 1, "name": "Lakshan Fernando", "runs": 85, "out": { "out": true } },
        { "order": 2, "name": "Shiran Tharanga",  "runs": 27, "out": { "out": false } }
      ]
    }
  }
}
```

Field semantics:
- `final / finalWick / finalOver / finalOverBall` — the final innings score in cricket "runs / wickets in overs.balls" format.
- `out.out: true` means the batter was dismissed; `false` means not out.
- The carousel only shows the top two scorers per side, so `batting` may be truncated server-side.
- `-256` is the placeholder the current mock uses when a value is unknown; the API SHOULD return real values and omit unknowns rather than encoding `-256`.

---

## 1.3 Match summaries (carousel)

Compact summary cards shown at the top of the cricket explorer page. Drives the `match-summary-carousel` component (10 cards in a horizontal PrimeNG carousel). Mixes past results, in-progress matches, and the next few upcoming fixtures so the same card shape can render any of them.

### `GET /match/cricket/summaries`

```
Query params (optional):
  limit?:      number   // default 10, hard cap 20
  tournament?: "string" // tournament slug; when omitted, mixes tournaments
  include?:    "PAST | LIVE | UPCOMING" (comma-separated; default "PAST,LIVE,UPCOMING")

Response 200: MatchSummary[]   // newest/next-up first
```

```jsonc
// MatchSummary
{
  "slug":            "sl-united-vs-sri-lankan-cc",      // canonical match slug
  "embedSlug":       "sl-united-cc-vs-sri-lankan-cc",   // optional carousel alias
  "sport":           "CRICKET",                         // sport code; drives the sport icon on the card
  "tournament":      "SLCL Season VII",                 // full tournament name
  "tournamentLabel": "SLCL S7",                         // short code shown top-right on the card
  "status":          "UPCOMING | LIVE | RESULT",        // drives the card status pill
  "startTime":       "2024-10-06T17:15:00+04:00",       // ISO8601 with timezone
  "venue":           "Seven District Oval 1",
  "venueLabel":      "SEVEN DISTRICT OVAL 1",           // optional pre-uppercased display string

  "homeTeam": {
    "name":      "SL United",
    "shortName": "SLU",                                 // optional 2–4 letter code
    "logoUrl":   "/assets/images/matchResult/SL%20United.png",   // empty string allowed
    "innings": [                                        // 0 (upcoming) / 1 (limited overs) / 1–2 (multi-day) entries
      { "runs": 184, "wickets": 7,  "overs": "20.0", "declared": false }
    ]
  },
  "awayTeam": {
    "name":      "Sri Lankan CC",
    "shortName": "SLCC",
    "logoUrl":   "/assets/images/matchResult/Sri%20Lankan%20CC.png",
    "innings": [
      { "runs": 182, "wickets": 10, "overs": "19.5", "declared": false }
    ]
  },

  "result": {                                           // omit / null when status !== "RESULT"
    "kind":  "WIN | DRAW | TIE | NO_RESULT",
    "label": "SL UNITED WON BY 2 RUNS"                  // already cased for display
  }
}
```

Field semantics:

- `status` is what the card renders as a coloured pill (`RESULT` red, `LIVE` green-pulsing, `UPCOMING` neutral). It MUST agree with the presence of `result`: `UPCOMING` / `LIVE` carry no `result`, `RESULT` MUST carry one.
- `innings[]` lets the same card render T20 (one entry per side, e.g. `"184/7 (20.0)"`) and multi-day formats (two entries joined with `" & "`, e.g. `"344/10 (119) & 54/1 (18)"`). For `UPCOMING`, both teams' arrays MUST be empty.
- `declared` is optional and only meaningful for multi-day matches; the card appends a `d` suffix (e.g. `"344/10d (119)"`) when true.
- `tournamentLabel` is the short code rendered in the top-right corner of the card (e.g. `"BOTG"`, `"BAN-W VS SL-W"`, `"SLCL S7"`). When omitted the client falls back to `tournament`.
- `venueLabel` is rendered verbatim under upcoming cards next to the date. When omitted the client uppercases `venue`.
- `logoUrl` may be an empty string; the client then falls back to `assets/images/matchResult/<name>.png`.

Caching: same expectations as `match/past-featured` — `shareReplay(1)` on the client, so `Cache-Control: public, max-age=60` is fine.

---

## 1.4 Match detail (full scorecard page)

Powers `match-page` — tabs: Summary, Scorecard, Commentary, Analysis, Heroes, MVP, Teams, Gallery. The frontend interface is `CricketMatchDetail` in `@core/services/cricket.service.ts`.

### `GET /match/cricket/:slug`

```
Path params:
  slug: string   // canonical match slug

Response 200: CricketMatchDetail
Response 404: { "error": "Match not found" }
```

```jsonc
// CricketMatchDetail (top-level)
{
  "slug":         "saints-xi-vs-victorians-final",
  "tournament":   "QTerminals Premier League- Season 7",
  "seriesName":   "QTerminals Premier League- Season 7",
  "status":       "Past | Live | Upcoming",
  "stage":        "Final",
  "venue":        "Hamad Port Cricket Ground",
  "city":         "Doha",
  "location":     "Hamad Port Cricket Ground, Doha",   // display string
  "overs":        "12 Ov.",                            // format used per innings
  "startTime":    "28-Nov-24 06:09 AM",                // display string (already formatted)
  "matchDate":    "28/11/2024",                        // display string

  "homeTeam":     { "name": "VICTORIANS", "score": "138/5", "overs": "12.0 Ov" },
  "awayTeam":     { "name": "SAINTS XI",  "score": "144/2", "overs": "10.1 Ov" },

  "toss":         "SAINTS XI opt to field",
  "result":       "SAINTS XI won by 8 wickets",

  "topBatters":   [ CricketBatter, ... ],   // see below
  "topBowlers":   [ CricketBowler, ... ],
  "scorecardInnings": [ ScorecardInnings, ... ],
  "commentary":   [ CommentaryInnings, ... ],
  "analysis":     MatchAnalysis,
  "heroes":       MatchHeroes,
  "mvp":          [ MvpPlayer, ... ],

  "videoEmbedUrl":"",                               // empty string when no video
  "streamTitle":  "SAINTS XI Vs VICTORIANS - GRAND FINAL",
  "rightsHolder": "Rangers Media",
  "totalViews":   306,
  "liveViewers":  1,

  "innings": [
    { "label": "1st Innings",    "durationMins": 53, "from": "06:09", "to": "07:03" },
    { "label": "Innings Break",  "durationMins": 8,  "from": "07:03", "to": "07:11" },
    { "label": "2nd Innings",    "durationMins": 39, "from": "07:11", "to": "07:50" }
  ],

  "lastUpdated":   "2024-11-29 at 05:54",
  "lastUpdatedBy": "Jins Manager"
}
```

### Sub-shapes

```jsonc
// CricketBatter (Summary tab)
{ "name": "Safni", "team": "SAINTS XI", "runs": 78, "balls": 27, "fours": 2, "sixes": 11, "strikeRate": 288.89 }

// CricketBowler (Summary tab)
{ "name": "Arun Raveendran", "team": "SAINTS XI", "overs": 3.0, "maidens": 0, "runs": 14, "wickets": 3, "economy": 4.67 }
```

```jsonc
// ScorecardInnings
{
  "teamName": "VICTORIANS",
  "score":    "138/5",
  "overs":    "12.0 Ov",
  "batters": [
    {
      "name":       "Rizwan Khan",
      "captain":    true,         // optional, defaults false
      "keeper":     false,        // optional, defaults false
      "dismissal":  "run out (Safni)",  // or "not out"
      "runs":       12,
      "balls":      9,
      "fours":      1,
      "sixes":      0,
      "strikeRate": 133.33,
      "minutes":    12
    }
  ],
  "extras":  { "total": 10, "breakdown": "wd 6, lb 2, nb 1, b 1" },
  "yetToBat": ["Omar Sheikh", "Bilal Hassan"],
  "fallOfWickets": [
    { "score": 38, "wicket": 1, "batter": "Talha Shah", "over": "4.2" }
  ],
  "bowlers": [
    {
      "name":     "Arun Raveendran",
      "captain":  false,
      "overs":    3.0,
      "maidens":  0,
      "runs":     14,
      "wickets":  3,
      "dots":     5,
      "fours":    1,
      "sixes":    1,
      "wides":    1,
      "noBalls":  0,
      "economy":  4.67
    }
  ]
}
```

```jsonc
// CommentaryInnings
{
  "teamName": "SAINTS XI",
  "entries": [ CommentaryEntry, ... ]   // newest first
}

// CommentaryEntry — discriminated union by "kind"

// kind: "ball"
{
  "kind":        "ball",
  "over":        "10.1",                                  // overs.balls
  "badge":       "dot | run | four | six | wide | noball | wicket",
  "badgeLabel":  "0 | 1 | 4 | 6 | WD | NB | W",
  "description": "Kamran Ali to Safni, SIX, launched over mid-wicket!",
  "isBoundary":  true
}

// kind: "notification"
{ "kind": "notification", "text": "Kamran Ali comes into the attack" }

// kind: "over-end"
{
  "kind":       "over-end",
  "overNumber": 10,
  "score":      "144/2",
  "runs":       16,        // runs in this over
  "wickets":    0,         // wickets in this over
  "batters": [
    { "name": "Safni", "notOut": true, "runs": 78, "balls": 27 }
  ],
  "bowlers": [
    { "name": "Kamran Ali", "overs": 2.0, "maidens": 0, "runs": 36, "wickets": 0 }
  ]
}
```

```jsonc
// MatchAnalysis (powers Manhattan, Run-rate, Worm, Wicket-types charts)
{
  "innings1": [
    { "over": 1, "runs": 6,  "wickets": 0, "runRate": 6.0 },
    { "over": 2, "runs": 14, "wickets": 1, "runRate": 10.0 }
  ],
  "innings2": [
    { "over": 1, "runs": 8,  "wickets": 0, "runRate": 8.0 }
  ],
  "innings1WicketTypes": [
    { "label": "Caught out", "count": 2 },
    { "label": "Bowled",     "count": 2 },
    { "label": "Caught behind", "count": 0 },
    { "label": "Run out",    "count": 1 },
    { "label": "LBW",        "count": 0 },
    { "label": "Stumped",    "count": 0 }
  ],
  "innings2WicketTypes": [ /* same shape */ ]
}
```

Notes:
- `innings1` MUST contain one row per over actually bowled in that innings.
- `runRate` is cumulative (runs per over up to and including that over).
- `innings1WicketTypes` and `innings2WicketTypes` MUST share the same label order so the pie chart can sum them by index.

```jsonc
// MatchHeroes
{
  "playerOfMatch": HeroPlayer,
  "bestBatter":    HeroPlayer,
  "bestBowler":    HeroPlayer
}

// HeroPlayer — include batting OR bowling (or both); omit irrelevant block
{
  "name":     "Safni",
  "team":     "SAINTS XI",
  "imageUrl": "",                                      // empty string allowed
  "batting":  { "runs": 78, "balls": 27, "fours": 2, "sixes": 11, "strikeRate": 288.89 },
  "bowling":  { "overs": 3.0, "maidens": 0, "runs": 14, "wickets": 3, "economy": 4.67 }
}
```

```jsonc
// MvpPlayer
{
  "rank":     1,
  "name":     "Safni",
  "team":     "SAINTS XI",
  "imageUrl": "",
  "batting":  7.620,
  "bowling":  2.554,
  "fielding": 0.000,
  "total":    10.174
}
```

`videoEmbedUrl` MUST be either an empty string or a full URL whose host is one of: `www.youtube.com`, `youtube.com`, `player.vimeo.com`. Any other host is rejected by the frontend sanitizer.

---

## 1.5 Live score (optional, for in-progress matches)

Lightweight payload for polling/SSE while a match is `LIVE`. Avoid returning the full `CricketMatchDetail` on every poll.

### `GET /match/cricket/:slug/live`

```jsonc
Response 200:
{
  "slug":      "saints-xi-vs-victorians-final",
  "status":    "LIVE",
  "homeTeam":  { "name": "VICTORIANS", "score": "138/5", "overs": "12.0 Ov", "batFirst": true },
  "awayTeam":  { "name": "SAINTS XI",  "score": "98/2",  "overs": "8.3 Ov",  "batFirst": false },
  "target":    139,                                   // null in 1st innings
  "required":  { "runs": 41, "balls": 69, "runRate": 3.57 },   // null in 1st innings
  "currentBatters": [
    { "name": "Safni", "onStrike": true,  "runs": 46, "balls": 18 },
    { "name": "Nawaz", "onStrike": false, "runs": 8,  "balls": 7  }
  ],
  "currentBowler":  { "name": "Talha Shah", "overs": 3.0, "maidens": 0, "runs": 29, "wickets": 1 },
  "lastBalls":      ["0","1","6","W","4","1"],       // most recent over, newest right
  "lastUpdated":    "2024-11-28T07:30:11Z"
}
```

---

# 2. Admin score editor

This section covers the endpoints the **Cricket Score Editor** (`/scores/:slug` → `CricketScoreEditorComponent`) calls when an admin updates a cricket match. Cricket scores piggy-back on the generic `/matches` resource (documented in [`MATCHES_API.md`](./MATCHES_API.md)); the editor never calls a cricket-specific endpoint. All cricket-specific state lives inside the `details` object on a `Match`.

## 2.1 Endpoints the editor calls

### `GET /matches/:slug` — load match into the editor

Called once when the editor opens. Response hydrates both the "Update Full Match" form and the "Over-by-Over (Live)" tab.

```json
Response 200: Match  // shape defined in MATCHES_API.md, with cricket details described in §2.2 below
```

### `PUT /matches/:slug` — save full match OR save live innings

The editor sends the **same payload shape** for both tabs:

- **Full Match save:** sends the form's `status` as-is.
- **Live (over-by-over) save:** forces `status: "LIVE"` and only patches the three fields for the currently-batting innings (`<innings>Runs`, `<innings>Wickets`, `<innings>Overs`); all other `details.*` keys are echoed back unchanged from the previously loaded match.

The backend MUST treat this as a full replacement of the match document — the client always sends the complete state it wants persisted, including all four innings. Do not attempt to diff or merge server-side.

```json
Response 200: Match  // the updated match, used to re-hydrate the editor
```

There is no separate "live" endpoint. The backend does not need to expose one.

---

## 2.2 Match payload (cricket)

The full `PUT /matches/:slug` request body the editor sends:

```json
{
  "sport":          "CRICKET",
  "tournamentSlug": "string | null",
  "matchType":      "LEAGUE | KNOCKOUT | FRIENDLY | INDEPENDENT | null",
  "homeTeamName":   "string",
  "awayTeamName":   "string",
  "venue":          "string?",
  "scheduledAt":    "ISO8601?",
  "referee":        "string?",
  "status":         "SCHEDULED | LIVE | COMPLETED | CANCELLED | POSTPONED",
  "notes":          "string?",          // free-text result / commentary
  "homeScore":      "number | null",    // match-total override (optional)
  "awayScore":      "number | null",
  "details": {
    "cricketFormat":   "T20 | OD | TEST | null",   // game length; drives which innings the editor shows

    "homeInn1Runs":    "number?",
    "homeInn1Wickets": "number? (0-10)",
    "homeInn1Overs":   "number? (X.Y, Y in 0..5)",

    "homeInn2Runs":    "number?",
    "homeInn2Wickets": "number? (0-10)",
    "homeInn2Overs":   "number? (X.Y, Y in 0..5)",

    "awayInn1Runs":    "number?",
    "awayInn1Wickets": "number? (0-10)",
    "awayInn1Overs":   "number? (X.Y, Y in 0..5)",

    "awayInn2Runs":    "number?",
    "awayInn2Wickets": "number? (0-10)",
    "awayInn2Overs":   "number? (X.Y, Y in 0..5)"

    // Other sports' detail keys may co-exist on the same document and MUST be
    // preserved untouched on cricket-only writes.
  },
  "homeTeamPlayers": [
    { "name": "string", "jerseyNumber": "number?", "position": "string?", "playerId": "string | null" }
  ],
  "awayTeamPlayers": [
    { "name": "string", "jerseyNumber": "number?", "position": "string?", "playerId": "string | null" }
  ]
}
```

The response is the same shape, plus server-managed fields the editor reads back (`slug`, `featured`, `createdAt`, `updatedAt`, etc. — see [`MATCHES_API.md`](./MATCHES_API.md)).

---

## 2.3 Field rules the backend MUST enforce

| Field | Rule |
|---|---|
| `sport` | Must equal `"CRICKET"` for this editor. Reject other values with `400`. |
| `status` | One of `SCHEDULED \| LIVE \| COMPLETED \| CANCELLED \| POSTPONED`. The editor will send `LIVE` automatically on every over-by-over save. |
| `details.cricketFormat` | One of `T20 \| OD \| TEST` or `null`. Discriminates game length: `T20` and `OD` are single-innings-per-side, `TEST` allows two innings per side. Informational only — the backend MUST NOT reject Inn2 fields when the format is `T20`/`OD`, since admins may switch format on an in-progress match. When the match has a `tournamentSlug` and this field is missing/null on write, the backend MUST inherit it from the linked tournament's `cricketFormat` (see [`TOURNAMENTS_API.md`](../TOURNAMENTS_API.md) → Cricket format inheritance). |
| `details.*Runs` | Integer `≥ 0` or `null`. No upper bound. |
| `details.*Wickets` | Integer in `[0, 10]` or `null`. |
| `details.*Overs` | Decimal `X.Y` where `X ≥ 0` and `Y ∈ {0,1,2,3,4,5}`. Examples: `0`, `4`, `4.3`, `19.5`. **`4.6` is invalid** — six legal balls roll the over to `5.0`. Store as a number (the client sends `parseFloat("X.Y")`). |
| `homeScore` / `awayScore` | Optional match-total overrides. The editor surfaces them as separate inputs and does **not** auto-derive them from innings. Treat as opaque numbers. |
| `notes` | Optional free text. Used for the result line (e.g. `"Home won by 5 wickets"`). |
| `details` (unknown keys) | Preserve unknown keys verbatim. Cricket writes must not drop football/rugby/etc. keys that may already be on the document. |

Innings consistency (`runs ≥ 0`, `wickets ≤ 10`, valid `X.Y` overs) and `cricketFormat` membership are the **only** cross-field validations required. The backend should not try to validate cricketing logic beyond that (e.g. don't reject "Inn2 set but Inn1 empty" — admins do edit partial state during a live match — and don't enforce a per-format maximum overs cap).

---

## 2.4 Status transitions

The editor sets `status` in two ways:

1. **Full Match tab:** whatever the admin picks in the dropdown.
2. **Over-by-Over tab:** always sends `LIVE`, regardless of previous state.

The backend MUST accept any of the five statuses on any update — there is no state machine. Going `COMPLETED → LIVE` (e.g. correcting a premature completion) is a legal admin action.

---

## 2.5 Example: live over save

Scenario: home team is batting in 1st innings, currently `45/2 in 7.4`. Admin clicks "+4 Four" then "Save Live Score".

Editor will `PUT /matches/{slug}` with:

```json
{
  "sport": "CRICKET",
  "tournamentSlug": "uae-t20-2026",
  "matchType": "LEAGUE",
  "homeTeamName": "Dubai Stars",
  "awayTeamName": "Sharjah Kings",
  "venue": "Dubai International Cricket Stadium",
  "scheduledAt": "2026-05-18T14:30:00.000Z",
  "referee": null,
  "status": "LIVE",
  "notes": null,
  "homeScore": null,
  "awayScore": null,
  "details": {
    "cricketFormat": "T20",
    "homeInn1Runs": 49,
    "homeInn1Wickets": 2,
    "homeInn1Overs": 7.5,
    "homeInn2Runs": null, "homeInn2Wickets": null, "homeInn2Overs": null,
    "awayInn1Runs": null, "awayInn1Wickets": null, "awayInn1Overs": null,
    "awayInn2Runs": null, "awayInn2Wickets": null, "awayInn2Overs": null
  },
  "homeTeamPlayers": [ /* ...unchanged... */ ],
  "awayTeamPlayers": [ /* ...unchanged... */ ]
}
```

Expected response is the persisted `Match` with the same `details` echoed back and `status: "LIVE"`.

---

## 2.6 Things the backend does NOT need (yet)

The editor does **not** currently call, and the backend does not need to implement, any of:

- A ball-by-ball event log endpoint. The live tab keeps `ballHistory` only in component memory; it is **not** posted to the server. Each "Save Live Score" click overwrites the innings totals, not a delta.
- The legacy `/api/cric-scores/*` routes (see §3). They are unused by this editor.
- Innings-level partial updates (`PATCH /matches/:slug/innings/:key`). Whole-document `PUT` is sufficient.
- Server-side recomputation of `homeScore` / `awayScore` from innings totals. Leave those as admin-managed fields.

If/when ball-by-ball persistence is needed (e.g. for a public live feed), add a new endpoint rather than changing this contract.

---

# 3. Legacy `/api/cric-scores` (deprecated)

> **Deprecated.** These routes predate the generic `/matches` resource and the cricket display endpoints in §1. They are kept here for compatibility with old clients but **must not** be used for new work. The admin editor (§2) and the public display (§1) supersede them entirely.

These routes are `/api` only — they are NOT mirrored at the bare path.

### `GET /api/cric-scores`
```json
Response 200: CricScore[]
```

### `GET /api/cric-scores/status/:status`
```json
Params:
status: "upcoming | live | completed"

Response 200: CricScore[]
```

### `GET /api/cric-scores/:id`
```json
Response 200: CricScore
```

### `POST /api/cric-scores`
```json
Request:
{
  "matchTitle": "string",
  "team1": { "name": "string", "score": "number?", "overs": "number?", "wickets": "number?" },
  "team2": { "name": "string", "score": "number?", "overs": "number?", "wickets": "number?" },
  "status": "upcoming | live | completed",
  "venue": "string?",
  "matchDate": "ISO8601?",
  "matchType": "ODI | T20 | Test | Other",
  "result": "string?"
}

Response 201: CricScore
```

### `PUT /api/cric-scores/:id`
```json
Request: same shape as POST

Response 200: CricScore
```

### `DELETE /api/cric-scores/:id`
```json
Response 200: { "message": "Score deleted" }
```

---

## Caching

- `GET /match/past-featured` and `GET /match/cricket/:slug` are cached on the client with `shareReplay(1)` per session, so a `Cache-Control: public, max-age=60` (or similar) on the server is safe.
- `GET /match/cricket/:slug/live` should set `Cache-Control: no-store`.
- All other read endpoints retry up to 3 times with a 1s backoff on the client.
- The admin editor endpoints (§2) MUST NOT be cached.
