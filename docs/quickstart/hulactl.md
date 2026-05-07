# `hulactl` Quick Start (macOS / Linux)

Install the `hulactl` CLI on your laptop and authenticate against an existing
Hula server. The audience here is a human operator — for CI / deploy-bots, see
[hulaagent Quick Start](hulaagent.md) instead.

## Prerequisites

- A running Hula server you have admin (or operator) credentials for.
- macOS or Linux laptop, amd64 or arm64.
- ~50 MB free disk for the binary.

## Step 1 — install

```bash
curl -fsSL https://raw.githubusercontent.com/tlalocweb/hulation/main/installtools.sh | bash
```

Drops `hulactl` into `~/.local/bin/`. Override with `INSTALL_DIR`:

```bash
# Install to /usr/local/bin (requires sudo)
INSTALL_DIR=/usr/local/bin \
  curl -fsSL https://raw.githubusercontent.com/tlalocweb/hulation/main/installtools.sh | sudo bash
```

Pin a specific version:

```bash
HULA_VERSION=v0.20.0-pre1 \
  curl -fsSL https://raw.githubusercontent.com/tlalocweb/hulation/main/installtools.sh | bash
```

Confirm:

```bash
hulactl --version
```

If `hulactl: command not found`, your shell PATH doesn't include `~/.local/bin/`.
Add it to your shell rc file:

```bash
echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.zshrc   # or ~/.bashrc
exec $SHELL -l
```

## Step 2 — authenticate

```bash
hulactl auth https://hula.example.com
```

Prompts for username (defaults to `admin`) and password. Authentication uses
**OPAQUE PAKE** — the plaintext password never leaves your laptop, even during
first login. The server only ever sees PAKE-derived material.

If TOTP 2FA is enabled on your account, you'll be prompted for the 6-digit code
after the password.

Credentials are stored in `~/.config/hulactl/hulactl.yaml` — a JWT (and the
OPAQUE-derived state) Hulactl needs for subsequent calls.

## Step 3 — sanity check

```bash
hulactl authok
# ok
```

Confirms the stored credentials still work against the server. Run this any time
you suspect the JWT has expired.

## Step 4 — what you can now do

Five commands worth knowing right away:

```bash
# What sites does this Hula serve?
hulactl listforms          # forms (lead-capture endpoints)
hulactl listlanders        # landers (campaign landing pages)

# Build a site
hulactl build <server-id>
hulactl build-status <build-id>
hulactl builds <server-id>     # last 10 builds

# Mount a remote staging site as a local folder
hulactl staging-mount <server-id> ./local-site --autobuild

# Who's been probing the server lately?
hulactl badactors

# Reload config without restart
hulactl reload
```

Full surface in the [`hulactl` reference](../reference/hulactl.md).

## Step 5 — rotate / logout

Rotate your password (OPAQUE — current and new never travel as plaintext):

```bash
hulactl set-password
# prompts for current, then new
```

Or non-interactively:

```bash
HULACTL_CURRENT_PASSWORD='current' \
HULACTL_NEW_PASSWORD='new-strong-password' \
  hulactl set-password
```

Remove stored credentials when you're done with this server:

```bash
hulactl logout
```

## macOS gotchas

**Gatekeeper blocks the unsigned binary on first run.**

```bash
xattr -d com.apple.quarantine ~/.local/bin/hulactl
```

Or right-click the binary in Finder and choose Open once. Subsequent runs
work normally.

**Homebrew tap.** None yet. The release tarball is the supported install path.

## Linux gotchas

**`~/.local/bin` not on PATH** on some minimal distros (Alpine without
`shadow-login`, Debian non-interactive shells). Add it manually as in Step 1.

**`Permission denied` on the binary** — `chmod +x ~/.local/bin/hulactl`. The
installer should set this, but a non-default umask can interfere.

## Next steps

- [`hulactl` reference](../reference/hulactl.md) — every command, flag, env var.
- [hulaagent Quick Start](hulaagent.md) — for CI runners that can't hold a JWT.
- [Configuration reference](../reference/config.md) — what's behind each `hulactl` command.
