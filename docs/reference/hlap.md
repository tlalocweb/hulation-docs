# HLAP — Hula Local Agent Protocol

Wire-level reference for the unix-socket text protocol between local apps
and `hula-agent`. Plain text, line-oriented, request-then-response. Each
HLAP request maps to exactly one authenticated mTLS call against hula.

## Connection model

The agent listens on a unix-socket. A client opens a connection, sends
one or more requests, reads each response in turn, and closes. Multiple
requests on the same connection are processed sequentially — there's no
pipelining and no multiplexing.

```
client                                agent
  | --- BUILD docs\n                    -->|
  |<-- OK build_id=abc123\n               |
  |<-- STATUS=complete\n                  |
  |<-- \n         (blank line, EOF resp)  |
```

A blank line terminates each response. The connection stays open for
the next request until the client closes it.

## Request grammar

```text
<VERB> <site> [args]\n
```

- **`<VERB>`** — uppercase. Case-sensitive. See the [verb table](#verbs).
- **`<site>`** — the server's `id` from `config.yaml`'s `servers[].id`.
  Must match a `sites.<id>` entry in the agent's YAML.
- **`[args]`** — optional, verb-specific. Free-form, whitespace-separated.
  Whatever args the agent accepts get passed through to the underlying
  hula API call **alongside** any options the registry attached.

Examples:

```text
BUILD docs
STAGING-BUILD docs
PULL docs
COMMIT docs feat: tweak hero copy
PUSH-FILE docs content/about.md
GET-FILE docs static/og.png
```

For verbs that take a request body (`COMMIT`, `STAGE`, `PUSH-FILE`),
the body comes after the request line — see
[Request bodies](#request-bodies) below.

## Response grammar

Two top-level forms:

### Success

```text
OK [key=value [key=value ...]]\n
[<extra status lines>\n]*
\n
```

The first line is `OK` followed by zero or more space-separated
`key=value` pairs of structured data. Subsequent lines are
human-readable status. A blank line terminates the response.

```text
OK build_id=abc123
STATUS=running
STATUS=complete

```

### Error

```text
ERR <reason>\n
\n
```

`<reason>` is a single line, free-form. The blank line terminates.

```text
ERR agent not registered

```

### Streaming responses

For `GET-FILE` (file content) and `BUILD` (large logs), the body is
streamed verbatim **after** the OK line, length-prefixed via
`content_length=`:

```text
OK content_length=12345
<... 12345 bytes ...>
\n
```

The trailing blank line still terminates. Clients should read
`content_length` bytes from the stream rather than scanning for the
terminator inside binary content.

## Permission model recap

Every HLAP call is gated by the agent's registry record on the server.
For verb `<V>` on site `<S>`:

1. The agent forwards the request as an mTLS POST/GET/PUT against
   hula's REST endpoint for `<V>`.
2. Hula's mTLS middleware extracts the agent's `<id>` from the cert's
   Subject CN and looks up the registry.
3. If the entry is missing / revoked / past expiry → `401 agent <reason>`
   → agent surfaces `ERR agent <reason>`.
4. Otherwise hula checks `record.IsAllowed(S, V)`. If false → `403 forbidden`
   → agent surfaces `ERR permission denied for verb <V> on site <S>`.
5. If allowed, hula appends the registered options string to the
   request and runs the underlying handler.

The permission string is opaque to the agent. Whatever the registry
says, gets applied — agents cannot inject extra options at call time.

## Verbs

| HLAP verb | Required permission | Maps to | Notes |
|-----------|---------------------|---------|-------|
| [`BUILD`](#build) | `build` | `POST /api/site/trigger-build` | Trigger production build, stream status. |
| [`STAGING-BUILD`](#staging-build) | `staging-build` | `POST /api/staging/build` | Trigger build in long-lived staging container. |
| [`PULL`](#pull) | `pull` | `POST /api/staging/{id}/git/pull` | Rebase staging working tree on origin. |
| [`PUSH`](#push) | `push` | `POST /api/staging/{id}/git/push` | Push staging HEAD to origin. |
| [`SYNC`](#sync) | `sync` | `POST /api/staging/{id}/git/sync` | Pull then push, server-side. |
| [`COMMIT`](#commit) | `commit` | `POST /api/staging/{id}/git/commit` | Commit staged edits. Body: message. |
| [`STAGE`](#stage) | `stage` | `POST /api/staging/{id}/git/stage` | `git add` paths. Body: paths, one per line. |
| [`PUSH-FILE`](#push-file) | `push-file` | `PUT /api/staging/{id}/dav/<path>` | Upload one file via WebDAV. |
| [`GET-FILE`](#get-file) | `get-file` | `GET /api/staging/{id}/dav/<path>` | Download one file via WebDAV. |

### `BUILD`

```text
> BUILD <site>
< OK build_id=<id>
< STATUS=<phase>             # repeated as the build progresses
< STATUS=complete            # or: STATUS=failed
< 
```

**Args:** none accepted from the client side; build options come from
the registered permission string.

**Phases:** `pending`, `cloning`, `preparing_image`, `starting_container`,
`transferring_source`, `running`, `extracting_result`, `deploying`,
`complete`, `failed`.

### `STAGING-BUILD`

```text
> STAGING-BUILD <site>
< OK
< STATUS=running
< STATUS=complete
<
```

Triggers a rebuild in the long-lived staging container. Faster than
`BUILD` because the source is already mounted and the builder image
doesn't restart.

### `PULL`

```text
> PULL <site>
< OK head=<sha-short>
<
```

Pulls origin/`<branch>` into the staging working tree, rebasing on top.
On a rebase conflict, hula automatically rewinds the tree to the
pre-pull HEAD; the response then carries:

```text
< OK head=<pre-pull-sha> rewound=true
< note=rebase conflict; restored to <pre-pull-sha>
<
```

Refuses with `ERR uncommitted edits` if the working tree has unstaged
changes — commit them first.

### `PUSH`

```text
> PUSH <site>
< OK head=<sha-short>
<
```

Pushes HEAD of `staging_src_dir` to origin/`<branch>`. Auth credentials
come from the same `root_git_autodeploy.creds` block the production
clone uses.

### `SYNC`

```text
> SYNC <site>
< OK head=<sha-short>
<
```

`PULL` then `PUSH` in a single server-side operation. If the pull
rebase conflicts, hula rewinds and reports the conflict — push is not
attempted. If pull succeeds but push is rejected, hula rewinds the
working tree to the pre-sync HEAD.

### `COMMIT`

```text
> COMMIT <site> <message>
< OK head=<sha-short>
<
```

Commits whatever's currently staged. The message is the rest of the
request line; for multi-line messages, use a body:

```text
> COMMIT <site>
> body_length=<n>\n
> <n bytes of message>
< OK head=<sha-short>
<
```

Hula appends a `Committed-by: Hula` trailer to the message on its own
line.

### `STAGE`

```text
> STAGE <site>
> body_length=<n>\n
> <path>\n
> <path>\n
> ...
< OK staged=<count>
<
```

`git add` the listed paths inside the staging working tree. With no
body, equivalent to `git add -A`. Paths must stay inside the working
tree (no `..`).

### `PUSH-FILE`

```text
> PUSH-FILE <site> <remote-path>
> content_length=<n>\n
> <n bytes of file content>
< OK
<
```

Upload one file to the staging WebDAV mount at `<remote-path>`. The
remote path is relative to `staging_src_dir`. Hula does the same
safety filtering `staging-mount --dangerous` applies — executables and
security-sensitive files are refused unless the registry's permission
string opted into them.

### `GET-FILE`

```text
> GET-FILE <site> <remote-path>
< OK content_length=<n>
< <n bytes of file content>
<
```

Download one file. Atomic on the receiving side via temp+rename is the
client's responsibility.

## Request bodies

Verbs that carry a body announce its size on the second line:

```text
<VERB> <site> [args]\n
body_length=<n>\n
<n bytes>
```

or, for binary file uploads, `content_length=<n>` is used in place of
`body_length=<n>` to match the response field. Both are accepted by the
agent for backward compatibility; new clients prefer the matching pair
(`content_length=` for binary, `body_length=` for text).

## Concurrency

One in-flight request per connection. The agent serialises requests on
a single connection — pipelining a second request before reading the
first response is not supported and may close the connection.

For parallel work, open multiple connections. Hula throttles per-cert
concurrency via the registry's `max_concurrent` field (Phase 6 — not
yet wired); for now the bottleneck is the agent's single tokio runtime.

## Error codes

Errors come back as `ERR <reason>` on a single line. Common values:

| Reason | Cause |
|--------|-------|
| `ERR agent not registered` | Cert's fingerprint isn't in the server's registry. Wrong server, or registry was wiped. |
| `ERR agent expired` | Cert's `NotAfter` has passed. Re-mint with a longer `--expires-in`. |
| `ERR agent revoked` | `hulactl revoke-agent` fired against this cert. Re-mint. |
| `ERR permission denied for verb X on site Y` | Registry's `allow.<verb>` doesn't include this combination. Re-mint with the right `--allow-*` flags. |
| `ERR <site> not found` | The site `id` doesn't match any `servers[].id` on this hula. |
| `ERR uncommitted edits` | `PULL` / `PUSH` / `SYNC` blocked by dirty working tree. |
| `ERR rebase conflict` | `PULL` couldn't rebase. The tree was rewound; resolve manually. |
| `ERR socket: <detail>` | Connection-level failure on the unix-socket leg. |
| `ERR upstream: <status>` | Hula returned a non-200 not covered above. |

## Worked example

```bash
# Open the socket via netcat
exec 3<>/tmp/hula-agent.sock

# Send a BUILD request
echo "BUILD docs" >&3

# Read until blank line
while read -ru 3 line; do
  [ -z "$line" ] && break
  echo "got: $line"
done
exec 3<&-
```

Or with `nc`:

```bash
printf 'BUILD docs\n' | nc -U /tmp/hula-agent.sock
```

## Where to go next

- [Agents & HLAP](../concepts/agents-hlap.md) — the conceptual model.
- [Agent YAML reference](agent-yaml.md) — how the agent gets its
  identity.
- [`hula-agent` binary reference](hula-agent.md) — the runtime that
  speaks HLAP on one side and mTLS on the other.
- [GitHub Actions → hulaagent](../examples/github-actions-hulaagent.md)
  — full CI walkthrough.
