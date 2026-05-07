# Backups & restore

What state needs backing up: the Bolt store (auth + agent registry + small operator tables), the cert cache directory, ClickHouse data, the per-server staging source directories. Recovery procedures for losing each piece in isolation, and the canonical "warm spare" replication pattern.

!!! note "Stub"

    This page is on the docs roadmap. See `PRD.md` in the repo for the
    planned outline. Contributions welcome via PR against
    [`tlalocweb/hulation-docs`](https://github.com/tlalocweb/hulation-docs).
