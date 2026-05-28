---
name: spaitial-api
description: Generate 3D worlds programmatically via the Spaitial v1 HTTP API. Use when integrating with `api.spaitial.ai`, the SpAItial developer API, or when the user mentions API keys (`spt_live_…` / `spt_test_…`), `/v1/worlds`, splat (`.spz`) generation, panorama generation, world request IDs (`req_…`), file IDs (`file_…`), webhook delivery for world generation, or building an SDK / integration on top of Spaitial.
license: MIT
---

# LLM Skills

A self-contained, single-page reference designed to be loaded as a skill by AI coding assistants (Cursor, Claude Code, Cline, etc.). Drop this URL (or its `.md` companion) into a prompt or skill manifest and an agent can integrate end-to-end without browsing the rest of the docs.

If you're a human reading this, the [Getting Started](https://docs.spaitial.ai/api/getting-started.md) page is friendlier. The content below intentionally repeats schema, examples, and edge cases that are split across multiple pages elsewhere, so it stays useful on its own.

## Base URL & Auth

```
https://api.spaitial.ai
```

Every request needs a Bearer token issued by the developers site (`https://developers.spaitial.ai`):

```
Authorization: Bearer spt_live_<32-char-base32>
```

Test keys use the `spt_test_` prefix (same scopes, lower rate limits, no GPU billing).

Required scopes per endpoint:

| Endpoint                                                           | Scope           |
| ------------------------------------------------------------------ | --------------- |
| `POST /v1/worlds`                                                  | `worlds:create` |
| `GET /v1/worlds/requests`                                          | `worlds:read`   |
| `GET /v1/worlds/requests/:id` (+ `/status`, `/splat`, `/panorama`) | `worlds:read`   |
| `GET /v1/worlds/requests/:id/exports` (+ `/:type`)                 | `worlds:read`   |
| `PATCH /v1/worlds/requests/:id`                                    | `worlds:write`  |
| `POST /v1/worlds/requests/:id/cancel`                              | `worlds:write`  |
| `POST /v1/worlds/requests/:id/exports/:type`                       | `worlds:write`  |
| `POST /v1/files`                                                   | `files:create`  |
| `GET /v1/files`                                                    | `files:read`    |
| `GET /v1/models`                                                   | `worlds:read`   |

## Endpoints at a glance

```
POST   /v1/worlds                                       Create a world generation job
GET    /v1/worlds/requests                              List jobs created by this API key
GET    /v1/worlds/requests/:request_id                  Full job result (post-completion)
GET    /v1/worlds/requests/:request_id/status           Poll status (cheap, cached 3s)
PATCH  /v1/worlds/requests/:request_id                  Update world visibility/title
POST   /v1/worlds/requests/:request_id/cancel           Best-effort cancel
GET    /v1/worlds/requests/:request_id/splat            302 → fresh signed splat URL
GET    /v1/worlds/requests/:request_id/panorama         302 → fresh signed panorama URL
POST   /v1/worlds/requests/:request_id/exports/:type    Start an export, e.g. mesh
GET    /v1/worlds/requests/:request_id/exports/:type    Export status; READY includes download_url
GET    /v1/worlds/requests/:request_id/exports          List export statuses
POST   /v1/files                                        Upload an input file, returns file_id
GET    /v1/files                                        List uploaded files for this API key
GET    /v1/models                                       List available generation models
GET    /v1/openapi.json                                 Machine-readable spec
GET    /v1/docs                                         Swagger UI
```

## Quickstart: submit + poll

