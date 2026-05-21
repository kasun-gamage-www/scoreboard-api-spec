# Polls API Specification

Base URL: `{API_BASE_URI}` (configured via environment).

All endpoints expect/return `application/json`. Routes are reachable both with and without the `/api` prefix unless explicitly noted.

This document covers the **public, reader-facing** poll endpoints consumed by the `/polls` page in `scoreboard-app`. CRUD/admin endpoints (`POST /polls`, `PUT /polls/:id`, `DELETE /polls/:id`) live in the main `API_SPEC.md` and are unchanged.

---

## Concepts

### Poll lifecycle / `status`

| Status   | Visible on `/polls`? | Votable? | Notes                                                                 |
|----------|----------------------|----------|-----------------------------------------------------------------------|
| `DRAFT`  | no                   | no       | Admin-only; backend MUST exclude from any public list.                |
| `ACTIVE` | yes                  | yes      | Accepting votes. Also auto-closes when `endDate` is in the past.      |
| `CLOSED` | yes (closed tab)     | no       | Results visible; vote endpoint MUST return `409`.                     |

The backend SHOULD treat an `ACTIVE` poll whose `endDate` has passed as effectively `CLOSED` (return it in the `closed` filter, reject votes with `409`). It MAY persistently flip the status on a scheduled job, but the GET responses MUST behave as if the flip already happened.

### Voter identity (anonymous voting)

Voting is anonymous — no signup required — but the backend MUST make a best-effort attempt to prevent the same person from voting twice on the same poll. Strategy:

1. **Voter cookie.** On the first vote request that has no `voterId` cookie, the backend issues a `Set-Cookie: voterId=<uuid>; Path=/; Max-Age=63072000; SameSite=Lax; HttpOnly; Secure` cookie. Subsequent vote requests echo this cookie back automatically; the backend uses it as the dedupe key.
2. **IP + UA fallback.** If the client refuses cookies, the backend SHOULD additionally key on a hash of `(remote IP, User-Agent)` per poll, so a cookie wipe alone doesn't enable rapid revoting.
3. **Authenticated voters.** If an `x-access-token` JWT is present, prefer the user id over the cookie as the dedupe key, and record `voterUserId` alongside the vote.

Duplicate votes return `409 Conflict` (see `POST /polls/:id/vote` below). Changing your vote is **not** supported in v1.

### CORS / cookies

The poll endpoints are called cross-origin from the public site. The backend MUST:
- send `Access-Control-Allow-Credentials: true`
- echo the request `Origin` in `Access-Control-Allow-Origin` (no wildcard, since credentials are in use)
- send `Vary: Origin`

The frontend MUST send `withCredentials: true` on `GET /polls*` and `POST /polls/:id/vote` so the `voterId` cookie round-trips.

---

## Schemas

### `Poll`

```jsonc
{
  "id":          "string",                          // stable id (uuid or slug)
  "question":    "string",
  "options": [
    {
      "id":      "string",                          // stable per option (uuid); index is NOT stable
      "text":    "string",
      "votes":   123                                // tally; always present, 0 if no votes yet
    }
  ],
  "status":      "ACTIVE | CLOSED",                 // DRAFT is never returned on public endpoints
  "startDate":   "ISO8601 | null",
  "endDate":     "ISO8601 | null",
  "totalVotes":  456,                               // sum of options[].votes
  "createdAt":   "ISO8601",
  "hasVoted":    false,                             // see "Per-request fields" below
  "votedOptionId": null                             // string when hasVoted is true, else null
}
```

#### Per-request fields

- `hasVoted` — `true` if the current voter (identified by cookie / JWT) has already voted on this poll. Always present; backend computes it per request. For unauthenticated callers with no `voterId` cookie, return `false`.
- `votedOptionId` — when `hasVoted` is `true`, the `options[].id` they chose; otherwise `null`. Lets the UI highlight the user's pick.

---

## Endpoints

### `GET /polls`

List polls for the public `/polls` page. Supports three mutually-exclusive sort/filter modes via the `filter` query param.

