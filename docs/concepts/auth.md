# Auth (OPAQUE, TOTP, OIDC)

Three authentication paths: OPAQUE PAKE for admin and operator passwords (passwords never travel the wire), TOTP 2FA at-rest-encrypted, and OIDC SSO for federated logins. This page covers the OPAQUE handshake, the TOTP enrollment flow, and how internal-password and OIDC users coexist.

!!! note "Stub"

    This page is on the docs roadmap. See `PRD.md` in the repo for the
    planned outline. Contributions welcome via PR against
    [`tlalocweb/hulation-docs`](https://github.com/tlalocweb/hulation-docs).
