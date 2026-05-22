# Cricket Score — Backend API Specification

This spec covers the endpoints the **Cricket Score Editor** (`/scores/:slug` →
`CricketScoreEditorComponent`) calls when an admin updates a cricket match.
Cricket scores piggy-back on the generic `/matches` resource: the editor never
calls a cricket-specific endpoint. All cricket-specific state lives inside the
`details` object on a `Match`.

Base URL: `{API_BASE_URI}` (from `environment.API_BASE_URI`).
All requests/responses are `application/json` unless stated otherwise.

> The legacy `/api/cric-scores` routes (see main `API_SPEC.md`) are **not** used
> by the admin score editor and should be treated as deprecated for new work.

---

## Endpoints the editor calls

### 1. GET /matches/:slug — load match into the editor

Called once when the editor opens. Response hydrates both the "Update Full Match"
form and the "Over-by-Over (Live)" tab.

```json
Response 200: Match  // shape defined below
```

### 2. PUT /matches/:slug — save full match OR save live innings

The editor sends the **same payload shape** for both tabs:

- **Full Match save:** sends the form's `status` as-is.
- **Live (over-by-over) save:** forces `status: "LIVE"` and only patches the
  three fields for the currently-batting innings (`<innings>Runs`,
  `<innings>Wickets`, `<innings>Overs`); all other `details.*` keys are echoed
  back unchanged from the previously loaded match.

The backend MUST treat this as a full replacement of the match document — the
client always sends the complete state it wants persisted, including all four
innings. Do not attempt to diff or merge server-side.

```json
Response 200: Match  // the updated match, used to re-hydrate the editor
```

There is no separate "live" endpoint. The backend does not need to expose one.

---

## Match payload (cricket)

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

The response is the same shape, plus server-managed fields the editor reads
back (`slug`, `featured`, `createdAt`, `updatedAt`, etc. — see main `API_SPEC.md`).

---

## Field rules the backend MUST enforce

| Field | Rule |
|---|---|
| `sport` | Must equal `"CRICKET"` for this editor. Reject other values with `400`. |
| `status` | One of `SCHEDULED \| LIVE \| COMPLETED \| CANCELLED \| POSTPONED`. The editor will send `LIVE` automatically on every over-by-over save. |
| `details.*Runs` | Integer `≥ 0` or `null`. No upper bound. |
| `details.*Wickets` | Integer in `[0, 10]` or `null`. |
| `details.*Overs` | Decimal `X.Y` where `X ≥ 0` and `Y ∈ {0,1,2,3,4,5}`. Examples: `0`, `4`, `4.3`, `19.5`. **`4.6` is invalid** — six legal balls roll the over to `5.0`. Store as a number (the client sends `parseFloat("X.Y")`). |
| `homeScore` / `awayScore` | Optional match-total overrides. The editor surfaces them as separate inputs and does **not** auto-derive them from innings. Treat as opaque numbers. |
| `notes` | Optional free text. Used for the result line (e.g. `"Home won by 5 wickets"`). |
| `details` (unknown keys) | Preserve unknown keys verbatim. Cricket writes must not drop football/rugby/etc. keys that may already be on the document. |

Innings consistency (`runs ≥ 0`, `wickets ≤ 10`, valid `X.Y` overs) is the
**only** cross-field validation required. The backend should not try to
validate cricketing logic beyond that (e.g. don't reject "Inn2 set but Inn1
empty" — admins do edit partial state during a live match).

---

## Status transitions

The editor sets `status` in two ways:

1. **Full Match tab:** whatever the admin picks in the dropdown.
2. **Over-by-Over tab:** always sends `LIVE`, regardless of previous state.

The backend MUST accept any of the five statuses on any update — there is no
state machine. Going `COMPLETED → LIVE` (e.g. correcting a premature
completion) is a legal admin action.

---

## Example: live over save

Scenario: home team is batting in 1st innings, currently `45/2 in 7.4`. Admin
clicks "+4 Four" then "Save Live Score".

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

Expected response is the persisted `Match` with the same `details` echoed back
and `status: "LIVE"`.

---

## Things the backend does NOT need (yet)

The editor does **not** currently call, and the backend does not need to
implement, any of:

- A ball-by-ball event log endpoint. The live tab keeps `ballHistory` only in
  component memory; it is **not** posted to the server. Each "Save Live Score"
  click overwrites the innings totals, not a delta.
- Separate cricket score routes (`/api/cric-scores/*` are unused by this
  editor).
- Innings-level partial updates (`PATCH /matches/:slug/innings/:key`). Whole-
  document `PUT` is sufficient.
- Server-side recomputation of `homeScore`/`awayScore` from innings totals.
  Leave those as admin-managed fields.

If/when ball-by-ball persistence is needed (e.g. for a public live feed), add
a new endpoint rather than changing this contract.
