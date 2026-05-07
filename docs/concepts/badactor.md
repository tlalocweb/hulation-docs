# Bot & abuse defense

A radix-tree of IPs scored by suspicious behaviour: known WordPress / vuln-probe paths, TCP protocol probes, TLS handshake failures. Scores TTL-expire, allowlists are CIDR-aware, and ClickHouse keeps an audit row per incident. This page covers the scoring model, the block thresholds, and the `hulactl badactors` listing.

!!! note "Stub"

    This page is on the docs roadmap. See `PRD.md` in the repo for the
    planned outline. Contributions welcome via PR against
    [`tlalocweb/hulation-docs`](https://github.com/tlalocweb/hulation-docs).
