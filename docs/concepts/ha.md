# High availability

Single-node Raft (`hashicorp/raft` + raft-boltdb) is the production storage default. Solo installs auto-bootstrap a `TeamID` + `NodeID` under `data_dir` on first boot — no `team:` block required. Multi-node clustering is opt-in via `team:`. This page covers the storage seam, the join workflow, and the consistency guarantees.

!!! note "Stub"

    This page is on the docs roadmap. See `PRD.md` in the repo for the
    planned outline. Contributions welcome via PR against
    [`tlalocweb/hulation-docs`](https://github.com/tlalocweb/hulation-docs).
