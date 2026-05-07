# Cookieless analytics, no banner

Run analytics without a cookie banner. `tracking_mode: cookieless` derives a per-day visitor ID via HMAC over a per-server salt, the visitor's IP, and the user-agent. Same-day visitors remain stitched; cross-day stitching is impossible by design. Covers the salt-rotation flow with `hulactl rotate-cookieless-salt`.

!!! note "Stub"

    This page is on the docs roadmap. See `PRD.md` in the repo for the
    planned outline. Contributions welcome via PR against
    [`tlalocweb/hulation-docs`](https://github.com/tlalocweb/hulation-docs).
