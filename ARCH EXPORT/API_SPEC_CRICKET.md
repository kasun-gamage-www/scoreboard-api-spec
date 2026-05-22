# Cricket Scores API Specification

Specifies the endpoints and payload shapes required by the frontend to display
cricket match scores. This complements the generic `/matches` CRUD documented in
`API_SPEC.md` — the routes below are the dedicated read endpoints the cricket
display components consume.

Base URL: `{API_BASE_URI}` (configured via environment, e.g. `http://localhost:8081/api/`).

All endpoints return `application/json`. Read endpoints are public. Times are
ISO 8601 unless explicitly marked as pre-formatted display strings.

---

## Display surfaces and the data each needs

| Component                       | Endpoint                                  | Purpose                                       |
|---------------------------------|-------------------------------------------|-----------------------------------------------|
| `featured-match`                | `GET match/past-featured`                 | Carousel of recent featured match scorecards  |
| `top-knock-ticker`              | `GET match/past-featured`                 | Top knocks ticker (same source)               |
| `match-summary-carousel`        | `GET match/cricket/summaries`             | Mixed past/upcoming/live summary card strip   |
| `match-schedule`                | `GET match/list-match`                    | Upcoming + recent matches + countdown         |
| `match-page` (match detail)     | `GET match/cricket/:slug`                 | Full scorecard, commentary, analysis, MVP     |
| Live in-page banner (optional)  | `GET match/cricket/:slug/live`            | Lightweight live score polling                |

The frontend caches `past-featured` and per-slug match detail with
`shareReplay(1)`, so these endpoints should be cheap and cacheable.

---

## 1. List matches (schedule)

Returns all matches the schedule view should know about (past + upcoming). The
client filters upcoming vs. past by `startTime` and selects the next one for the
countdown.

### GET match/list-match

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
- `startTime` MUST include a timezone offset; the countdown widget parses it
  with `new Date(startTime)`.
- The schedule component groups matches by date in the user's local timezone.

---

## 2. Past featured matches (carousel + ticker)

Recent completed matches with a compact scorecard summary. Drives the
`featured-match` carousel and `top-knock-ticker`.

### GET match/past-featured

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
- `final / finalWick / finalOver / finalOverBall` — the final innings score in
  cricket "runs / wickets in overs.balls" format.
- `out.out: true` means the batter was dismissed; `false` means not out.
- The carousel only shows the top two scorers per side, so `batting` may be
  truncated server-side.
- `-256` is the placeholder the current mock uses when a value is unknown;
  the API SHOULD return real values and omit unknowns rather than encoding -256.

---

## 2a. Match summaries (carousel)

Compact summary cards shown at the top of the cricket explorer page. Drives the
`match-summary-carousel` component (10 cards in a horizontal PrimeNG carousel).
Mixes past results, in-progress matches, and the next few upcoming fixtures so
the same card shape can render any of them.

### GET match/cricket/summaries

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

- `sport` is the sport code that drives the small sport icon rendered next to
  the tournament label on each card. For this endpoint the value is always
  `"CRICKET"`, but the field is required so the same `MatchSummary` shape can
  be reused by a future cross-sport summaries endpoint. Recognised values
  (case-insensitive on the client): `"CRICKET"`, `"SOCCER"`, `"RUGBY"`,
  `"BASKETBALL"`, `"VOLLEYBALL"`, `"NETBALL"`, `"BADMINTON"`, `"BOWLING"`,
  `"CYCLING"`, `"MOTOR-SPORTS"`. Unknown values fall back to a generic icon.
- `status` is what the card renders as a coloured pill (`RESULT` red, `LIVE`
  green-pulsing, `UPCOMING` neutral). It MUST agree with the presence of
  `result`: `UPCOMING`/`LIVE` carry no `result`, `RESULT` MUST carry one.
- `innings[]` lets the same card render T20 (one entry per side, e.g.
  `"184/7 (20.0)"`) and multi-day formats (two entries joined with `" & "`,
  e.g. `"344/10 (119) & 54/1 (18)"`). For `UPCOMING`, both teams' arrays MUST
  be empty.
- `declared` is optional and only meaningful for multi-day matches; the card
  appends a `d` suffix (e.g. `"344/10d (119)"`) when true.
- `tournamentLabel` is the short code rendered in the top-right corner of the
  card (e.g. `"BOTG"`, `"BAN-W VS SL-W"`, `"SLCL S7"`). When omitted the
  client falls back to `tournament`.
- `venueLabel` is rendered verbatim under upcoming cards next to the date.
  When omitted the client uppercases `venue`.
- `logoUrl` may be an empty string; the client then falls back to
  `assets/images/matchResult/<name>.png`.

Caching: same expectations as `match/past-featured` — `shareReplay(1)` on the
client, so `Cache-Control: public, max-age=60` is fine.

---

## 3. Match detail (full scorecard page)

Powers `match-page` — tabs: Summary, Scorecard, Commentary, Analysis, Heroes,
MVP, Teams, Gallery. The frontend interface is `CricketMatchDetail` in
`@core/services/cricket.service.ts`.

### GET match/cricket/:slug

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
- `innings1WicketTypes` and `innings2WicketTypes` MUST share the same label
  order so the pie chart can sum them by index.

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

`videoEmbedUrl` MUST be either an empty string or a full URL whose host is one
of: `www.youtube.com`, `youtube.com`, `player.vimeo.com`. Any other host is
rejected by the frontend sanitizer.

---

## 4. Live score (optional, for in-progress matches)

Lightweight payload for polling/SSE while a match is `LIVE`. Avoid returning
the full `CricketMatchDetail` on every poll.

### GET match/cricket/:slug/live

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

## Field conventions

- **Slugs**: every match is addressed by `slug` (kebab-case, URL-safe). The
  carousel and ticker may use a separate `embedSlug` alias; both MUST resolve
  to the same match detail.
- **Scores**: surface both the structured form (`final`, `finalWick`,
  `finalOver`, `finalOverBall`) and the display string (`score: "138/5"`,
  `overs: "12.0 Ov"`). The carousel uses the structured form; the detail page
  uses the strings as-is.
- **Times**:
  - Schedule and live endpoints — ISO 8601 with timezone (`2024-10-06T17:15:00+04:00`).
  - Match detail `startTime`, `matchDate`, `lastUpdated`, and the `innings[].from/to`
    fields are pre-formatted display strings and are rendered verbatim.
- **Unknowns**: prefer omitting the field or returning `null` over the `-256`
  sentinel that appears in legacy mocks.
- **Empty strings**: `videoEmbedUrl`, `imageUrl`, and `result` may be empty
  strings when not yet known; the frontend treats them the same as missing.

---

## Caching

- `match/past-featured` and `match/cricket/:slug` are cached on the client with
  `shareReplay(1)` per session, so a `Cache-Control: public, max-age=60` (or
  similar) on the server is safe.
- `match/cricket/:slug/live` should set `Cache-Control: no-store`.
- All other read endpoints retry up to 3 times with a 1s backoff on the client.
