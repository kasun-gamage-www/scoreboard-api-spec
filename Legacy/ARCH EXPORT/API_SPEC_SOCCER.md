# Soccer Scores API Specification

Specifies the endpoints and payload shapes required by the frontend to display
soccer match scores. This complements the generic `/matches` CRUD documented in
`API_SPEC.md` — the routes below are the dedicated read endpoints the soccer
display components consume.

Base URL: `{API_BASE_URI}` (configured via environment, e.g. `http://localhost:8081/api/`).

All endpoints return `application/json`. Read endpoints are public. Times are
ISO 8601 unless explicitly marked as pre-formatted display strings.

---

## Display surfaces and the data each needs

| Component                       | Endpoint                                  | Purpose                                       |
|---------------------------------|-------------------------------------------|-----------------------------------------------|
| `featured-match`                | `GET match/soccer/past-featured`          | Carousel of recent featured match scorecards  |
| `top-knock-ticker`              | `GET match/soccer/past-featured`          | Top scorers ticker (same source)              |
| `match-summary-carousel`        | `GET match/soccer/summaries`              | Mixed past/upcoming/live summary card strip   |
| `match-schedule`                | `GET match/list-soccer`                   | Upcoming + recent matches + countdown         |
| `match-page` (match detail)     | `GET match/soccer/:slug`                  | Full scorecard, commentary, analysis, MVP     |
| Live in-page banner (optional)  | `GET match/soccer/:slug/live`             | Lightweight live score polling                |

The frontend caches `past-featured` and per-slug match detail with
`shareReplay(1)`, so these endpoints should be cheap and cacheable.

---

## 1. List matches (schedule)

Returns all soccer matches the schedule view should know about (past + upcoming).
The client filters upcoming vs. past by `startTime` and selects the next one
for the countdown.

### GET match/list-soccer

```
Query params (optional):
  sport?:      "SOCCER"            // when omitted, returns all soccer matches
  tournament?: "string"             // tournament slug
  from?:       "ISO8601"
  to?:         "ISO8601"
  status?:     "SCHEDULED | LIVE | COMPLETED | CANCELLED | POSTPONED"

Response 200: MatchScheduleItem[]
```

```jsonc
// MatchScheduleItem
{
  "slug":        "colombo-fc-vs-jaffna-united",     // URL-friendly id used by /match/soccer/:slug
  "embedSlug":   "colombo-fc-vs-jaffna-united",     // optional alias used by carousel
  "sport":       "SOCCER",
  "tournament":  "Lanka Super League 2026",
  "stage":       "Group | Quarter Final | Semi Final | Final | null",
  "match":       "Match 8",                         // optional bracket label
  "matchNumber": 8,                                 // optional sequence number
  "homeTeam":    "Colombo FC",
  "awayTeam":    "Jaffna United",
  "startTime":   "2026-05-15T19:00:00+05:30",       // ISO8601 with timezone
  "group":       "A",                               // optional, group-stage only
  "venue":       "Sugathadasa Stadium",
  "ground":      "Main",                            // optional short label
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

### GET match/soccer/past-featured

```
Query params (optional):
  limit?: number   // default 10

