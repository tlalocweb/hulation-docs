# Server-side forwarders

Server-to-server forwarding to ad platforms — Meta CAPI and GA4 Measurement Protocol — without client-side beacons or third-party cookies. Each forwarder has a `purpose:` field that gates events behind the matching consent flag. This page covers the supported adapters, the config shape, and the consent-gating semantics.

!!! note "Stub"

    This page is on the docs roadmap. See `PRD.md` in the repo for the
    planned outline. Contributions welcome via PR against
    [`tlalocweb/hulation-docs`](https://github.com/tlalocweb/hulation-docs).
