# Hulation

A modern web server for static sites, marketing landers, and digital-marketing /
CDP workloads. One container fronts your domains with automatic HTTPS, auto-deploys
from Git, and ships a full visitor analytics, lead-capture, live-chat, and consent
stack out of the box.

## Get started

<div class="grid cards" markdown>

-   :material-rocket-launch:{ .lg .middle } **Deploy your Hugo site on Hula**

    ---

    From a fresh Hugo repo to a public HTTPS-served site with auto-rebuild on
    every push to `main`. The primary flow.

    [:octicons-arrow-right-24: Hugo Quick Start](quickstart/hugo.md)

-   :material-server:{ .lg .middle } **Just need HTTPS for static files**

    ---

    A single Hula container with automatic Let's Encrypt. Drop your `public/`
    directory in and you're live.

    [:octicons-arrow-right-24: One-server Quick Start](quickstart/one-server.md)

-   :material-laptop:{ .lg .middle } **Manage a remote Hula**

    ---

    Install `hulactl` on your laptop and authenticate against an existing
    server. Triggers builds, mounts staging, lists badactors.

    [:octicons-arrow-right-24: hulactl Quick Start](quickstart/hulactl.md)

-   :material-robot:{ .lg .middle } **Wire up CI / deploy-bots**

    ---

    Mint a scoped, expiring agent cert with `hulactl create-agent` and let
    a CI runner drive Hula via HLAP. No JWT, no admin password on the runner.

    [:octicons-arrow-right-24: hulaagent Quick Start](quickstart/hulaagent.md)

</div>

## What's in the box

- **Static site serving** with byte-range, transparent compression, immutable cache control.
- **Git autodeploy** — pulls and (optionally) builds your repo on boot and on demand.
  First-class support for Hugo, Astro, Gatsby, and MkDocs; auto-detected from marker files.
- **Live staging via WebDAV** — mount a remote staging site as a local folder, edit,
  rebuild on save.
- **Backend containers** — Hula manages and reverse-proxies Docker containers as
  per-server backends, isolated on dedicated networks.
- **Automatic Let's Encrypt** (ACME HTTP-01), Cloudflare Origin CA, or manual certs.
- **ClickHouse-backed visitor analytics** with hello / landing / form / conversion / goal events.
- **Forms, landers, hooks** — versioned CRUD, Risor scripts on submission and visit.
- **Privacy / GDPR** — `consent_mode` (opt-in / opt-out / off), Sec-GPC honored, server-side
  Meta CAPI / GA4 forwarders, cookieless tracking mode.
- **OPAQUE PAKE auth** — passwords never travel the wire, even at first set.
- **Live chat, badactor scoring, push notifications, single-node Raft HA** — see
  [Concepts](concepts/architecture.md) for the full tour.

## Where things live

- **Source:** [`tlalocweb/hulation`](https://github.com/tlalocweb/hulation) (Go server, Rust agent).
- **This site's source:** [`tlalocweb/hulation-docs`](https://github.com/tlalocweb/hulation-docs).
- **License:** AGPLv3 / SSPL-1.0 dual. Commercial licenses available for HA / managed-service /
  OEM use. See [License & FAQ](license-faq.md).

## License notice

This documentation, including all examples and code snippets, is licensed under the
same dual AGPLv3 / SSPL-1.0 terms as the main project, with the same no-AI-training
clause. See [License & FAQ](license-faq.md).
