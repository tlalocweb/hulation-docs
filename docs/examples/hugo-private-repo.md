# Hugo + private repo

Deploy a Hugo site from a private GitHub repo. Two auth options: a
fine-grained personal access token (recommended) or an SSH deploy key.

## Personal access token (recommended)

GitHub PAT scope requirements:

| Scope | Why |
|-------|-----|
| `Contents: read` | Clone the repo and read commits. |
| `Metadata: read` | Required by GitHub for any access. |

Use a **fine-grained** PAT scoped to the single repo, not a classic PAT
with `repo` (which grants access to every private repo the token's owner
can see).

### Server config

```yaml
servers:
  - host: example.com
    id: example
    ssl:
      acme:
        email: admin@example.com
        cache_dir: /var/hula/certs
    root_git_autodeploy:
      repo: https://github.com/myorg/private-site.git
      creds:
        username: x-access-token
        password: ${GITHUB_TOKEN}
      ref:
        branch: main
      hula_build: production
```

The `username: x-access-token` is required by GitHub when authenticating
with a PAT — the token is the password.

### Setting the env var

The `${GITHUB_TOKEN}` placeholder is resolved against the **Hula
process's environment** at config-load time. Where to set it depends on
how Hula is launched:

=== "start-with-docker.sh"

    ```bash
    GITHUB_TOKEN='ghp_...' ./start-with-docker.sh
    ```

=== "Docker run directly"

    ```bash
    docker run -d \
      --name hula \
      -e GITHUB_TOKEN='ghp_...' \
      ...
    ```

=== "systemd"

    ```ini
    [Service]
    Environment=GITHUB_TOKEN=ghp_...
    EnvironmentFile=/etc/hula/secrets.env
    ```

After updating the env var, `hulactl reload` re-resolves the substitution
without restarting.

### Rotating the PAT

1. Mint a new token in GitHub with the same scopes.
2. Update the env var (edit `EnvironmentFile`, etc.).
3. `hulactl reload` to re-resolve.
4. Confirm the next build succeeds: `hulactl build example`.
5. Revoke the old token in GitHub.

The overlap window prevents downtime — both tokens are valid until
step 5.

## Deploy key

For organisations that prefer SSH or want machine-bound credentials.

### Generate the key on the Hula host

```bash
ssh-keygen -t ed25519 -f /etc/hula/keys/example-deploy -N "" -C "hula-deploy-example"
```

`chmod 600 /etc/hula/keys/example-deploy` — Hula's process needs read
access; nothing else does.

### Add the public key to GitHub

In the GitHub repo: **Settings → Deploy keys → Add deploy key**. Paste
the contents of `/etc/hula/keys/example-deploy.pub`. **Allow write
access** is **not** required for autodeploy (read-only is enough); leave
it off unless you'll also be pushing back via `hulactl push`.

### Server config

```yaml
servers:
  - host: example.com
    id: example
    root_git_autodeploy:
      repo: git@github.com:myorg/private-site.git
      creds:
        ssh_key_path: /etc/hula/keys/example-deploy
      ref:
        branch: main
      hula_build: production
```

The `git@github.com:` URL form is required for SSH; HTTPS URLs ignore
SSH credentials.

### Host-key verification

On first contact with `github.com`, Hula populates `~/.ssh/known_hosts`
inside the build container. To pre-seed (avoid the trust-on-first-use
window):

```bash
ssh-keyscan -t rsa,ecdsa,ed25519 github.com > /etc/hula/keys/known_hosts
```

Then mount it via `/etc/hula/keys` (already covered by the typical
`-v /etc/hula:/etc/hula:ro` mount).

## Triggering a build

Same as the [public-repo flow](hugo-github-autodeploy.md):

```bash
hulactl build example
# Build started: build-id=...
```

Auto-rebuild on push works identically.

## Troubleshooting

**`fatal: could not read Username for 'https://github.com'`.** The
PAT username is wrong. GitHub requires `x-access-token` (literally that
string) when authenticating with a PAT — see the snippet above.

**`Permission denied (publickey)`.** SSH deploy key isn't being read,
or wasn't added to the repo's deploy keys. Test from the Hula host:

```bash
docker run --rm -v /etc/hula/keys:/keys:ro alpine/git \
  ssh -i /keys/example-deploy -o StrictHostKeyChecking=no \
  -T git@github.com
# Expected: "Hi <repo>! You've successfully authenticated..."
```

**Token works locally but not in Hula.** The env var isn't getting into
the container. `docker exec hula env | grep GITHUB_TOKEN` to confirm —
if absent, fix the launcher's environment plumbing.

**PAT expired.** GitHub fine-grained PATs default to 30 days. Either
extend the expiry, switch to a deploy key, or set up a calendar reminder
for rotation.

## Next

- [Hugo Quick Start](../quickstart/hugo.md) — full walk-through.
- [Hugo + GitHub autodeploy](hugo-github-autodeploy.md) — public-repo
  variant.
- [Live staging via WebDAV](live-staging-webdav.md) — edit live without
  a Git push.
