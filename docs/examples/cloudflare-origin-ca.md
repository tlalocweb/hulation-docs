# Behind Cloudflare (Origin CA)

Cloudflare terminates the public TLS handshake; Hula presents a
Cloudflare-issued **Origin CA** certificate on the back-end leg. End
users see a Cloudflare cert on `https://example.com`; Hula sees a
Cloudflare-only cert that's free, long-lived (15 years), and never
visible outside Cloudflare's network.

## When to use this

- You're already on Cloudflare (DNS, CDN, WAF) and want end-to-end
  encryption from Cloudflare to Hula without ACME on the origin.
- You don't want port 80 reachable from the public internet on the
  origin (Origin CA works when only Cloudflare can reach the origin).
- You want a long-lived cert (15 years) without renewal automation.

If you're not on Cloudflare, use [ACME](../quickstart/one-server.md)
instead.

## Generate the Origin CA cert

Cloudflare dashboard → **SSL/TLS → Origin Server → Create Certificate**.
Defaults are sensible:

- Key type: **ECC (recommended)**.
- Hostnames: `example.com`, `*.example.com`. Add specific subdomains if
  you don't want wildcard.
- Validity: **15 years**.

Click Create. Cloudflare shows the certificate (PEM) and the private
key. **The private key is shown once** — copy it now.

Save both to disk on the Hula host:

```bash
mkdir -p /etc/hula/cf-origin
chmod 700 /etc/hula/cf-origin
# Paste cert.pem and key.pem
chmod 600 /etc/hula/cf-origin/key.pem
chmod 644 /etc/hula/cf-origin/cert.pem
```

## Server config

```yaml
servers:
  - host: example.com
    aliases: [www.example.com]
    id: example
    ssl:
      cloudflare_origin_ca:
        cert: /etc/hula/cf-origin/cert.pem
        key: /etc/hula/cf-origin/key.pem
    root: /var/hula/sites/example/public
```

Reload:

```bash
hulactl reload
```

Hula now presents the Cloudflare Origin CA cert on every TLS handshake
for `example.com`.

## Cloudflare side: enforce Origin CA

In the Cloudflare dashboard:

1. **SSL/TLS → Overview**. Set encryption mode to **Full (strict)**.
   Cloudflare will only connect to the origin over TLS *and* will verify
   the cert chains to the Cloudflare Origin CA root.
2. **SSL/TLS → Edge Certificates → Authenticated Origin Pulls** (optional).
   Cloudflare presents a client cert to the origin; Hula verifies it.
   Defends against an attacker with a Cloudflare Origin CA cert (anyone
   can mint one for any domain) bypassing Cloudflare and connecting
   directly. **Recommended for production.**

For Authenticated Origin Pulls:

```yaml
servers:
  - host: example.com
    ssl:
      cloudflare_origin_ca:
        cert: /etc/hula/cf-origin/cert.pem
        key: /etc/hula/cf-origin/key.pem
        require_cf_client_cert: true
```

`require_cf_client_cert: true` makes Hula reject any handshake whose
client cert isn't signed by the Cloudflare authenticated-origin-pull CA.
The CA cert is bundled with Hula; no extra config required.

## Locking down the origin

With Cloudflare in front, the origin doesn't need port 80 open:

```bash
sudo ufw delete allow 80/tcp     # nothing connects to port 80 anymore
sudo ufw allow 443/tcp           # only Cloudflare reaches port 443
```

Optionally, restrict 443 to Cloudflare's IP ranges. Hula has built-in
support for trusting Cloudflare's `CF-Connecting-IP` header — see the
[`config.cloudflare`](../reference/config.md) section. Combined with a
host-firewall allow-list for Cloudflare's IPs, this keeps everything
non-Cloudflare out at the network layer.

```bash
# Get Cloudflare's IP ranges (refresh periodically)
curl -s https://www.cloudflare.com/ips-v4
curl -s https://www.cloudflare.com/ips-v6
```

Apply to your firewall:

```bash
for ip in $(curl -s https://www.cloudflare.com/ips-v4); do
  sudo ufw allow from "$ip" to any port 443 proto tcp
done
sudo ufw default deny incoming
```

(For production, automate the refresh — Cloudflare's IPs do change.)

## Headers to honor

Behind Cloudflare, Hula reads the trusted edge headers automatically:

| Header | What Hula uses it for |
|--------|----------------------|
| `CF-Connecting-IP` | Real client IP (overrides `X-Forwarded-For`). |
| `CF-IPCountry` | Visitor country (no IP geolocation hop needed). |
| `CF-Ray` | Cloudflare request ID — surfaced in logs for cross-correlation. |

These are automatically trusted because Hula sees only Cloudflare's IPs
on the back-end leg. **Do not enable** `CF-Connecting-IP` trust if Hula
is *not* exclusively reachable through Cloudflare — an attacker
connecting directly could spoof it.

## Renewing the cert

Origin CA certs are 15 years. Practically never. Set a calendar
reminder for year 14, mint a fresh cert, swap, reload. The `ssl.cert`
and `ssl.key` paths are the only thing that changes; SIGHUP-reloadable.

If you ever need to rotate (key compromise, etc.):

1. Mint new cert in Cloudflare dashboard.
2. Replace `cert.pem` + `key.pem` on the host.
3. `hulactl reload`.
4. Revoke the old cert in Cloudflare's dashboard (defense-in-depth).

## Mixing with ACME

You can run **some servers on Origin CA, others on ACME** in the same
Hula. The unified listener handles both. Useful when transitioning a
fleet onto Cloudflare gradually:

```yaml
servers:
  - host: cloudflared.example.com
    ssl:
      cloudflare_origin_ca:
        cert: /etc/hula/cf-origin/cert.pem
        key: /etc/hula/cf-origin/key.pem
  - host: direct.example.com
    ssl:
      acme:
        email: admin@example.com
        cache_dir: /var/hula/certs
```

Each server's TLS config is independent; the listener picks the right
chain at SNI-handshake time.

## Troubleshooting

**Cloudflare error 525 (SSL handshake failed).** Encryption mode is
**Full (strict)** but Hula isn't presenting a Cloudflare-trusted cert.
Either downgrade to **Full** (insecure — accepts any cert) or fix the
`cloudflare_origin_ca:` block.

**Cloudflare error 526 (invalid SSL certificate).** Cert is being
presented but doesn't chain to Cloudflare's Origin CA. Check
`cert.pem` is the **certificate** Cloudflare gave you (not the
key, not a CA-bundle file).

**`require_cf_client_cert` rejects connections that should work.**
Either Cloudflare's Authenticated Origin Pulls isn't enabled, or
you're testing from somewhere other than Cloudflare. Disable
temporarily to test the underlying TLS first.

**Visitor IPs all show as Cloudflare's range in analytics.** Hula
isn't reading `CF-Connecting-IP`. Confirm the request is reaching Hula
through Cloudflare (look at the Hula log for `CF-Connecting-IP`
present). If absent, your firewall is letting non-Cloudflare traffic
through; restrict 443.

## Next

- [One-server Quick Start](../quickstart/one-server.md) — pure-ACME flow.
- [Behind nginx / Traefik](behind-reverse-proxy.md) — a different
  reverse-proxy model.
- [Cookieless analytics](cookieless-analytics.md) — the natural pair for
  privacy-forward Cloudflare deployments.
