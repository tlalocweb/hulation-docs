# `hula-agent` binary reference

The Rust sidecar that consumes an [agent YAML](agent-yaml.md), opens a
unix-socket, and translates [HLAP](hlap.md) requests into authenticated
mTLS calls against a Hula server.

!!! warning "Implementation status"

    Phase 1–3 of the agent design have shipped on the server side
    (cert minting, registry, mTLS handshake validation). The Rust
    runtime described on this page (Phase 4+) is in development —
    flags and behaviour below are the target shape; build from source
    until the binary ships in the standard release tarball. Track
    progress in
    [`HULAAGENT_PLAN.md`](https://github.com/tlalocweb/hulation/blob/main/HULAAGENT_PLAN.md).

## Install

Once Phase-4 ships, the binary lives in the same release tarball as
`hulactl`:

```bash
curl -fsSL https://raw.githubusercontent.com/tlalocweb/hulation/main/installtools.sh | bash
```

Until then, build from source:

```bash
git clone https://github.com/tlalocweb/hulation.git
cd hulation/hulaagent
cargo build --release
cp target/release/hula-agent ~/.local/bin/
```

Supported triples: `x86_64-unknown-linux-gnu`, `aarch64-unknown-linux-gnu`,
`x86_64-apple-darwin`, `aarch64-apple-darwin`. Single binary, ~2 MB
stripped, no runtime dependencies (rustls — no openssl).

## Synopsis

```text
hula-agent --config <path> --socket <path> [flags]
```

Both `--config` and `--socket` are required.

## Flags

| Flag | Default | Description |
|------|---------|-------------|
| `--config <path>` | — | Path to the agent YAML written by `hulactl create-agent`. **Required.** |
| `--socket <path>` | — | Path to the unix-socket the agent listens on. **Required.** Cleaned up on graceful shutdown. |
| `--socket-mode <octal>` | `0600` | Permission bits applied to the socket after creation. `0660` is acceptable when the agent and clients run as different users in the same group. |
| `--log-level <level>` | `info` | One of `error`, `warn`, `info`, `debug`, `trace`. |
| `--log-format <fmt>` | `json` | `json` or `text`. JSON for log shippers, text for tail-and-grep. |
| `--dump` | — | Parse the YAML, print the parsed (sanitised) view to stdout, and exit. Used to validate secret-injection without starting the agent. The `key` block is replaced with `<redacted>` in the dump. |
| `--version` | — | Print the version and exit. |
| `--help` | — | Print this usage and exit. |

## Lifecycle

```mermaid
sequenceDiagram
    participant OS
    participant agent as hula-agent
    participant client as Local app
    participant hula as Hula host

    OS->>agent: spawn
    agent->>agent: parse --config YAML
    agent->>agent: bind --socket
    agent->>agent: load mTLS material
    Note over agent: ready

    client->>agent: open socket; HLAP request
    agent->>hula: mTLS POST/GET/PUT
    hula-->>agent: 200 + body (or error)
    agent-->>client: OK ... / ERR ...

    OS->>agent: SIGTERM
    agent->>agent: drain in-flight requests
    agent->>agent: unlink socket
    agent->>OS: exit 0
```

### Signals

| Signal | Behaviour |
|--------|-----------|
| `SIGTERM` | Graceful shutdown. Stop accepting new connections, drain in-flight, unlink the socket, exit `0`. |
| `SIGINT` | Same as SIGTERM. |
| `SIGHUP` | Reload the YAML in place. Existing in-flight requests finish on the old config; new requests use the new config. Convenience for cert-rotation flows that don't want to drop the socket. |
| `SIGKILL` | Immediate termination, socket file may persist. Avoid; the agent never holds an unrecoverable lock. |

### Exit codes

| Code | Meaning |
|------|---------|
| `0` | Graceful shutdown after SIGTERM/SIGINT. |
| `1` | Bad YAML (parse error, missing required field). |
| `2` | Socket bind failed (permissions, address-in-use). |
| `3` | mTLS material rejected at load time (malformed cert, key/cert mismatch). |
| `4` | Hula host unreachable on first health check. |
| `5` | Internal error (panic, unrecoverable async task). |

## Logging

Each log line is a single record. JSON shape:

```json
{
  "ts": "2026-05-07T18:00:00.123Z",
  "level": "info",
  "msg": "request",
  "verb": "BUILD",
  "site": "docs",
  "agent_id": "Y2k0...",
  "duration_ms": 412,
  "result": "ok"
}
```

Stable field names:

| Field | Description |
|-------|-------------|
| `ts` | RFC3339 with milliseconds. UTC. |
| `level` | `error`, `warn`, `info`, `debug`, `trace`. |
| `msg` | Short event name — `boot`, `request`, `response`, `mtls_error`, `socket_error`, `shutdown`. |
| `verb` | The HLAP verb on a request line. |
| `site` | The HLAP site target. |
| `agent_id` | Truncated agent ID (first 8 chars). Useful for cross-correlating with the server's audit log. Never the cert's full SAN. |
| `duration_ms` | Wall-clock duration of the upstream call. |
| `result` | `ok` or an error class — `agent_revoked`, `permission_denied`, `upstream_error`, `socket_error`. |

`--log-format text` produces a key=value line:

```text
2026-05-07T18:00:00.123Z INFO request verb=BUILD site=docs duration_ms=412 result=ok
```

## Resource footprint

- **Binary size:** ~2 MB stripped (release profile uses `opt-level = "z"`,
  `lto = true`, `panic = abort`).
- **Cold startup:** under 50 ms on a modern CPU.
- **Steady-state RSS:** ~3–5 MB. The single-threaded tokio runtime
  doesn't allocate worker pools.
- **Concurrency:** designed for single-digit concurrent HLAP requests
  per agent instance. For higher throughput, run multiple agents on
  different sockets.

## Security notes

- **YAML file mode `0600`.** The `key` block is unencrypted. Run the
  agent under its own UID on shared CI hosts.
- **Socket file mode `0600` by default.** Change to `0660` and ensure
  the local app shares a group with the agent if you want
  same-host-different-user access.
- **No bearer tokens, no JWTs.** The cert is the only credential. If
  the YAML leaks, the cert leaks; revoke and re-mint.
- **`panic = abort` consequences.** A panic terminates the process
  rather than unwinding. In-flight requests are dropped (the upstream
  call may or may not have completed — clients see `ERR socket: closed`).
  Investigate via the host's coredump configuration; the binary's small
  size makes core analysis quick.
- **No outbound DNS resolution beyond `hula_host`.** The agent only ever
  speaks to the host named in the YAML. If you want it to traverse a
  proxy, run a TCP-level forwarder on the host and point `hula_host` at
  the forwarder.

## Operational tips

**Verify the YAML before trusting it.** `--dump` parses and prints the
sanitised view, so you can confirm secret injection worked without
exposing the key:

```bash
echo "$HULA_AGENT_YAML" | hula-agent --config /dev/stdin --dump
```

**Tail the log as a smoke test.** Run with `--log-level=debug` for the
first few requests after deploying a fresh agent — the boot record
includes the resolved `hula_host` and the agent's truncated ID, both
of which should match the registry record on the server.

**Rotate via overlap, not hot-swap.** When rotating an agent's cert:
mint the new one, deploy alongside the old, switch traffic, revoke the
old. SIGHUP-reload also works for the swap step but the overlap window
gives you a rollback if the new cert was issued wrong.

**One agent, one runner.** Sharing a single agent across multiple
runners works (just point them at the same socket via NFS or
similar), but a compromise of one rotates them all. Per-runner agents
are the supported shape.

## Where to go next

- [Agent YAML reference](agent-yaml.md) — schema for `--config`.
- [HLAP reference](hlap.md) — the protocol the socket speaks.
- [Agents & HLAP](../concepts/agents-hlap.md) — the conceptual model.
- [GitHub Actions → hulaagent](../examples/github-actions-hulaagent.md)
  — full CI deployment of the binary.
