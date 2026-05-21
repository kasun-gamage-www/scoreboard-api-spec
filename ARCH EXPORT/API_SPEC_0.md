# Backend API Specification

Base URL: `{API_BASE_URI}` (configured via environment)

All endpoints expect/return `application/json`.
Most resource routes are registered both with and without `/api` prefixes. For example, `/articles` and `/api/articles` are equivalent unless a route is explicitly listed as `/api/...` only.

Auth endpoints return a JWT token. Protected endpoints currently expect it in the `x-access-token` header.

---

## Users

### GET /user
```json
Response 200: User[]
```

---

## Tournaments

### GET /tournaments
```json
Response 200: Tournament[]  // must include slug + name (used in match/event dropdowns)
```

### GET /tournaments/:slug
```json
Response 200: Tournament
```

### POST /tournaments
```json
Request:
{
  "name":        "string",   // required
  "sport":       "CRICKET | RUGBY | SOCCER | BADMINTON | BOWLING | MOTOR_SPORTS | BASKETBALL | NETBALL | VOLLEYBALL | CYCLING", // required
  "format":      "LEAGUE | KNOCKOUT | GROUP_STAGE | ROUND_ROBIN", // required
  "status":      "UPCOMING | ONGOING | COMPLETED | CANCELLED",    // required, default UPCOMING
  "description": "string?",
  "startDate":   "ISO8601?",
  "endDate":     "ISO8601?",
  "location":    "string?",
  "logoUrl":     "string?"
}

Response 201: Tournament
```

### PUT /tournaments/:slug
```json
Request: same shape as POST

Response 200: Tournament
```

### DELETE /tournaments/:slug
```json
Response 204
```

### POST /tournaments/upload
Protected. Same response shape as `/media/presign`, default folder `tournaments`. See [`MEDIA_API.md`](../MEDIA_API.md).

---

## Other Events

### GET /other-events
```json
Response 200: OtherEvent[]
```

### GET /other-events/:slug
```json
Response 200: OtherEvent
```

### POST /other-events
```json
Request:
{
  "title":          "string",  // required
  "eventType":      "TRAINING | EXHIBITION | CEREMONY | AWARD_CEREMONY | PRESS_CONFERENCE | CLINIC | OTHER", // required
  "status":         "SCHEDULED | LIVE | COMPLETED | CANCELLED | POSTPONED", // required, default SCHEDULED
  "sport":          "CRICKET | RUGBY | SOCCER | BADMINTON | BOWLING | MOTOR_SPORTS | BASKETBALL | NETBALL | VOLLEYBALL | CYCLING | null",
  "tournamentSlug": "string | null",
  "venue":          "string?",
  "scheduledAt":    "ISO8601?",
  "organizer":      "string?",
  "description":    "string?"
}

Response 201: OtherEvent
```

### PUT /other-events/:slug
```json
Request: same shape as POST

Response 200: OtherEvent
```

### DELETE /other-events/:slug
```json
Response 204
```

---

## Venues

### GET /venues
```json
Response 200: Venue[]
```

### GET /venues/:slug
```json
Response 200: Venue
```

### POST /venues
```json
Request:
{
  "name":        "string",  // required
  "type":        "STADIUM | ARENA | GROUND | INDOOR_COURT | OUTDOOR_COURT | CLUBHOUSE | OTHER", // required
  "status":      "ACTIVE | INACTIVE | CLOSED",  // required, default ACTIVE
  "description": "string?",
  "capacity":    "number?",
  "address":     "string?",
  "emirate":     "ABU_DHABI | DUBAI | SHARJAH | AJMAN | UMM_AL_QUWAIN | RAS_AL_KHAIMAH | FUJAIRAH | null",
  "city":        "string?",
  "country":     "string?",
  "website":     "string?",
  "email":       "string?",
  "phone":       "string?",
  "imageUrl":    "string?"
}

Response 201: Venue
```

### PUT /venues/:slug
```json
Request: same shape as POST

Response 200: Venue
```

### DELETE /venues/:slug
```json
Response 204
```

