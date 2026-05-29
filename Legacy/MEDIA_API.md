# Media Storage API Specification

Base URL: `{API_BASE_URI}` (configured via environment).

All endpoints expect/return `application/json`. Routes are reachable both with and without the `/api` prefix.

This document defines the contract for uploading image / video assets. All image and video assets are uploaded **directly to S3 from the browser** using a presigned `PUT`. The frontend never sees S3 credentials, and the file bytes never pass through the API server — the API only signs a short-lived URL.

The same presign contract is exposed under the generic `POST /media/presign` endpoint and via per-resource aliases (`POST /articles/upload`, `POST /clubs/upload`, `POST /photos/upload`, `POST /players/upload`, `POST /sponsors/upload`, `POST /teams/upload`, `POST /tournaments/upload`, `POST /venues/upload`, `POST /videos/upload`). Each alias hard-codes a folder so the client can omit the `folder` field.

This contract applies to team logos, sponsor logos, article images, venue images, photo gallery assets, hosted video files, and every other media field across the API.

---

## Concepts

### Two URLs per upload

Every presign response carries two distinct URLs on different hosts:

- `uploadUrl` — the short-lived presigned **S3 PUT** URL. The client `PUT`s raw file bytes here, browser → S3 directly.
- `url` — the canonical public / CDN URL the client stores on the resource (`logoUrl`, `imageUrl`, `photoUrl`, `source`, …). If the bucket is served via CloudFront, this is the CDN host, not `s3.amazonaws.com`.

These are NOT the same host. The frontend MUST `PUT` to `uploadUrl` and PERSIST `url`.

### Per-resource aliases

The aliases are pure conveniences — same request/response shape as `POST /media/presign`, just with `folder` hard-coded:

| Alias                       | Folder         |
|-----------------------------|----------------|
| `POST /articles/upload`     | `articles`     |
| `POST /clubs/upload`        | `clubs`        |
| `POST /photos/upload`       | `photos`       |
| `POST /players/upload`      | `players`      |
| `POST /sponsors/upload`     | `sponsors`     |
| `POST /teams/upload`        | `teams`        |
| `POST /tournaments/upload`  | `tournaments`  |
| `POST /venues/upload`       | `venues`       |
| `POST /videos/upload`       | `videos`       |

On an alias, the backend MUST ignore any client-supplied `folder` field and use the alias default.

### Allowed folders

When calling the generic `POST /media/presign`, `folder` must be one of:

`articles`, `clubs`, `general`, `photos`, `players`, `sponsors`, `teams`, `tournaments`, `venues`, `videos`.

Anything else → `400`.

### Authentication

All endpoints in this document are **protected** — admin-only. Requests without a valid `x-access-token` JWT MUST be rejected with `401`. See [`AUTH_API.md`](./AUTH_API.md) for the token issuance flow.

---

## Endpoints

### `POST /media/presign`

```jsonc
Headers:
{ "x-access-token": "JWT" }

Request:
{
  "filename":    "sponsor-logo.png",   // used only to derive extension; never echoed into the key path
  "contentType": "image/png",          // MIME; whitelist enforced per folder
  "folder":      "sponsors"            // must be one of the allowed folders above; ignored on per-resource aliases
}

Response 200:
{
  "uploadUrl":      "https://...",                              // presigned S3 PUT URL, short-lived
  "method":         "PUT",                                      // always PUT
  "headers":        { "Content-Type": "image/png" },            // headers the client MUST send on the PUT
  "key":            "scoreboard/sponsors/images/2026/05/...",   // S3 object key
  "url":            "https://...",                              // final public / CDN URL to store on the resource
  "maxUploadBytes": 52428800,                                   // hard limit baked into the signature (50 MB default)
  "expiresIn":      300                                         // seconds the uploadUrl is valid
}

Response 400: { "error": "Invalid contentType" }
              | { "error": "Invalid folder" }
              | { "error": "Invalid filename" }
Response 401: { "error": "Unauthorized" }
```

### `DELETE /media`

