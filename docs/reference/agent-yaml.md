# Agent YAML reference

The YAML file emitted by [`hulactl create-agent`](hulactl.md#agents) and
consumed by [`hula-agent`](hula-agent.md). The schema is stable: same
shape between Phase-1 offline mode and Phase-2+ server-issued mode.

## Shape

```yaml
agent:
  id: <base64url-16>
  hula_host: hula.example.com:443
  mTLS:
    ca: |
      -----BEGIN CERTIFICATE-----
      ...
      -----END CERTIFICATE-----
    cert: |
      -----BEGIN CERTIFICATE-----
      ...
      -----END CERTIFICATE-----
    key: |
      -----BEGIN EC PRIVATE KEY-----
      ...
      -----END EC PRIVATE KEY-----

sites:
  <site_id>:
    allow:
      <verb>: "<options-string>"
```

## `agent`

The runtime identity block.

| Field | Type | Description |
|-------|------|-------------|
| `agent.id` | string | Server-assigned, base64url-encoded 16-byte random ID. Encoded into the leaf cert's Subject CN as `agent:<id>`. The middleware extracts it at handshake time to look up the registry. **Don't edit on the runner — it's set in the cert; changing the YAML alone has no effect.** |
| `agent.hula_host` | string | `host:port` the agent connects to. Mandatory. The agent verifies the server's TLS hostname against this value. |
| `agent.mTLS.ca` | string (PEM) | The Agent CA certificate that signed `cert`. Used by the agent to verify the server's identity at handshake (see [open question §11.1 in the PRD](https://github.com/tlalocweb/hulation-docs/blob/main/PRD.md) on whether this also pins the server's serving cert chain). |
| `agent.mTLS.cert` | string (PEM) | The leaf certificate the agent presents at handshake. Subject CN = `agent:<id>`. NotAfter = the cert's expiry. |
| `agent.mTLS.key` | string (PEM) | The leaf cert's private key. Encoded as a PKCS#8 EC private key. **Unencrypted** — the YAML itself is the secret. |

## `sites`

The per-site permission map.

```yaml
sites:
  docs:
    allow:
      build: ""
      staging-build: ""
  blog:
    allow:
      build: ""
```

| Field | Description |
|-------|-------------|
| `sites.<site_id>` | A server's `id` from `config.yaml`'s `servers[].id`. |
| `sites.<site_id>.allow.<verb>` | One of `build`, `staging-build`, `pull`, `push`, `sync`, `commit`, `stage`, `push-file`, `get-file`. The string value is an opaque options blob applied server-side. Empty string = "verb permitted with no options". |

The permission string is **opaque to the agent** — agents can't pass
arbitrary options at HLAP-call time; whatever the registered `allow`
string says is what they get. Wildcards (`"*"`) are deliberately **not
supported** in v1.

### Permission semantics worked example

```yaml
sites:
  docs:
    allow:
      build: ""                    # plain BUILD docs, no flags
      staging-build: "OPT1,OPT2"   # STAGING-BUILD docs always with OPT1,OPT2
```

Given this YAML:

- `BUILD docs` → server runs the build with no extra options. ✅
- `STAGING-BUILD docs` → server runs the staging build with `OPT1,OPT2`
  applied. ✅
- `STAGING-BUILD docs other-opts` → ❌ rejected. The agent doesn't pass
  options through; the server enforces the registered options.
- `PUSH docs` → ❌ rejected. `push` not in `allow`.
- `BUILD blog` → ❌ rejected. `blog` not in `sites`.

## Trust boundary: server registry vs. runner YAML

Editing the YAML on the runner does **not** grant the agent any new
authority. The cert's Subject CN names a registry entry on the server;
the registry holds the canonical permission map. Hula consults the
registry at every HLAP call, not the YAML.

So:

- Adding `sites:` entries to the YAML on the runner — server still rejects
  them.
- Removing `sites:` entries — the agent stops sending those verbs (won't
  bother), but the cert remains capable until revoked. Don't rely on YAML
  edits as an access-control mechanism.
- Editing `id` — breaks the cert's Subject CN match; handshake fails.
- Editing `hula_host` — agent talks to a different host, with no
  authority there. Server's mTLS handshake fails.

The **only** authoritative mutation is `hulactl revoke-agent <id>` (or
re-minting a fresh cert and decommissioning the old one).

## Generating the YAML

### Server-issued (Phase 2+, the production path)

```bash
hulactl create-agent \
    --allow-build=docs \
    --allow-staging-build=docs \
    --expires-in=1yr \
    > docs-ci-agent.yaml
```

`hulactl` calls `POST /api/v1/agent/create` against the authenticated
hula server. The server mints a fresh ID, signs a leaf under the
persistent Agent CA, writes the registry record, and returns the
rendered YAML.

### Offline (Phase-1 dev mode)

```bash
hulactl create-agent --offline \
    --allow-build=docs \
    --expires-in=30d \
    > docs-dev-agent.yaml
```

Generates a one-off Agent CA inside `hulactl` and emits a self-signed
agent. **Useful only for dev / demo** — the resulting cert has no
matching registry entry on any real hula server, so a real handshake
will fail with `agent not registered`.

## Storage on the runner

- File mode: **`0600`**. The `key` block is unencrypted; treat the file
  as you would any other secret.
- Inject via your runner's secret manager (GitHub Actions repo /
  org secret, Jenkins credentials, Kubernetes Secret) rather than
  committing to a repo.
- Mount into the runner's process at runtime; don't bake into a
  container image.
- One agent per runner is the supported shape; sharing across runners
  works but means a compromise of one rotates them all.

See [`hula-agent` binary reference](hula-agent.md) for runtime
operational details (socket permissions, signal handling, exit codes).

## Deprecation policy

The schema is **stable** for the v1 line. Adding optional fields
(e.g., `agent.metadata`, `sites.<id>.allow.<verb>` becoming a
structured object) is allowed and won't break existing YAMLs. Removing
or repurposing existing fields requires a major-version bump.

## Where to go next

- [Agents & HLAP](../concepts/agents-hlap.md) — the conceptual model
  this YAML feeds.
- [HLAP reference](hlap.md) — what verbs do.
- [`hula-agent` binary reference](hula-agent.md) — what consumes this
  YAML at runtime.
- [`hulactl create-agent`](hulactl.md#agents) — what produces it.
