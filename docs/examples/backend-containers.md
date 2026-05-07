# Backend container reverse proxy

Hula can manage Docker containers on behalf of a virtual server and
reverse-proxy a configured URL prefix to each. The static site lives
under `root:`, the dynamic API runs in a sibling container, and Hula
ties them together at the same hostname.

## Why this exists

- Marketing sites with a small embedded API (lead capture, search,
  feature-flag lookup) don't need a separate edge proxy in front of
  Hula + a dedicated app server.
- Hula's TLS, badactor, and analytics already cover the front edge.
  Adding a backend just hangs an app behind a path prefix.
- Per-server Docker networks isolate one site's backends from another
  site's, so a compromise in one tenant's app doesn't grant lateral
  access.

For multi-tier production stacks (microservice fleets, mesh-managed
backends), Hula's backend support is intentionally minimal — front
those with a real orchestrator instead.

## How it works

```mermaid
flowchart LR
  Visitor -- "https://example.com/" --> Hula
  Visitor -- "https://example.com/api/*" --> Hula
  Hula -- "static" --> Static[/static files /var/hula/.../public]
  Hula -- "/api/* -> /api/v2/*" --> Backend[(myapi container<br/>example-net)]
  Hula -- "/admin/* -> /" --> Admin[(admin-ui container<br/>example-net)]
```

- Hula starts the listed Docker containers at boot (`docker create` +
  `docker start`).
- Each backend gets attached to a per-server Docker network
  (`hula-net-<server-id>`). Containers on the same server's network can
  reach each other; containers on different servers' networks can't.
- Hula's reverse proxy maps `virtual_path` (URL prefix on the public
  side) to `container_path` (URL prefix on the container side).

## Minimal config

```yaml
servers:
  - host: example.com
    id: example
    ssl:
      acme:
        email: admin@example.com
        cache_dir: /var/hula/certs
    root: /var/hula/sites/example/public
    backends:
      - container_name: myapi
        image: registry.example.com/myapi:1.4.2
        virtual_path: /api
        container_path: /
        expose: ["8080"]
```

Result:

- `https://example.com/anything.html` → static file from
  `/var/hula/sites/example/public/anything.html`.
- `https://example.com/api/anything` → proxied to `myapi`'s `/anything`
  (the `/api` prefix is stripped because `container_path: /`).
- `https://example.com/api/users/42` → `myapi`'s `/users/42`.

If you want the `/api` prefix preserved on the backend side, set
`container_path: /api`:

```yaml
backends:
  - container_name: myapi
    image: registry.example.com/myapi:1.4.2
    virtual_path: /api
    container_path: /api
    expose: ["8080"]
```

Now `/api/users/42` → `myapi`'s `/api/users/42`.

## Mounting the Docker socket

Backends require Hula to manage Docker. When Hula runs inside Docker
itself (the typical case), mount the host's socket:

```bash
docker run -d \
  --name hula \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v "$(pwd)/config.yaml":/etc/hula/config.yaml:ro \
  -v "$(pwd)/public":/var/hula/public:ro \
  --restart unless-stopped \
  ghcr.io/tlalocweb/hula:latest
```

The default `start-with-docker.sh` from the installer already does this.

**Security note:** mounting the Docker socket is effectively giving Hula
root on the host. Standard tradeoff for a single-machine deployment;
not appropriate for multi-tenant hosts where Hula's process boundary
needs to mean something.

## Full backend config

```yaml
backends:
  - container_name: myapi                            # unique within server
    image: registry.example.com/myapi:1.4.2          # exact pin in production
    virtual_path: /api                               # public URL prefix
    container_path: /                                # backend URL prefix
    expose: ["8080"]                                 # ports the container listens on
    restart: always                                  # always | unless-stopped | no
    environment:                                     # env vars passed in
      - DATABASE_URL=${MYAPI_DB_URL}
      - LOG_LEVEL=info
    command: ["/app/server", "--port", "8080"]       # override CMD
    volumes:                                         # bind mounts
      - "/etc/hula/myapi-secrets:/run/secrets:ro"
    pull_policy: missing                             # always | missing | never
    logs:
      passthrough: true                              # stream stdout/err to hula's log
      colorize: true                                 # ANSI colors
```

Field-by-field:

| Field | Description |
|-------|-------------|
| `container_name` | Unique name within the server. Hula creates the container under this name on first boot. |
| `image` | Image ref. **Pin a version in production** — `:latest` makes upgrades non-deterministic. |
| `virtual_path` | URL prefix Hula strips (or preserves, see `container_path`) before forwarding. |
| `container_path` | Path prefix prepended on the backend side. |
| `expose` | Ports the container listens on. Hula reverse-proxies to the **first** port in the list. |
| `restart` | Standard Docker restart policy. Default: `unless-stopped`. |
| `environment` | List of `KEY=VALUE`. `${VAR}` substitutes against Hula's process env. |
| `command` | Override the image's CMD. |
| `volumes` | Bind mounts. Same syntax as `docker run -v`. |
| `pull_policy` | `always` / `missing` / `never`. Default `missing`. |
| `logs.passthrough` | Stream backend's stdout/stderr into Hula's log stream (default: on). |
| `logs.colorize` | ANSI colorize the streamed lines. |

## Multiple backends per server

Stack as many as you need:

```yaml
servers:
  - host: example.com
    id: example
    root: /var/hula/sites/example/public
    backends:
      - container_name: myapi
        image: registry.example.com/myapi:1.4.2
        virtual_path: /api
        container_path: /
        expose: ["8080"]
      - container_name: admin-ui
        image: registry.example.com/admin-ui:0.9.0
        virtual_path: /admin
        container_path: /
        expose: ["3000"]
      - container_name: search
        image: registry.example.com/search:2.1.0
        virtual_path: /search
        container_path: /
        expose: ["7700"]
```

Routing precedence: longest-prefix match. `/admin/users/42` goes to
`admin-ui`; `/api/orders/100` goes to `myapi`; `/anything-else` falls
through to the static `root:`.

## Container-to-container communication

Containers on the same server's `hula-net-<server-id>` network resolve
each other by `container_name`:

```bash
# inside myapi:
curl http://search:7700/health
# inside admin-ui:
curl http://myapi:8080/users/42
```

Cross-server (`example` → `other-site`'s backend) is **deliberately not
possible** — different Docker networks, different DNS scope.

## Image pulls and registries

Configure registry credentials at the top level:

```yaml
registries:
  registry.example.com:
    username: ${REGISTRY_USER}
    password: ${REGISTRY_PASSWORD}
  ghcr.io:
    username: ${GITHUB_ACTOR}
    password: ${GITHUB_TOKEN}
```

Hula uses these on first pull. Public images don't need entries.

## Lifecycle

| Trigger | Behaviour |
|---------|-----------|
| Hula boot | All backends start. |
| Backend exits non-zero with `restart: always` | Docker restarts it. Hula tracks the restart count. |
| `hulactl reload` after a `backends[]` change | Affected backends are re-created (image diff, env diff). |
| Hula stops (SIGTERM) | Backends are stopped (not removed) and persist for the next boot. |
| Hula restart | `sweepOrphanBuilders` removes only orphan **builder** containers; backends survive untouched. |

## Health checks

Hula honors Docker's native `HEALTHCHECK` from the image. There's no
Hula-side liveness probe today — if the backend is up, requests get
forwarded; if not, Hula returns 502.

For app-aware health, expose a `/health` endpoint and have your
upstream monitoring poll it via Hula's reverse proxy.

## Logs

With `logs.passthrough: true` (default), backend stdout/stderr stream
into Hula's log stream tagged by `container_name`. To split out:

```bash
./start-with-docker.sh --logs | grep '\[myapi\]'
```

Disable passthrough to keep backend logs only in Docker:

```yaml
logs:
  passthrough: false
```

## Resource limits

Hula doesn't expose CPU / memory limits in the `backends:` shape today.
For tightly-controlled limits, run the backend outside Hula's
backend-containers feature and reverse-proxy to it directly via
`proxies:`.

## Troubleshooting

**`502 Bad Gateway` on every request.** Backend isn't running, or
isn't listening on the first `expose:` port. Check
`docker logs <container_name>`.

**Backend exits immediately on boot.** Image's CMD is wrong, or env
vars aren't getting through. `docker inspect <container_name>` shows
the resolved environment.

**Cross-backend calls fail with `Could not resolve host`.** They're on
different per-server networks. Either move both backends under the
same `servers[]` entry, or use external service discovery.

**`hulactl reload` doesn't pick up image changes.** Image diff requires
the registry to confirm — `pull_policy: missing` won't pull a newer
tag. Pin a new tag (`:1.4.3` instead of `:1.4.2`) or set
`pull_policy: always` for dev environments.

## Next

- [Configuration reference — backends](../reference/config.md#backends-reverse-proxy) —
  full field-by-field.
- [Multi-site](multi-site.md) — running many `servers[]` with their
  own backends.
- [Behind nginx / Traefik](behind-reverse-proxy.md) — when Hula isn't
  the edge.