Response 200: FeaturedMatchSummary[]
```

```jsonc
// FeaturedMatchSummary
{
  "embedSlug":  "kandy-strikers-vs-galle-mariners",
  "slug":       "kandy-strikers-vs-galle-mariners",     // canonical slug for match detail
  "sport":      "SOCCER",
  "tournament": "Lanka Super League 2026",
  "homeTeam":   "Kandy Strikers",
  "awayTeam":   "Galle Mariners",
  "startTime":  "2026-05-12T18:30:00+05:30",
  "group":      "B",
  "venue":      "Pallekele Stadium",
  "derived": {
    "wonBy":   "by 1 goal",                             // human-readable result
    "comment": "Full time"                              // e.g. "After ET", "On penalties"
  },
  "scoreboard": {
    "winner": "Kandy Strikers",                         // team name; empty string if drawn/no result
    "homeTeam": {
      "kickOff":            true,
      "final":              3,                          // total goals
      "finalGoals":         3,
      "finalShots":         14,
      "finalShotsOnTarget": 7,
      "finalCorners":       6,
      "finalYellowCards":   2,
      "finalRedCards":      0,
      "scorers": [
        {
          "order":    1,                                // scorer order; -256 means "unknown"
          "name":     "Sanjaya Perera",
          "goals":    2,
          "assists":  0,
          "minutes":  ["18", "67"]                      // minutes of each goal as strings ("45+2" supported)
        }
      ]
    },
    "awayTeam": {
      "kickOff":            false,
      "final":              2,
      "finalGoals":         2,
      "finalShots":         11,
      "finalShotsOnTarget": 5,
      "finalCorners":       4,
      "finalYellowCards":   3,
      "finalRedCards":      1,
      "scorers": [
        { "order": 1, "name": "Roshan Madushanka", "goals": 1, "assists": 0, "minutes": ["29"] },
        { "order": 2, "name": "Charith Fernando",  "goals": 1, "assists": 1, "minutes": ["81"] }
      ]
    }
  }
}
```

Field semantics:
- `final` — total goals (mirrors `finalGoals`; surfaced for parity with cricket
  and rugby `final` fields).
- For knockout matches decided by penalties, the `final` figure is the regulation
  + extra-time score; the shootout result is conveyed via `derived.wonBy`
  (e.g. `"on penalties (4-2)"`).
- The carousel only shows the top two scorers per side, so `scorers` may be
  truncated server-side.
- `-256` is the placeholder the legacy mock uses when a value is unknown;
  the API SHOULD return real values and omit unknowns rather than encoding -256.

---

## 2a. Match summaries (carousel)

Compact summary cards shown at the top of the soccer explorer page. Drives the
`match-summary-carousel` component (10 cards in a horizontal PrimeNG carousel).
Mixes past results, in-progress matches, and the next few upcoming fixtures so
the same card shape can render any of them.

### GET match/soccer/summaries

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
  "slug":            "kandy-strikers-vs-galle-mariners",
  "embedSlug":       "kandy-strikers-vs-galle-mariners",
  "sport":           "SOCCER",                            // sport code; drives the sport icon on the card
  "tournament":      "Lanka Super League 2026",           // full tournament name
  "tournamentLabel": "LSL 26",                            // short code shown top-right on the card
  "status":          "UPCOMING | LIVE | RESULT",          // drives the card status pill
  "startTime":       "2026-05-12T18:30:00+05:30",         // ISO8601 with timezone
  "venue":           "Pallekele Stadium",
  "venueLabel":      "PALLEKELE STADIUM",                 // optional pre-uppercased display string

  "homeTeam": {
    "name":      "Kandy Strikers",
    "shortName": "KAN",                                   // optional 2–4 letter code
    "logoUrl":   "/assets/images/matchResult/Kandy%20Strikers.png",   // empty string allowed
    "halves": [                                           // 0 (upcoming) / 1–3 (live, finished, after ET)
      { "goals": 1, "shots": 7, "shotsOnTarget": 3, "corners": 3, "label": "1st Half" },
      { "goals": 2, "shots": 7, "shotsOnTarget": 4, "corners": 3, "label": "2nd Half" }
    ]
  },
  "awayTeam": {
    "name":      "Galle Mariners",
    "shortName": "GAL",
    "logoUrl":   "/assets/images/matchResult/Galle%20Mariners.png",
    "halves": [
      { "goals": 1, "shots": 6, "shotsOnTarget": 3, "corners": 2, "label": "1st Half" },
      { "goals": 1, "shots": 5, "shotsOnTarget": 2, "corners": 2, "label": "2nd Half" }
    ]
  },

  "result": {                                             // omit / null when status !== "RESULT"
    "kind":  "WIN | DRAW | NO_RESULT",
    "label": "KANDY STRIKERS WON 3-2"                     // already cased for display
  }
}
```

Field semantics:

- `sport` is the sport code that drives the small sport icon rendered next to
  the tournament label on each card. For this endpoint the value is always
  `"SOCCER"`, but the field is required so the same `MatchSummary` shape can
  be reused by a future cross-sport summaries endpoint.
- `status` is what the card renders as a coloured pill (`RESULT` red, `LIVE`
  green-pulsing, `UPCOMING` neutral). It MUST agree with the presence of
  `result`: `UPCOMING`/`LIVE` carry no `result`, `RESULT` MUST carry one.
- `halves[]` lets the same card render 90-minute matches (two entries per side)
  and knockout matches that went to extra time (third entry with
  `label: "Extra Time"`). For `UPCOMING`, both teams' arrays MUST be empty.
- `tournamentLabel` is the short code rendered in the top-right corner of the
  card. When omitted the client falls back to `tournament`.
