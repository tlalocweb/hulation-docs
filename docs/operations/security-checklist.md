# Security hardening checklist

Items to verify before going live and quarterly thereafter. Each item
links to the relevant config or concept page.

The checklist is opinionated — defaults are sensible, but a few
fields default to the *convenient* setting rather than the
*security-conservative* one. The asterisk (★) marks items that need
explicit operator action beyond accepting defaults.

## TLS

- [ ] **TLS via ACME or Cloudflare Origin CA, not manual self-signed.**
      [TLS modes](../concepts/tls.md). Self-signed certs are dev-only.
- [ ] **`ssl.acme.email` is set** to a monitored address. Let's
      Encrypt sends pre-expiry warnings if renewals stall. (★)
- [ ] **Port 80 is reachable for ACME**, **OR** the front proxy
      forwards `/.well-known/acme-challenge/` to Hula's
      `ssl.acme.http_port`. Confirm with a fresh cert issuance after
      every config change.
- [ ] **HSTS is enabled** if appropriate for your audience. Hula
      doesn't set HSTS by default — add it via the front proxy or the
      `csp:` block. (★)
      ```yaml
      servers:
        - host: example.com
          csp:
            other:
              Strict-Transport-Security: "max-age=63072000; includeSubDomains; preload"
      ```
- [ ] **TLS protocol versions** restricted to TLS 1.2+ at the listener.
      Hula's defaults exclude 1.0 / 1.1; if you've custom-configured a
      front proxy, confirm the same.
- [ ] **Cloudflare Origin CA users:** `require_cf_client_cert: true`
      is set. Without it, anyone can mint a Cloudflare Origin CA cert
      for any domain and bypass Cloudflare. (★)

## Auth

- [ ] **OPAQUE keys pinned** in `config.yaml` or `HULA_OPAQUE_*` env
      vars. Auto-generated keys log a loud warning at boot — paste them
      in. Pinning prevents fresh generation on every restart, which
      would invalidate every operator credential. (★)
- [ ] **`jwt_key` set explicitly** (not auto-generated). Auto-gen
      regenerates on restart, invalidating active sessions.
- [ ] **`totp_encryption_key` set** if any operator has TOTP enabled.
      Auto-gen has the same restart-survival problem. Generate with
      `hulactl totp-key`. (★)
- [ ] **Admin password is OPAQUE-enrolled**, not the legacy
      `admin.hash`. New installs default to OPAQUE; legacy installs
      should migrate via `hulactl set-password`.
- [ ] **TOTP enabled for the admin account.** `hulactl totp-setup`. (★)
- [ ] **Per-user roles configured.** Don't grant `admin` to everyone
      who needs operator access — the `viewer` and `manager` roles are
      lower-privilege. (★)
- [ ] **OIDC providers' redirect URIs match exactly** what's
      registered with the upstream IdP. Mismatch is the #1 OIDC
      production failure.

## Network

- [ ] **Hula not directly exposed if behind a proxy.** Bind to
      `127.0.0.1:8080` (or a private network), not a public IP. (★)
- [ ] **Firewall rules allow only the necessary ingress.** For a
      front-faced Hula: 80 + 443 from anywhere. For a behind-proxy
      Hula: only the proxy's IP. Cloudflare-fronted: only Cloudflare
      IPs (auto-update the allowlist). (★)
- [ ] **Outbound egress restricted** to Let's Encrypt, your Git
      remote, your container registry, your push providers (Apple /
      Google), and your forwarder destinations (Meta CAPI, GA4 MP).
      Lock down the rest if your org's posture demands it. (★)
- [ ] **Docker socket mount** (`/var/run/docker.sock`) is required for
      backend / builder containers. **Equivalent to root on the host.**
      Acceptable for a dedicated Hula host; not acceptable on a
      multi-tenant host where Hula's process boundary needs to mean
      something. (★)

## Storage

- [ ] **`data_dir` permissions are 0700**. Bolt file 0600. Agent CA
      key (`agent-ca.key`) 0600. Verify:
      ```bash
      stat -c '%a %n' /var/hula/data /var/hula/data/hula.db /var/hula/data/agent-ca.key
      ```
- [ ] **`data_dir` is on encrypted storage** (LUKS, dm-crypt, or
      cloud provider equivalent). The Bolt file contains OPAQUE
      records, agent registry, the agent CA private key. (★)
- [ ] **Cert cache (`ssl.acme.cache_dir`) on persistent storage.**
      Losing it means re-issuance under Let's Encrypt rate limits.
- [ ] **Backups go off-host.** A backup on the same disk as the
      primary protects nothing. (★) See [Backups & restore](backups.md).
- [ ] **Backups are encrypted in transit and at rest.** The Bolt file
      is secret material. (★)

## Secrets handling

- [ ] **No secrets as literals in `config.yaml`.** Use `${ENV_VAR}`
      substitution with values supplied at process launch. (★)
- [ ] **Secrets stored in a secrets manager** (Vault, AWS Secrets
      Manager, GCP Secret Manager, Doppler, sealed-secrets) — not in
      a plain `.env` file in version control. (★)
- [ ] **Operator laptops:** `~/.config/hulactl/hulactl.yaml` mode
      `0600`. (★)
