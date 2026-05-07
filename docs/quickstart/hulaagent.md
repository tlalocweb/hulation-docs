# hulaagent Quick Start (CI runner)

Drive a remote Hula from a CI runner / deploy-bot using a scoped, expiring agent
certificate — no admin password or JWT on the runner.

!!! warning "Implementation status"

    The admin-side `hulactl create-agent` flow is live in `hulation` today
    (Phases 1–3 of [`HULAAGENT_PLAN.md`](https://github.com/tlalocweb/hulation/blob/main/HULAAGENT_PLAN.md)).
    The runner-side `hula-agent` Rust binary (Phase 4+) is still in development —
    the workflow below is the target shape; sections marked *(in development)*
    are not yet runnable end-to-end. Track progress on the linked plan.

## Why an agent

Cert-only auth, scoped to specific sites + verbs, expires automatically. No JWT
to rotate, no admin password on the runner. Contrast with
[`hulactl auth`](hulactl.md), which is for human operators.

The flow is **two-machine**: an admin runs `hulactl create-agent` once, hands
the resulting YAML to the CI runner, and the runner thereafter speaks only to
its local `hula-agent` over a unix-socket. The cert is the only credential.
Revoking it is instant.

## Step 1 — admin: mint the agent

On the Hula host (or any laptop with admin `hulactl auth`):

```bash
hulactl create-agent \
    --allow-build=docs \
    --allow-staging-build=docs \
    --expires-in=1yr \
    > docs-ci-agent.yaml
```

The generated YAML contains:

- `agent.id` — a base64url 16-byte server-assigned ID.
- `agent.mTLS.ca` — the Agent CA cert (PEM).
- `agent.mTLS.cert` — the leaf cert this agent will present at handshake (PEM).
- `agent.mTLS.key` — the private EC key (PEM).
- `agent.hula_host` — host:port the agent connects to.
- `sites:` — the per-site `allow` map, mirroring the `--allow-*` flags.

Treat the YAML as a secret — the `key` block is unencrypted.

For repeatable agent configs in version control, a template form is supported:

```bash
hulactl create-agent -c agent-template.yaml > docs-ci-agent.yaml
```

## Step 2 — admin: hand the YAML to the runner

GitHub Actions:

```yaml
# .github/workflows/deploy.yml
env:
  HULA_AGENT_YAML: ${{ secrets.HULA_AGENT_YAML }}
```

Add the YAML content as a repository or organisation secret named
`HULA_AGENT_YAML`.

Jenkins: store as a "Secret text" credential.
Local dev: `chmod 600 ~/.hula/agent.yaml`.

## Step 3 — runner: install `hula-agent`  *(in development)*

The Rust sidecar will ship in the same release tarball as `hulactl`
(see [decision D6](../index.md) — pending Phase 4). Until that ships, build
from source:

```bash
git clone https://github.com/tlalocweb/hulation.git
cd hulation/hulaagent
cargo build --release
cp target/release/hula-agent ~/.local/bin/
```

Single binary, ~2 MB stripped, no runtime dependencies.

## Step 4 — runner: start the sidecar  *(in development)*

```bash
echo "$HULA_AGENT_YAML" > /tmp/agent.yaml
chmod 600 /tmp/agent.yaml
hula-agent --config /tmp/agent.yaml --socket /tmp/hula-agent.sock &
```

The agent listens on the unix-socket, waits for HLAP commands, and translates
each into an authenticated mTLS call against `agent.hula_host`.

## Step 5 — runner: drive it via HLAP  *(in development)*

Trigger a production build:

```bash
printf 'BUILD docs\n' | nc -U /tmp/hula-agent.sock
# < OK build_id=abc123
# < STATUS=complete
```

A staging build:

```bash
printf 'STAGING-BUILD docs\n' | nc -U /tmp/hula-agent.sock
```

Push a single file via the staging WebDAV:

```bash
printf 'PUSH-FILE docs content/changelog.md\n%s' "$(cat ./local-changelog.md)" \
  | nc -U /tmp/hula-agent.sock
```

The full HLAP verb set (`BUILD`, `STAGING-BUILD`, `PULL`, `PUSH`, `SYNC`,
`COMMIT`, `STAGE`, `PUSH-FILE`, `GET-FILE`) is documented in the
[HLAP reference](../reference/hlap.md).

## Step 6 — admin: revoke when decommissioned

```bash
hulactl revoke-agent <id>
```

Sets the `revoked` flag in the agent registry. The next mTLS handshake from
that cert is refused (401 *agent revoked*); in-flight requests finish.
Revocation is permanent — re-mint a fresh agent when the runner comes back.

List active agents:

```bash
hulactl list-agents
```

## Permission semantics

`sites.<site_id>.allow.<verb>` carries the option string for that verb (empty
string = verb permitted with no options). The permission string is opaque to
the agent — agents can't pass arbitrary options at HLAP-call time; whatever
the registered `allow` says is what they get.

- `allow.build: ""` lets the agent do a plain `BUILD` against that site, no
  flags.
- `allow.staging-build: "OPT1,OPT2"` lets the agent do
  `STAGING-BUILD OPT1,OPT2` and nothing else.
- `allow.staging-build` absent means no `STAGING-BUILD` for that site.

Wildcards (`"*"` to allow any options) are explicitly not supported. Operators
who need flexibility issue multiple agents.

## Troubleshooting

**`agent not registered`.**
The cert's leaf isn't in the server's agent registry. Either the registry was
wiped, or the agent YAML is for a different Hula host. Re-mint.

**`agent expired`.**
NotAfter on the cert has passed. Re-mint with a longer `--expires-in`.

**`agent revoked`.**
Someone ran `hulactl revoke-agent` against this ID. Re-mint.

**`permission denied for verb X on site Y`.**
The agent's `allow:` map doesn't grant that verb on that site. Re-mint with
the right `--allow-*` flags — permissions are immutable on existing certs by
design.

## Next steps

- [HLAP reference](../reference/hlap.md) — wire-level protocol, every verb.
- [`hula-agent` reference](../reference/hula-agent.md) — flags, lifecycle, security notes.
- [Agent YAML reference](../reference/agent-yaml.md) — schema and field semantics.
- [GitHub Actions → hulaagent](../examples/github-actions-hulaagent.md) — full CI walkthrough.
- [Local dev loop with hulaagent](../examples/hulaagent-local-dev.md) — edit / commit / push via HLAP.
