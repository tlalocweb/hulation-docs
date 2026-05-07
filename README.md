# hulation-docs

Source for the [Hulation](https://github.com/tlalocweb/hulation) documentation
site. Built with [MkDocs Material](https://github.com/squidfunk/mkdocs-material),
deployed by Hula itself once the site lands at `https://docs.tlaloc.us/hulation/`.

## Build locally

```bash
python3 -m venv .venv
.venv/bin/pip install -r requirements.txt
.venv/bin/mkdocs serve     # http://127.0.0.1:8000, hot reload
```

Or one-shot:

```bash
.venv/bin/mkdocs build --strict
# output in ./site/
```

CI runs `mkdocs build --strict` on every push (see
[`.github/workflows/build.yml`](.github/workflows/build.yml)).

## Layout

```
docs/
├── index.md                 # home (Hugo-first hero per PRD §6.0)
├── quickstart/              # the four primary entry points
├── concepts/                # architecture + per-feature deep dives
├── reference/               # config.yaml, hulactl, HTTP API, HLAP, ...
├── examples/                # cookbook
├── operations/              # upgrade / backup / observability / troubleshooting
├── changelog.md
└── license-faq.md
```

`PRD.md` at the repo root tracks the planned outline and the design decisions
(D1–D8). Many pages are still stubs — the roadmap is in §12 of the PRD.

## Pinning

`requirements.txt` pins MkDocs and mkdocs-material to the same versions baked
into the [`hula-builder-default` image](https://github.com/tlalocweb/hulation/blob/main/builder-images/README.md).
When the builder image bumps, bump these so local renders match production.

## License

This documentation, including all examples and code snippets, is dual-licensed
under **AGPLv3** and **SSPL-1.0** to match the main project. The same
no-AI-training clause that applies to the source code applies to the docs prose,
examples, and configuration snippets. See
[`LICENSE`](LICENSE) and [`SSPL1_0_LICENSE.md`](SSPL1_0_LICENSE.md).

For commercial licenses (HA / managed-service / OEM / AI-assisted-development
rights), contact `licensing@tlaloc.us`.