### POST /venues/upload
Protected. Same response shape as `/media/presign`, default folder `venues`. See [`MEDIA_API.md`](../MEDIA_API.md).

---

## Photos

### GET /photos
```json
Response 200: Photo[]
```

### GET /photos/:slug
```json
Response 200: Photo
```

### POST /photos
```json
Request:
{
  "title":       "string",  // required
  "status":      "PUBLISHED | DRAFT",  // required, default DRAFT
  "description": "string?",
  "imageUrl":    "string?"
}

Response 201: Photo
```

### PUT /photos/:slug
```json
Request: same shape as POST

Response 200: Photo
```

### DELETE /photos/:slug
```json
Response 204
```

### POST /photos/upload
Protected. Same response shape as `/media/presign`, default folder `photos`. See [`MEDIA_API.md`](../MEDIA_API.md).

---

## Videos

### GET /videos
```json
Response 200: Video[]
```

### GET /videos/:slug
```json
Response 200: Video
```

### POST /videos
```json
Request:
{
  "caption": "string",  // required
  "sport":   "CRICKET | RUGBY | SOCCER | BADMINTON | BOWLING | MOTOR_SPORTS | BASKETBALL | NETBALL | VOLLEYBALL | CYCLING", // required
  "date":    "ISO8601",  // required
  "source":  "string (YouTube video ID or hosted media URL)", // required
  "status":  "ACTIVE | HIDDEN",  // required, default ACTIVE
  "title":   "string?"
}

Response 201: Video
```

### PUT /videos/:slug
```json
Request: same shape as POST

Response 200: Video
```

### DELETE /videos/:slug
```json
Response 204
```

### POST /videos/upload
Protected. Same response shape as `/media/presign`, default folder `videos`. Used when the admin uploads a hosted video file (mp4/webm/etc) instead of pasting a YouTube URL — the returned `url` is stored in `source`. See [`MEDIA_API.md`](../MEDIA_API.md).

---

## Sponsors

### GET /sponsors
```json
Response 200: Sponsor[]
```

### POST /sponsors
```json
Request:
{
  "name":    "string",  // required
  "logoUrl": "string?"
}

Response 201: Sponsor  // { id, name, logoUrl, ... }
```

### DELETE /sponsors/:id
```json
Response 204
```

### POST /sponsors/upload
Protected. Same response shape as `/media/presign`, default folder `sponsors`. See [`MEDIA_API.md`](../MEDIA_API.md).

---

## Polls

### GET /polls
```json
Response 200: Poll[]
```

### GET /polls/:id
```json
Response 200: Poll
```

### POST /polls
```json
Request:
{
  "question":  "string",                        // required
  "options":   ["string", "string", ...],       // required, min 2 items
  "status":    "DRAFT | ACTIVE | CLOSED",       // required, default DRAFT
  "startDate": "ISO8601?",
  "endDate":   "ISO8601?"
}

Response 201: Poll  // { id, question, options, status, startDate, endDate, totalVotes, createdAt }
```

### PUT /polls/:id
```json
Request: same shape as POST

Response 200: Poll
```

### DELETE /polls/:id
```json
Response 204
```

---

## Common Patterns

**Slug**: every resource is identified by a URL-friendly slug (not numeric ID). The backend generates it, likely from the name. It must be returned in all GET responses.

**Upload endpoints**: return presigned S3 upload details. Called before the main save; upload the file directly to S3, then store the returned `url` in the resource's `*Url` field.

**Protected endpoints**: send `x-access-token: <JWT>`. The current CRUD routes are not protected in code; media presign/delete and resource upload aliases are protected.

**Sport enum** (shared across entities):
`CRICKET | RUGBY | SOCCER | BASKETBALL | NETBALL | VOLLEYBALL | BADMINTON | CYCLING | BOWLING | MOTOR_SPORTS | OTHER`

**Enums used in multiple places**:
- Status (match/other-event): `SCHEDULED | LIVE | COMPLETED | CANCELLED | POSTPONED`
- Status (generic resource): `ACTIVE | INACTIVE`
