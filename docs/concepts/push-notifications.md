# Push notifications

Operator alerts and report digests fan out across email, APNs (iOS), and FCM (Android). Push is optional — when creds are missing, those channels degrade silently. This page covers the alert routing, per-recipient delivery accounting, and operational notes around APNs key rotation.

!!! note "Stub"

    This page is on the docs roadmap. See `PRD.md` in the repo for the
    planned outline. Contributions welcome via PR against
    [`tlalocweb/hulation-docs`](https://github.com/tlalocweb/hulation-docs).
