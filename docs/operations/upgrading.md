# Upgrading Hula

Upgrading the running container, the builder image, the ClickHouse schema, and `hulactl` on operator laptops. Covers the safe-rollback model (current container stops cleanly, certs survive in their cache dir, ClickHouse schema migrations are forward-only), and the per-major-release upgrade notes.

!!! note "Stub"

    This page is on the docs roadmap. See `PRD.md` in the repo for the
    planned outline. Contributions welcome via PR against
    [`tlalocweb/hulation-docs`](https://github.com/tlalocweb/hulation-docs).