```
Query params:
  filter?   "popular" | "latest" | "closed"   (default: "latest")
  limit?    number                             (default: 20, max: 100)
  offset?   number                             (default: 0)
```

Filter semantics:

| `filter`  | Includes statuses | Sort order                                                |
|-----------|-------------------|-----------------------------------------------------------|
| `latest`  | `ACTIVE` only     | `createdAt` DESC                                          |
| `popular` | `ACTIVE` only     | `totalVotes` DESC, then `createdAt` DESC as tiebreaker    |
| `closed`  | `CLOSED` only (plus `ACTIVE` polls whose `endDate` has passed) | `endDate` DESC, then `createdAt` DESC |

Unknown `filter` values MUST return `400 Bad Request`.

```jsonc
Response 200:
{
  "filter": "latest",
  "limit": 20,
  "offset": 0,
  "total": 37,                // total matching the filter, ignoring limit/offset
  "items": [ Poll, Poll, ... ]
}
```

The endpoint MUST be safe to call without credentials. If a `voterId` cookie / JWT is present, populate `hasVoted` / `votedOptionId` on each returned poll; otherwise both are `false` / `null`.

---

### `GET /polls/:id`

Fetch a single poll by `id`. Returns `404` for `DRAFT` polls or unknown ids.

```jsonc
Response 200: Poll
Response 404: { "error": "Poll not found" }
```

Same `hasVoted` / `votedOptionId` rules as the list endpoint.

---

### `POST /polls/:id/vote`

Record one vote on the given poll.

```jsonc
Request:
{
  "optionId": "string"   // must match one of the poll's options[].id values
}
```

Behavior:

1. Resolve voter identity (JWT > `voterId` cookie > IP+UA fallback; see "Voter identity" above).
2. If no `voterId` cookie was on the request, mint one and set it on the response.
3. Validate:
   - Poll exists and is not `DRAFT` → else `404`.
   - Poll status is `ACTIVE` AND (`endDate` is null OR `endDate` is in the future) → else `409 { "error": "Poll is closed" }`.
   - `optionId` belongs to this poll's `options` → else `400 { "error": "Invalid optionId" }`.
   - This voter has not already voted on this poll → else `409 { "error": "Already voted", "votedOptionId": "..." }`.
4. Atomically increment `options[i].votes` and `totalVotes`, and record the vote (with the voter identity key) for dedupe.
5. Return the updated poll (so the client can re-render bars without a second round-trip).

```jsonc
Response 200: Poll   // hasVoted=true, votedOptionId=<the option they just picked>
Response 400: { "error": "Invalid optionId" }
Response 404: { "error": "Poll not found" }
Response 409: { "error": "Poll is closed" }
           | { "error": "Already voted", "votedOptionId": "..." }
```

The response MUST include the `Set-Cookie: voterId=...` header on the first vote from a given browser. Subsequent requests reuse the same cookie.

#### Atomicity

The increment + dedupe-record MUST happen in a single transaction (or via a unique index on `(pollId, voterKey)` + retry on conflict). Two simultaneous votes from the same voter MUST result in exactly one tally bump and one `409` — never two bumps and never two `409`s.

#### Rate limiting

Per-IP rate limit of ~30 vote attempts/minute is recommended to slow brute-force tampering. Returns `429 Too Many Requests` with a `Retry-After` header when tripped.

---

## Error format

All non-2xx responses use:

```json
{ "error": "human-readable message", "code": "OPTIONAL_MACHINE_CODE" }
```

---

## Notes for the frontend

- Treat `options[].id` as the stable identifier — never the array index.
- Send `withCredentials: true` on every poll request so the voter cookie persists.
- After a successful `POST /polls/:id/vote`, replace the cached poll with the response body rather than re-fetching the list.
- The `closed` tab can mix polls with `status: "CLOSED"` and `status: "ACTIVE"` whose `endDate` has passed; the client SHOULD render both as read-only and rely on the server to disallow the vote.
