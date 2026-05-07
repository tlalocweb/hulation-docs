# Upgrading Hula

Hula is shipped as a Docker image. Upgrading is "stop, pull, start" plus
attention to schema and key-material continuity. This page covers the
mechanics and the safe-rollback model.

## What gets upgraded

| Artifact | How | Upgrade cadence |
|----------|-----|-----------------|
| `hula` container | `docker pull` + restart | Per release |
| `hula-builder-default` image | `./build-images.sh` or pulled separately | Per release (independent of `hula`) |
| `hulactl` (operator laptops) | `installtools.sh` | Per release |
| `hula-agent` (CI runners) | Same release tarball as `hulactl` (Phase 4+) | Per release |
| ClickHouse schema | `hulactl initdb` (idempotent) | Migrations run automatically on Hula boot |
| Bolt + Raft state | Survives across upgrades | Format-stable within v1 |
| Issued Let's Encrypt certs | Survive across upgrades (cached on disk) | Renew automatically |

## Standard upgrade

For a single-Hula deployment using the installer's `start-with-docker.sh`:

```bash
cd /path/to/hula
./start-with-docker.sh --pull --restart
```

What this does:

1. Pulls the latest `ghcr.io/tlalocweb/hula:latest`.
2. Stops the running container (default 5s graceful shutdown).
3. Restarts with the new image, mounting the same `config.yaml`,
   `hula_certs/`, and `public/` directories.

The persistent state directories (`hula_certs/`, the data dir, the
ClickHouse volume) are unaffected — that's the whole point of having
them as host-mounted volumes.

For pinned versions, override `HULA_IMAGE`:

```bash
HULA_IMAGE=ghcr.io/tlalocweb/hula:v0.21.0 ./start-with-docker.sh --pull --restart
```

**Pin in production.** `:latest` is convenient but makes a stuck node a
puzzle. Pin to a specific tag and bump deliberately.

## Builder image upgrade

Builder images version separately from the `hula` server image. Bump
when there's a new Hugo / mkdocs / mkdocs-material pin:

```bash
cd /path/to/hulation/builder-images
./build-images.sh
docker tag hula-builder-alpine-default:latest hula-builder-default:latest
```

Hula picks up the new image on the next build (no restart required).
For a fully clean break, force-remove any running staging containers
first:

```bash
docker ps --filter "name=hula-builder-" -q | xargs -r docker rm -f
```

A subsequent `hulactl staging-build <site>` will start fresh with the
new image.

## ClickHouse schema migrations

Hula migrations run automatically on boot. Forward-only — there's no
"migrate down". The boot log shows the schema version reached:

```text
clickhouse: schema migrated to version 24
```

If a migration fails:

- Hula logs the failure, retries per `dbconfig.retries` × `delay_retry`,
  then exits non-zero.
- Boot does not proceed until the schema applies cleanly. Old container
  stays stopped.
- Roll back by starting the previous Hula image (it understands its
  own older schema).

Most migrations are additive (new tables, new columns). The forward-only
constraint means: **don't downgrade past a migration boundary unless
you've also restored the ClickHouse data from a pre-migration backup.**

