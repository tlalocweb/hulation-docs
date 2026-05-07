# Observability

Three signals to watch:

- **Logs** — Hula's stdout. Structured (JSON or human-readable), tagged
  by subsystem.
- **Health endpoints** — `/healthz` (liveness) and `/readyz` (readiness)
  for load-balancer probes.
- **ClickHouse** — visitor and event analytics. Queryable directly for
  live operational visibility.

This page covers all three, plus how to wire each into your existing
stack.

## Logs

### Format

Hula's logs are level-tagged. The default format is human-readable:

```text
14:22:01 INF [unified] listening on 443
14:22:02 INF [acme] obtained certificate for example.com
14:22:03 WRN [badactor] 1.2.3.4 score=57 (path=/wp-login.php)
14:22:04 ERR [sitedeploy] build abc123 failed: hugo not found in builder
```

Levels: `TRC`, `DBG`, `INF`, `WRN`, `ERR`, `FTL`. Subsystem tags in
square brackets — `unified`, `acme`, `sitedeploy`, `badactor`,
`chat`, `agent`, `analytics`, `forms`, `notify`, `team`.

### Filtering

Two config knobs select which subsystems log:

```yaml
log_tags: ""           # comma-separated allow-list (empty = all)
no_log_tags: ""        # comma-separated deny-list
```

Useful filters:

```yaml
# Only sitedeploy + analytics, drop the rest
log_tags: sitedeploy,analytics

# Everything except chat (chatty during normal operation)
no_log_tags: chat
```

Set via env at startup:

```bash
HULA_LOG_TAGS=sitedeploy,analytics ./start-with-docker.sh
```

### Tail and grep

```bash
./start-with-docker.sh --logs                    # follow live
./start-with-docker.sh --logs | grep '\[acme\]'  # ACME-only
./start-with-docker.sh --logs | grep ERR         # errors-only
```

