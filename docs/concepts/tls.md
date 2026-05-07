# TLS modes

Hula supports three TLS configurations on a per-server basis: manual cert/key paths, Cloudflare Origin CA, and automatic Let's Encrypt via ACME HTTP-01. This page covers when to pick which, how the unified listener handles mixed configs across servers sharing a port, and the operational shape of certificate caching and renewal.

!!! note "Stub"

    This page is on the docs roadmap. See `PRD.md` in the repo for the
    planned outline. Contributions welcome via PR against
    [`tlalocweb/hulation-docs`](https://github.com/tlalocweb/hulation-docs).
