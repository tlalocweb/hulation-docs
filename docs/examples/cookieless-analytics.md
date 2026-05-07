# Cookieless analytics, no banner

Run visitor analytics without setting any cookie. Hula derives a
per-day visitor ID by HMAC-ing the visitor's IP and User-Agent against
a per-server salt. Same-day visitors are recognisable; cross-day
stitching is **mathematically impossible** by design. The documented
answer to "I want analytics without a cookie banner."

## When this works for you

- Your site falls under jurisdictions where even consented cookies
  trigger banner requirements (CNIL, parts of the EU).
- You only care about same-day metrics: page views, conversion rate,
  funnel completion within a session.
- You can live without cross-day return-visitor analytics.

## When it doesn't

- You need cohort retention reports across weeks / months. Switch back
  to `tracking_mode: cookie` and accept the banner.
- You're feeding ad-platform forwarders (Meta CAPI, GA4) that need
  cross-session user identifiers. Cookieless mode emits per-day
  identifiers; the ad platform won't stitch them.

## Enable

Per-server config:

```yaml
servers:
  - host: example.com
    id: example
    tracking_mode: cookieless
    consent_mode: off          # see "Consent interaction" below
    ssl:
      acme:
        email: admin@example.com
        cache_dir: /var/hula/certs
    root: /var/hula/sites/example/public
```

`hulactl reload`, and from that point the visitor JS stops setting
cookies. No banner needed (in most jurisdictions — confirm with your
DPO).

## How the visitor ID is derived

```text
visitor_id = HMAC-SHA256(
  per-server salt,
  YYYYMMDD || normalized(IP) || User-Agent
)
```

- **Per-server salt** is 32 random bytes generated automatically on
  first boot, stored in the `cookieless_salts` Bolt bucket. Distinct
  salts per server prevent cross-domain correlation.
- **YYYYMMDD** rolls the salt input over at midnight UTC. The same
  visitor at 23:59 and 00:01 has different IDs.
- **Normalized IP** truncates IPv4 to /24 and IPv6 to /48 to defend
  against client IP rotation breaking the within-day stitch.
- **User-Agent** is taken verbatim from the request.

Visitors who wipe one of {IP, User-Agent} mid-day get a fresh ID.
That's the design — no cookie, no localStorage, no fingerprinting
beyond what the request already carries.

## Consent interaction

`tracking_mode: cookieless` and `consent_mode:` are independent:

| `tracking_mode` | `consent_mode` | Behaviour |
|-----------------|----------------|-----------|
| `cookie` | `off` | Cookie set unconditionally. Marketing events still respect `Sec-GPC: 1`. |
| `cookie` | `opt_in` | Cookie set only after consent. `/v/hello` returns 204 + `Hula-Consent-Required: 1` until then. |
| `cookieless` | `off` | No cookie. Visitor ID derived per-request. Events written. **No banner needed in most jurisdictions.** |
| `cookieless` | `opt_in` | No cookie. Events still gated behind explicit consent — the visitor ID is computed but not persisted. Use only when your DPO insists. |

The "no banner" case is `cookieless` + `consent_mode: off`. Marketing
events still drop on `Sec-GPC: 1`.

## Salt rotation (kill-the-stitch)

The salt determines visitor identity. Rotating it makes today's
visitors un-recognisable starting tomorrow — useful for:

- A privacy incident where you want to wipe identifier continuity.
- Routine annual rotation as policy.
- Compliance with a "right to erasure" request that needs to extend to
  derivable identifiers.

```bash
# Stop hula first — Bolt is single-writer.
./start-with-docker.sh --stop

# Rotate
hulactl --bolt /var/hula/data/hula.db rotate-cookieless-salt example

# Restart
./start-with-docker.sh
```

After rotation, the same-day same-visitor stitching breaks for any
visit that crosses the rotation moment. Visits before rotation
remain anchored to the old identifier in stored events; visits after
rotation start fresh.

## What ends up in ClickHouse

```sql
SELECT visitor_id, count() AS events, min(ts), max(ts)
FROM hula.events
WHERE server_id = 'example'
  AND ts >= today()
GROUP BY visitor_id
ORDER BY events DESC
LIMIT 10;
```

Each row is a same-day session. The `visitor_id` column carries the
HMAC; it changes each day. Joining today's IDs to yesterday's is
impossible without the salt — and the salt rotates if you choose.

## Comparison vs. CNIL Matomo cookieless

Matomo's cookieless mode (CNIL-cleared in France) uses the same shape:
salt + per-day rolling. Hula's implementation is independent but the
privacy properties are equivalent.

## Forwarders interact awkwardly

Server-side forwarders (Meta CAPI, GA4 MP) work with cookieless
visitor IDs but the receiving platforms can't stitch users across
days. For ad-platform attribution you almost certainly want
`tracking_mode: cookie` with `consent_mode: opt_in` and a CMP banner.

You can split the difference:

- One server: privacy-forward, cookieless, no forwarders.
- Another server (subdomain or separate domain): cookie + opt-in +
  full forwarders for explicit-consent campaigns.

## What about local-storage / IndexedDB?

Hula's visitor JS doesn't touch `localStorage` or `IndexedDB` in
cookieless mode. Same for fingerprinting libraries — none are
loaded. The only visitor-side state is what the browser's HTTP layer
exposes (IP, UA), which the request already carries.

## Tradeoffs and limits

- **No return-visitor metrics across days.** Period.
- **NAT and shared IPs collide.** All users behind a corporate NAT
  share an IP and (often) similar UAs. Same-day stitch will under-count
  unique visitors. Acceptable for most marketing-funnel use; talk to
  your data team if you depend on precise unique counts.
- **VPN / Tor users get fresh IDs every connection.** They look like
  separate visitors. Same problem any analytics has.
- **Mobile carrier IP rotation** breaks within-day stitches. The
  /24 / /48 truncation defends against this somewhat.

## Migration from cookie mode

Switching from `tracking_mode: cookie` to `cookieless` is a config
change + reload. Active visitors keep their cookies for the rest of
their session; new visits compute the cookieless ID. Reports the next
day reflect cookieless data only.

To go back, flip the config and reload — Hula will start setting
cookies again. Existing cookie-visitor IDs in ClickHouse don't get
rewritten; queries just need to handle both shapes.

## Troubleshooting

**Reports show 1.5x more visitors than I expected.** Cookieless mode
counts each {IP-/24, UA} as one visitor; mobile carrier traffic
inflates this slightly. For an accurate-cookie-comparable count, run
both modes on parallel staging and benchmark.

**`hulactl rotate-cookieless-salt` fails with "boltdb: timeout".**
Hula is still running. Stop the process first (`./start-with-docker.sh
--stop`) — Bolt is single-writer.

**Events stop being written after enabling.** Check `consent_mode`. If
it's `opt_in`, no events get committed until the visitor signals
consent. Set `consent_mode: off` for true no-banner operation.

## Next

- [Consent & privacy](../concepts/consent-privacy.md) — the
  consent-mode state machine.
- [Server-side forwarders](server-side-forwarders.md) — Meta CAPI / GA4
  MP, including consent-gate behaviour.
- [Configuration reference — consent](../reference/config.md#consent-privacy-per-server) —
  per-server `tracking_mode` / `consent_mode` reference.