```jsonc
Headers:
{ "x-access-token": "JWT" }

Request:
{ "key": "scoreboard/sponsors/images/2026/05/..." }

Response 204
Response 400: { "error": "Key outside scoreboard/ prefix" }
Response 401: { "error": "Unauthorized" }
```

The backend MUST validate that `key` is inside the `scoreboard/` prefix and reject anything else with `400`. This prevents an authenticated admin from accidentally (or maliciously) deleting arbitrary bucket objects via this endpoint.

### Per-resource aliases

Identical request/response shape to `POST /media/presign`, with `folder` ignored / hard-coded. Example:

```
POST /clubs/upload
Headers: { "x-access-token": "JWT" }
Body:    { "filename": "manchester-fc.png", "contentType": "image/png" }
```

The response uses `folder: "clubs"` and returns a `key` under `scoreboard/clubs/images/.../<uuid>.png`.

---

## Client flow

1. `POST` to `/media/presign` (or a per-resource alias) with `filename` + `contentType`.
2. `PUT` the raw `File` bytes to the returned `uploadUrl`, using exactly the headers returned in `headers`. The request goes browser → S3; the API server is not involved.
3. On S3 `2xx`, store the returned `url` on the resource field (`logoUrl`, `imageUrl`, `photoUrl`, `source`, …) and `POST` / `PUT` the resource as usual.

If the `PUT` fails (S3 rejects oversize / wrong content-type, expiry elapsed, network drop), the client SHOULD start over with a fresh presign — presigned URLs are single-use and short-lived.

---

## Backend responsibilities

The backend implementation MUST:

1. **Require auth.** Reject requests without a valid `x-access-token` JWT (`401`). Presign and delete are admin-only operations.
2. **Validate inputs:**
   - `filename`: non-empty string. Use only for deriving an extension — never echo it back into the key path verbatim. Strip directory separators and control characters.
   - `contentType`: non-empty MIME string. Whitelist per folder (e.g. `image/*` for `photos`, `image/*` + `video/*` for `videos`, `image/png|jpeg|webp|svg+xml` for logo folders). Reject anything else with `400`.
   - `folder`: must be one of the allowed values above. On per-resource aliases the backend ignores any client-supplied folder and uses the alias default.
3. **Generate a unique, opaque key.** Pattern: `scoreboard/<folder>/<images|videos>/YYYY/MM/<uuid>.<ext>`. Never trust the client filename for the key. Use the date for partitioning and a UUID for the basename.
4. **Sign with AWS SigV4** using a server-side IAM role/user whose only permissions are `s3:PutObject` (and `s3:DeleteObject` for the delete endpoint) on this bucket/prefix. Credentials never leave the server.
5. **Bake size + type into the signature.** Include `Content-Type` as a signed header and a `Content-Length` range condition equal to `maxUploadBytes`. This means S3 itself rejects oversize uploads or content-type swaps — the client cannot bypass it. Recommended defaults: images 10 MB, videos 200 MB.
6. **Set a short expiry.** 5 minutes (300 s) is plenty for option-1 single-PUT uploads. Return `expiresIn` so the client can show a sensible error.
7. **Return the canonical public URL.** If the bucket is served via CloudFront / CDN, `url` should be the CDN URL, not the raw `s3.amazonaws.com` URL. The S3 PUT URL (`uploadUrl`) and the public URL (`url`) are different hosts.

---

## S3 bucket requirements

- **Block public ACLs**, but allow public reads via bucket policy on the chosen prefix (or, preferably, keep the bucket private and serve via a CDN / signed reads).
- **CORS** must allow `PUT` from the admin and public origins, and expose `ETag`:
  ```json
  [{
    "AllowedOrigins": ["https://admin.example.com", "https://example.com"],
    "AllowedMethods": ["PUT", "GET", "HEAD"],
    "AllowedHeaders": ["*"],
    "ExposeHeaders":  ["ETag"],
    "MaxAgeSeconds":  3000
  }]
  ```
- Optionally enable a lifecycle rule that deletes unreferenced objects after N days, since a client may presign and then never PUT.

---

## Error format

All non-2xx responses use the project-wide error shape:

```json
{ "error": "human-readable message", "code": "OPTIONAL_MACHINE_CODE" }
```