- `venueLabel` is rendered verbatim under upcoming cards next to the date.
  When omitted the client uppercases `venue`.
- `logoUrl` may be an empty string; the client then falls back to
  `assets/images/matchResult/<name>.png`.

Caching: same expectations as `match/soccer/past-featured` —
`shareReplay(1)` on the client, so `Cache-Control: public, max-age=60` is fine.

---

## 3. Match detail (full scorecard page)

Powers `match-page` — tabs: Summary, Lineups, Commentary, Analysis, Heroes,
MVP, Teams, Gallery. The frontend interface is `SoccerMatchDetail` in
`@core/services/soccer.service.ts`.

### GET match/soccer/:slug

```
Path params:
  slug: string   // canonical match slug

Response 200: SoccerMatchDetail
Response 404: { "error": "Match not found" }
```

```jsonc
// SoccerMatchDetail (top-level)
{
  "slug":         "matara-eagles-vs-negombo-storm-final",
  "sport":        "SOCCER",
  "tournament":   "President Cup 2026",
  "seriesName":   "President Cup 2026",
  "status":       "Past | Live | Upcoming",
  "stage":        "Final",
  "venue":        "Race Course Stadium",
  "city":         "Colombo",
  "location":     "Race Course Stadium, Colombo",   // display string
  "duration":     "90 + ET + Penalties",            // pre-formatted duration label
  "startTime":    "10-May-26 07:30 PM",             // display string (already formatted)
  "matchDate":    "10/05/2026",                     // display string

  "homeTeam":     { "name": "Matara Eagles", "score": "1", "halftime": 0, "penalties": 2 },
  "awayTeam":     { "name": "Negombo Storm", "score": "1", "halftime": 1, "penalties": 4 },

  "referee":      "Hasitha Wijesinghe",
  "attendance":   12480,
  "result":       "Negombo Storm won 4-2 on penalties (1-1 a.e.t.)",

  "topScorers":   [ SoccerScorer, ... ],            // see below
  "keepers":      [ SoccerKeeper, ... ],
  "teamStats":    TeamStats,
  "lineups":      [ TeamLineup, ... ],              // exactly two entries (home, away)
  "events":       [ MatchEvent, ... ],              // chronological event log
  "commentary":   [ CommentaryHalf, ... ],
  "analysis":     MatchAnalysis,
  "heroes":       MatchHeroes,
  "mvp":          [ MvpPlayer, ... ],

  "videoEmbedUrl":"",                               // empty string when no video
  "streamTitle":  "Matara Eagles Vs Negombo Storm - PRESIDENT CUP FINAL",
  "rightsHolder": "Lanka Sports Network",
  "totalViews":   6210,
  "liveViewers":  0,

  "halves": [
    { "label": "1st Half",   "durationMins": 45, "from": "19:30", "to": "20:15" },
    { "label": "Halftime",   "durationMins": 15, "from": "20:15", "to": "20:30" },
    { "label": "2nd Half",   "durationMins": 45, "from": "20:30", "to": "21:15" },
    { "label": "Extra Time", "durationMins": 30, "from": "21:25", "to": "21:55" },
    { "label": "Penalties",  "durationMins":  8, "from": "21:55", "to": "22:03" }
  ],

  "lastUpdated":   "2026-05-10 at 22:18",
  "lastUpdatedBy": "PC Manager"
}
```

`homeTeam.penalties` / `awayTeam.penalties` is the shootout score; omit when
the match did not go to penalties.

### Sub-shapes

```jsonc
// SoccerScorer (Summary tab)
{ "name": "Lasantha Gunaratne", "team": "Negombo Storm", "goals": 1, "assists": 0, "shots": 3, "shotsOnTarget": 2 }

// SoccerKeeper (Summary tab)
{ "name": "Sahan Liyanage", "team": "Negombo Storm", "saves": 3, "goalsConceded": 1, "cleanSheet": false, "penaltiesSaved": 2 }
```

```jsonc
// TeamStats — every nested object is { home, away }
{
  "possession":     { "home": 53, "away": 47 },     // percentage; sums to 100
  "shots":          { "home": 13, "away": 12 },
  "shotsOnTarget":  { "home":  4, "away":  6 },
  "corners":        { "home":  7, "away":  5 },
  "fouls":          { "home": 14, "away": 16 },
  "offsides":       { "home":  3, "away":  4 },
  "yellowCards":    { "home":  3, "away":  2 },
  "redCards":       { "home":  0, "away":  0 }
}
```