- [ ] **CI runners:** agent YAML mode `0600`. The `key:` block is
      unencrypted. (★) See [Agent YAML reference](../reference/agent-yaml.md).
- [ ] **Rotation policy** for the secrets that have one:
      - Operator passwords: per your password-policy.
      - GitHub PATs: at most every 90 days.
      - Agent certs: pick a sensible `--expires-in` (1 year is fine
        for stable runners; 30d for ephemeral).
      - APNs keys: per Apple's expiry, set a calendar reminder.
      - Cloudflare API tokens: scoped to the minimum permission set.

## Bot defense

- [ ] **`badactor` config set, not relying on defaults.** Tune
      `block_threshold` to your traffic shape. The default `50` is
      reasonable; if you serve a lot of legit traffic that triggers
      probe heuristics (e.g., a security-research site), bump higher.
      (★)
- [ ] **`allow_cidrs` includes operator office IPs and monitoring
      probes.** Otherwise a routine sweep against the staging admin
      will lock the office out. (★)
- [ ] **The badactor allowlist is reviewed quarterly.** CIDRs change.

## Privacy

- [ ] **`consent_mode` set per-server appropriate to the audience's
      jurisdiction.** EU-default sites: `opt_in`. US-default: `off`
      with `Sec-GPC: 1` honored.
- [ ] **`tracking_mode: cookieless`** considered for jurisdictions
      that reject even consented cookies (CNIL, etc.).
- [ ] **Per-server cookieless salt rotated periodically** if you've
      promised that to users / regulators. `hulactl
      rotate-cookieless-salt <id>`. (★)
- [ ] **Server-side forwarders' `purpose:` matches what each platform
      actually does.** Meta CAPI as `purpose: marketing` is correct;
      mis-tagging it as `analytics` would forward marketing-purpose
      data under an analytics-purpose consent flag.
- [ ] **A privacy policy is published** and links to a CMP that
      handles `consent_mode` correctly. (★)

## Logging

- [ ] **Logs ship to a centralised store** off-host, with retention
      that satisfies your audit requirements. (★)
- [ ] **JWTs and session tokens scrubbed** at the shipping layer
      (Vector / Fluent Bit filter). Hula doesn't log them, but if a
      future bug introduces leakage, a redaction filter is a good
      backstop. (★)
- [ ] **Cookies scrubbed** from logs (same logic).
- [ ] **`log_tags` / `no_log_tags` configured** to drop noisy
      subsystems if your log budget is tight. Don't drop `acme`,
      `agent`, `team`, or `badactor` — they carry security signal.

## Updates

- [ ] **`hula` image tag pinned in production**, not `:latest`.
      Predictable upgrades. (★)
- [ ] **Subscribed to release announcements** at
      [github.com/tlalocweb/hulation/releases](https://github.com/tlalocweb/hulation/releases).
- [ ] **Builder image rebuilt periodically** to pick up base-image
      security patches, even when Hugo / mkdocs versions don't
      change. (★)

## Backups

- [ ] **Daily `data_dir` backup**, retained for 30 days. (★)
- [ ] **Weekly cert cache backup.** (★)
- [ ] **Daily ClickHouse backup** (incremental + weekly full). (★)
- [ ] **Backups verified quarterly** by restoring to a throwaway
      VM. (★) An unverified backup is a hope.

## License compliance

- [ ] **License terms understood.** AGPLv3 / SSPL-1.0 dual-license:
      AGPL for ordinary self-hosted use, SSPL for managed-service /
      multi-tenant / HA-clustered deployments unless commercially
      licensed.
- [ ] **AI-content clause respected.** The license forbids using Hula's
      source / docs / examples / configuration as training data,
      retrieval-augmented context, or other input for AI models intended
      to replace or compete with Hula.
- [ ] **Commercial license arranged** for HA / managed-service / OEM /
      AI-assisted-development use cases. Contact `licensing@tlaloc.us`.
      (★)

## Incident response

- [ ] **Response plan written.** Even one paragraph: who's on-call,
      where the runbook lives, how to quarantine. (★)
- [ ] **`hulactl revoke-agent` rehearsed.** Practice on a throwaway
      agent so the muscle memory is there.
- [ ] **`hulactl deletedb` is locked down.** Anyone with admin can
      drop analytics. Audit who has admin.
- [ ] **`forget-opaque-record` recovery flow rehearsed.** When the
      admin password is lost, knowing this command beats discovering
      it under stress.

## Quarterly re-review

This list isn't a one-time exercise. Schedule a quarterly review
where you walk it top-to-bottom against the running deployment. The
items most likely to drift:

- Pinned image tags (something gets bumped, then nobody bumps it back).
- Allowlist CIDRs (offices change ISPs).
- Outbound egress rules (a new forwarder gets added without updating
  the firewall).
- Backup verification (everyone is "going to" test, nobody does).
- Secrets rotation (PATs always expire at 2am).

## Where to go next

- [Configuration reference](../reference/config.md) — every field
  this checklist references.
- [Backups & restore](backups.md) — the backup half of the picture.
- [Troubleshooting](troubleshooting.md) — when an item on this list
  fails.
- [License & FAQ](../license-faq.md) — the license landscape this
  checklist commits you to.
