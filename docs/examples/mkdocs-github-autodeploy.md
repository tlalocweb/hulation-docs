# MkDocs site with GitHub auto-deploy

Goal: a public MkDocs site, hosted on GitHub, that rebuilds and redeploys on
every push to `main` — with HTTPS, no nginx, no separate CI pipeline. Hula
clones the repo, runs `mkdocs build`, and serves the output.

## Prereqs

- A Hula server running and reachable on ports 80 and 443. If you don't have
  one yet, see [One-server quick start](../quickstart/one-server.md) — five
  minutes.
- A MkDocs site in a Git repo. The repo can be public or private; private
  repos need a personal access token or deploy key.
- DNS A record for the docs domain pointed at the server.

## Step 1 — server config

Add a server block to `config.yaml`:

```yaml
servers:
  - host: docs.example.com
    id: docs
    root_git_autodeploy:
      repo: https://github.com/myorg/my-docs.git
      ref:
        branch: main
      hula_build: production
```

Reload Hula:

```bash
hulactl reload
```

That's the minimum. With no `.hula/sitebuild.yaml` in the repo, Hula
auto-detects MkDocs from `mkdocs.yml` and runs:

```text
WORKDIR /builder
MKDOCS build --strict --site-dir _hula_out
FINALIZE /builder/site/_hula_out
```

For a basic mkdocs-material site, that's all you need.

## Step 2 — trigger and verify

```bash
hulactl build docs
```

Wait for `STATUS=complete`, then:

```bash
curl -sI https://docs.example.com/ | head -1
# HTTP/2 200
```

Subsequent pushes to `main` rebuild automatically.

## Pinning versions

The default builder image ships pinned versions (see
[builder-images/README.md](https://github.com/tlalocweb/hulation/blob/main/builder-images/README.md)
for the current pins). Sites that need different versions declare them in
`.hula/sitebuild.yaml`:

```yaml
mkdocs:
  version: "1.6.1"
  material: "9.5.49"
  extra_packages:
    - pymdown-extensions==10.11.2

configs:
  production:
    commands: |
      WORKDIR /builder
      MKDOCS build --strict --site-dir _hula_out
      FINALIZE /builder/site/_hula_out
```

Hula synthesises a derived image, content-hash-cached, that pip-installs
the pinned versions over the base builder. The cache means subsequent
builds are fast; only the first build after a pin change pays for the
derive step.

## Self-hosted Mermaid

mkdocs-material supports Mermaid via `pymdownx.superfences`. To avoid
loading from a CDN at runtime:

1. Vendor `mermaid.min.js` into `docs/assets/javascripts/`.
2. Reference it in `mkdocs.yml`:

   ```yaml
   markdown_extensions:
     - pymdownx.superfences:
         custom_fences:
           - name: mermaid
             class: mermaid
             format: !!python/name:pymdownx.superfences.fence_code_format

   extra_javascript:
     - assets/javascripts/mermaid.min.js
   ```

3. Use it:

   ````markdown
   ```mermaid
   sequenceDiagram
       Client->>Hula: HTTPS request
       Hula->>Builder: trigger build
   ```
   ````

No `extra_packages` needed — the integration is built into mkdocs-material.
Do **not** install `mkdocs-mermaid2-plugin`; that plugin is for non-Material
themes and conflicts with the Material superfences integration.

## Private repo

GitHub PAT in `config.yaml`:

```yaml
servers:
  - host: docs.example.com
    id: docs
    root_git_autodeploy:
      repo: https://github.com/myorg/private-docs.git
      creds:
        username: x-access-token
        password: ${GITHUB_TOKEN}
      ref:
        branch: main
      hula_build: production
```

`${GITHUB_TOKEN}` is read from the environment of the Hula process at
config-load time. Rotate by setting a new env var and `hulactl reload`.

## Live staging via WebDAV

For a docs writer who wants to edit and see the result without a Git
push, switch the deploy mode and use `staging-mount`:

```yaml
hula_build: staging
```

Then on a laptop:

```bash
hulactl staging-mount docs ./local-edit --autobuild
```

Edits to `./local-edit/` sync to the staging container, which auto-rebuilds
on every change. `staging-mount` syncs both directions and refuses to upload
executables without `--dangerous`. See
[Live staging via WebDAV](./live-staging-webdav.md) for the full flow.

## Common gotchas

**`mkdocs build --strict` fails on warnings you didn't see locally.**
Material's `--strict` mode promotes warnings to errors. Run it locally first:
`mkdocs build --strict`. Common offenders: broken cross-links, missing nav
entries, malformed YAML in front-matter.

**Mermaid diagrams render as code blocks.** You forgot the
`pymdownx.superfences` config block. The fence is `mermaid`, not `mmd`.

**Wrong theme: site builds but looks like the default mkdocs theme.**
`theme: { name: material }` is required in `mkdocs.yml` — a missing theme
block falls back to the built-in `mkdocs` theme. mkdocs-material is
installed but only used when explicitly selected.

**Third-party theme not found.** Themes other than `mkdocs`, `readthedocs`,
and `material` need to be installed via `extra_packages`:

```yaml
mkdocs:
  extra_packages:
    - mkdocs-cinder
```

**Build fails: `mkdocs: not found`.** The sitebuild.yaml's `commands:` is
calling `mkdocs` but the builder image is one that doesn't ship it. Both
default builder images ship MkDocs; if you're using a custom builder image,
add `RUN pip3 install mkdocs` to the Dockerfile or `extra_packages` to
`sitebuild.yaml`.

**Permission to merge `sitebuild.yaml` is permission to run code on the
builder.** `extra_packages`, `dockerfile_prebuild`, and `RUN` commands all
execute in the build container — review changes the same way you review
Dockerfiles.