```jsonc
// TeamLineup
{
  "teamName":  "Matara Eagles",
  "formation": "4-3-3",                             // free-form, e.g. "4-2-3-1"
  "starters": [
    {
      "number":   10,                               // shirt number
      "name":     "Asanka Bandara",
      "position": "GK | CB | RB | LB | CDM | CM | CAM | RW | LW | ST | RM | LM",
      "captain":  false                             // optional, defaults false
    }
  ],
  "substitutes": [
    { "number": 12, "name": "Kavindu Bandaranayake", "position": "GK" }
  ]
}
```

```jsonc
// MatchEvent — discriminated union by "kind"

// kind: "goal"
{
  "minute":      "56",
  "kind":        "goal",
  "team":        "Matara Eagles",
  "player":      "Asanka Bandara",
  "assist":      "Pradeep Senanayake",              // optional
  "description": "Curling shot into the top corner."
}

// kind: "yellow" | "red"
{ "minute": "78", "kind": "yellow", "team": "Negombo Storm", "player": "Sanjeewa Weerakoon", "description": "Tactical foul." }

// kind: "sub"
{ "minute": "67", "kind": "sub", "team": "Negombo Storm", "playerOff": "Pasindu Madhushanka", "playerOn": "Sahan Madawela" }

// kind: "fulltime" — half-time / full-time / end of extra time
{ "minute": "90+5", "kind": "fulltime", "score": "1-1", "description": "Match goes to extra time." }

// kind: "penalty-shootout"
{ "minute": "PEN", "kind": "penalty-shootout", "score": "4-2", "description": "Sahan Liyanage saves twice; Negombo Storm lift the cup." }
```

`minute` is always a string so injury-time values like `"45+2"`, `"90+4"` and
the literal `"PEN"` are all representable.

```jsonc
// CommentaryHalf
{
  "teamName": "Negombo Storm",
  "entries":  [ CommentaryEntry, ... ]              // newest first
}

// CommentaryEntry — discriminated union by "kind"

// kind: "event" — scoring or non-scoring in-play moment
{
  "kind":        "event",
  "minute":      "90+4",
  "badge":       "goal | yellow | red | corner | whistle | trophy | sub",
  "badgeLabel":  "GOAL | YC | RC | CK | FT | CUP | SUB",
  "description": "Charith Fernando heads in. 3-2.",
  "isScoring":   true                               // true for goals only
}

// kind: "notification"
{ "kind": "notification", "text": "Negombo Storm bring on Sahan Madawela for Pasindu Madhushanka" }
```

```jsonc
// MatchAnalysis (powers Shot, Possession and Goal-type charts)
{
  "half1": [
    { "minute":  5, "homeShots": 1, "awayShots": 0, "possessionHome": 56 },
    { "minute": 45, "homeShots": 6, "awayShots": 5, "possessionHome": 52 }
  ],
  "half2": [
    { "minute": 50, "homeShots": 7,  "awayShots": 5,  "possessionHome": 53 },
    { "minute": 90, "homeShots": 11, "awayShots": 10, "possessionHome": 53 }
  ],
  "extraTime": [                                    // optional; omit when no ET played
    { "minute": 100, "homeShots": 12, "awayShots": 11, "possessionHome": 51 },
    { "minute": 120, "homeShots": 13, "awayShots": 12, "possessionHome": 53 }
  ],
  "goalTypes": [
    { "label": "Open play", "count": 1 },
    { "label": "Set piece", "count": 1 },
    { "label": "Penalty",   "count": 0 },
    { "label": "Own goal",  "count": 0 }
  ]
}
```

Notes:
- `half1` / `half2` SHOULD contain one row per 5–10-minute bucket actually
  played in that half. The client linearly interpolates between buckets.
- `homeShots` / `awayShots` are cumulative shot totals at that minute.
- `possessionHome` is the cumulative possession percentage; the client derives
  `possessionAway` as `100 - possessionHome`.
- `goalTypes` MUST contain exactly the four labels above in that order so the
  pie chart can index into them.

