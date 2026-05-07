# Servers, hosts, aliases

Each entry under `servers:` in `config.yaml` is a virtual host. `host` is the canonical name, `aliases` are alternate names that resolve to the same site, and `id` is the opaque short ID used in CLI commands and analytics rows. This page covers how requests get routed, how aliases interact with TLS, and the special host/alias values `use:hostname` and `use:interfaceips`.

!!! note "Stub"

    This page is on the docs roadmap. See `PRD.md` in the repo for the
    planned outline. Contributions welcome via PR against
    [`tlalocweb/hulation-docs`](https://github.com/tlalocweb/hulation-docs).
