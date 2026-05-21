# Auth API Specification

Base URL: `{API_BASE_URI}` (configured via environment).

All endpoints expect/return `application/json`. Routes are reachable both with and without the `/api` prefix.

This document covers the authentication endpoints. The token they return is the same `x-access-token` JWT consumed by every protected endpoint in [`API_SPEC.md`](./ARCH%20EXPORT/API_SPEC.md) and the extracted resource specs (e.g. `POST /<resource>/upload`).

---

## Concepts

### Token

Successful auth returns an opaque JWT string. The client MUST send it on protected endpoints in the `x-access-token` header (NOT `Authorization: Bearer …`):

```
x-access-token: <jwt>
```

Endpoints currently treated as protected:

- `POST /media/presign` and `DELETE /media`
- All per-resource upload aliases: `/articles/upload`, `/clubs/upload`, `/photos/upload`, `/players/upload`, `/sponsors/upload`, `/teams/upload`, `/tournaments/upload`, `/venues/upload`, `/videos/upload`
- `POST /matches/upload-schedule`

CRUD on the resource collections (`POST /articles`, `PUT /clubs/:slug`, etc.) is not protected in code today; treat that as an existing gap rather than the documented contract.

### Obfuscated route names

The signin and signup routes intentionally use non-obvious suffixes (`/auth/signin1988`, `/auth/signup351`) rather than the bare `/auth/signin` / `/auth/signup`. Treat the suffixes as part of the canonical path — they are NOT a typo and MUST NOT be normalised away by the router. Aliases like `/auth/signin` SHOULD return `404`.

### Password handling on signup

The signup endpoint accepts a client-supplied `passHash` rather than a raw password. The backend MUST still apply its own server-side hash (e.g. bcrypt/argon2) on top of whatever the client sends before storing — the `passHash` field is **not** a substitute for server-side hashing, and the backend MUST NOT store the value verbatim.

The signin endpoint accepts a raw `password` (legacy contract) and applies the same comparison routine the backend would apply to the signup-derived stored hash.

---

## Endpoints

### `POST /auth/signin1988`

Exchange username + password for a JWT.

```jsonc
Request:
{
  "username": "string",
  "password": "string"
}

Response 200:
{
  "token": "string",                  // JWT to be sent as x-access-token on protected calls
  "user": {
    "id":   "string",                 // stable user id
    "priv": "string"                  // privilege / role marker (e.g. "admin")
  }
}

Response 400: { "error": "Missing credentials" }
Response 401: { "error": "Invalid credentials" }
```

The backend MUST return the same `401` shape regardless of whether the username exists or the password is wrong, to avoid leaking account existence.

### `POST /auth/signup351`

Create a new account.

```jsonc
Request:
{
  "username": "string",
  "passHash": "string"                // client-side hashed; backend MUST re-hash before storing
}

Response 200:
{
  "token": "string"                   // JWT for the newly created account
}

Response 400: { "error": "Missing fields" }
Response 409: { "error": "Username already taken" }
```

---

## Error format

All non-2xx responses use the project-wide error shape:

```json
{ "error": "human-readable message", "code": "OPTIONAL_MACHINE_CODE" }
```
