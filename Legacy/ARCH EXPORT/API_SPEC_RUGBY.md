# Rugby Scores API Specification

Specifies the endpoints and payload shapes required by the frontend to display
rugby match scores. This complements the generic `/matches` CRUD documented in
`API_SPEC.md` — the routes below are the dedicated read endpoints the rugby
display components consume.

Base URL: `{API_BASE_URI}` (configured via environment, e.g. `http://localhost:8081/api/`).

All endpoints return `application/json`. Read endpoints are public. Times are
ISO 8601 unless explicitly marked as pre-formatted display strings.

---

## Display surfaces and the data each needs

| Component                       | Endpoint                                  | Purpose                                       |
|---------------------------------|-------------------------------------------|-----------------------------------------------|
| `featured-match`                | `GET match/rugby/past-featured`           | Carousel of recent featured match scorecards  |
| `top-knock-ticker`              | `GET match/rugby/past-featured`           | Top scorers ticker (same source)              |
| `match-summary-carousel`        | `GET match/rugby/summaries`               | Mixed past/upcoming/live summary card strip   |
| `match-schedule`                | `GET match/list-rugby`                    | Upcoming + recent matches + countdown         |
| `match-page` (match detail)     | `GET match/rugby/:slug`                   | Full scorecard, commentary, analysis, MVP     |
| Live in-page banner (optional)  | `GET match/rugby/:slug/live`              | Lightweight live score polling                |

The frontend caches `past-featured` and per-slug match detail with
`shareReplay(1)`, so these endpoints should be cheap and cacheable.

---

## 1. List matches (schedule)

Returns all rugby matches the schedule view should know about (past + upcoming).
The client filters upcoming vs. past by `startTime` and selects the next one for
the countdown.

### GET match/list-rugby

```
Query params (optional):
  sport?:      "RUGBY"             // when omitted, returns all rugby matches
  tournament?: "string"             // tournament slug
  from?:       "ISO8601"
  to?:         "ISO8601"
  status?:     "SCHEDULED | LIVE | COMPLETED | CANCELLED | POSTPONED"

Response 200: MatchScheduleItem[]
```

```jsonc
// MatchScheduleItem
{
  "slug":        "kandy-highlanders-vs-galle-southerners",  // URL-friendly id used by /match/rugby/:slug
  "embedSlug":   "kandy-highlanders-vs-galle-southerners",  // optional alias used by carousel
  "sport":       "RUGBY",
  "tournament":  "Ceylon Rugby Premier League 2026",
  "stage":       "Group | Quarter Final | Semi Final | Final | null",
  "match":       "Match 7",                       // optional bracket label
  "matchNumber": 7,                               // optional sequence number
  "homeTeam":    "Kandy Highlanders",
  "awayTeam":    "Galle Southerners",
  "startTime":   "2026-05-11T15:30:00+05:30",     // ISO8601 with timezone
  "group":       "A",                             // optional, group-stage only
  "venue":       "Racecourse Ground",
  "ground":      "Pitch 1",                       // optional short label
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

### GET match/rugby/past-featured

```
Query params (optional):
  limit?: number   // default 10

