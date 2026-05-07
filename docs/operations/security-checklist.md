# Security hardening checklist

Pre-launch and post-launch security review: TLS config (modern ciphers, OCSP stapling, HSTS), credentials at rest (encrypted Bolt, OPAQUE-only auth), badactor thresholds tuned, secrets via env vars not config-file literals, log shipping doesn't capture cookies, license clauses understood. Each item links to the relevant config / reference page.

!!! note "Stub"

    This page is on the docs roadmap. See `PRD.md` in the repo for the
    planned outline. Contributions welcome via PR against
    [`tlalocweb/hulation-docs`](https://github.com/tlalocweb/hulation-docs).