```bash
API_KEY="spt_live_..."

# 1. Submit
RES=$(curl -sX POST https://api.spaitial.ai/v1/worlds \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "input": {
      "type": "url",
      "image_url": "https://images.unsplash.com/photo-1600596542815-ffad4c1539a9?w=800"
    },
    "title": "Cozy reading nook"
  }')
REQ_ID=$(echo "$RES" | jq -r .request_id)

# 2. Poll
while true; do
  STATUS=$(curl -s "https://api.spaitial.ai/v1/worlds/requests/$REQ_ID/status" \
    -H "Authorization: Bearer $API_KEY" | jq -r .status)
  echo "status=$STATUS"
  [[ "$STATUS" == "COMPLETED" || "$STATUS" == "FAILED" || "$STATUS" == "CANCELLED" ]] && break
  sleep 5
done

# 3. Fetch result
curl -s "https://api.spaitial.ai/v1/worlds/requests/$REQ_ID" \
  -H "Authorization: Bearer $API_KEY" | jq

# 4. Download splat (302 → signed URL, follow with -L)
curl -L -o world.spz "https://api.spaitial.ai/v1/worlds/requests/$REQ_ID/splat" \
  -H "Authorization: Bearer $API_KEY"
```

Generation takes 5–10 minutes per world.

## Input types

`POST /v1/worlds` accepts a discriminated `input.type`:

### `url` — external HTTPS image

```json
{ "input": { "type": "url", "image_url": "https://example.com/photo.jpg" } }
```

Server fetches under SSRF guards. HTTPS only, ≤25 MB, JPEG/PNG/WebP/GIF.

For a 360 panorama image, keep the same input shape and add `is_pano: true`:

```json
{
  "input": {
    "type": "url",
    "image_url": "https://example.com/panorama.jpg",
    "is_pano": true
  }
}
```

The API trusts that the image is an equirectangular 360 panorama, starts generation after the image-to-panorama stage, and skips suitability validation even if `validation.skip` is `false`. Content moderation still applies.

### `base64` — inline data URI

```json
{
  "input": {
    "type": "base64",
    "image_base64": "data:image/png;base64,iVBORw0KGgo..."
  }
}
```

≤25 MB after decode.

### `file_id` — upload first, reference later

```bash
# Step 1: upload
FILE=$(curl -sX POST https://api.spaitial.ai/v1/files \
  -H "Authorization: Bearer $API_KEY" \
  -F "file=@photo.jpg" | jq -r .file_id)

# Step 2: submit
curl -sX POST https://api.spaitial.ai/v1/worlds \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/json" \
  -d "{\"input\":{\"type\":\"file_id\",\"file_id\":\"$FILE\"}}"
```

Uploads live in a private bucket with a 24-hour TTL. Owner-scoped — another caller's `file_id` returns 404. Using a `file_id` marks it as consumed; `GET /v1/files` returns `available`, `consumed`, and `expired` statuses.

Uploaded panoramas use the same file flow:

```bash
PANO_FILE=$(curl -sX POST https://api.spaitial.ai/v1/files \
  -H "Authorization: Bearer $API_KEY" \
  -F "file=@panorama.jpg" | jq -r .file_id)

curl -sX POST https://api.spaitial.ai/v1/worlds \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/json" \
  -d "{\"input\":{\"type\":\"file_id\",\"file_id\":\"$PANO_FILE\",\"is_pano\":true}}"
```

List existing uploads for the current API key:

```bash
curl -s "https://api.spaitial.ai/v1/files?limit=20&offset=0" \
  -H "Authorization: Bearer $API_KEY"
```

The response includes `status` (`available`, `consumed`, or `expired`) plus `expires_at` and `consumed_at`; it never exposes private storage keys.

### `text` — prompt → image → world

```json
{
  "input": {
    "type": "text",
    "prompt": "a cozy sunlit reading nook with bookshelves"
  }
}
```

Charged for both prompt-to-image and world generation.

## Full request shape

```json
{
  "input": { "type": "url", "image_url": "https://..." },
  "model": "default",
  "title": "My world",
  "validation": { "skip": true, "error_on_fail": false },
  "visibility": { "is_public": false, "is_listed": false },
  "webhook": { "url": "https://example.com/hooks/spaitial" }
}
```