Response 200: FeaturedMatchSummary[]
```

```jsonc
// FeaturedMatchSummary
{
  "embedSlug":  "kandy-highlanders-vs-galle-southerners",
  "slug":       "kandy-highlanders-vs-galle-southerners",   // canonical slug for match detail
  "sport":      "RUGBY",
  "tournament": "Ceylon Rugby Premier League 2026",
  "homeTeam":   "Kandy Highlanders",
  "awayTeam":   "Galle Southerners",
  "startTime":  "2026-05-11T15:30:00+05:30",
  "group":      "A",
  "venue":      "Racecourse Ground",
  "derived": {
    "wonBy":   "by 8 points",                     // human-readable result
    "comment": "Full time"                        // e.g. "Full time", "After ET"
  },
  "scoreboard": {
    "winner": "Kandy Highlanders",                // team name; empty string if drawn/no result
    "homeTeam": {
      "kickFirst":         true,
      "final":             27,                    // total points
      "finalTries":        3,
      "finalConversions":  3,
      "finalPenalties":    2,
      "finalDropGoals":    0,
      "scorers": [
        {
          "order":       1,                       // scorer order; -256 means "unknown"
          "name":        "Sahan Ratnayake",
          "tries":       2,
          "conversions": 0,
          "penalties":   0,
          "dropGoals":   0,
          "points":      10
        }
      ]
    },
    "awayTeam": {
      "kickFirst":         false,
      "final":             19,
      "finalTries":        2,
      "finalConversions":  0,
      "finalPenalties":    3,
      "finalDropGoals":    0,
      "scorers": [
        { "order": 1, "name": "Lasitha de Silva",   "tries": 1, "conversions": 0, "penalties": 0, "dropGoals": 0, "points": 5 },
        { "order": 2, "name": "Roshan Edirisinghe", "tries": 0, "conversions": 0, "penalties": 3, "dropGoals": 0, "points": 9 }
      ]
    }
  }
}
```

Field semantics:
- `final` — total points (try = 5, conversion = 2, penalty = 3, drop goal = 3).
- `finalTries / finalConversions / finalPenalties / finalDropGoals` — count of
  each scoring action across the match.
- The carousel only shows the top two scorers per side, so `scorers` may be
  truncated server-side.
- `-256` is the placeholder the legacy mock uses when a value is unknown;
  the API SHOULD return real values and omit unknowns rather than encoding -256.

---

## 2a. Match summaries (carousel)

Compact summary cards shown at the top of the rugby explorer page. Drives the
`match-summary-carousel` component (10 cards in a horizontal PrimeNG carousel).
Mixes past results, in-progress matches, and the next few upcoming fixtures so
the same card shape can render any of them.

### GET match/rugby/summaries

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
  "slug":            "kandy-highlanders-vs-galle-southerners",
  "embedSlug":       "kandy-highlanders-vs-galle-southerners",
  "sport":           "RUGBY",                              // sport code; drives the sport icon on the card
  "tournament":      "Ceylon Rugby Premier League 2026",   // full tournament name
  "tournamentLabel": "CRPL 26",                            // short code shown top-right on the card
  "status":          "UPCOMING | LIVE | RESULT",           // drives the card status pill
  "startTime":       "2026-05-11T15:30:00+05:30",          // ISO8601 with timezone
  "venue":           "Racecourse Ground",
  "venueLabel":      "RACECOURSE GROUND",                  // optional pre-uppercased display string

  "homeTeam": {
    "name":      "Kandy Highlanders",
    "shortName": "KAN",                                    // optional 2–4 letter code
    "logoUrl":   "/assets/images/matchResult/Kandy%20Highlanders.png",  // empty string allowed
    "halves": [                                            // 0 (upcoming) / 1–2 (live or finished) entries
      { "points": 17, "tries": 2, "conversions": 2, "penalties": 1, "dropGoals": 0, "label": "1st Half" },
      { "points": 10, "tries": 1, "conversions": 1, "penalties": 1, "dropGoals": 0, "label": "2nd Half" }
    ]
  },
  "awayTeam": {
    "name":      "Galle Southerners",
    "shortName": "GAL",
    "logoUrl":   "/assets/images/matchResult/Galle%20Southerners.png",
    "halves": [
      { "points":  8, "tries": 1, "conversions": 0, "penalties": 1, "dropGoals": 0, "label": "1st Half" },
      { "points": 11, "tries": 1, "conversions": 0, "penalties": 2, "dropGoals": 0, "label": "2nd Half" }
    ]
  },

  "result": {                                              // omit / null when status !== "RESULT"
    "kind":  "WIN | DRAW | NO_RESULT",
    "label": "KANDY HIGHLANDERS WON BY 8 POINTS"           // already cased for display
  }
}
```