```jsonc
// MatchHeroes
{
  "playerOfMatch": HeroPlayer,
  "topScorer":     HeroPlayer,
  "topAssist":     HeroPlayer
}

// HeroPlayer — `stats` is a free-form object whose fields depend on the role
{
  "name":     "Sahan Liyanage",
  "team":     "Negombo Storm",
  "imageUrl": "",                                   // empty string allowed
  "stats":    { "saves": 3, "penaltiesSaved": 2, "goalsConceded": 1 }
  // OR scoring stats: { "goals": 1, "assists": 0, "shots": 3, "shotsOnTarget": 2 }
  // OR creator stats: { "assists": 1, "keyPasses": 4, "crosses": 7 }
}
```

```jsonc
// MvpPlayer
{
  "rank":         1,
  "name":         "Sahan Liyanage",
  "team":         "Negombo Storm",
  "imageUrl":     "",
  "attack":       0.000,                            // attack rating (0–10)
  "defense":      8.500,                            // defense rating (0–10)
  "distribution": 1.500,                            // distribution / playmaking rating (0–10)
  "total":        10.000                            // sum of the three; tie-broken by rank
}
```

`videoEmbedUrl` MUST be either an empty string or a full URL whose host is one
of: `www.youtube.com`, `youtube.com`, `player.vimeo.com`. Any other host is
rejected by the frontend sanitizer.

---

## 4. Live score (optional, for in-progress matches)

Lightweight payload for polling/SSE while a match is `LIVE`. Avoid returning
the full `SoccerMatchDetail` on every poll.

### GET match/soccer/:slug/live

```jsonc
Response 200:
{
  "slug":      "colombo-fc-vs-jaffna-united",
  "sport":     "SOCCER",
  "status":    "LIVE",
  "half":      "1st Half | 2nd Half | Extra Time 1 | Extra Time 2 | Penalties",
  "minute":    79,                                  // current match minute (0–90 + ET)
  "addedTime": 0,                                   // injury time minutes already added in this period
  "homeTeam":  { "name": "Colombo FC",    "score": 2, "shots": 11, "shotsOnTarget": 5, "corners": 5, "yellowCards": 2, "redCards": 0 },
  "awayTeam":  { "name": "Jaffna United", "score": 1, "shots": 10, "shotsOnTarget": 4, "corners": 3, "yellowCards": 2, "redCards": 0 },
  "possession":{ "home": 54, "away": 46 },          // percentage; sums to 100
  "lastEvents":["CK-JAF", "GOAL-CFC-71", "SUB-CFC-63", "GOAL-JAF-54-PEN"],  // newest right
  "lastUpdated":"2026-05-15T20:22:11Z"
}
```

`lastEvents` is a compact event log — `<kind>-<team-short>[-<minute>][-<note>]`
tokens. Recognised kinds: `GOAL`, `YC` (yellow card), `RC` (red card), `SUB`,
`CK` (corner kick), `OFF` (offside). Optional notes: `PEN` (penalty),
`OG` (own goal). The client only uses it for an at-a-glance ticker; full event
detail comes from the match detail commentary.

---

## Field conventions

- **Slugs**: every match is addressed by `slug` (kebab-case, URL-safe). The
  carousel and ticker may use a separate `embedSlug` alias; both MUST resolve
  to the same match detail.
- **Scores**: surface both the structured form (`final`, `finalGoals`,
  `finalShots`, `finalShotsOnTarget`, `finalCorners`, `finalYellowCards`,
  `finalRedCards`) and the display string (`score: "3"`). The carousel uses
  the structured form; the detail page uses the strings as-is.
- **Times**:
  - Schedule and live endpoints — ISO 8601 with timezone (`2026-05-15T19:00:00+05:30`).
  - Match detail `startTime`, `matchDate`, `lastUpdated`, and the `halves[].from/to`
    fields are pre-formatted display strings and are rendered verbatim.
  - All `minute` fields (commentary, events, live) are strings so injury-time
    values like `"45+2"`, `"90+4"`, and the literal `"PEN"` are representable.
- **Unknowns**: prefer omitting the field or returning `null` over the `-256`
  sentinel that appears in legacy mocks.
- **Empty strings**: `videoEmbedUrl`, `imageUrl`, and `result` may be empty
  strings when not yet known; the frontend treats them the same as missing.

---

## Caching

- `match/soccer/past-featured` and `match/soccer/:slug` are cached on the
  client with `shareReplay(1)` per session, so a `Cache-Control: public,
  max-age=60` (or similar) on the server is safe.
- `match/soccer/:slug/live` should set `Cache-Control: no-store`.
- All other read endpoints retry up to 3 times with a 1s backoff on the client.
