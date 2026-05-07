# `config.yaml` reference

Every key Hula reads from `config.yaml`. Sections mirror the top-level YAML
keys in the order they appear in
[`config/config.go`](https://github.com/tlalocweb/hulation/blob/main/config/config.go).
The struct tags are the source of truth — when this page disagrees with the
code, the code wins.

!!! tip "Environment-variable substitution"

    Any string value in `config.yaml` may include `${ENV_VAR}` references that
    Hula resolves at config-load time. Used heavily for secrets:
    `password: ${SMTP_PASSWORD}`. The variables themselves come from the Hula
    process's environment — set them in `start-with-docker.sh`, your systemd
    unit, or your container orchestrator.

!!! tip "Per-field environment overrides"

    Some fields also accept a dedicated env var that takes precedence over the
    YAML value (see the `env:` column in each table). Example:
    `HULA_HOST=hula.example.com` overrides whatever `hula_host:` says.

## Top-level keys

| Key | Type | Default | Env | Description |
|-----|------|---------|-----|-------------|
| `admin` | object | — | — | Legacy admin password hash. New installs use OPAQUE — see [`hulactl set-password`](hulactl.md). Omit for OPAQUE-only auth. |
| `port` | int | `8080` | `APP_PORT` | Default port the unified listener binds to when a `servers[].port` isn't set. Production: `443`. |
| `publish_port` | bool | `false` | — | Include the port in the URLs published in the generated `hula.js` script. Set when running on a non-standard port that's externally visible. |
| `external_publish_port` | int | — | — | Override the published port (for reverse-proxy setups where the external port differs from the internal one). |
| `external_http_scheme` | string | `https` | `HULA_EXTERNAL_HTTP_SCHEME` | Scheme used in published URLs. Set to `https` when behind a TLS-terminating reverse proxy. |
| `dbconfig` | object | — | `DB_*` | ClickHouse connection. See [`dbconfig`](#dbconfig). |
| `servers` | list | — | — | Virtual servers — one entry per host. See [`servers`](#servers). |
| `cors` | object | — | — | Default CORS policy applied across all servers. Per-server overrides via `servers[].cors`. |
| `ssl` | object | — | — | Default TLS config inherited by servers without their own `ssl:` block. See [`ssl`](#ssl). |
| `hula_ssl` | object | — | — | Separate TLS config for Hula's own admin / API endpoints (covers `localhost`, `127.0.0.1`, `hula_host`). |
| `registries` | map | — | — | Container-registry credentials for pulling backend images. Keyed by registry hostname. |
| `bad_actors` | object | — | — | IP-scoring config. See [Bot & abuse defense](../concepts/badactor.md). |
| `backend_logs` | object | (passthrough on) | — | Default for backend-container log passthrough. Per-backend `logs:` blocks override. |
| `log_tags` | string | — | — | Comma-separated list of log tags to include (only these tags log). Inverse of `no_log_tags`. |
| `no_log_tags` | string | — | — | Comma-separated list of log tags to exclude. |
| `proxies` | list | — | — | Top-level reverse-proxy targets. Lower priority than per-server backends. |
| `jwt_key` | string | random | — | HMAC-SHA256 key for signing operator JWTs. **Set explicitly in production** — random regeneration invalidates all sessions across restarts. |
| `totp_encryption_key` | string | — | `HULA_TOTP_ENCRYPTION_KEY` | Base64url-encoded 32-byte key for at-rest TOTP secret encryption. Generate with `hulactl totp-key`. |
| `totp_issuer` | string | `Hulation` | — | Issuer name shown in authenticator apps. |
| `auth` | object | — | — | Auth provider list (OIDC SSO + internal). See [`auth`](#auth). |
| `analytics` | object | — | — | ClickHouse-side tunables. |
| `chat` | object | — | — | Visitor-chat config. See [Live chat](../concepts/live-chat.md). |
| `mailer` | object | — | — | SMTP config for scheduled report dispatch. See [`mailer`](#mailer). |
| `apns` | object | — | `HULA_APNS_*` | Apple Push Notification creds. See [`apns`](#apns-fcm). |
| `fcm` | object | — | `HULA_FCM_*` | Firebase Cloud Messaging creds. |
| `opaque` | object | random | `HULA_OPAQUE_*` | OPAQUE PAKE seed + AKE secret. **Pin these** — changing invalidates every operator credential. |
| `team` | object | auto | — | Raft cluster identity. Solo installs auto-bootstrap. See [High availability](../concepts/ha.md). |
| `jwt_expiration` | duration | `72h` | — | Operator JWT lifetime. |
| `hula_host` | string | `localhost` | `HULA_HOST` | The hostname Hula publishes in cookie domains and CORS headers for its own admin / API endpoints. |
| `hostname` | string | — | — | Alias for `hula_host` (compat with adopted izcr code). |
| `hula_aliases` | list | — | — | Other hostnames the Host header can carry for the admin / API endpoint. |
| `hula_domain` | string | derived | — | Override the cookie domain Hula uses. Rare. |
| `listen_on` | string | `0.0.0.0` | — | Listen address or interface name for the unified listener. |
| `hello_script_filename` | string | `hula.js` | `PUBLISHED_HELLO_SCRIPT_FILENAME` | Filename of the dynamically-served visitor JS. Change to dodge ad blockers. |
| `forms_script_filename` | string | `forms.js` | `PUBLISHED_FORMS_SCRIPT_FILENAME` | Filename of the forms JS. |

---

## `admin`

Legacy admin password hash. The first thing on a fresh install is `hulactl
set-password`, which writes the OPAQUE record to Bolt — the `admin:` block
only matters if you're upgrading from a pre-OPAQUE deployment.

```yaml
admin:
  hash: "$argon2id$v=19$m=16384,t=12,p=4$..."
```

Generate the hash with `hulactl generatehash`. New installs should leave the
block empty and use `hulactl set-password` instead.

---

## `dbconfig`

ClickHouse connection. Every field has an env-var override.

| Field | Type | Default | Env | Description |
|-------|------|---------|-----|-------------|
| `host` | string | `localhost` | `DB_HOST` | ClickHouse host. In Docker compose: `hula-clickhouse`. |
| `port` | int | `9000` | `DB_PORT` | TCP port (native protocol). HTTP port `8123` is unused. |
| `user` | string | `hula` | `DB_USERNAME` | ClickHouse user. |
| `pass` | string | — | `DB_PASSWORD` | ClickHouse password. Use `${VAR}` substitution. |
| `dbname` | string | `hula` | `DB_NAME` | Database name. |
| `retries` | int | `5` | — | Connection retries on boot. |
| `delay_retry` | int | `5` | — | Seconds between retries. |

Example:

```yaml
dbconfig:
  host: hula-clickhouse
  port: 9000
  user: hula
  pass: ${CLICKHOUSE_PASSWORD}
  dbname: hula
  retries: 10
  delay_retry: 3
```

---

## `ssl`

Three modes — manual paths, ACME (Let's Encrypt), or Cloudflare Origin CA.
Configurable per-server (`servers[].ssl`) or globally as a default.

### Manual

```yaml
ssl:
  cert: /path/to/cert.pem
  key: /path/to/key.pem
```

`cert` and `key` may be filesystem paths or inline PEM. Reload the running
process to pick up rotated certs.

### ACME (Let's Encrypt)

```yaml
ssl:
  acme:
    email: admin@example.com
    cache_dir: /var/hula/certs
    http_port: 80
    domains:
      - example.com
      - www.example.com
```

| Field | Required | Default | Description |
|-------|----------|---------|-------------|
| `email` | recommended | — | ACME account contact. Let's Encrypt sends expiry warnings here. |
| `cache_dir` | no | `<confdir>/certs` | Where issued certs are cached. Persistent volume in production. |
| `http_port` | no | `80` | Port for HTTP-01 challenges. Override when behind a reverse proxy. |
| `domains` | no | host + aliases | Explicit cert SANs. If omitted, derived from the server's `host` + `aliases`. |

### Cloudflare Origin CA

```yaml
ssl:
  cloudflare_origin_ca:
    cert: /etc/hula/cf-origin.pem
    key: /etc/hula/cf-origin-key.pem
    # or inline:
    # cert: |
    #   -----BEGIN CERTIFICATE-----
    #   ...
```

For deployments where Cloudflare terminates public TLS and Hula presents an
Origin CA cert on the back-end leg. See [Behind Cloudflare](../examples/cloudflare-origin-ca.md).

---

## `servers`

The deepest section — each entry is a virtual host. Hula matches by `host`
(or `aliases`) on the `Host:` header / SNI.

### Required keys

| Field | Type | Description |
|-------|------|-------------|
| `host` | string | Canonical hostname this server responds for. `SERVER_HOST` env override. |
| `id` | string | Short opaque identifier used in CLI commands and analytics rows. **Required, no default.** Not a secret. |

### Routing & TLS

| Field | Default | Description |
|-------|---------|-------------|
| `aliases` | — | List of alternate hostnames that resolve to this server. |
| `redirect_aliases` | — | Aliases that 301-redirect to the canonical `host` instead of serving. |
| `listen_if` | — | IP or interface name to bind to. Defaults to inheriting from the listener. |
| `port` | inherit | Port (defaults to global `port`). |
| `ssl` | inherit | TLS config (manual / ACME / Cloudflare). Inherits global `ssl:` when unset. |
| `path_prefix` | — | Prefix for Hula's own endpoints. Currently UNIMPLEMENTED. |
| `api_path` | `/api` | Where the admin API lives. `SERVER_API_PATH` env override. |
| `domain` | derived | Cookie domain. Defaults to the parent of `host`. |
| `http_scheme` | `https` | Scheme published in `hula.js`. Set to `http` for unencrypted local dev. |

### Static serving

| Field | Default | Description |
|-------|---------|-------------|
| `root` | — | Filesystem path to serve. `SERVER_ROOT` env override. |
| `root_compress` | `false` | Transparent gzip / brotli for compressible types. |
| `root_byte_range` | `false` | Honor `Range:` requests for partial content. |
| `root_browse` | `false` | Directory index when `index.html` is absent. **Off in production.** |
| `root_index` | `index.html` | Filename to serve for directory requests. |
| `root_cache_duration` | — | Go duration string (`5m`, `1h`). Sets `Cache-Control: max-age`. |
| `root_max_age` | `3600` | `max-age` in seconds (used when `root_cache_duration` is unset). |
| `static_folders` | — | Additional static-file mounts at custom URL prefixes. See [`StaticFolder`](#additional-static-folders). |

### Git autodeploy

```yaml
servers:
  - host: example.com
    id: example
    root_git_autodeploy:
      repo: https://github.com/myorg/example-site.git
      creds:
        username: x-access-token
        password: ${GITHUB_TOKEN}
      ref:
        branch: main
      hula_build: production    # or "staging"
      build_env:
        - HUGO_ENV=production
```

| Field | Default | Description |
|-------|---------|-------------|
| `repo` | — | HTTPS or SSH clone URL. **Required** when the block is present. |
| `creds.username` | — | Git username (use `x-access-token` for GitHub PATs). |
| `creds.password` | — | Git password / token. Use `${VAR}` substitution. |
| `ref.branch` | — | Branch to track. |
| `ref.tag` | — | Tag to deploy (`semver` for newest semver tag, `any` for newest tag, or an exact tag name). |
| `hula_build` | `production` | Either `production` (ephemeral build → tarballed output) or `staging` (long-lived builder + WebDAV). |
| `data_dir` | `/var/hula/sitedeploy/{{serverid}}/repo` | Where the repo is cloned. Templates expand at runtime. |
| `deploy_dir` | `/var/hula/sitedeploy/{{serverid}}/site` | Where production output gets unpacked. |
| `staging_dir` | `/var/hula/sitedeploy/{{serverid}}/staging-site` | Where the staging container's served output lives. |
| `staging_src_dir` | `/var/hula/sitedeploy/{{serverid}}/staging-src` | Where the staging source tree is editable via WebDAV. |
| `build_env` | — | List of `KEY=VALUE` pairs passed into the build container. |
| `no_pull_on_start` | `false` | When `true`, don't auto-pull on boot. Useful for offline / air-gapped restarts. |
| `committer.name` | `hula-staging` | Author name used by `hulactl staging-mount` commits. |
| `committer.email` | `staging@hula.local` | Author email. |

Per-site builder behaviour (Hugo version, mkdocs version, COMMANDLIST) lives
in `.hula/sitebuild.yaml` inside the **site repo** — see
[builder-images/README.md](https://github.com/tlalocweb/hulation/blob/main/builder-images/README.md).

### Forms, landers, hooks

```yaml
servers:
  - host: example.com
    forms:
      - name: newsletter
        description: Newsletter signup
        feedback: "Thanks!"
        captcha: turnstile
        schema: '{"fields":[{"name":"email","required":true}]}'
    landers:
      - name: launch
        url_id: /launch
        redirect: https://example.com/landed
    hooks:
      on_new_form_submission:
        - name: notify-slack
          risor: |
            // Risor source
      on_lander_visit: []
      on_new_visitor: []
```

Form / lander / hook details: [Forms, landers, hooks](../concepts/forms-landers.md).

### Consent & privacy (per-server)

| Field | Values | Default | Description |
|-------|--------|---------|-------------|
| `consent_mode` | `off` / `opt_in` / `opt_out` | `off` | When events are persisted. See [Consent & privacy](../concepts/consent-privacy.md). |
| `tracking_mode` | `cookie` / `cookieless` | `cookie` | Visitor-ID strategy. Cookieless = HMAC of IP + UA + per-day salt. |
| `forwarders` | list | — | Server-side ad-platform forwarders. See [`forwarders`](#forwarders). |

### Backends (reverse-proxy)

```yaml
servers:
  - host: example.com
    backends:
      - container_name: myapi
        image: registry.example.com/myapi:latest
        virtual_path: /api
        container_path: /api/v2
        expose: ["8002"]
        restart: always
        environment:
          - API_KEY=${API_KEY}
        command: /app/server --port 8002
```

Per-backend keys: `container_name`, `image`, `virtual_path`, `container_path`,
`expose`, `restart`, `environment`, `command`, `volumes`, `logs`. Full surface
documented at [Backend container reverse proxy](../examples/backend-containers.md).

### Cookie options (per-server)

```yaml
servers:
  - host: example.com
    cookie_opts:
      cookie_prefix: hula
      expire_days: 365
      same_site: Strict      # Strict | Lax | None | (empty for autodetect)
      no_secure: false
      no_use_domain: false
```

### Misc

| Field | Default | Description |
|-------|---------|-------------|
| `hello_cookie_max_age` | `30` | Days the visitor cookie persists. |
| `captcha_secret` | — | Fallback captcha secret when individual forms don't set their own. `CAPTCHA_SECRET` env. |
| `csp` | — | Content Security Policy directives. See `CSP` struct. |
| `publish_port` | `false` | Override global `publish_port` per server. |

---

## `forwarders`

Server-to-server forwarders to ad / analytics platforms. Each forwarder is
**consent-gated by `purpose`** — events whose consent flag is false for that
purpose are silently dropped.

```yaml
servers:
  - host: example.com
    forwarders:
      - kind: meta_capi
        pixel_id: "1234567890"
        access_token: ${META_CAPI_TOKEN}
        purpose: marketing
      - kind: ga4_mp
        measurement_id: "G-XXXXXXXX"
        api_secret: ${GA4_API_SECRET}
        purpose: analytics
```

| Field | Description |
|-------|-------------|
| `kind` | Adapter type: `meta_capi` or `ga4_mp`. |
| `purpose` | Consent purpose this forwarder is gated by: `analytics` or `marketing`. |
| _per-adapter fields_ | `pixel_id` + `access_token` for Meta CAPI; `measurement_id` + `api_secret` for GA4 MP. |

See [Server-side forwarders](../concepts/forwarders.md) for the full event /
purpose matrix.

---

## `auth`

Auth providers — internal (OPAQUE password) and OIDC SSO. The `internal`
provider is auto-registered if absent so the admin break-glass account
always works.

```yaml
auth:
  providers:
    - name: internal
      provider: internal
    - name: google
      provider: oidc
      config:
        display_name: Google
        discovery_url: https://accounts.google.com/.well-known/openid-configuration
        client_id: ${HULA_GOOGLE_CLIENT_ID}
        client_secret: ${HULA_GOOGLE_CLIENT_SECRET}
        redirect_url: https://${HULA_HOST}/api/v1/auth/callback/google
        scopes: [openid, email, profile]
        icon_url: /analytics/icons/google.svg
```

| Field | Description |
|-------|-------------|
| `name` | Unique identifier for this provider entry. |
| `provider` | `internal` or `oidc`. |
| `config` | Provider-specific blob. For OIDC: `display_name`, `discovery_url`, `client_id`, `client_secret`, `redirect_url`, `scopes`, `icon_url`. |

---

## `chat`

Visitor chat tunables. Optional — when omitted, chat runs with sane defaults.

```yaml
chat:
  retention_days: 30
  captcha:
    provider: turnstile
    site_key: 0x...
    secret_key: ${TURNSTILE_SECRET}
  email_verifier:
    smtp_check: false
    disposable_check: true
    role_check: true
    misspell_check: true
  openai:
    enabled: false
    api_key: ${OPENAI_KEY}
    model: gpt-5.4-nano
    timeout_ms: 3000
    on_error: allow
  disable_new_sessions: false
```

| Field | Default | Description |
|-------|---------|-------------|
| `retention_days` | `365` | TTL on `chat_sessions` and `chat_messages` tables. |
| `captcha.provider` | `turnstile` | `turnstile` or `recaptcha`. |
| `captcha.test_bypass` | `false` | Treat any captcha token as valid. **Never set in production.** |
| `email_verifier.*` | various | Per-check toggles for email validation on `/chat/start`. |
| `openai.enabled` | `false` | Pre-screen the visitor's first message via OpenAI moderation. |
| `disable_new_sessions` | `false` | Operator kill-switch — `/chat/start` returns 503. Existing sessions stay live. |

---

## `mailer`

SMTP for scheduled report dispatch and operator alerts.

```yaml
mailer:
  host: smtp.example.com
  port: 587
  username: hula@example.com
  password: ${SMTP_PASSWORD}
  from: "Hula Analytics <hula@example.com>"
  starttls: true
```

When the block is unset or incomplete, the dispatcher logs reports without
sending and continues boot. Operator alerts fall through to push channels.

---

## `apns` / `fcm`

Mobile push for operator alerts. Both are optional — missing creds degrade
silently to email.

```yaml
apns:
  team_id: ABCDE12345
  key_id: KEY1234567
  key_pem_path: /etc/hula/apns.p8
  bundle_id: us.tlaloc.hulaadmin
  endpoint: api.push.apple.com           # or api.sandbox.push.apple.com

fcm:
  project_id: my-firebase-project
  service_account_json_path: /etc/hula/fcm.json
```

All fields support env overrides (`HULA_APNS_*`, `HULA_FCM_*`).

---

## `opaque`

OPAQUE PAKE keying material. **When unset Hula generates fresh values at boot
and logs them once with a loud warning** — paste the logged values into
config (or env vars) to pin them.

```yaml
opaque:
  oprf_seed: ${HULA_OPAQUE_OPRF_SEED}
  ake_secret: ${HULA_OPAQUE_AKE_SECRET}
```

**Changing either value invalidates every existing OPAQUE record** — every
operator and agent has to re-register. Treat them as part of the server's
identity. `hulactl opaque-seed` regenerates them off-line.

---

## `team`

Raft cluster identity and membership. Solo deployments omit the entire block;
Hula auto-bootstraps a single-node cluster on first boot and persists the
generated IDs under `data_dir`.

```yaml
team:
  team_id: 0192fc2b-...
  node_id: hula-node-01
  data_dir: /var/hula/data
  bind_addr: 0.0.0.0:5959
  bootstrap: first-of-team
  peers:
    - id: hula-node-02
      addr: 10.0.0.2:5959
    - id: hula-node-03
      addr: 10.0.0.3:5959
```

Multi-node cluster setup is detailed in
[`HA_PLAN2.md`](https://github.com/tlalocweb/hulation/blob/main/HA_PLAN2.md)
and the [3-node HA cluster](../examples/ha-cluster.md) example.

---

## `bad_actors`

```yaml
bad_actors:
  block_threshold: 50
  ttl_seconds: 86400
  allow_cidrs:
    - 198.51.100.0/24
    - 10.0.0.0/8
```

| Field | Default | Description |
|-------|---------|-------------|
| `block_threshold` | `50` | Score at which an IP is auto-blocked. |
| `ttl_seconds` | `86400` | How long scored entries persist before TTL-expiring. |
| `allow_cidrs` | — | CIDRs that are never scored or blocked. |

---

## `proxies`

Top-level reverse-proxy targets. Lower-priority than per-server backends —
mostly used to forward whole hosts elsewhere.

```yaml
proxies:
  - target: http://localhost:1314
    by_path: "/"
  - target: http://internal-app:8000
    by_path: "/internal"
```

---

## Additional static folders

```yaml
servers:
  - host: example.com
    static_folders:
      - root: /opt/assets
        url_prefix: /assets
        compress: true
        byte_range: true
        index: index.html
        cache_duration: 1h
        max_age: 3600
```

Same field semantics as the server-level `root_*` keys, just at a custom URL
prefix.

---

## Path-template variables

Several path fields expand templates at runtime:

| Variable | Resolves to |
|----------|-------------|
| `{{serverid}}` | The server's `id`. |
| `{{host}}` | The server's `host`. |
| `{{confdir}}` | Directory `config.yaml` was loaded from. |

Used in `data_dir`, `deploy_dir`, `staging_dir`, `staging_src_dir`,
`form_schema_folder`, and `ssl.acme.cache_dir`.

---

## Reload semantics

`hulactl reload` sends `SIGHUP` to the running process. Hula re-reads
`config.yaml` and re-applies what's safe in-place: hooks recompile,
backends restart if their config changed, autodeploys re-pull. The
following changes **require a full restart**:

- `port`, `listen_on`, top-level `ssl:` mode change
- Adding or removing `servers[]` entries
- `team:` block changes (Raft membership)
- `opaque:` keys (would invalidate every operator credential)

Verify with `./start-with-docker.sh --logs` immediately after reload — Hula
logs which sections were and weren't picked up.
