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

## Common Patterns

**Slug**: every resource is identified by a URL-friendly slug (not numeric ID). The backend generates it, likely from the name. It must be returned in all GET responses.

**Upload endpoints**: return presigned S3 upload details. Called before the main save; upload the file directly to S3, then store the returned `url` in the resource's `*Url` field.

**Protected endpoints**: send `x-access-token: <JWT>`. The current CRUD routes are not protected in code; media presign/delete and resource upload aliases are protected.

**Sport enum** (shared across entities):
`CRICKET | RUGBY | SOCCER | BASKETBALL | NETBALL | VOLLEYBALL | BADMINTON | CYCLING | BOWLING | MOTOR_SPORTS | OTHER`

**Enums used in multiple places**:
- Status (generic resource): `ACTIVE | INACTIVE`
