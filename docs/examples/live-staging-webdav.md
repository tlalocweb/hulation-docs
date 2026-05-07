# Live staging via WebDAV

Edit a remote site as if it were a local folder: `hulactl staging-mount` syncs both directions, debounces rapid edits, and triggers `--autobuild` on every save. This page covers the full setup, the safety filters (`--dangerous`), and the conflict-resolution semantics when the remote tree changes underneath you.

!!! note "Stub"

    This page is on the docs roadmap. See `PRD.md` in the repo for the
    planned outline. Contributions welcome via PR against
    [`tlalocweb/hulation-docs`](https://github.com/tlalocweb/hulation-docs).