Field semantics:

- `sport` is the sport code that drives the small sport icon rendered next to
  the tournament label on each card. For this endpoint the value is always
  `"RUGBY"`, but the field is required so the same `MatchSummary` shape can
  be reused by a future cross-sport summaries endpoint.
- `status` is what the card renders as a coloured pill (`RESULT` red, `LIVE`
  green-pulsing, `UPCOMING` neutral). It MUST agree with the presence of
  `result`: `UPCOMING`/`LIVE` carry no `result`, `RESULT` MUST carry one.
- `halves[]` lets the same card render 80-minute matches (two entries per side,
  e.g. `"1H 17 / 2H 10 = 27"`). Sevens or tens variants may use a single entry
  per half. For `UPCOMING`, both teams' arrays MUST be empty. The optional
  `label` ("1st Half", "2nd Half", "Extra Time") drives the per-row label.
- `tournamentLabel` is the short code rendered in the top-right corner of the
  card. When omitted the client falls back to `tournament`.
- `venueLabel` is rendered verbatim under upcoming cards next to the date.
  When omitted the client uppercases `venue`.
- `logoUrl` may be an empty string; the client then falls back to
  `assets/images/matchResult/<name>.png`.

Caching: same expectations as `match/rugby/past-featured` — `shareReplay(1)` on
the client, so `Cache-Control: public, max-age=60` is fine.

---

## 3. Match detail (full scorecard page)

Powers `match-page` — tabs: Summary, Scorecard, Commentary, Analysis, Heroes,
MVP, Teams, Gallery. The frontend interface is `RugbyMatchDetail` in
`@core/services/rugby.service.ts`.

### GET match/rugby/:slug

```
Path params:
  slug: string   // canonical match slug

Response 200: RugbyMatchDetail
Response 404: { "error": "Match not found" }
```

```jsonc
// RugbyMatchDetail (top-level)
{
  "slug":         "colombo-titans-vs-jaffna-jets-final",
  "sport":        "RUGBY",
  "tournament":   "Lanka Rugby Cup 2026",
  "seriesName":   "Lanka Rugby Cup 2026",
  "status":       "Past | Live | Upcoming",
  "stage":        "Final",
  "venue":        "Sugathadasa Stadium",
  "city":         "Colombo",
  "location":     "Sugathadasa Stadium, Colombo",   // display string
  "duration":     "80 Min",                         // pre-formatted duration label
  "startTime":    "12-May-26 05:00 PM",             // display string (already formatted)
  "matchDate":    "12/05/2026",                     // display string

  "homeTeam":     { "name": "Colombo Titans", "score": "24", "tries": 3, "halftime": 14 },
  "awayTeam":     { "name": "Jaffna Jets",    "score": "35", "tries": 5, "halftime": 21 },

  "toss":         "Jaffna Jets opt to receive",
  "result":       "Jaffna Jets won by 11 points",

  "topScorers":   [ RugbyScorer, ... ],   // see below
  "topTacklers":  [ RugbyTackler, ... ],
  "scorecardHalves": [ ScorecardHalf, ... ],
  "commentary":   [ CommentaryHalf, ... ],
  "analysis":     MatchAnalysis,
  "heroes":       MatchHeroes,
  "mvp":          [ MvpPlayer, ... ],

  "videoEmbedUrl":"",                               // empty string when no video
  "streamTitle":  "Colombo Titans Vs Jaffna Jets - GRAND FINAL",
  "rightsHolder": "Lanka Rugby Media",
  "totalViews":   542,
  "liveViewers":  0,

  "halves": [
    { "label": "1st Half",  "durationMins": 40, "from": "17:00", "to": "17:40" },
    { "label": "Halftime",  "durationMins": 10, "from": "17:40", "to": "17:50" },
    { "label": "2nd Half",  "durationMins": 40, "from": "17:50", "to": "18:30" }
  ],

  "lastUpdated":   "2026-05-12 at 19:05",
  "lastUpdatedBy": "LRC Manager"
}
```

