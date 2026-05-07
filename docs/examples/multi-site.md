# Multi-site (one Hula, many domains)

A single Hula instance can serve dozens of domains — each with its own
TLS, content root, analytics, forms, and badactor scoring — all sharing
the same listener. This page covers the shared-listener semantics, the
operational shape, and the resource ceilings.

## Why one Hula

- One persistent Bolt + Raft state. One ClickHouse. One operator
  credential set. Reduced surface area to monitor and back up.
- One ACME account. Shared cert cache across servers — Let's Encrypt
  rate limits hit once, not per-domain.
- Shared badactor radix tree across all servers — an IP probing one
  domain gets blocked from all of them at threshold.
- One cluster of Risor hooks compiled per-server.

Splitting per-domain Hulas only makes sense when domains belong to
fully isolated tenants (in which case use the
[multi-tenant model](../concepts/architecture.md) — out of scope here).

## Config shape

Every domain is an entry under `servers:`:

```yaml
servers:
  - host: example.com
    aliases: [www.example.com]
    id: example
    ssl:
      acme:
        email: admin@example.com
        cache_dir: /var/hula/certs
    root: /var/hula/sites/example/public

  - host: blog.example.com
    id: blog
    ssl:
      acme:
        email: admin@example.com
        cache_dir: /var/hula/certs
    root_git_autodeploy:
      repo: https://github.com/myorg/blog.git
      ref: { branch: main }
      hula_build: production

  - host: docs.example.com
    id: docs
    ssl:
      acme:
        email: admin@example.com
        cache_dir: /var/hula/certs
    root_git_autodeploy:
      repo: https://github.com/myorg/docs.git
      ref: { branch: main }
      hula_build: production

  - host: store.example.com
    id: store
    ssl:
      acme:
        email: admin@example.com
        cache_dir: /var/hula/certs
    root: /var/hula/sites/store/public
    backends:
      - container_name: store-api
        image: registry.example.com/store-api:latest
        virtual_path: /api
        expose: ["8080"]
```

Four wildly different sites; one Hula process; one TLS listener.

## How routing works

Per-request routing is two steps:

1. **TLS handshake** — SNI selects the right cert. Hula's unified
   listener has all servers' certs loaded; the cert that matches the
   `Host:` header (or, for `https://IP/`, falls back to a default cert)
   gets presented.
2. **HTTP routing** — the `Host:` header (lowercased, port stripped)
   picks the matching server entry. Aliases are folded into the same
   match.

Hosts that don't match any `servers[]` entry get the unified-listener
fallback (404 or the configured default).

## ACME at scale

The ACME manager merges domains across all servers that use the same
`cache_dir`:

- One ACME account, shared.
- One challenge listener on port 80.
- Cert renewal is per-domain but batched.

Let's Encrypt rate limits to be aware of:

| Limit | Value | Mitigation |
|-------|-------|------------|
| Certificates per registered domain | 50 / week | Don't loop ACME on a misconfigured server (a tight retry would burn through this). |
| Duplicate certificates | 5 / week | Make sure `domains:` lists are stable across reloads — same set every time. |
| Failed validations | 5 / hour | If a single domain keeps failing (stale DNS, port 80 unreachable), pause it rather than retry. |

Practical advice: when bulk-adding 20+ servers, stage them in over a few
hours rather than all at once, so any misconfiguration that triggers
challenge failures has a chance to be noticed before the limit hits.

## Per-server analytics

Every analytics row carries the server's `id`. Roll up by site or
across sites with standard ClickHouse:

```sql
SELECT server_id, count() AS visitors
FROM hula.events
WHERE event_type = 'hello'
  AND ts >= now() - INTERVAL 1 DAY
GROUP BY server_id
ORDER BY visitors DESC;
```

The admin UI does the same query under the hood and shows per-server
breakdown by default.

## Per-server forms and hooks

Forms / landers / hooks are per-server (under each `servers[]` entry),
not global. Two domains submitting the same `name: newsletter` form
get separate event streams. To share a Risor hook across servers, drop
it in a shared file and reference from each:

```yaml
servers:
  - host: example.com
    hooks:
      on_new_form_submission:
        - name: notify-slack
          risor: ./hooks/slack.risor      # path relative to confdir
  - host: blog.example.com
    hooks:
      on_new_form_submission:
        - name: notify-slack
          risor: ./hooks/slack.risor
```

Both compile from the same source; per-server isolation is preserved.

## Resource ceilings

Practical limits on a single Hula process. These aren't hard limits —
just where you'll start to feel pressure:

| Resource | Comfortable | Tightening |
|----------|-------------|-----------|
| Total `servers[]` entries | 100s | 1000+ — boot time grows linearly with hook compilation; consider splitting. |
| Concurrent live ACME challenges | ~50 | 100+ — port 80 challenge handling becomes a noticeable cold-start tax. |
| Builder containers active | 1 per site, ephemeral | Spin-up dominates total build time. |
| ClickHouse rows / sec | bottlenecks first | Tune ClickHouse, not Hula. |

Pure static-serving scales horizontally well — Hula is mostly
TLS-terminator + static file server + analytics ingest queue at that
point.

## Reload semantics

Adding or removing entries from `servers[]` is **not SIGHUP-safe** —
restart Hula. Most other in-place changes (root paths, Risor hooks,
ACME emails, cert/key paths) reload cleanly:

```bash
hulactl reload
# logs which sections were and weren't picked up
```

For deployments where adding a server is common (multi-tenant, dev),
schedule restart windows or accept short connection drops on
in-place reconfigs.

## Backups

One backup target for the whole fleet:

- `data_dir` — Bolt + Raft state.
- `cache_dir` — issued certs.
- ClickHouse — analytics data (separate ClickHouse backup story).

Per-server `staging_src_dir` is part of `data_dir` only when the
operator hasn't overridden it.

## Operational notes

- **Logs share a stream.** Per-server log entries carry `server_id` —
  filter with `--log-tags=server.example` (when log-tag config supports
  it) or grep.
- **`hulactl` commands take a server ID.** Multi-site operators
  typically use shell completion or a wrapper script that maps
  human-friendly names to server IDs.
- **Analytics queries scope to a server by default.** The admin UI
  prompts for a server-id picker; cross-server reports are an explicit
  "all sites" toggle.

## Next

- [Architecture overview](../concepts/architecture.md) — how the
  unified listener works.
- [Servers, hosts, aliases](../concepts/servers-hosts.md) — the
  per-virtual-host model in depth.
- [Backend container reverse proxy](backend-containers.md) — for
  per-server backend services.
