# Backend container reverse proxy

Hula manages and reverse-proxies Docker containers as backend services for any virtual server: a FastAPI app, a Node service, a Rails worker. Each backend gets its own Docker network, isolated from other servers' backends. Covers the `backends:` config block, the lifecycle (start / restart / health), and the environment-variable wiring.

!!! note "Stub"

    This page is on the docs roadmap. See `PRD.md` in the repo for the
    planned outline. Contributions welcome via PR against
    [`tlalocweb/hulation-docs`](https://github.com/tlalocweb/hulation-docs).