### Sub-shapes

```jsonc
// RugbyScorer (Summary tab)
{ "name": "Naveen Selvaraj", "team": "Jaffna Jets", "tries": 0, "conversions": 4, "penalties": 0, "dropGoals": 1, "points": 11 }

// RugbyTackler (Summary tab)
{ "name": "Sanjeewa Kumara", "team": "Jaffna Jets", "tackles": 18, "missed": 1, "turnovers": 2 }
```

```jsonc
// ScorecardHalf
{
  "teamName":  "Colombo Titans",
  "score":     "24",
  "breakdown": "3T 3C 1P 0DG",                      // human-readable scoring breakdown
  "scorers": [
    {
      "name":     "Anura Bandara",
      "position": "Wing",
      "minute":   12,                               // minute of the action (0–80 + ET)
      "kind":     "Try | Conversion | Penalty | Drop Goal | Penalty Try",
      "points":   5
    }
  ],
  "squad": ["Anura Bandara", "Pradeep Jayawardena", "..."]    // matchday squad listing
}
```

```jsonc
// CommentaryHalf
{
  "teamName": "Jaffna Jets",
  "entries":  [ CommentaryEntry, ... ]              // newest first
}

// CommentaryEntry — discriminated union by "kind"

// kind: "event" — scoring or non-scoring in-play moment
{
  "kind":        "event",
  "minute":      "71",                              // string so injury time like "40+2" is supported
  "badge":       "try | conversion | penalty | drop | yellow | red | pending | trophy | whistle",
  "badgeLabel":  "TRY | C | P | DG | YC | RC | ... | CUP | FT",
  "description": "Ravi Sivanathan touches down from a quick tap penalty.",
  "isScoring":   true                               // true for try/conversion/penalty/drop/penalty try
}

// kind: "notification"
{ "kind": "notification", "text": "Jaffna Jets bring on Karthik Rajan for Mahesh Anand" }

// kind: "minute-end" — half/full-time / quarter break marker
{
  "kind":      "minute-end",
  "minute":    80,
  "score":     "24-35",                             // running score at that point
  "halfLabel": "Half Time | Full Time | After Extra Time"
}
```

```jsonc
// MatchAnalysis (powers Run-rate, Worm, Scoring-types charts)
{
  "half1": [
    { "minute": 5,  "points": 0,  "runRate": 0.00 },
    { "minute": 10, "points": 7,  "runRate": 0.70 },
    { "minute": 40, "points": 21, "runRate": 0.53 }
  ],
  "half2": [
    { "minute": 50, "points": 7,  "runRate": 0.70 },
    { "minute": 80, "points": 14, "runRate": 0.35 }
  ],
  "extraTime": [                                    // optional; omit when no ET played
    { "minute": 90,  "points": 3, "runRate": 0.30 },
    { "minute": 100, "points": 7, "runRate": 0.35 }
  ],
  "scoringTypes": [
    { "label": "Tries",       "count": 8 },
    { "label": "Conversions", "count": 7 },
    { "label": "Penalties",   "count": 1 },
    { "label": "Drop Goals",  "count": 1 }
  ]
}
```

Notes:
- `half1` / `half2` SHOULD contain one row per 5-minute bucket actually played
  in that half. The client linearly interpolates between buckets.
- `runRate` is cumulative (points per minute up to and including that bucket).
- `scoringTypes` MUST contain exactly the four labels above in that order so
  the pie chart can index into them.

```jsonc
// MatchHeroes
{
  "playerOfMatch": HeroPlayer,
  "topTryScorer":  HeroPlayer,
  "topTackler":    HeroPlayer
}

// HeroPlayer — `stats` is a free-form object whose fields depend on the role
{
  "name":     "Sanjeewa Kumara",
  "team":     "Jaffna Jets",
  "imageUrl": "",                                   // empty string allowed
  "stats":    { "tackles": 18, "missed": 1, "turnovers": 2 }
  // OR scoring stats: { "tries": 2, "conversions": 0, "penalties": 0, "dropGoals": 0, "points": 10 }
}
```

