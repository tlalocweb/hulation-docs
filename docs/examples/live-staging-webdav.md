# Live staging via WebDAV

Edit a remote site as if it were a local folder. `hulactl staging-mount`
keeps a local directory in sync with a Hula staging server, debounces
rapid edits, and (with `--autobuild`) triggers a rebuild on every save —
useful for content writers who'd rather use VS Code than `git push`.

## When to use this

- A non-developer (writer, designer) needs to preview changes against a
  live site before merging.
- You want a fast edit-build-preview loop without committing every keystroke.
- You need to push small content tweaks (`content/about.md`) without
  scoping a new branch.

For pure developer flow with version control, just `git push` against
the [Hugo auto-deploy](hugo-github-autodeploy.md) — no staging needed.

## Server config (staging mode)

The site server runs in `hula_build: staging` instead of `production`:

```yaml
servers:
  - host: staging.example.com
    id: example-staging
    ssl:
      acme:
        email: admin@example.com
        cache_dir: /var/hula/certs
    root_git_autodeploy:
      repo: https://github.com/myorg/example-site.git
      creds:
        username: x-access-token
        password: ${GITHUB_TOKEN}
      ref:
        branch: staging
      hula_build: staging
      committer:
        name: writer
        email: writer@example.com
```

**Differences from production mode:**

- Hula keeps a long-lived builder container around (no per-build spin-up
  cost — rebuilds are fast).
- The cloned source lives in `staging_src_dir` and is **editable** via
  WebDAV; production's source is rebuilt fresh on every clone.
- `hulactl push` writes back to the Git remote — staging is the
  authoring path.

After updating, `hulactl reload`. Hula brings up the staging container
on first request (or immediately if `no_pull_on_start: false`).

## Mount on the laptop

```bash
hulactl staging-mount example-staging ~/sites/example
```

What this does:

1. Bidirectional sync between the local folder and Hula's
   `staging_src_dir` over WebDAV.
2. Watches both sides for changes; debounces ~500 ms before syncing.
3. With `--autobuild`, triggers `hulactl staging-build` on every change
   that lands.

Runs until <kbd>Ctrl-C</kbd>. The local folder doesn't have to exist
beforehand — Hula populates it on first sync.

```bash
hulactl staging-mount example-staging ~/sites/example --autobuild
```

## Auto-build behaviour

With `--autobuild`:

- A change synced **from local → remote** triggers a build.
- A change synced **from remote → local** does **not** (avoids loops).
- Builds debounce: rapid saves coalesce into one rebuild.

The site at `https://staging.example.com` reflects the latest build.
Watch the live log:

```bash
hulactl staging-mount example-staging ~/sites/example --autobuild --verbose
# [sync] uploaded content/about.md
# [build] STATUS=running
# [build] STATUS=complete
# [sync] downloaded content/_index.md   (someone else edited remotely)
```

## Safety filter

`staging-mount` refuses to upload by default:

- Executables (mode `+x`).
- `.git/`, `.ssh/`, `.env`, `*.key`, `*.pem`.
- Anything matching the configured `staging_blocklist` (operator-side).

Override with `--dangerous`:

```bash
hulactl staging-mount example-staging ~/sites/example --dangerous
```

Use rarely — the filter exists so an editor that auto-saves a backup of
your shell history doesn't end up on the staging server.

## Committing and pushing

Once the staged tree is in good shape, commit and push back:

```bash
hulactl stage example-staging
hulactl commit example-staging "feat: tweak hero copy"
hulactl push example-staging
```

Or one shot (after staging):

```bash
hulactl commit example-staging "tweak hero copy"
hulactl sync example-staging
```

`sync` is `pull --rebase` then `push`, server-side, with automatic
rewind on conflict.

The committer name / email come from `committer:` in the server config
(default `hula-staging` / `staging@hula.local`). Override per-commit:

```bash
hulactl commit example-staging "..." \
    --author-name "Alice Editor" \
    --author-email alice@example.com
```

## Conflict resolution

When the remote tree changes underneath you (someone else edited via
WebDAV, or `hulactl pull` brought down upstream changes):

- **Local clean, remote ahead.** `staging-mount` downloads the changes —
  your local folder updates, build triggers, no editor confusion.
- **Local ahead, remote ahead.** `staging-mount` reports the conflict
  and refuses to sync the diverging file. Resolve by hand (favoring
  whichever copy is correct), then re-save to retry.
- **Pull / rebase conflict during `sync`.** The server-side `pull`
  rewinds to the pre-pull HEAD; the staged site keeps serving the last
  good content; `sync` reports the conflict in stderr.

`staging-mount` never silently overwrites diverging content. The
worst-case is "edit got rejected, fix it manually."

## Multi-writer caveats

Two writers running `staging-mount` against the same staging server
work, with caveats:

- Each writer sees the other's edits via the remote→local direction.
- Last save wins on rapid simultaneous edits to the same file (~500 ms
  window).
- Build queueing: rapid alternating saves coalesce into bursts of 1–2
  rebuilds; the staging container doesn't fall behind.

For more than 2–3 concurrent writers, prefer Git branches with separate
staging servers per writer.

## What gets uploaded

The sync targets `staging_src_dir` — the **source** tree, not the built
output. So:

- `content/`, `themes/`, `assets/`, `static/`, `data/`, `layouts/` — yes.
- `public/`, `resources/_gen/` — no, ignored (they're build artefacts).
- `.git/` — no (filtered by safety rules).
- `node_modules/` — no by default; toggle via the operator-side
  `staging_blocklist` if your build needs them.

## Stopping the mount

<kbd>Ctrl-C</kbd>. The local folder remains; the next `staging-mount`
picks up where you left off.

## Troubleshooting

**`webdav: 401 unauthorized`.** Stored credentials expired. Run
`hulactl auth` to refresh.

**Edits don't appear on `https://staging.example.com`.** Either
`--autobuild` isn't set, or the build is queued behind a previous one.
Check `hulactl staging-build example-staging` runs cleanly.

**`refused to upload <file>: matches blocklist`.** Safety filter caught
it. Either rename the file or pass `--dangerous` if you really mean it.

**`upload-no-progress` for minutes on a large file.** Hula doesn't
chunk WebDAV PUTs — single-shot uploads are bounded by the configured
`max_upload_size`. For files > 100 MB, consider committing to git
instead.

**Two writers both see "remote-ahead" all the time.** Clock skew
between laptops can confuse the sync; ensure both have NTP. Or use
separate per-writer staging servers.

## Next

- [Hugo Quick Start](../quickstart/hugo.md) — production flow.
- [`hulactl staging-mount`](../reference/hulactl.md#staging-mount) —
  full flag reference.
- [Local dev loop with hulaagent](hulaagent-local-dev.md) — same shape
  but driven by `hula-agent` over HLAP.
