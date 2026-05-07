# Kubernetes deployment

Deploy Hula to a Kubernetes cluster with a `StatefulSet` for state, a `Deployment` for the unified listener, and `PersistentVolumeClaim`s for cert cache and Bolt/Raft data. Covers the manifests, the ClickHouse hook-up (in-cluster or external), and the ingress shape.

!!! note "Stub"

    This page is on the docs roadmap. See `PRD.md` in the repo for the
    planned outline. Contributions welcome via PR against
    [`tlalocweb/hulation-docs`](https://github.com/tlalocweb/hulation-docs).
