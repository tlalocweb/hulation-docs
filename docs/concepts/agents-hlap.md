# Agents & HLAP

The mTLS sidecar model for CI / deploy-bots: `hulactl create-agent` mints a scoped, expiring cert; the runner runs `hula-agent` and drives Hula via HLAP over a unix-socket. Cert-only auth, no JWTs on the runner. This page covers the trust model, the Agent CA, the HLAP wire shape, and the revocation flow.

!!! note "Stub"

    This page is on the docs roadmap. See `PRD.md` in the repo for the
    planned outline. Contributions welcome via PR against
    [`tlalocweb/hulation-docs`](https://github.com/tlalocweb/hulation-docs).