| Field                      | Default                     | Notes                                                                                |
| -------------------------- | --------------------------- | ------------------------------------------------------------------------------------ |
| `model`                    | server default for the user | `GET /v1/models` to list                                                             |
| `title`                    | unset                       | User-facing world caption (≤200 chars)                                               |
| `validation.skip`          | `true`                      | Skip suitability check (saves cost + latency)                                        |
| `validation.error_on_fail` | `false`                     | When `skip:false`, reject (422) on flagged input instead of proceeding with warnings |
| `visibility.is_public`     | `false`                     | Anyone with `viewer_url` can view                                                    |
| `visibility.is_listed`     | `false`                     | Eligible for public gallery (requires `is_public:true`)                              |
| `webhook.url`              | unset                       | HTTPS callback on terminal state                                                     |

## Statuses

```
PENDING → PROCESSING → COMPLETED
                    ↘ FAILED
                    ↘ CANCELLED
```

`progress` (0-1) is coarse; reflects the current pipeline stage.

## Result envelope (`GET /v1/worlds/requests/:id`)

```json
{
  "request_id": "req_...",
  "model": "default",
  "status": "COMPLETED",
  "created_at": "2026-05-14T12:00:00Z",
  "updated_at": "2026-05-14T12:09:48Z",
  "completed_at": "2026-05-14T12:09:48Z",
  "world": {
    "id": "<world-uuid>",
    "title": "Cozy reading nook",
    "splat_url": "https://api.spaitial.ai/v1/worlds/requests/req_.../splat",
    "splat_format": "spz",
    "thumbnail_url": "https://img.spaitial.ai/.../thumbnail.webp",
    "panorama_url": "https://api.spaitial.ai/v1/worlds/requests/req_.../panorama",
    "viewer_url": "https://app.spaitial.ai/worlds/<world-uuid>",
    "visibility": { "is_public": false, "is_listed": false },
    "created_at": "2026-05-14T12:00:01Z",
    "updated_at": "2026-05-14T12:09:48Z",
    "completed_at": "2026-05-14T12:09:48Z"
  },
  "validation": { "passed": true, "issues": [] },
  "input": {
    /* echoed request */
  }
}
```

`splat_url` and `panorama_url` are **stable API endpoints**, not signed URLs. Each `GET` to them returns a `302 Found` with a fresh 5-minute signed URL — safe to store the API URL in your DB forever. Auth on every download.

## Exports

Exports are optional artifacts derived from a completed world. They are keyed by `type` so the API can grow beyond mesh without changing the route shape.

Supported export types today:

| Type              | Description                                 |
| ----------------- | ------------------------------------------- |
| `mesh`            | Full-resolution reconstructed mesh (`.ply`) |
| `mesh-simplified` | Simplified mesh optimized for real-time use |

Start or retrieve an export:

```bash
curl -sX POST "https://api.spaitial.ai/v1/worlds/requests/$REQ_ID/exports/mesh" \
  -H "Authorization: Bearer $API_KEY" | jq
```

Poll export status using the same typed endpoint:

```bash
curl -s "https://api.spaitial.ai/v1/worlds/requests/$REQ_ID/exports/mesh" \
  -H "Authorization: Bearer $API_KEY" | jq
```

Not ready:

```json
{
  "type": "mesh",
  "status": "PROCESSING"
}
```

Ready:

```json
{
  "type": "mesh",
  "status": "READY",
  "download_url": "https://api.spaitial.ai/v1/worlds/requests/req_.../exports/mesh?download=1",
  "created_at": "2026-05-21T09:00:00Z",
  "updated_at": "2026-05-21T09:00:00Z"
}
```

`download_url` is a stable API proxy endpoint. Calling it redirects to a short-lived signed file URL, so store the API URL and fetch a fresh redirect when needed. Requesting either mesh type starts the shared mesh pipeline; both `mesh` and `mesh-simplified` become ready when processing completes.

## Idempotency

Send the same `Idempotency-Key` header to safely retry a POST without double-charging:

```bash
KEY=$(uuidgen)
curl -X POST https://api.spaitial.ai/v1/worlds \
  -H "Authorization: Bearer $API_KEY" \
  -H "Idempotency-Key: $KEY" \
  -d '{...}'
```