```jsonc
// MvpPlayer
{
  "rank":     1,
  "name":     "Naveen Selvaraj",
  "team":     "Jaffna Jets",
  "imageUrl": "",
  "attack":   7.200,                                // attack rating (0–10)
  "defense":  1.800,                                // defense rating (0–10)
  "kicking":  9.600,                                // kicking rating (0–10)
  "total":    18.600                                // sum of the three; tie-broken by rank
}
```

`videoEmbedUrl` MUST be either an empty string or a full URL whose host is one
of: `www.youtube.com`, `youtube.com`, `player.vimeo.com`. Any other host is
rejected by the frontend sanitizer.

---

## 4. Live score (optional, for in-progress matches)

Lightweight payload for polling/SSE while a match is `LIVE`. Avoid returning
the full `RugbyMatchDetail` on every poll.

### GET match/rugby/:slug/live

```jsonc
Response 200:
{
  "slug":      "havelock-bulls-vs-negombo-knights-semi",
  "sport":     "RUGBY",
  "status":    "LIVE",
  "half":      "1st Half | 2nd Half | Extra Time",
  "minute":    63,                                  // current match minute (0–80 + ET)
  "homeTeam":  { "name": "Havelock Bulls",   "score": 21, "tries": 3, "conversions": 3, "penalties": 0, "dropGoals": 0 },
  "awayTeam":  { "name": "Negombo Knights",  "score": 16, "tries": 1, "conversions": 1, "penalties": 3, "dropGoals": 0 },
  "possession":{ "home": 58, "away": 42 },          // percentage; sums to 100
  "territory": { "home": 62, "away": 38 },          // percentage; sums to 100
  "lastEvents":["P-NEG", "TRY-HAV", "C-HAV", "P-NEG"],  // most recent events, newest right
  "lastUpdated":"2026-05-15T17:03:11Z"
}
```

`lastEvents` is a compact event log — `<kind>-<team-short>` pairs, optionally
with a minute suffix. Recognised kinds: `TRY`, `C` (conversion), `P` (penalty),
`DG` (drop goal), `YC` (yellow card), `RC` (red card), `SUB`. The client only
uses it for an at-a-glance ticker; full event detail comes from the match
detail commentary.

---

## Field conventions

- **Slugs**: every match is addressed by `slug` (kebab-case, URL-safe). The
  carousel and ticker may use a separate `embedSlug` alias; both MUST resolve
  to the same match detail.
- **Scores**: surface both the structured form (`final`, `finalTries`,
  `finalConversions`, `finalPenalties`, `finalDropGoals`) and the display string
  (`score: "24"`, `tries: 3`). The carousel uses the structured form; the
  detail page uses the strings as-is.
- **Times**:
  - Schedule and live endpoints — ISO 8601 with timezone (`2026-05-11T15:30:00+05:30`).
  - Match detail `startTime`, `matchDate`, `lastUpdated`, and the `halves[].from/to`
    fields are pre-formatted display strings and are rendered verbatim.
  - Commentary `minute` is a string so injury-time markers like `"40+2"` and
    `"80+5"` are representable.
- **Unknowns**: prefer omitting the field or returning `null` over the `-256`
  sentinel that appears in legacy mocks.
- **Empty strings**: `videoEmbedUrl`, `imageUrl`, and `result` may be empty
  strings when not yet known; the frontend treats them the same as missing.

---

## Caching

- `match/rugby/past-featured` and `match/rugby/:slug` are cached on the client
  with `shareReplay(1)` per session, so a `Cache-Control: public, max-age=60`
  (or similar) on the server is safe.
- `match/rugby/:slug/live` should set `Cache-Control: no-store`.
- All other read endpoints retry up to 3 times with a 1s backoff on the client.
