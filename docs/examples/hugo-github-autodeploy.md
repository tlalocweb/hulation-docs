# Hugo + GitHub auto-deploy

Goal: a public Hugo site, hosted on GitHub, that rebuilds and redeploys
on every push to `main` — with HTTPS, no nginx, no separate CI pipeline.

This is the search-friendly landing page for the most common Hula use
case. The fuller walk-through with troubleshooting lives in the
[Hugo Quick Start](../quickstart/hugo.md).

## Prereqs

- A Hula server reachable on ports 80 and 443. If you don't have one,
  the [One-server Quick Start](../quickstart/one-server.md) sets one up
  in five minutes.
- A Hugo site in a public GitHub repo with `hugo.toml` (or `config.toml`)
  at the root.
- DNS A record for the docs domain pointed at the Hula host.

## Server config

```yaml
servers:
  - host: example.com
    aliases:
      - www.example.com
    id: example
    ssl:
      acme:
        email: admin@example.com
        cache_dir: /var/hula/certs
    root_git_autodeploy:
      repo: https://github.com/myorg/example-site.git
      ref:
        branch: main
      hula_build: production
      build_env:
        - HUGO_ENV=production
```

Reload Hula:

```bash
hulactl reload
```

## First build

Trigger and poll:

```bash
hulactl build example
# Build started: build-id=ab12cd34
# STATUS=cloning
# STATUS=running
# STATUS=complete
```

Verify HTTPS:

```bash
curl -sI https://example.com/ | head -1
# HTTP/2 200
```

Subsequent pushes to `main` trigger automatic rebuilds — Hula polls the
remote periodically (default: every minute).

## Submodules

If your theme is a Git submodule, override the default profile by
dropping `.hula/sitebuild.yaml` into the **site** repo:

```yaml
configs:
  production:
    commands: |
      WORKDIR /builder
      RUN git submodule update --init --recursive
      HUGO --minify
      FINALIZE /builder/site/public
```

## Pinning Hugo

The default builder image ships Hugo extended (currently
`0.147.6` — see
[`builder-images/README.md`](https://github.com/tlalocweb/hulation/blob/main/builder-images/README.md)).
For a different version, use a custom builder image — the `hugo:` block
in `sitebuild.yaml` parses but is not yet wired into per-site overrides.

## What it costs

- One ephemeral builder container per build (auto-removed).
- One persistent volume per server for cloned source + last-known-good
  output (`/var/hula/sitedeploy/{{serverid}}/`).
- Bandwidth: full clone on first build, shallow incremental thereafter.

## Common pitfalls

**`hugo: not found` in build log.** You're using a non-default builder
image. Either point `builder_image:` back at `default` or add Hugo via
`dockerfile_prebuild`.

**`git clone` 401.** Repo isn't public. Switch to the
[private-repo flow](hugo-private-repo.md).

**ACME pending.** First request to `https://example.com` triggers cert
issuance. Watch the log: `acme: certificate issued for example.com`. If
you see `connection refused on port 80`, port 80 isn't reachable from
the public internet.

## Next

- [Hugo + private repo](hugo-private-repo.md) — PAT and deploy-key flows.
- [Live staging via WebDAV](live-staging-webdav.md) — edit in place
  without a Git push.
- [Behind Cloudflare](cloudflare-origin-ca.md) — Origin CA mode.