- Same key + same body → cached 202 response (no new job)
- Same key + different body → `409 IDEMPOTENCY_KEY_REUSED`
- Keys are not retry tokens; to retry a `FAILED` job, submit a fresh POST with a new key (or no key).

## Webhooks

Set `webhook.url` on the POST to receive a callback on terminal state.

### Headers

```
Content-Type: application/json
User-Agent: SpaitialWebhook/1.0
X-Spaitial-Event: world.completed | world.failed | world.cancelled | world.export.completed | world.export.failed
X-Spaitial-Request-ID: req_...
X-Spaitial-Delivery-ID: wd_...
X-Spaitial-Delivery-Attempt: 1
X-Spaitial-Signature: sha256=<hmac-sha256(body, webhook_secret)>
```

Verify with the `webhook_secret` from your API key's settings page.

### Payload (flat envelope)

```json
{
  "event": "world.completed",
  "delivery_id": "wd_<uuid>",
  "timestamp": "2026-05-15T10:00:02Z",
  "request_id": "req_...",
  "status": "COMPLETED",
  "created_at": "2026-05-14T12:00:00Z",
  "updated_at": "2026-05-14T12:09:48Z",
  "completed_at": "2026-05-14T12:09:48Z",
  "validation": { "passed": true, "issues": [] },
  "data": {
    /* same shape as `world` in the result envelope */
  }
}
```

For `world.failed`: `data` is `null` + top-level `error: { code, message }`. For `world.cancelled`: `data` is `null`, no `error`.

Export webhooks use the same envelope with `data: null` and an `export` block:

```json
{
  "event": "world.export.completed",
  "delivery_id": "wd_<uuid>",
  "timestamp": "2026-05-15T10:00:02Z",
  "request_id": "req_...",
  "status": "COMPLETED",
  "created_at": "2026-05-14T12:00:00Z",
  "updated_at": "2026-05-14T12:09:48Z",
  "completed_at": "2026-05-14T12:09:48Z",
  "data": null,
  "export": {
    "type": "mesh",
    "status": "READY"
  }
}
```

For `world.export.failed`, `export.status` is `FAILED` and both the top-level `error` and `export.error` include `{ code, message }`.

### Delivery semantics

- HTTPS only. Private/loopback hostnames rejected (SSRF).
- 30s timeout per attempt.
- Up to 5 retries with backoff (≈10s / 60s / 600s / …) on non-2xx or timeout.
- Idempotent on `X-Spaitial-Delivery-ID` — same delivery may arrive twice; dedupe on this header.

### Signature verification (Node example)

```js
import { createHmac } from "crypto";
function verify(rawBody, signature, secret) {
  const expected =
    "sha256=" + createHmac("sha256", secret).update(rawBody).digest("hex");
  return signature === expected;
}
```

## Cancellation

```bash
curl -X POST "https://api.spaitial.ai/v1/worlds/requests/$REQ_ID/cancel" \
  -H "Authorization: Bearer $API_KEY"
# → { "success": true }   when intent was recorded
# → { "success": false }  when the job was already terminal
```

Cancel is best-effort. The API reports `CANCELLED` immediately; in-flight processing stops within seconds.

## Updating a World

Update visibility or title of a completed world:

```bash
curl -X PATCH "https://api.spaitial.ai/v1/worlds/requests/$REQ_ID" \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "title": "My updated world",
    "visibility": {
      "is_public": true,
      "is_listed": true
    }
  }'
```

All fields are optional, but at least one must be provided:

| Field                  | Type                | Description                                              |
| ---------------------- | ------------------- | -------------------------------------------------------- |
| `title`                | string (≤200 chars) | User-facing world caption                                |
| `visibility.is_public` | boolean             | Anyone with `viewer_url` can view                        |
| `visibility.is_listed` | boolean             | Eligible for public gallery (requires `is_public: true`) |

Returns the updated world object on success. Returns `409 RESOURCE_NOT_READY` if the world is not yet completed.

## Error envelope

Every non-2xx response:

```json
{
  "error": {
    "code": "VALIDATION_FAILED",
    "message": "Human-readable reason",
    "details": {
      /* optional structured context */
    }
  }
}
```

Stable codes:

| Code                     | HTTP | Meaning                                                       |
| ------------------------ | ---- | ------------------------------------------------------------- |
| `UNAUTHORIZED`           | 401  | Missing / invalid API key                                     |
| `FORBIDDEN`              | 403  | API key lacks required scope                                  |
| `MODEL_NOT_FOUND`        | 400  | Unknown `model`                                               |
| `MODEL_FORBIDDEN`        | 403  | Restricted model                                              |
| `MODEL_UNAVAILABLE`      | 503  | Model deployment unhealthy                                    |
| `INVALID_INPUT`          | 400  | Malformed body / unsupported input                            |
| `INSUFFICIENT_CREDITS`   | 402  | Top up at the developers site                                 |
| `MODERATION_REJECTED`    | 403  | Content moderation blocked the input                          |
| `VALIDATION_FAILED`      | 422  | Suitability check rejected (with `details.validation.issues`) |
| `FILE_NOT_FOUND`         | 404  | `file_id` unknown or not owned by caller                      |
| `FILE_EXPIRED`           | 404  | `file_id` older than 24 hours or already consumed             |
| `REQUEST_NOT_FOUND`      | 404  | Unknown `request_id` or not owned by caller                   |
| `RESOURCE_NOT_READY`     | 409  | World not yet `COMPLETED` for artifact/export operations       |
| `IDEMPOTENCY_KEY_REUSED` | 409  | Same key used with different body                             |
| `RATE_LIMIT_EXCEEDED`    | 429  | Back off; check `Retry-After` + `X-RateLimit-*`               |
| `INTERNAL_ERROR`         | 500  | Retry with backoff                                            |

## Rate limits

Each response carries:

```
X-RateLimit-Limit:     60
X-RateLimit-Remaining: 58
X-RateLimit-Reset:     1778836080
```

Defaults per key:

| Bucket            | Routes                      | Limit   |
| ----------------- | --------------------------- | ------- |
| `v1-world-create` | `POST /v1/worlds`           | 10/min  |
| `v1-status`       | `GET /…/status`             | 300/min |
| `v1-download`     | `GET /…/splat`, `/panorama` | 120/min |
| `v1-files`        | `POST /v1/files`            | 20/min  |
| `v1-default`      | everything else             | 120/min |

`429 RATE_LIMIT_EXCEEDED` includes `Retry-After` (seconds).

## Models

```bash
curl https://api.spaitial.ai/v1/models -H "Authorization: Bearer $API_KEY"
```

```json
{
  "models": [
    { "id": "default", "description": "Standard pipeline", "is_default": true },
    {
      "id": "experimental",
      "description": "Latest in-development",
      "is_default": false
    }
  ]
}
```

Pass `model: "<id>"` on `POST /v1/worlds`. Omit to use the server-side default for your account.

## Conventions worth knowing

- IDs are opaque UUIDs with type prefixes: `req_`, `file_`, `wd_` (delivery). World IDs are returned as raw UUIDs (in `world.id`).
- Times are ISO-8601 UTC. `completed_at` is `null` until terminal.
- `world` is the **artifact**; `request` is the **operation**. They have different IDs.
- `validation` is advisory by default. Issues are surfaced as warnings on the world unless you opt into `error_on_fail: true`.
- Submitting the same body with a new `Idempotency-Key` makes a fresh job. Reuse the same key only for genuine network retries.
- Splats and panoramas are served from a private bucket. Always go through `/v1/worlds/requests/:id/splat` or `/panorama`.
- Export `download_url` values are backend proxy URLs returned by `GET /v1/worlds/requests/:id/exports/:type`; calling one redirects to a short-lived signed file URL.

## Reference

- Spec: `https://api.spaitial.ai/v1/openapi.json`
- Swagger UI: `https://api.spaitial.ai/v1/docs`
- Developers portal (keys, usage, webhooks): `https://developers.spaitial.ai`
