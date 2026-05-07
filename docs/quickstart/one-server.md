# One-server Quick Start (Docker)

A single Hula container with automatic HTTPS, serving a static `public/` directory.
For Hugo sites that auto-deploy from Git, see [Hugo Quick Start](hugo.md) — start here
if you just want HTTPS in front of static files.

## Prerequisites

- **Docker Engine 20.10+** ([install](https://docs.docker.com/engine/install/) or
  `curl -fsSL https://get.docker.com | sh`).
- A **Linux host** with public IP.
- **Ports 80 and 443** reachable from the internet. Port 80 is required for
  ACME / Let's Encrypt HTTP-01 challenges.
- **DNS A record** pointing at the host.

## Step 1 — install

```bash
curl -fsSL https://raw.githubusercontent.com/tlalocweb/hulation/main/install.sh | bash
```

Creates `./hula/`, pulls the Hula and ClickHouse images, drops a default
`config.yaml`, and starts both containers.

Customise via env vars:

```bash
HULA_PORT=443 HULA_DIR=/opt/hula \
  curl -fsSL https://raw.githubusercontent.com/tlalocweb/hulation/main/install.sh | bash
```

## Step 2 — set the admin password

OPAQUE PAKE registration. The plaintext password never leaves the laptop / shell,
even on first set:

```bash
cd hula
HULACTL_CURRENT_PASSWORD='' \
HULACTL_NEW_PASSWORD='your-strong-password' \
  ./hulactl set-password
```

The empty `HULACTL_CURRENT_PASSWORD=''` is required on a fresh install. Subsequent
rotations need the actual current password.

## Step 3 — edit config.yaml

```bash
nano config.yaml
```

Minimal config for a single static site with ACME:

```yaml
jwt_key: "change-me-to-a-long-random-string"

port: 443

ssl:
  acme:
    email: you@example.com
    cache_dir: /var/hula/certs

servers:
  - host: example.com
    aliases:
      - www.example.com
    id: mysite
    root: /var/hula/public

cors:
  allow_credentials: true

dbconfig:
  host: hula-clickhouse
  port: 9000
  user: hula
  pass: hula
  dbname: hula
```

Key fields:

- `host` / `aliases` — every domain that should resolve to this site.
- `id` — short opaque ID used in CLI commands and analytics rows.
- `root` — directory inside the container where static files live. Maps to
  `./public` on the host.
- `ssl.acme.email` — Let's Encrypt expiry-notice address.

## Step 4 — drop your static files

```bash
cp -r /path/to/your/site/public/* ./public/
```

The `public/` directory maps to `/var/hula/public` inside the container.

## Step 5 — open the firewall

```bash
# Ubuntu/Debian (ufw)
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp

# RHEL/Fedora (firewalld)
sudo firewall-cmd --permanent --add-service=http --add-service=https
sudo firewall-cmd --reload
```

Cloud provider security groups need the same — consult your provider.

## Step 6 — restart on 80/443

```bash
./start-with-docker.sh --stop
HULA_PORT=443 ./start-with-docker.sh
```

The default installer maps a single port (8088); to expose both 80 and 443, run
the container manually:

```bash
docker run -d \
  --name hula \
  --network hula-net \
  -p 443:443 -p 80:80 \
  -v "$(pwd)/config.yaml":/etc/hula/config.yaml:ro \
  -v "$(pwd)/hula_certs":/var/hula/certs \
  -v "$(pwd)/public":/var/hula/public:ro \
  --restart unless-stopped \
  ghcr.io/tlalocweb/hula:latest
```

## Step 7 — verify

```bash
curl -sI https://example.com/ | head -1
# HTTP/2 200
```

First request triggers ACME — expect a brief delay while the cert is issued and
cached to `hula_certs/`. Subsequent requests are immediate. Renewals happen
automatically before expiry.

Check the log if something's off:

```bash
./start-with-docker.sh --logs
```

Look for `acme: certificate issued` confirming Let's Encrypt succeeded.

## Management

```bash
# Tail logs
./start-with-docker.sh --logs

# Stop everything (Hula + ClickHouse)
./start-with-docker.sh --stop

# Restart after editing config.yaml
./start-with-docker.sh --restart

# Pull a newer image and restart
./start-with-docker.sh --pull --restart

# Hot reload after editing config.yaml (no container restart)
./hulactl reload
```

## Troubleshooting

**ACME fails: `connection refused on port 80`.**
Port 80 isn't reachable from the public internet. Open it on the firewall
*and* the cloud provider security group. Behind a reverse proxy that
terminates 80, set `ssl.acme.http_port` to the internal port.

**ACME fails: `dns problem`.**
DNS A record doesn't resolve to this host yet, or hasn't propagated. Check with
`dig +short example.com`.

**`hulactl set-password` fails: `current password mismatch`.**
On a brand-new install the current password is empty — pass
`HULACTL_CURRENT_PASSWORD=''` explicitly. On a rotation, supply the actual
current password.

**Site shows ClickHouse errors at boot.**
ClickHouse is still starting. Hula retries the connection per `dbconfig.retries`
and `dbconfig.delay_retry`. Default is 5 retries × 5 seconds = 25 seconds.
Check `./start-with-docker.sh --logs` after that window.

## Next steps

- [Hugo Quick Start](hugo.md) — auto-rebuild a Hugo site on every push.
- [Configuration reference](../reference/config.md) — every key.
- [Live staging via WebDAV](../examples/live-staging-webdav.md) — edit live, no Git push.
- [Backend containers](../examples/backend-containers.md) — reverse-proxy a FastAPI / Node / Rails service alongside the static site.