For a schema-version compatibility matrix, watch the
[main repo's release notes](https://github.com/tlalocweb/hulation/releases).

## hulactl on operator laptops

```bash
curl -fsSL https://raw.githubusercontent.com/tlalocweb/hulation/main/installtools.sh | bash
```

Or pin a version:

```bash
HULA_VERSION=v0.21.0 \
  curl -fsSL https://raw.githubusercontent.com/tlalocweb/hulation/main/installtools.sh | bash
```

`hulactl` is forward-compatible across patch releases. Major-release
upgrades may need re-`auth` (the JWT format itself is stable; the
underlying handshake additions sometimes are not).

## Pre-upgrade checklist

Before starting a non-trivial upgrade:

- [ ] **Backup the data dir.** `tar czf hula-data-$(date +%F).tgz /var/hula/data/`
- [ ] **Backup ClickHouse.** Either `BACKUP TABLE` or volume snapshot.
- [ ] **Note the current image tag** in case of rollback.
- [ ] **Skim the release notes** for breaking changes.
- [ ] **Heads-up to live-staging users** — staging containers restart on
      `hula` restart, taking 10–30s.

For HA clusters, additionally:

- [ ] **Confirm leader.** `hulactl team-status` shows the current leader
      and which node you're targeting.
- [ ] **Plan the upgrade order.** Followers first, leader last, so the
      cluster never has a non-upgraded follower talking to a downgraded
      leader.

## Rolling upgrade (HA)

Multi-node Raft cluster. Upgrade one node at a time:

```bash
# On follower node 2:
./start-with-docker.sh --stop
HULA_IMAGE=ghcr.io/tlalocweb/hula:v0.21.0 ./start-with-docker.sh

# Verify it rejoined the cluster
hulactl team-status <leader-addr> --pki-dir /etc/hula/pki
# expected: node-2 status=voter version=v0.21.0

# Repeat on node 3, then on the leader.
```

Step-down + re-election happens cleanly when the leader stops. Followers
elect a new leader within ~1s; the upgrading node rejoins as a follower
when it comes back up.

## Rollback

Keep the previous image tagged locally so you don't have to pull on
rollback:

```bash
docker tag ghcr.io/tlalocweb/hula:v0.21.0 hula:rollback
```

To roll back:

```bash
./start-with-docker.sh --stop
HULA_IMAGE=hula:rollback ./start-with-docker.sh
```

Caveats:

- **ClickHouse migrations are forward-only.** If the new release ran a
  migration, the old `hula` may not understand the new schema. Restore
  ClickHouse from backup before rolling back across a schema change.
- **Bolt format is stable within v1**, so the data dir survives both
  directions.
- **Cert cache survives** — Let's Encrypt rate limits don't bite the
  rollback.

## OPAQUE keys must persist

Hula's OPAQUE PAKE state is keyed off `oprf_seed` and `ake_secret`.
**Changing these invalidates every operator credential.** Keep them
pinned across upgrades — either in `config.yaml`'s `opaque:` block or
in `HULA_OPAQUE_*` env vars.

If you're upgrading a deployment where the keys were auto-generated
(and never pinned), pin them now from the Bolt store before upgrading:

```bash
hulactl --bolt /var/hula/data/hula.db opaque-seed --print-current
# Capture the values into config.yaml.opaque or env vars.
```

(The `--print-current` flag is on the roadmap; until it lands,
inspect `hula.db` with `bolt` directly or pin fresh keys and re-enroll.)

## Agent CA continuity

The agent CA at `<data_dir>/agent-ca.{pem,key}` is generated on first
boot and never rotated automatically. Survives upgrades.

Rotate manually only when the CA key is compromised — every existing
agent has to be re-minted.

## Common upgrade pitfalls

**`hula` won't start: "schema migration failed".** A migration's
forward step refuses to run on the existing data. Fix paths:

1. Read the migration's failure detail in the log.
2. If reversible (rare): start the previous image, drain, fix, retry.
3. If permanent: restore ClickHouse from the pre-upgrade snapshot,
   then start the previous image.

**Builds fail with "image not found" after upgrade.** The
`hula-builder-*` image isn't on the host. Pull / rebuild:

```bash
cd /path/to/hulation/builder-images && ./build-images.sh
```

**Staging-mount loses connection during restart.** Expected — the
staging container restarts with the host. `staging-mount --autobuild`
reconnects automatically; manual sessions need to re-run.

**`hulactl authok` returns 401 after upgrade.** OPAQUE handshake added
fields the server understands but old `hulactl` doesn't (or vice-versa).
Update `hulactl` on the laptop and re-`auth`.

**Live chat sessions drop.** WebSocket connections terminate on
container stop. Visitors and operator agents reconnect within seconds;
chat history is preserved in ClickHouse.

## Where to go next

- [Backups & restore](backups.md) — what to back up before upgrades.
- [Troubleshooting](troubleshooting.md) — when an upgrade goes
  sideways.
- [Configuration reference](../reference/config.md) — what each
  upgrade-relevant config field does.
