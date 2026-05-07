# `hulactl` command reference

Every `hulactl` subcommand. Source of truth is
[`model/tools/hulactl/cmddef.go`](https://github.com/tlalocweb/hulation/blob/main/model/tools/hulactl/cmddef.go);
the inline `hulactl` (no args) prints the same surface in compact form.

## Conventions

- **Server URL** comes from the stored `~/.config/hulactl/hulactl.yaml`
  written by `hulactl auth`. Override per-invocation with the `--server`
  flag.
- **Authentication** uses the OPAQUE-derived JWT in `hulactl.yaml`. Run
  `hulactl authok` to confirm the JWT is still valid.
- **Flags must come before the command name.** This is the Go `flag`
  package convention — `hulactl --bolt /path foo bar`, not
  `hulactl foo --bolt /path bar`.
- **Exit codes:** `0` = success, `1` = command failed, `2` = usage error
  (bad flags / args). Network and auth errors exit `1` with a stderr
  message.
- **Environment variables** consumed by all commands:
  - `HULACTL_NEW_PASSWORD` / `HULACTL_CURRENT_PASSWORD` — password input
    for `set-password`.
  - `HULACTL_SERVER` — overrides the stored server URL.
  - `HULA_*` env vars referenced by individual commands (`HULA_TEAM_BOOTSTRAP_TOKEN`,
    etc.) are listed in their respective sections.

## Quick index

| Group | Commands |
|-------|----------|
| [Authentication & accounts](#authentication-accounts) | `auth`, `authok`, `logout`, `set-password`, `generatehash`, `updateadminhash`, `totp-key`, `totp-setup`, `opaque-seed`, `forget-opaque-record` |
| [Users & access](#users-access) | `createuser`, `modifyuser`, `deleteuser`, `listusers` |
| [Forms & landers](#forms-landers) | `createform`, `modifyform`, `submitform`, `deleteform`, `listforms`, `createlander`, `modifylander`, `deletelander`, `listlanders` |
| [Site builds](#site-builds) | `build`, `build-status`, `builds` |
| [Staging](#staging) | `staging-build`, `staging-update`, `staging-get`, `staging-mount`, `stage`, `commit`, `push`, `pull`, `sync` |
| [Operations](#operations) | `badactors`, `initdb`, `deletedb`, `rotate-cookieless-salt`, `reload` |
| [Agents](#agents) | `create-agent` |
| [HA / Team](#ha-team) | `genteamcerts`, `team-init`, `team-join`, `team-leave`, `team-status`, `team-rotate-bootstrap-token` |

---

## Authentication & accounts

### `auth`

Authenticate against a Hula server and store credentials.

```text
hulactl auth [URL]
```

`URL` can be a full URL or just a hostname (`https://` is assumed).

```bash
hulactl auth hula.example.com
hulactl auth https://hula.example.com:8443
```

Prompts for username (default `admin`) and password. Uses **OPAQUE PAKE** —
the plaintext password never leaves the laptop. If TOTP is enabled, prompts
for the 6-digit code after the password.

Stores the JWT and OPAQUE-derived state in `~/.config/hulactl/hulactl.yaml`.

### `authok`

Confirm stored credentials still authenticate.

```bash
hulactl authok
# ok
```

Exit code `0` if the stored JWT is still valid, non-zero otherwise.

### `logout`

Remove stored credentials from `hulactl.yaml`. No network call.

```bash
hulactl logout
```

### `set-password`

Set or rotate a password via OPAQUE PAKE registration. Defaults to the
`admin` user.

```text
set-password [--username admin] [--provider admin]
```

Prompts for the new password, or reads `HULACTL_NEW_PASSWORD` from env.
On rotation, also prompts for the current password (or reads
`HULACTL_CURRENT_PASSWORD`). On a fresh install, pass an empty current
password:

```bash
HULACTL_CURRENT_PASSWORD='' \
HULACTL_NEW_PASSWORD='strong' \
  hulactl set-password
```

The server stores an OPAQUE registration record; the password itself is
never sent over the wire.

### `generatehash`

Generate an argon2 password hash for the legacy `admin.hash` config field.
Prefer OPAQUE (`set-password`) for new installs.

```bash
hulactl generatehash
# prompts for password, prints argon2id hash
```

### `updateadminhash`

Generate a hash and write it to a Hula config file in place. Convenience
wrapper around `generatehash` for legacy installs.

```text
hulactl --hulaconf /etc/hula/config.yaml updateadminhash
```

The `--hulaconf` flag is required and points at the YAML to update.

### `totp-key`

Generate a base64url-encoded 32-byte key for the
[`totp_encryption_key`](config.md#top-level-keys) config field. At-rest
encryption for TOTP secrets.

```bash
hulactl totp-key
# prints base64url key
```

### `totp-setup`

Set up TOTP for the admin user. Interactive — shows a QR code and a
fallback secret.

```bash
hulactl totp-setup
```

### `opaque-seed`

Generate a base64url OPAQUE OPRF seed and AKE secret for the
[`opaque:`](config.md#opaque) config block. Pin these to prevent fresh
generation on every restart.

```bash
hulactl opaque-seed
# prints two base64url-encoded values
```

### `forget-opaque-record`

**Emergency offline recovery.** Delete an OPAQUE record from a Bolt file
when the live admin password is lost.

```text
hulactl --bolt <path> forget-opaque-record <provider> <username>
```

Hula **must be stopped first** — Bolt is single-writer. Caller is
responsible for copy-out / edit / copy-back.

```bash
# Stop hula, then:
hulactl --bolt /var/hula/data/hula.db forget-opaque-record admin admin
# Restart hula and run set-password to enroll a fresh credential.
```

Flags must come **before** the command name (Go flag convention).

---

## Users & access

### `createuser`

Create an operator user with a per-server access role (`viewer` or
`manager`).

```bash
hulactl createuser '{
  "username": "alice",
  "email": "alice@example.com",
  "role": "manager",
  "servers": ["mysite", "blog"]
}'
```

### `modifyuser`

Modify an existing user (role, servers, email).

### `deleteuser`

Delete a user.

### `listusers`

List configured users with their roles and per-server access.

```bash
hulactl listusers
```

---

## Forms & landers

### `createform`

Create a form (lead-capture endpoint).

```bash
hulactl createform '{
  "name": "newsletter",
  "description": "Newsletter signup",
  "schema": "{\"fields\":[{\"name\":\"email\",\"required\":true}]}"
}'
```

### `modifyform`

Modify an existing form by ID.

```text
modifyform [form ID] [PAYLOAD]
```

Example:

```bash
hulactl modifyform abc123 '{"name":"newname"}'
```

### `submitform`

Submit a form data row as if from a web client. Useful for testing.

```bash
hulactl submitform newsletter '{"email":"alice@example.com"}'
```

### `deleteform`

Delete a form by ID.

### `listforms`

List configured forms.

### `createlander` / `modifylander` / `deletelander` / `listlanders`

Same shape as the form commands — landers are campaign landing pages with
versioned CRUD.

```bash
hulactl createlander '{"name":"launch","url_id":"/launch","redirect":"https://example.com/landed"}'
hulactl listlanders
```

---

## Site builds

Production builds. For staging, see [Staging](#staging) below.

### `build`

Trigger a build and poll until complete.

```text
build <server-id>
```

```bash
hulactl build mysite
# Build started: build-id=ab12cd34
# STATUS=running
# STATUS=complete
```

### `build-status`

Show status of a build by ID.

```bash
hulactl build-status ab12cd34
```

### `builds`

List recent builds for a server (last 10, newest first).

```bash
hulactl builds mysite
```

---

## Staging

When a server is configured with `hula_build: staging`, Hula keeps a
long-lived builder container around. These commands operate against that
container.

### `staging-build`

Trigger a rebuild in the staging container.

```bash
hulactl staging-build mysite
```

### `staging-update`

Upload a single file to the staging site via WebDAV.

```text
staging-update <server-id> <local-file> <remote-path>
```

```bash
hulactl staging-update mysite ./about.md content/about.md
```

### `staging-get`

Download a file from the staging site via WebDAV. Mirror of
`staging-update`. Atomic via temp+rename.

```text
staging-get <server-id> <remote-path> <local-file>
```

### `staging-mount`

Live-sync a local folder with the staging site. Runs until <kbd>Ctrl-C</kbd>.

```text
staging-mount <server-id> <folder-mount-point>
```

| Flag | Description |
|------|-------------|
| `--autobuild` | Trigger a staging build automatically after each sync. |
| `--dangerous` | Allow syncing executables and security-sensitive files (default: refused). |

```bash
hulactl staging-mount mysite ./local-site --autobuild
```

### `stage`

Stage edits in a staging server's git working tree (`git add`).

```text
stage <server-id> [<path> ...]
```

With no `<path>` arguments, stages every change (`git add -A`).
With one or more paths, stages only those. Paths must stay inside the
staging working tree.

The server refuses if the named server isn't `hula_build: staging` or its
`staging_src_dir` isn't a git working tree.

### `commit`

Commit staged edits in a staging server's git working tree.

```text
commit <server-id> <message>
```

Hula appends a `Committed-by: Hula` trailer to the message on its own line.

| Flag | Default | Description |
|------|---------|-------------|
| `--author-name` | `hula-staging` | Override committer name. |
| `--author-email` | `staging@hula.local` | Override committer email. |

### `push`

Push HEAD of `staging_src_dir` to `origin/<branch>`, where `<branch>` is
configured under `root_git_autodeploy.ref.branch`.

```bash
hulactl push mysite
```

Auth credentials come from the same `root_git_autodeploy.creds` block —
make sure those env vars are still in scope.

### `pull`

Pull `origin/<branch>` updates into the staging working tree (rebase on
top, rewind on conflict).

```bash
hulactl pull mysite
```

Refuses if the working tree has uncommitted edits — commit them first.
On a rebase conflict, Hula automatically rewinds the tree to the pre-pull
HEAD so the served site keeps working; you'll see a `rewound to <SHA>`
notice in the output.

### `sync`

Pull then push in a single server-side operation.

```bash
hulactl sync mysite
```

If the pull rebase conflicts, Hula rewinds and reports the conflict — push
is not attempted. If pull succeeds but push is rejected, Hula rewinds the
working tree to the pre-sync HEAD so the served site reverts to its
known-good state, then reports the push failure.

---

## Operations

### `badactors`

List scored IPs with their score and block status.

```bash
hulactl badactors
# IP            score   blocked   last seen
# 1.2.3.4       57      yes       2026-05-07 14:22:01
# ...
```

### `initdb`

Initialise the analytics schema (ClickHouse). Idempotent — safe to re-run.
Mostly used during first-boot bootstrap.

```bash
hulactl initdb
```

### `deletedb`

**Destructive.** Drop the analytics schema. Confirm prompt.

```bash
hulactl deletedb
```

### `rotate-cookieless-salt`

Replace the cookieless visitor-id salt for a server. Yesterday's visitors
become unrecognisable today — this is the correct behaviour for
"wipe everyone".

```text
hulactl --bolt <path> rotate-cookieless-salt <server_id>
```

Hula **must be stopped first** (Bolt single-writer).

```bash
hulactl --bolt /var/hula/data/hula.db rotate-cookieless-salt mysite
```

### `reload`

Send `SIGHUP` to the running Hula process to reload `config.yaml` without
restart. See [Reload semantics](config.md#reload-semantics) for which
changes are picked up in-place.

```bash
hulactl reload
```

---

## Agents

### `create-agent`

Generate an mTLS-secured agent config YAML for `hula-agent`. Two ways to
declare permissions:

```text
create-agent [-c template.yaml] [--allow-<verb>=<site>[,opts]]... [--expires-in=DUR] [--hula-host=HOST] > agent.yaml
```

**Flag form:**

```bash
hulactl create-agent \
    --allow-build=docs \
    --allow-staging-build=docs \
    --expires-in=1yr \
    > docs-ci-agent.yaml
```

**Template form:**

```bash
hulactl create-agent -c agent-template.yaml > agent.yaml
```

The two compose: flags override template entries.

| Flag | Description |
|------|-------------|
| `-c <path>` | Read an `agent-template.yaml` for permissions and config. |
| `--allow-<verb>=<site>[,opts]` | Grant `<verb>` on `<site>` with optional opaque option string. Repeatable. Verbs: `build`, `staging-build`, `pull`, `push`, `sync`, `commit`, `push-file`, `get-file`. |
| `--expires-in <DUR>` | Cert validity. Accepts Go durations (`8760h`), days (`30d`), or years (`1yr`). |
| `--hula-host <host>` | Override the `hula_host` written into the agent YAML. |
| `--offline` | Phase-1 mode: generate a one-off Agent CA without talking to a server. |

Output is YAML on stdout; redirect to a secret-managed file. See the
[Agent YAML reference](agent-yaml.md) for the schema.

---

## HA / Team

Multi-node Raft cluster commands. Solo deployments don't need any of these
— Hula auto-bootstraps a single-node cluster on first boot.

### `genteamcerts`

Generate a Team CA + per-node mTLS bundle + bootstrap token. Offline
ceremony.

```text
hulactl genteamcerts --nodes <id1>,<id2>,... [--team-id <uuid>] [--validity 365d] [--out ./team-bundles]
```

Produces:

```text
<out>/ca.pem            # deploy to every node
<out>/ca.key            # operator-secured; do NOT deploy
<out>/bootstrap-token   # 32 random bytes, base64
<out>/team-id
<out>/<node-id>/{cert.pem,key.pem,ca.pem}
```

Distribute per-node bundles + bootstrap token out-of-band (secrets manager).

### `team-init`

Generate `team_id` + `bootstrap_token` bytes (offline). Operator stuffs
both into the seed node's config before first boot.

```bash
hulactl team-init
```

Doesn't talk to a running Hula.

### `team-join`

Join this node to an existing team via the leader's `MembershipService`.

```text
hulactl team-join <leader-addr> --token <bootstrap-token> --pki-dir <dir>
```

| Flag | Description |
|------|-------------|
| `--token` | The team's bootstrap token (base64). |
| `--pki-dir` | Local dir holding `ca.pem`, `cert.pem`, `key.pem`. |
| `--node-id` | Override the joining node's ID (default: hostname). |
| `--node-hostname` | Operator-provisioned per-node hostname for chat-WS pinning. |

### `team-leave`

Remove a node from the team.

```text
hulactl team-leave <leader-addr> [<node-id>] --pki-dir <dir>
```

### `team-status`

Print the team's membership table.

```text
hulactl team-status <node-addr> --pki-dir <dir> [--team-id <uuid>]
```

`--team-id` hard-exits when the polled node belongs to a different team —
useful as a guard in scripts.

### `team-rotate-bootstrap-token`

Generate a fresh bootstrap token and write it to the team's Raft FSM. Must
be run against the current leader.

```bash
hulactl team-rotate-bootstrap-token <leader-addr> --pki-dir <dir>
```

Existing nodes are unaffected; update `HULA_TEAM_BOOTSTRAP_TOKEN` before
issuing any new `team-join`.

---

## Inline help

Run `hulactl` with no arguments to print the full command list with
one-line descriptions. Run `hulactl <cmd>` with no arguments (where
applicable) to print the per-command usage block.

```bash
hulactl              # full list
hulactl create-agent # detailed usage for create-agent
```
