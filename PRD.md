# Hulation Documentation Site — PRD (Draft)

**Status:** Draft v0.1
**Owner:** Tlaloc / Hulation team
**Repo:** `github.com/tlalocweb/hulation-docs`
**Production URL:** `https://docs.tlaloc.us/hulation` (decided)

---

## 1. Summary

The Hulation documentation site is the canonical, public-facing reference for installing, configuring, and operating [Hula](https://github.com/tlalocweb/hulation). Today, all documentation lives in a 558-line `README.md` and a 1,284-line `DEPLOYMENT.md` inside the main repo. That works for engineers who already cloned the source — it does not work for new evaluators, Hugo users, or operators looking up a single config flag.

This site replaces the "scroll the README" experience with a structured, searchable, fast-loading documentation site, built with **MkDocs Material** and **deployed by Hula itself** — proving the product on its own home page.

## 2. Goals

| # | Goal | Success signal |
|---|------|----------------|
| G1 | A new user with Docker installed has a working Hula server in **under 5 minutes** | "Quick Start (1 server)" page is self-contained, copy-paste, no forward references |
| G2 | A Hugo user can deploy their existing site on Hula in **under 15 minutes** | Dedicated Hugo quick start covers `root_git_autodeploy`, `hulabuild`, and HTTPS |
| G3 | A macOS or Linux operator can install `hulactl` and authenticate against a remote Hula server in **under 5 minutes** | Standalone "Configure hulactl" guide; no need to read Docker / server docs |
| G4 | Every config key in `config.yaml` is discoverable by search | Full reference page generated/curated from `config/*.go` structs |
| G5 | Every `hulactl` subcommand is documented with flags, env vars, and at least one example | Full reference page; `hulactl help` and the doc site agree |
| G6 | The HTTP/gRPC API surface is documented for integrators | API reference with request/response shapes |
| G7 | The site is **deployed by Hula** (dogfooding) | Live site is served from a Hula instance with `root_git_autodeploy` |

## 3. Non-goals

- Marketing site for Tlaloc / Hula (separate property — links over).
- Pricing, commercial licensing flows (those live on tlaloc.us / `licensing@tlaloc.us`).
- Auto-generated Go API reference (godoc.org / pkg.go.dev covers this).
- Replacing `BACKLOG.md`, `HA_PLAN*.md`, `PHASE_*_STATUS.md` and other internal planning docs — those stay in-repo.
- Multi-version docs in v1 (single "latest" channel; revisit when there's a stable v1.0 release line).

## 4. Audience & primary user journeys

**Persona A — Hobbyist self-hoster.** Has a VPS and Docker. Wants HTTPS for a static site without nginx/certbot ceremony.
→ Lands on Quick Start (1 server) → has a live site.

**Persona B — Hugo user with an existing site.** Already has a Hugo repo on GitHub.
→ Lands on Quick Start (Hugo) → adds `root_git_autodeploy` → site rebuilds on push.

**Persona C — Operator managing a remote Hula instance.** Server already runs; needs CLI access from their laptop.
→ Lands on Quick Start (hulactl) → installs binary → `hulactl auth` → can list forms, trigger builds, mount staging.

**Persona D — Evaluator / integrator.** Wants to know if Hula has feature X, or how to wire up consent / forwarders / live chat.
→ Uses search → lands on a feature page → hits the relevant config / CLI reference.

**Persona E — CI / deploy-bot operator.** Has a GitHub Actions runner, a Jenkins box, or a developer laptop that needs to trigger builds / push files to a remote Hula. **Cannot hold an admin password or JWT.**
→ Lands on Quick Start (hulaagent) → an admin issues a scoped, expiring agent cert with `hulactl create-agent` → drops the YAML on the runner → `hula-agent` runs as a sidecar → CI scripts speak HLAP over a unix-socket. No long-lived bearer tokens anywhere.

## 5. Information architecture

The top-level nav is intentionally short — the three quick starts come first because they are the primary conversion path.

```
Home
├── Quick Start
│   ├── Hugo Site                   ← Persona B (PRIMARY — home page hero)
│   ├── One Server (Docker)         ← Persona A (and prereq for Hugo flow)
│   ├── hulactl on macOS / Linux    ← Persona C
│   └── hulaagent on a CI runner    ← Persona E
├── Concepts
│   ├── Architecture overview
│   ├── Servers, hosts, aliases
│   ├── TLS modes (manual / Cloudflare Origin CA / ACME)
│   ├── Git autodeploy & staging
│   ├── Visitor analytics & events
│   ├── Forms, landers, hooks
│   ├── Consent & privacy (opt_in / opt_out / cookieless)
│   ├── Server-side forwarders (Meta CAPI, GA4)
│   ├── Live chat
│   ├── Bot & abuse defense (badactor)
│   ├── Auth (OPAQUE, TOTP, OIDC)
│   ├── Push notifications (APNs / FCM)
│   ├── High availability (Raft, single & multi-node)
│   └── Agents & HLAP (mTLS sidecar for CI / deploy-bots)
├── Reference
│   ├── config.yaml             ← every key, type, default, example
│   ├── hulactl command reference
│   ├── HTTP API
│   ├── gRPC API
│   ├── Hooks (Risor) API
│   ├── HLAP (Hula Local Agent Protocol)
│   ├── hula-agent binary & agent YAML
│   └── Environment variables
├── Examples / Cookbook
│   ├── Hugo + GitHub autodeploy
│   ├── Hugo + private repo (PAT, deploy key)
│   ├── Live staging via WebDAV mount
│   ├── Multi-site (one Hula, many domains)
│   ├── Behind Cloudflare (Origin CA)
│   ├── Behind nginx / Traefik (custom http_port)
│   ├── Backend container reverse proxy (FastAPI / Node / Rails)
│   ├── Cookieless analytics, no banner
│   ├── Meta CAPI + GA4 server-side forwarding
│   ├── Newsletter form with Risor → webhook
│   ├── Lander A/B redirect
│   ├── Operator alerts to email + iOS push
│   ├── 3-node HA cluster (Raft)
│   ├── GitHub Actions → hulaagent → trigger Hugo build
│   ├── Local dev loop: hulaagent + edit/commit/push via HLAP
│   └── Kubernetes deployment
├── Operations
│   ├── Upgrading Hula
│   ├── Backups & restore
│   ├── Observability (logs, ClickHouse queries)
│   ├── Troubleshooting (ACME failures, DB retries, badactor lockout)
│   └── Security hardening checklist
├── Releases / Changelog
└── License & FAQ
```

## 6. Detailed page specs (v1 must-haves)

### 6.0 Home page

The home page hero is the **Hugo path**. The single dominant call-to-action is "Deploy your Hugo site on Hula" → §6.2. Below that, three secondary tiles: "Just need HTTPS for static files" → §6.1, "Manage a remote Hula" → §6.3, "Wire up CI / deploy-bots" → §6.3.5. No marketing copy, no scroll-to-features carousel — this is a docs site, not a landing page. One paragraph explaining what Hula is, then the tiles.

### 6.1 Quick Start — One Server (Docker)

**Constraint:** Must work on a fresh Ubuntu 22.04 / Debian 12 / Fedora VPS with Docker installed and ports 80/443 reachable. **No forward references.** No "see config reference" mid-flow. Copy-paste only.

Outline:

1. **Prerequisites box** — Docker 20.10+, Linux server with public IP, ports 80+443 open, A record pointing at the server.
2. **Run the installer** —
   ```bash
   curl -fsSL https://raw.githubusercontent.com/tlalocweb/hulation/main/install.sh | bash
   ```
3. **Set the admin password** —
   ```bash
   cd hula
   HULACTL_CURRENT_PASSWORD='' HULACTL_NEW_PASSWORD='your-strong-password' ./hulactl set-password
   ```
4. **Edit `config.yaml`** — minimal example (host, ACME email, server id, root). Inline, no link-out.
5. **Drop your static files** — `cp -r site/public/* ./public/`.
6. **Restart on 80/443** — `./start-with-docker.sh --stop && HULA_PORT=443 ./start-with-docker.sh` plus the manual `docker run` snippet for port 80.
7. **Verify** — `curl https://example.com`. ACME log line to look for.
8. **Next steps** — three links: Hugo quick start, hulactl quick start, configuration reference.

### 6.2 Quick Start — Hugo Site

**This is the primary entry point for the docs site** — most visitors land here from the home page. Audience already knows Hugo, so we skip the Hugo install / `hugo new site` step (link to Hugo docs).

Outline:

0. **Prereq callout (top of page).** "You need a running Hula server first — 5 min Docker install →" linking prominently to §6.1. Inline the one-line installer command so motivated readers don't context-switch.
1. Prereqs: Hula running (link back to 6.1), Hugo site in a Git repo, GitHub PAT or deploy key.
2. The `root_git_autodeploy` block — copy-paste, with `repo`, `creds`, `ref.branch`, `hula_build: production`, `build_env`.
3. `hulactl reload` to pick up config changes without restarting.
4. Trigger a build — `hulactl build <server-id>`, poll with `hulactl build-status <id>`.
5. (Optional) Live staging — point the same server at `hula_build: staging` instead, then `hulactl staging-mount <id> ./local-site --autobuild`.
6. Troubleshooting box — "build failed: hugo not found in builder" → use the `tlalocweb/hula-builder` image; "git clone 401" → PAT scope.

### 6.3 Quick Start — hulactl on macOS / Linux

Audience: an operator whose Hula server already runs (cloud, colleague's machine, prod). They need the CLI on their laptop.

Outline:

1. **Install** —
   ```bash
   curl -fsSL https://raw.githubusercontent.com/tlalocweb/hulation/main/installtools.sh | bash
   ```
   Note PATH (`~/.local/bin`); homebrew tap if/when available (track as later work).
2. **Authenticate** —
   ```bash
   hulactl auth https://hula.example.com
   ```
   What gets stored (`~/.config/hulactl/hulactl.yaml`), how OPAQUE means the password never leaves the laptop, TOTP prompt if enabled.
3. **Sanity check** — `hulactl authok`.
4. **What you can now do** — five-line tour: `listforms`, `listlanders`, `builds <id>`, `badactors`, `reload`.
5. **Logout / rotate** — `hulactl logout`, `hulactl set-password`.
6. **macOS gotchas** — Gatekeeper unsigned binary on first run (`xattr -d com.apple.quarantine ./hulactl`); shell-completion (track as later work).

### 6.3.5 Quick Start — hulaagent on a CI runner

Audience: Persona E. They have admin (or admin-equivalent) access on a remote Hula server long enough to issue an agent cert, and a CI runner / dev box that needs to drive that Hula without holding an admin credential afterwards.

The flow is **two-machine**: an admin runs `hulactl create-agent` once, hands the resulting YAML to the CI runner, and the runner thereafter speaks only to its local `hula-agent` over a unix-socket. The cert is the only credential. Revoking it is instant.

Outline:

1. **Why an agent (one paragraph).** Cert-only auth, scoped to specific sites + verbs, expires automatically. No JWT to rotate, no admin password on the runner. Contrast with `hulactl auth`, which is for human operators.
2. **(Admin step) Mint the agent on the Hula host** —
   ```bash
   hulactl create-agent \
       --allow-build=docs \
       --allow-staging-build=docs \
       --expires-in=1yr \
       > docs-ci-agent.yaml
   ```
   What's in the YAML: the agent ID, the Agent CA `ca.pem`, the leaf `cert.pem`, the private `key.pem`, the `hula_host`, and the per-site `allow` map. Treat the YAML as a secret — the `key` block is unencrypted.
3. **(Admin step) Hand it to the runner securely.** GitHub Actions: paste into a repo or org secret as `HULA_AGENT_YAML`. Jenkins: credentials store. Local dev: `chmod 600` it.
4. **(Runner step) Install `hula-agent`.** Pre-built binaries from the Hulation releases page (Linux amd64 / arm64, macOS amd64 / arm64). One file, ~2 MB stripped, no runtime dependencies. `chmod +x` and run.
5. **(Runner step) Start the sidecar** —
   ```bash
   echo "$HULA_AGENT_YAML" > /tmp/agent.yaml
   chmod 600 /tmp/agent.yaml
   hula-agent --config /tmp/agent.yaml --socket /tmp/hula-agent.sock &
   ```
6. **(Runner step) Drive it via HLAP.** The "hello-world" example — trigger a production build:
   ```bash
   printf 'BUILD docs\n' | nc -U /tmp/hula-agent.sock
   # < OK build_id=...
   # < STATUS=complete
   ```
   Plus a one-liner for a file push (`PUSH-FILE`) and a one-liner for `STAGING-BUILD`.
7. **Revoke when the runner is decommissioned** —
   ```bash
   hulactl revoke-agent <id>
   ```
   What "revoke" actually does (registry flag → handshake refused on next connection; existing in-flight requests finish).
8. **Troubleshooting box.** `agent not registered` (registry wiped / wrong server), `agent expired` (re-mint), `agent revoked`, `permission denied for verb X on site Y` (re-mint with the right `--allow-*` flag — permissions are immutable on existing certs by design).
9. **Status callout.** v1 of the docs ships with Phase 1–3 of hulaagent live (cert minting, registry, mTLS validation). Phases 4–6 — the Rust runtime, full HLAP verb set, `list-agents` / `revoke-agent` — are tracked in `HULAAGENT_PLAN.md`. Sections 5–8 above will be marked **Preview** until those phases land; we do not ship the Quick Start as "stable" until end-to-end works on a fresh runner.

### 6.4 Configuration reference (`config.yaml`)

A single long, anchored, searchable page. Sections mirror the YAML root keys, in the order they appear in `config/config.go`:

- `admin` (legacy hash, OPAQUE flow)
- `jwt_key`, `port`, `ssl` (manual, ACME, Cloudflare Origin CA)
- `proxies` (top-level reverse-proxy)
- `servers[]` — the deepest section: host, aliases, id, root, `root_git_autodeploy`, `backends`, `consent_mode`, `tracking_mode`, `forwarders`, hooks, captcha
- `cors`, `dbconfig`, `chat`, `mailer`, `apns`, `fcm`, `badactor`, `team` (HA), `auth` (OIDC)

Each key documented as: type, default, required?, env-var override (where applicable from the `env:` struct tag), one-paragraph description, minimal example. **Source of truth:** the struct tags in `config/*.go`. Generation strategy noted in §9.

### 6.5 hulactl command reference

One page per command group (Auth, Users, Forms, Landers, Builds, Staging, Operations) — same groupings as in the README block. Each command page includes:

- One-line summary
- Synopsis (`hulactl <cmd> [flags] <args>`)
- All flags with types and defaults
- Environment variables read (`HULACTL_*`)
- Exit codes
- Two examples: minimal, and one realistic
- Links to the underlying API endpoint or RPC where relevant

### 6.6 API reference

- **HTTP visitor API** — `/v/hello`, `/v/landing`, `/v/form`, `/v/conversion`, `/v/goal`, the iframe variants, `/chat/*` WS endpoints. Document headers (`Hula-Consent-Required`, `Sec-GPC`), request bodies, response codes.
- **HTTP admin API** — auth flow, forms/landers/users CRUD, build endpoints, badactor list. Mark which require admin vs operator.
- **gRPC** — generated reference from the .proto files in `protoext/`. Link out to the Postman collection that already exists.
- **Risor hooks** — `on_new_form_submission`, `on_lander_visit`, `on_new_visitor` — globals available, return contract, examples.

### 6.6.1 Agents reference (HLAP, `hula-agent`, agent YAML)

Three sibling pages under **Reference → Agents**:

**Reference → Agents → HLAP protocol.** Wire-level reference of the unix-socket text protocol:

- Connection model (one connection = one or more line-oriented requests; blank line terminates a response).
- Request grammar: `VERB <site> [args]\n`. Body conventions for `COMMIT`, `STAGE`, `PUSH-FILE`.
- Response grammar: `OK key=value [...]` or `ERR <reason>`, with a streamed body for `GET-FILE` / `BUILD` (length-prefixed via `content_length=`).
- Verb table — `BUILD`, `STAGING-BUILD`, `PULL`, `PUSH`, `SYNC`, `COMMIT`, `STAGE`, `PUSH-FILE`, `GET-FILE` — each with: required permission key, REST endpoint it maps to, request args, response keys, error codes.
- Permission semantics. `allow.<verb>: ""` means "verb permitted, no options"; a non-empty string is the **exact** option string applied server-side. Wildcards are not supported in v1; operators issue multiple agents instead.

**Reference → Agents → `hula-agent` binary.** Operator reference for the Rust sidecar:

- Install (release tarball, checksums, supported triples).
- Flags: `--config`, `--socket`, `--log-level`, `--dump` (echo the parsed YAML and exit — for verifying secret-injection).
- Lifecycle: graceful shutdown on SIGTERM, socket cleanup, exit codes.
- Resource footprint and tuning (single-threaded tokio runtime, `opt-level = "z"`, `panic = abort` — consequences for crash diagnosis).
- Logging format and what to ship to the operator's logging system.
- **Security notes:** unix-socket permission bits, why the YAML must be `0600`, why agents should run under their own UID on shared CI hosts.

**Reference → Agents → Agent YAML.** Schema reference for the YAML emitted by `hulactl create-agent`:

```yaml
agent:
  id: <base64url-16>
  hula_host: hula.example.com:443
  mTLS:
    ca:   |  # PEM, Agent CA — used to verify hula's serving cert? (clarify w/ §11)
    cert: |  # PEM, leaf cert
    key:  |  # PEM, EC private key
sites:
  <site_id>:
    allow:
      <verb>: "<options-string>"
```

Field-by-field: type, required, what it's used for, what changes if you edit it (mostly: nothing — the registry on the server side is the source of truth for permissions; editing `sites:` on the runner does not grant the agent any new authority). Plus the `--offline` mode caveat (Phase 1 dev mode where the YAML is generated without a server round-trip).

### 6.6.2 Agent management — `hulactl create-agent / list-agents / revoke-agent`

Sits inside §6.5 (the `hulactl` reference) but called out here because it's the admin-side counterpart to §6.3.5. Each command page documents:

- Required role (admin / OPAQUE-authenticated).
- Flag form vs template form for `create-agent`.
- Output shape (the agent YAML — link to §6.6.1).
- `list-agents` output columns: `id`, `created_at`, `expires_at`, `revoked`, allowed sites/verbs.
- `revoke-agent <id>` semantics — registry flag, takes effect on next handshake, no way to un-revoke (re-mint instead). What happens to in-flight requests.
- Cert rotation guidance (re-mint, distribute, revoke old — overlap window is fine because each agent has its own cert).

### 6.7 Examples / Cookbook

Each example is a self-contained page: the problem, the full `config.yaml` (or fragment), the commands, what success looks like, what to check if it fails. Optimised for being landed on directly via search.

## 7. Look & feel

- **Voice & tone.** Terse, normal developer-doc register — same posture as the existing `README.md`. Imperative ("Run this", "Set this") over instructional ("You can run this if you'd like to"). No exclamation points, no "great!", no faux-friendly framing. Code first, prose second; if a sentence doesn't change what the reader does, cut it. This applies to every page including the quick starts — first-timer accessibility is achieved through correctness and brevity, not warmth.
- **Theme:** [MkDocs Material](https://github.com/squidfunk/mkdocs-material).
- **Color:** primary derived from the Hula/Tlaloc brand (placeholder: indigo + amber accent — to be locked once brand reviews).
- **Logo / favicon:** existing Hula mark from `web/` if one exists, otherwise placeholder.
- **Features (Material):** instant nav, search w/ highlighting, copy-button on code blocks, content tabs (for Linux/macOS variants), admonitions (`!!! warning`, `!!! tip`), versioned snippets via `pymdownx.snippets`, dark/light toggle, sitemap+robots.
- **Markdown extensions:** `pymdownx.superfences`, `tabbed`, `details`, `tasklist`, `keys`, `mermaid` (for the architecture diagram).
- **Diagrams:** Mermaid via `pymdownx.superfences` custom-fence, **self-hosted** (`extra_javascript: [assets/javascripts/mermaid.min.js]`). No CDN — the docs site has zero third-party JS dependencies at runtime. Vendor `mermaid.min.js` into `docs/assets/javascripts/` and bump it deliberately, the same way we'd bump any other pinned dep.
- **Typography:** Material defaults (Roboto / Roboto Mono) — no custom fonts in v1.

## 8. Deployment (Hula dogfooding)

**Decision:** Mode A — Hula's `root_git_autodeploy` builds and serves the docs. This is non-negotiable because the docs site is the public proof that Hula's autodeploy works on real-world site generators, not just Hugo.

The work splits across two repos:

**(a) `hulation` repo — refine the existing latent MkDocs support in `hulabuild`.** *(Important correction from the original PRD draft: MkDocs is **already wired end-to-end** in `hulabuild` — `MKDOCS` is an accepted COMMANDLIST verb, both builder images pre-install `mkdocs-material`, and `cmdStaticGen` dispatches it identically to Hugo. The work is much smaller than originally scoped.)* Tracked in detail at `hulation/MKDOCS_BUILDER_PLAN.md`; the four items that matter are:
- **Auto-detect** mkdocs sites so `sitebuild.yaml` is optional (currently the default profile when `sitebuild.yaml` is absent is hard-coded Hugo).
- **`MkDocsVersionConfig`** — analog of the existing `HugoVersionConfig`, so version pinning of `mkdocs` + `mkdocs-material` is first-class instead of "whatever the builder image happened to install".
- **Pin** the baked-in `mkdocs` / `mkdocs-material` versions in the builder Dockerfiles (today they're unpinned).
- **Tests** — fixture mkdocs site in `test/fixtures/`, e2e build, content-hash-cache integration test.

The docs PRD is the proximate driver, but mkdocs is the second-most-common generator after Hugo, so the polish earns its keep beyond this site.

**(b) `hulation-docs` repo — author the site, plus a `mkdocs.yml` that the new builder consumes.**

Production config (lives on the docs Hula host):

```yaml
servers:
  - host: docs.tlaloc.us
    id: hulation-docs
    path_prefix: "/hulation"
    root_git_autodeploy:
      repo: https://github.com/tlalocweb/hulation-docs.git
      ref: { branch: main }
      hula_build: production
      builder: mkdocs
      build_env:
        - MKDOCS_STRICT=1
```

The deployment target is one Hula instance behind ACME, no backends. We turn on Hula's own analytics on the docs site once it's stable — useful product feedback (which pages get visited, search terms, conversion to install-link clicks).

**Bootstrapping note.** Until the `mkdocs` builder ships in `hulabuild`, CI builds the site and commits the rendered output to a `gh-pages`-style branch that Hula serves with a plain `root:` — this is the *temporary* fallback during M0–M1, not a long-term mode. The fallback is removed at M4.

## 9. Authoring & build

- **Source format:** Markdown (CommonMark + Material extensions) under `docs/`.
- **Repo layout:**
  ```
  hulation-docs/
  ├── docs/
  │   ├── index.md
  │   ├── quickstart/{one-server,hugo,hulactl}.md
  │   ├── concepts/*.md
  │   ├── reference/{config,hulactl,http-api,grpc-api,hooks,env}.md
  │   ├── examples/*.md
  │   ├── operations/*.md
  │   └── assets/{images,css}
  ├── overrides/                  # Material theme overrides if needed
  ├── mkdocs.yml
  ├── requirements.txt            # mkdocs-material + plugins, pinned
  └── PRD.md                      # this file (until v1 ships, then archive)
  ```
- **Local dev:**
  ```bash
  pip install -r requirements.txt
  mkdocs serve   # http://127.0.0.1:8000, hot reload
  ```
- **CI:** GitHub Actions (`.github/workflows/build.yml`) — install Python, `mkdocs build --strict`, upload `site/` as artifact, optionally rsync to a release branch that Hula pulls.
- **Linting:** `markdownlint` and `lychee` (link check) on PRs. Fail on broken internal links.
- **Code-snippet hygiene:** every shell block tagged with the language; YAML examples validated against a JSON Schema generated from `config/*.go` (stretch goal — see §11).
- **Config-reference generation:** v1 is hand-authored from the struct tags. Stretch: a small Go program in this repo that walks `github.com/tlalocweb/hulation/config` and emits `docs/reference/config.md` from the `yaml`/`env`/`default`/`test` tags. That keeps drift bounded.

## 10. Success metrics

| Metric | Target (90 days post-launch) |
|--------|------------------------------|
| Median time-to-success on Quick Start (1 server) | < 5 min, validated with two external testers |
| % of `config.yaml` keys with at least one example | 100% |
| % of `hulactl` subcommands with at least one example | 100% |
| Broken-link CI failures merged to main | 0 |
| Search → page-view conversion (search led to a click) | > 60% (Hula analytics, once enabled) |
| Site Lighthouse performance | > 95 mobile, > 99 desktop (Material defaults already get us most of the way) |
| Total page weight (no-cache, home page) | < 250 KB |

## 11. Open questions

Still open:

1. **Agent CA verification semantics.** The plan says the agent YAML's `ca:` block lets the agent verify hula's serving cert, but hula's serving cert today is Let's Encrypt (or operator-provided), not signed by the Agent CA. Clarify: does the agent pin via the Agent CA, the system trust store, or hostname pinning to `hula_host`? *(Action: read `pkg/agent/mtls/middleware.go` and the rust client TLS setup; needed before §6.6.1 can be authored accurately.)*
2. **Versioning.** Single-channel v1 is fine today. When Hula cuts a stable v1.0, adopt `mike` for versioned docs (`/latest`, `/0.20`, etc.)? *(Defer to v2 unless Ed wants this from launch.)*
3. **Auto-generated config reference.** Hand-author for v1, or invest a day in a tag-walker that emits `docs/reference/config.md` from the `yaml`/`env`/`default` struct tags? *(Recommend hand-author for launch, file follow-up issue.)*
4. **Search backend.** Material's built-in lunr is fine for ~150 pages. Algolia DocSearch is the upgrade if we outgrow it. No action needed at launch.
5. **Comments / feedback widget.** Material supports a "Was this page helpful?" widget. Out of scope for v1, easy add later.

### 11.1 Decided

| # | Question | Decision |
|---|----------|----------|
| D1 | Primary persona / home-page hero | **Hugo flow** (Persona B). Home page leads with "Deploy your Hugo site on Hula"; other quick starts are secondary tiles. |
| D2 | Tone / voice | **Terse, normal developer-doc register.** Match the README. Imperative, code-first. No friendliness theater. |
| D3 | Builder mode (Hula deploying its own docs) | **Mode A required.** MkDocs support already exists in `hulabuild`; the gating work is refinement (auto-detection, version pinning, tests) tracked at `hulation/MKDOCS_BUILDER_PLAN.md`. The CI-built `gh-pages` bootstrap is used only during M0–M1 while those refinements land. |
| D4 | Production domain | `https://docs.tlaloc.us/hulation` (path-prefixed on the existing tlaloc.us host). |
| D5 | hulaagent docs preview banner | **No banner.** Ship hulaagent docs as the implementation lands; iterate. |
| D6 | `hula-agent` distribution channel | **Same release as the Go binaries.** `installtools.sh` installs `hulactl` *and* `hula-agent` from one release tarball. Implies a release-pipeline change in the main repo to cross-build the Rust binary alongside the Go ones. |
| D7 | Docs prose license | **Same dual AGPLv3 / SSPL-1.0 as the main repo**, including the no-AI-training clause carried into the docs footer. No CC-BY split. |
| D8 | Drift control via CI | **Approved.** Add a check in the main `hulation` repo that fails (or nags — TBD severity) when `config/`, `client/`, or `handler/` change without a paired `hulation-docs` PR or issue reference in the commit trailer. |

## 12. Milestones

| Milestone | Scope | Target |
|-----------|-------|--------|
| **M0 — Skeleton** | mkdocs-material scaffold, theme configured, CI green (`mkdocs build --strict`, markdownlint, lychee link-check). Bootstrap deploy: CI commits rendered output to a `gh-pages` branch that Hula serves from a plain `root:`. License files copied from main repo. | Week 1 |
| **M1 — Quick Starts** | Hugo Quick Start (the hero path) + One Server (Docker) + hulactl. Externally tested by at least two non-author readers. Home page (§6.0) live. | Week 2 |
| **M2 — Reference** | Full `config.yaml` reference, full `hulactl` reference (incl. `create-agent` admin flow), HTTP API outline, Agents concept page, Agent YAML reference. | Week 3–4 |
| **M3 — Cookbook** | At least 8 of the listed examples written and verified end-to-end. hulaagent Quick Start (§6.3.5) and HLAP wire reference written as the underlying phases land — no banners. | Week 5 |
| **M4 — Public launch** | **Mode A live** — MkDocs auto-detection + version pinning landed in `hulabuild` (per `MKDOCS_BUILDER_PLAN.md` P0–P2), docs site auto-deploys via `root_git_autodeploy`, the bootstrap gh-pages branch is retired. Site live at `docs.tlaloc.us/hulation`. README in main repo links here. Drift-control CI (D8) merged in main repo. | Week 6 |
| **M5 — Polish** | gRPC reference, HA examples, Hula's own analytics enabled on docs site, auto-generated config reference (if invested). | Week 7–8 |

## 13. Risks

- **Drift between docs and code.** Mitigation per D8: CI in the main `hulation` repo runs a "docs touched?" check whenever `config/`, `client/`, or `handler/` change, and links to this repo's issue template.
- **`hulabuild` mkdocs polish slips past M4.** Per D3, Mode A is required. The work is now smaller than originally scoped (see `MKDOCS_BUILDER_PLAN.md`) — auto-detection, `MkDocsVersionConfig`, builder-image version pinning, tests. Each item is independently shippable. Mitigation: P0 (pin builder-image versions) lands this week, P1 (auto-detection) aligns with M0, P2 (`MkDocsVersionConfig`) aligns with M1 — so M4's Mode-A switch only depends on whichever of the four items have actually merged by then. The temporary `gh-pages` bootstrap covers any slippage gracefully.
- **Cross-language release pipeline.** D6 says `hula-agent` ships in the same release as the Go binaries. The release process today builds Go for linux/macOS × amd64/arm64; adding cross-compiled Rust (rustls, no openssl) is a real CI-pipeline change. Mitigation: prove out the Rust release flow before M3 so the hulaagent Quick Start in M3 can rely on it. If it's not ready, the Quick Start ships pointing at "build from source" until the release pipeline catches up — explicit fallback, not a blocker.
- **Snippet rot.** Quick Start snippets must be re-run on every Hula release. Add to the main repo's release checklist as part of D8's drift-control work.
- **Documenting partially-shipped surface.** hulaagent (Phases 1–3 landed, 4–6 ahead) is the live case. Per D5, no preview-banner mechanism — instead, sections are written as the implementation lands and we iterate. Risk: a reader hits a doc page describing a verb the runtime doesn't yet implement. Mitigation: do not merge a doc page until the corresponding runtime is in a Hula release; M3 owns this discipline.

---

*Draft — feedback welcome. Reviewers please leave comments inline rather than rewriting; we'll merge once §11 is closed.*