For multi-host or production deployments, ship to a centralised log
store rather than tailing — see [Shipping](#shipping) below.

### Per-server correlation

Many log lines carry a `server_id` field for cross-correlation with
analytics rows:

```text
14:22:01 INF [forms] server=example form=newsletter submit ok
```

Filter to a single server:

```bash
./start-with-docker.sh --logs | grep 'server=example'
```

### Build logs

Builds emit their full builder-container output, prefixed by the
build ID:

```text
14:22:30 INF [sitedeploy] build=abc123 STATUS=cloning
14:22:32 INF [sitedeploy] build=abc123 [hugo] Total in 412 ms
14:22:33 INF [sitedeploy] build=abc123 STATUS=complete
```

Programmatic access:

```bash
hulactl build-status abc123
```

returns the same lines via the admin API.

## Health endpoints

Two unauthenticated endpoints designed for load-balancer probes:

### `GET /healthz`

Liveness — does the process exist and respond? Returns `200 OK` with
body `ok` if the listener is accepting connections; nothing else
matters.

```bash
curl -sf https://hula.example.com/healthz && echo alive
```

Use as a Docker `HEALTHCHECK` or k8s liveness probe.

### `GET /readyz`

Readiness — is Hula ready to serve traffic? Returns `200 OK` only when
all of these are true:

- Bolt + Raft state is open.
- ClickHouse is reachable.
- The agent CA is loaded.
- Any backend containers configured for this server are running and
  healthy (per Docker `HEALTHCHECK`, when set).

Returns `503 Service Unavailable` with a JSON body listing what's not
ready:

```json
{
  "ready": false,
  "checks": [
    {"name": "clickhouse", "ok": true},
    {"name": "raft", "ok": true},
    {"name": "agent_ca", "ok": true},
    {"name": "backend:myapi", "ok": false, "error": "container not running"}
  ]
}
```

Use as a k8s readiness probe — when `/readyz` returns 503, k8s stops
sending traffic until it recovers, but the pod stays running.

## ClickHouse-side observability

Hula's ClickHouse holds visitor / event analytics. Query directly for
live operational visibility — much richer than the admin UI can show.

### Connection

```bash
docker exec -it hula-clickhouse clickhouse-client \
  --user hula --password hula --database hula
```

Or any ClickHouse client pointing at `dbconfig.host:dbconfig.port`.

### Useful queries

**Per-server live visitors (last 5 min):**

```sql
SELECT server_id, count(DISTINCT visitor_id) AS active_visitors
FROM hula.events
WHERE event_type = 'hello'
  AND ts >= now() - INTERVAL 5 MINUTE
GROUP BY server_id
ORDER BY active_visitors DESC;
```

**Today's hello/landing/conversion mix:**

```sql
SELECT event_type, count() AS n
FROM hula.events
WHERE ts >= today()
GROUP BY event_type
ORDER BY n DESC;
```

**Form submissions, hourly:**

```sql
SELECT toStartOfHour(ts) AS hour, server_id, form_name, count() AS subs
FROM hula.events
WHERE event_type = 'form'
  AND ts >= now() - INTERVAL 24 HOUR
GROUP BY hour, server_id, form_name
ORDER BY hour DESC, subs DESC;
```

**Top probed paths today (badactor signals):**

```sql
SELECT server_id, url, count() AS hits, max(score) AS peak_score
FROM hula.badactor_audit
WHERE ts >= today()
GROUP BY server_id, url
ORDER BY hits DESC
LIMIT 20;
```

**Build duration distribution:**

```sql
SELECT
  server_id,
  quantile(0.5)(duration_ms) AS p50,
  quantile(0.95)(duration_ms) AS p95,
  quantile(0.99)(duration_ms) AS p99,
  count() AS n
FROM hula.builds
WHERE ts >= now() - INTERVAL 7 DAY
  AND status = 'complete'
GROUP BY server_id
ORDER BY p95 DESC;
```

### CSV export

The admin API supports `?format=csv` on most analytics endpoints:

```bash
curl -H "Authorization: Bearer $JWT" \
  "https://hula.example.com/api/v1/analytics/visitors?server_id=example&days=30&format=csv" \
  > visitors-30d.csv
```

Useful for ad-hoc reporting against a stored JWT.

## Shipping

Most production setups want logs in Loki / Elasticsearch / Datadog
rather than tail-and-grep.

### Vector

`/etc/vector/vector.toml`:

```toml
[sources.hula]
type = "docker_logs"
include_containers = ["hula"]

[sinks.loki]
type = "loki"
inputs = ["hula"]
endpoint = "https://loki.example.com"
encoding.codec = "json"
labels.host = "{{ host }}"
labels.app = "hula"
```

### Fluent Bit

```ini
[INPUT]
    Name        forward
    Listen      0.0.0.0
    Port        24224

[FILTER]
    Name        parser
    Match       hula.**
    Key_Name    log
    Parser      hula

[OUTPUT]
    Name        loki
    Match       hula.**
    Host        loki.example.com
```

Plus the corresponding parser in `parsers.conf` — Hula's text format is
parseable with a small regex.

### Promtail (Loki-native)

`/etc/promtail/config.yml`:

```yaml
scrape_configs:
  - job_name: hula
    docker_sd_configs:
      - host: unix:///var/run/docker.sock
        filters:
          - name: name
            values: [hula]
    relabel_configs:
      - source_labels: [__meta_docker_container_name]
        target_label: container
```

## Operator alerts

Independent of log shipping, Hula has a built-in operator-alert path
that fans out across email + APNs + FCM:

- Build failures
- Badactor blocks at threshold
- Goal anomaly (today's count is X% off from the trailing average)

Configured via the `mailer:`, `apns:`, `fcm:` blocks (see
[Configuration reference](../reference/config.md)) and the per-rule
configuration in the admin UI.

For "wake me up at 3am" coverage, prefer push (APNs / FCM) over email
— email through SMTP can stall on the server you're trying to alert
about.

## Metrics

Hula does **not** expose Prometheus metrics today. The roadmap has a
`/metrics` endpoint; track progress in
[the issue tracker](https://github.com/tlalocweb/hulation/issues).

For now, derive operational metrics from ClickHouse queries (visitor
counts, build durations, error rates) and from log volume / error rate
shipped to your log stack.

## Tracing

OpenTelemetry support is on the roadmap. Until then, trace context
propagation works at the HTTP-header level (Hula doesn't strip
`traceparent`) but Hula doesn't emit its own spans.

## Common observability questions

**How do I see what build a hulactl push triggered?**

```bash
hulactl builds <site> | head -1
# build-id, started_at, status, commit_sha, branch
```

**How do I see why a request was rate-limited?**

ClickHouse:

```sql
SELECT ts, client_ip, score, reason
FROM hula.badactor_audit
WHERE client_ip = '1.2.3.4'
ORDER BY ts DESC LIMIT 20;
```

**How do I see live agent connections?**

`hulactl list-agents` (Phase 6 — pending) shows active agents and last
handshake time. Until then, the audit log:

```sql
SELECT ts, agent_id, verb, site, result
FROM hula.agent_audit
ORDER BY ts DESC LIMIT 50;
```

**How do I correlate a build failure with the underlying cause?**

```bash
hulactl build-status <build-id>      # build's own log tail
./start-with-docker.sh --logs | grep build=<id>   # surrounding host log
```

## Where to go next

- [Troubleshooting](troubleshooting.md) — what to do when an alert
  fires.
- [Operator alerts (email + push)](../examples/operator-alerts.md) —
  how to wire up the alert fan-out.
- [Configuration reference — log_tags](../reference/config.md#top-level-keys) —
  log-filtering controls.
