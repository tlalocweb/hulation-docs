# HTTP API reference

Hula exposes two distinct HTTP surfaces:

- **Visitor API** — public, unauthenticated. The `/v/*` endpoints visitors'
  browsers hit (driven by `hula.js`). Carry analytics events.
- **Admin API** — authenticated. The `/api/v1/*` endpoints `hulactl` and the
  admin UI hit. Generated from the gRPC services in
  [`pkg/apispec/v1/`](https://github.com/tlalocweb/hulation/tree/main/pkg/apispec/v1)
  via grpc-gateway.

Plus the WebSocket endpoints under `/chat/*` and `/api/v1/chat/*`.

## Authentication

| Caller | Mechanism | Header / state |
|--------|-----------|----------------|
| Visitor browser | none | — |
| `hulactl` (human operator) | OPAQUE-derived JWT | `Authorization: Bearer <jwt>` |
| `hula-agent` (CI runner) | mTLS client cert | TLS handshake (Subject CN = `agent:<id>`) |
| Same-host scripts (admin UI) | JWT cookie | `Authorization` or cookie |

The admin API enforces a permissions model expressed via
[`hula.auth.permission`](https://github.com/tlalocweb/hulation/blob/main/protoext/hula/auth/permission.proto)
annotations on each gRPC method. Permissions take the form
`server.{server_id}.{resource}.{action}` — e.g.,
`server.docs.build.trigger`, `server.docs.staging.build`. JWT users are
mapped to permissions through their per-server roles (`viewer`, `manager`,
or admin). Agents are mapped through their registry record's
`allow:<verb>` map.

## Conventions

- **Base URL.** `https://hula.example.com` (or whatever `hula_host:` resolves
  to). The unified listener handles both 80 and 443.
- **JSON.** Request and response bodies are JSON unless explicitly
  documented otherwise (e.g., WebDAV PUT bodies are file content).
- **Status codes.** `200` success with body, `204` success no body,
  `400` request validation error, `401` auth missing / invalid / expired,
  `403` auth valid but insufficient permission, `404` not found,
  `409` conflict (e.g. build already in progress), `422` semantic
  validation error, `500` server error, `503` dependency unavailable
  (typically ClickHouse during boot).
- **Pagination.** List endpoints accept `?limit=N` (default varies by
  endpoint, max 100). Cursor-based pagination is on the roadmap; today
  most listings are bounded by the registry's natural size.
- **Error shape.**
  ```json
  {
    "code": 7,
    "message": "permission denied",
    "details": []
  }
  ```
  `code` follows
  [google.rpc.Code](https://github.com/googleapis/googleapis/blob/master/google/rpc/code.proto).

---

## Visitor API

Public endpoints visitors' browsers hit via the dynamically-served
`hula.js` script. No authentication; consent-gated per server (see
[Consent & privacy](../concepts/consent-privacy.md)).

### `POST /v/hello`

The first event a new visitor's browser sends. Establishes (or refreshes)
the visitor's identity and records a page view.

```http
POST /v/hello?h=<server-id>&b=<bounce-id> HTTP/1.1
Host: example.com
Content-Type: application/json

{
  "u": "https://example.com/landing",
  "d": "{...page metadata...}",
  "consent": {
    "analytics": true,
    "marketing": false
  }
}
```

| Query param | Description |
|-------------|-------------|
| `h` | The server's `id` from `config.yaml`. Validated against the `Host:` header. |
| `b` | A short-lived bounce ID generated client-side. Hula uses it to correlate the iframe and direct-POST paths. |

| Body field | Description |
|------------|-------------|
| `b` | (alternative to query param) Bounce ID. |
| `e` | Event code (`int`). Used for sub-classification — most clients omit. |
| `u` | Current page URL (visitor-side). |
| `d` | Free-form page metadata blob (JSON-encoded string). |
| `consent.analytics` | bool — opt-in to analytics-purpose events. |
| `consent.marketing` | bool — opt-in to marketing-purpose events / forwarders. |

**Response.**

- `204 No Content` with `Hula-Consent-Required: 1` header when
  `consent_mode: opt_in` and consent isn't supplied.
- `200 OK` with no meaningful body otherwise (visitor JS doesn't read it).

**Headers honored:**

- `Sec-GPC: 1` — binding marketing opt-out at any consent mode.
- `CF-Connecting-IP`, `CF-IPCountry` — Cloudflare-trusted edge headers
  for IP / country.
- `X-Forwarded-For`, `X-Real-IP` — fallback IP source when not behind
  Cloudflare.

### `POST /v/landing`

Records a landing-page visit (driven by a configured `landers[]` entry).
Used when a campaign URL maps to a `lander` rather than a static file.

Request shape mirrors `/v/hello`.

### `POST /v/form`

Submit a form (e.g., newsletter signup). Driven by `forms[]` config + the
`forms.js` script.

```http
POST /v/form?h=<server-id>&form=<form-name> HTTP/1.1
Content-Type: application/json

{
  "email": "alice@example.com",
  "captcha_token": "...",
  "consent": { "marketing": true }
}
```

| Field | Description |
|-------|-------------|
| `form` (query) | The form's `name` from `config.yaml`. |
| Body | Validated against the form's JSON Schema. Captcha token added per the form's `captcha:` setting. |

**Response:** `200` with `feedback` template rendered, or `400` /  `422` on
validation failure. Triggers `on_new_form_submission` Risor hooks
asynchronously.

### `POST /v/conversion`

Records a conversion event (typically fired after a form submit or a
post-purchase callback).

```http
POST /v/conversion?h=<server-id> HTTP/1.1
Content-Type: application/json

{
  "name": "purchase",
  "value": 49.99,
  "currency": "USD"
}
```

### `POST /v/goal`

Records a goal hit (defined per-server via `hulactl goals`).

### Iframe-mode endpoints

When `hula.js` is loaded as an iframe (`tracking_mode: cookieless` does
not require iframe; this is for embedded-script fallback):

| Endpoint | Description |
|----------|-------------|
| `GET /hula.js` | Dynamically-built visitor script. Filename overridable via `hello_script_filename`. |
| `GET /forms.js` | Forms submit helper. Override via `forms_script_filename`. |
| `GET /hula_hello.html` | Iframe HTML wrapper. |
| `GET /hulans.html` | Iframe noscript fallback. |

The `.js` filenames can be overridden per-deployment to dodge ad-blocker
heuristics — see the [config reference](config.md#top-level-keys).

---

## Admin API

The full surface is generated from the proto files in
[`pkg/apispec/v1/`](https://github.com/tlalocweb/hulation/tree/main/pkg/apispec/v1).
Each section below names the service, its proto file (the source of
truth), and the REST endpoints it exposes.

For exhaustive request / response schemas, read the proto files
directly — they're concise.

### Site builds — `SiteService`

[`pkg/apispec/v1/site/site.proto`](https://github.com/tlalocweb/hulation/blob/main/pkg/apispec/v1/site/site.proto)

| Method | REST | Permission |
|--------|------|------------|
| `TriggerBuild` | `POST /api/v1/site/{server_id}/build` | `server.{server_id}.build.trigger` |
| `GetBuildStatus` | `GET /api/v1/site/builds/{build_id}` | `server.{server_id}.build.read` |
| `ListBuilds` | `GET /api/v1/site/{server_id}/builds` | `server.{server_id}.build.list` |

`TriggerBuild` body:

```json
{
  "branch": "main",
  "commit": ""
}
```

Empty `branch` uses the server's configured default.

### Staging — `StagingService`

[`pkg/apispec/v1/staging/staging.proto`](https://github.com/tlalocweb/hulation/blob/main/pkg/apispec/v1/staging/staging.proto)

gRPC-gateway methods:

| Method | REST | Permission |
|--------|------|------------|
| `StagingBuild` | `POST /api/v1/staging/{server_id}/build` | `server.{server_id}.staging.build` |

WebDAV-style file operations are intentionally **not** part of the
gRPC service — they're served by the unified server's fallback mux:

| HTTP | Path | Purpose |
|------|------|---------|
| `GET` | `/api/staging/{server_id}/dav/<remote-path>` | Download file. |
| `PUT` | `/api/staging/{server_id}/dav/<remote-path>` | Upload file (full replace). |
| `PATCH` | `/api/staging/{server_id}/dav/<remote-path>` | Partial update via `X-Update-Range:` or `X-Patch-Format: diff`. |

Plus the git-aware operations driven by `hulactl stage`/`commit`/`push`/
`pull`/`sync`:

| HTTP | Path | Purpose |
|------|------|---------|
| `POST` | `/api/staging/{id}/git/stage` | `git add` paths. Body: paths, one per line. |
| `POST` | `/api/staging/{id}/git/commit` | Commit staged. Body: `{"message": "..."}`. |
| `POST` | `/api/staging/{id}/git/push` | Push HEAD. |
| `POST` | `/api/staging/{id}/git/pull` | Pull + rebase. |
| `POST` | `/api/staging/{id}/git/sync` | Pull + push. |

### Forms — `FormsService`

[`pkg/apispec/v1/forms/forms.proto`](https://github.com/tlalocweb/hulation/blob/main/pkg/apispec/v1/forms/forms.proto)

CRUD over the form definitions in `servers[].forms[]`.

| Method | REST |
|--------|------|
| `CreateForm` | `POST /api/v1/forms` |
| `ListForms` | `GET /api/v1/forms` |
| `GetForm` | `GET /api/v1/forms/{id}` |
| `UpdateForm` | `PATCH /api/v1/forms/{id}` |
| `DeleteForm` | `DELETE /api/v1/forms/{id}` |

### Landers — `LandersService`

Same shape as Forms — see
[`pkg/apispec/v1/landers/landers.proto`](https://github.com/tlalocweb/hulation/blob/main/pkg/apispec/v1/landers/landers.proto).

### Auth — `AuthService`

[`pkg/apispec/v1/auth/auth.proto`](https://github.com/tlalocweb/hulation/blob/main/pkg/apispec/v1/auth/auth.proto)

OPAQUE PAKE handshake, JWT lifecycle, OIDC callback, TOTP enroll /
verify. The OPAQUE flow is two round-trips:

1. `POST /api/v1/auth/opaque/init` — client sends ephemeral; server
   responds with envelope material.
2. `POST /api/v1/auth/opaque/finish` — client sends finishing material;
   server responds with JWT (and a TOTP challenge if 2FA is enabled).

Plus:

| Method | REST | Notes |
|--------|------|-------|
| OIDC callback | `GET /api/v1/auth/callback/{provider}` | Per-provider OAuth2 callback. |
| TOTP enroll | `POST /api/v1/auth/totp/enroll` | — |
| TOTP verify | `POST /api/v1/auth/totp/verify` | — |
| Logout | `POST /api/v1/auth/logout` | Invalidates the JWT server-side (when JWT-revocation is enabled). |

### Bad actors — `BadActorService`

[`pkg/apispec/v1/badactor/badactor.proto`](https://github.com/tlalocweb/hulation/blob/main/pkg/apispec/v1/badactor/badactor.proto)

| Method | REST |
|--------|------|
| `ListBadActors` | `GET /api/v1/badactors` |
| `GetBadActor` | `GET /api/v1/badactors/{ip}` |
| `Allowlist` | `POST /api/v1/badactors/allow` |

### Analytics — `AnalyticsService`

[`pkg/apispec/v1/analytics/analytics.proto`](https://github.com/tlalocweb/hulation/blob/main/pkg/apispec/v1/analytics/analytics.proto)

Read-only queries against ClickHouse. Most endpoints accept a `?format=csv`
override to return CSV instead of JSON.

| Method | REST | Notes |
|--------|------|-------|
| `Visitors` | `GET /api/v1/analytics/visitors` | Per-server, per-day visitor counts. |
| `Events` | `GET /api/v1/analytics/events` | Filter by event type, date range. |
| `Goals` | `GET /api/v1/analytics/goals` | Goal hit / value rollups. |
| `Forms` | `GET /api/v1/analytics/forms` | Form submission counts. |
| `Funnels` | `GET /api/v1/analytics/funnels` | Goal-to-conversion funnels. |

### Goals — `GoalsService`

[`pkg/apispec/v1/goals/goals.proto`](https://github.com/tlalocweb/hulation/blob/main/pkg/apispec/v1/goals/goals.proto)

CRUD over goals (matching predicates against analytics events).

### Reports — `ReportsService`

[`pkg/apispec/v1/reports/reports.proto`](https://github.com/tlalocweb/hulation/blob/main/pkg/apispec/v1/reports/reports.proto)

Scheduled report definitions, ad-hoc render endpoint.

### Alerts — `AlertsService`

[`pkg/apispec/v1/alerts/alerts.proto`](https://github.com/tlalocweb/hulation/blob/main/pkg/apispec/v1/alerts/alerts.proto)

Operator alert rules: badactor block thresholds, build-failure alerts,
goal anomaly alerts.

### Mobile — `MobileService`

[`pkg/apispec/v1/mobile/mobile.proto`](https://github.com/tlalocweb/hulation/blob/main/pkg/apispec/v1/mobile/mobile.proto)

Device registration for the operator iOS / Android push apps.

### Notify — `NotifyService`

[`pkg/apispec/v1/notify/notify.proto`](https://github.com/tlalocweb/hulation/blob/main/pkg/apispec/v1/notify/notify.proto)

Push channel registration, test-send endpoints.

### Status — `StatusService`

[`pkg/apispec/v1/status/status.proto`](https://github.com/tlalocweb/hulation/blob/main/pkg/apispec/v1/status/status.proto)

| Method | REST |
|--------|------|
| `Status` | `GET /api/v1/status` |
| `Healthz` | `GET /healthz` |
| `Readyz` | `GET /readyz` |

`/healthz` and `/readyz` are unauthenticated — designed for load-balancer
probes.

### Agents — `/api/v1/agent/*`

Admin-side cert minting and registry management.

| HTTP | Path | Description |
|------|------|-------------|
| `POST` | `/api/v1/agent/create` | Mint and register an agent cert. Powers `hulactl create-agent`. |
| `GET` | `/api/v1/agent/list` | List active agents. Powers `hulactl list-agents` (Phase 6). |
| `POST` | `/api/v1/agent/{id}/revoke` | Revoke an agent. Powers `hulactl revoke-agent` (Phase 6). |

See [Agents & HLAP](../concepts/agents-hlap.md) for the trust model.

### Membership / HA — `MembershipService`

[`pkg/apispec/v1/membership/membership.proto`](https://github.com/tlalocweb/hulation/blob/main/pkg/apispec/v1/membership/membership.proto)

Multi-node Raft cluster join / leave / status. mTLS-only, signed by the
Team CA — distinct trust domain from the Agent CA.

---

## Chat WebSockets

| HTTP | Path | Purpose |
|------|------|---------|
| `POST` | `/api/v1/chat/start` | Visitor starts a chat session. Returns a session token. |
| `GET` | `/api/v1/chat/ws?token=<sess>` | Visitor WebSocket. Token is the value from `/start`. |
| `GET` | `/api/v1/chat/admin/agent-ws` | Operator agent WebSocket. JWT-authenticated. |
| `GET` | `/api/v1/chat/admin/agent-control-ws` | Operator control channel (joining sessions, sending presence). |

Visitor-side WS is gated by the chat-session token signed at `/start`.
Operator-side WS is JWT-authenticated. See [Live chat](../concepts/live-chat.md).

---

## Postman collection

A maintained Postman collection covering the most-used admin endpoints
lives at:

[https://www.getpostman.com/collections/0e83876e0f2a0c8ecd70](https://www.getpostman.com/collections/0e83876e0f2a0c8ecd70)

Useful for poking at the API while writing client code. The collection
expects a JWT bearer token; run `hulactl auth` first and copy the token
out of `~/.config/hulactl/hulactl.yaml`.

---

## Versioning

The admin API lives under `/api/v1/`. Within v1, additive changes
(new fields, new methods) ship in normal releases. Breaking changes
(removed fields, changed semantics) require a `/api/v2/` parallel
surface; v1 stays available through the deprecation window.

The visitor API (`/v/*`) follows the same shape, but the bar for
breaking changes is much higher — embedded `hula.js` snippets across
the live web don't get redeployed. Plan for v1 to live indefinitely.

## Where to go next

- [Configuration reference](config.md) — what each `server[]` config key
  changes about the admin / visitor APIs.
- [`hulactl` reference](hulactl.md) — the CLI that drives the admin API.
- [Hooks (Risor) reference](hooks.md) — the visitor APIs trigger Risor
  hooks; this page covers the hook ABI.
- [HLAP reference](hlap.md) — the agent-side socket protocol that wraps
  the admin API for CI runners.
