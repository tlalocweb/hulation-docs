# Git autodeploy & staging

`root_git_autodeploy` clones (and optionally builds) a repo on boot and on demand. Production mode produces an output tarball that gets unpacked to `deploy_dir`; staging mode keeps a long-lived builder container around so writers can edit via WebDAV. This page covers the full lifecycle: clone, build, finalize, deploy, and how staging differs.

!!! note "Stub"

    This page is on the docs roadmap. See `PRD.md` in the repo for the
    planned outline. Contributions welcome via PR against
    [`tlalocweb/hulation-docs`](https://github.com/tlalocweb/hulation-docs).
