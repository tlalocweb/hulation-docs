# Hugo Quick Start

Deploy a Hugo site on Hula with automatic HTTPS and Git auto-deploy. Most visitors
to this site land here — it's the primary flow.

!!! note "Prerequisite: a running Hula server"

    The Hugo flow assumes Hula is already running on a Linux host with public IP
    and ports 80 / 443 reachable. If you don't have one yet:

    ```bash
    curl -fsSL https://raw.githubusercontent.com/tlalocweb/hulation/main/install.sh | bash
    ```

    Five-minute version covered in detail in [One-server Quick Start](one-server.md).

## What you'll set up

- A `servers:` block in `config.yaml` that points Hula at your Hugo repo.
- Auto-rebuild on every push to `main` (or whatever branch you pick).
- Optional: live staging via WebDAV so writers can edit without a Git push.

## Step 1 — confirm your Hugo repo

Hula clones your repo into the builder container and runs `hugo --minify`. The
repo needs to be a standard Hugo site — `hugo.toml` (or `config.toml`) at the
root. Anything you'd build locally with `hugo` works.

For a public repo, no credentials are needed. For a private repo, you'll need a
GitHub PAT or deploy key — covered in
[Hugo + private repo](../examples/hugo-private-repo.md).

## Step 2 — server config

Add a server block to `config.yaml` on the Hula host:

```yaml
servers:
  - host: example.com
    aliases:
      - www.example.com
    id: example
    ssl:
      acme:
        email: you@example.com
        cache_dir: /var/hula/certs
    root_git_autodeploy:
      repo: https://github.com/myorg/example-site.git
      ref:
        branch: main
      hula_build: production
      build_env:
        - HUGO_ENV=production
```

Reload Hula to pick up the new config:

```bash
hulactl reload
```

That's the minimum. Hula auto-detects Hugo from `hugo.toml` (or `config.toml`)
in the repo root and runs:

```text
WORKDIR /builder
HUGO --minify
FINALIZE /builder/site/public
```

For a typical Hugo site, this is all you need.

## Step 3 — trigger and verify

Trigger the first build:

```bash
hulactl build example
```

Wait for `STATUS=complete`, then:

```bash
curl -sI https://example.com/ | head -1
# HTTP/2 200
```

Subsequent pushes to `main` rebuild automatically. To trigger an out-of-band
rebuild any time:

```bash
hulactl build example
hulactl build-status <build-id>
hulactl builds example   # last 10 builds, newest first
```

## Pinning the Hugo version

The default builder image ships an extended Hugo build. To pin per-site, drop a
`.hula/sitebuild.yaml` in the repo:

```yaml
hugo:
  at_least: "0.147.0"

configs:
  production:
    commands: |
      WORKDIR /builder
      HUGO --minify
      FINALIZE /builder/site/public
```

The `hugo:` block declares a version constraint. Today the constraint is parsed
but not enforced at the per-site level; if you need a specific Hugo version, use
the builder-image override path described in
[`builder-images/README.md`](https://github.com/tlalocweb/hulation/blob/main/builder-images/README.md).

## Live staging via WebDAV (optional)

When `hula_build: staging`, Hula keeps a long-lived builder container around so
you can edit the site as if it were a local folder:

```bash
# One-shot file upload
hulactl staging-update example /tmp/about.md content/about.md
hulactl staging-build example

# Live sync — runs until Ctrl-C; rebuilds automatically on every change
hulactl staging-mount example ./local-site --autobuild
```

`staging-mount` syncs both directions, debounces rapid edits, and refuses to
upload executables or security-sensitive files unless `--dangerous` is set.

Full walkthrough in [Live staging via WebDAV](../examples/live-staging-webdav.md).

## Troubleshooting

**Build fails: `hugo: not found`.**
The builder image is one that doesn't ship Hugo, or the wrong builder is being
used. Default is `hula-builder-default` (alpine), which ships Hugo extended. If
you've overridden `builder_image` in `sitebuild.yaml`, point it back to the
default or add Hugo via `dockerfile_prebuild`.

**`git clone` fails with 401 / 403.**
Private repo without credentials. Add `creds:`:

```yaml
root_git_autodeploy:
  repo: https://github.com/myorg/private-site.git
  creds:
    username: x-access-token
    password: ${GITHUB_TOKEN}
  ref:
    branch: main
```

`${GITHUB_TOKEN}` is read from the environment of the Hula process at
config-load time. Restart or `hulactl reload` after rotating.

**ACME challenge fails: `403 / no resource`.**
Port 80 must be reachable from the internet for HTTP-01 challenges. Behind a
reverse proxy, set `ssl.acme.http_port` to whatever port the proxy forwards
challenge traffic to.

**Theme submodule not pulled.**
Hula does a shallow clone by default. For Hugo themes that live as Git submodules,
add to `sitebuild.yaml`:

```yaml
configs:
  production:
    commands: |
      WORKDIR /builder
      RUN git submodule update --init --recursive
      HUGO --minify
      FINALIZE /builder/site/public
```

## Next steps

- [Configuration reference](../reference/config.md) — every key in `config.yaml`.
- [`hulactl` reference](../reference/hulactl.md) — full CLI surface.
- [Hugo + private repo](../examples/hugo-private-repo.md) — PAT, deploy key.
- [Behind Cloudflare](../examples/cloudflare-origin-ca.md) — Origin CA setup.
