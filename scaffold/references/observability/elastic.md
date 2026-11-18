# Observability: structured logs → Elastic stack (local, free)

## Architecture

```
api (structlog/pino, ECS JSON) ──► stdout (always)
                                └─► logs/*.ndjson (when LOG_FILE set)
                                          │
                                   Filebeat (filestream + ndjson parser)
                                          │
                                   Elasticsearch ──► Kibana (Discover, dashboards)
```

- **ECS (Elastic Common Schema)** field names (`@timestamp`, `log.level`, `message`, `service.name`, `http.request.method`, `url.path`, `http.response.status_code`, `event.duration`, `error.stack_trace`, plus our `request_id`) mean Kibana understands logs without custom mappings. The same fields work with OpenSearch or Loki later.
- **Apps never talk to Elasticsearch directly.** They write lines, and a shipper moves them. The app stays decoupled, and logging can't take the app down.
- Elastic's self-managed default distribution (Basic tier) is free. Elasticsearch and Kibana are also offered under AGPLv3. **OpenSearch** (Apache-2.0 fork) and **Grafana Loki** are the drop-in alternatives if licensing ever matters; app-side logging doesn't change.

## Local stack

From `assets/compose.yaml`, behind the `observability` profile so Postgres can run alone:

```
docker compose up -d                              # postgres only
docker compose --profile observability up -d      # + elasticsearch, kibana, filebeat
```

- Single node, **security disabled for local only**. That's acceptable only because every port binds to `127.0.0.1`. Never reuse this config on a server.
- Elasticsearch takes ~1–2 GB RAM (`ES_JAVA_OPTS` caps the heap at 1 GB). On Windows, make sure Docker Desktop/WSL has at least 4 GB.
- Version comes from `ELASTIC_VERSION` in `.env`. Check the current release; Elasticsearch, Kibana and Filebeat must match.
- Filebeat reads `./logs/*.ndjson` (mounted read-only) with `assets/filebeat.yml`. `--strict.perms=false` is needed because Windows bind mounts can't satisfy its config-permission check.
- First run: open Kibana at http://localhost:5601 → Discover → create a data view for `filebeat-*` with `@timestamp` as the time field. Put this step in the README.

## What to log

| Event | Level | Fields |
|---|---|---|
| App start/stop | info | `service.version`, config summary (no secrets) |
| Each HTTP request (one line, at completion) | info (warning for 4xx you care about, error for 5xx) | method, path, status, duration, request_id |
| Business events worth searching ("order placed") | info | domain IDs, never personal data |
| Handled anomalies (retry, fallback) | warning | what happened, what was done |
| Failures | error | `error.type`, `error.message`, `error.stack_trace` |
| Slow DB query | warning | duration, statement name (not the parameters) |

**Never:** passwords, tokens, auth headers, cookies, full request/response bodies, emails or names, card data. Redaction is configured in the logger (see the backend references) so it can't be forgotten at a call site.

## Correlation

- `request_id` binds at the start of each request and appears on every line logged during it. It's also returned to the client as `X-Request-ID`. Searching `request_id: "…"` in Kibana shows the whole story of one request.
- The frontend passes the ID it gets back into error reports, so a user-visible error can be traced to server logs.

## Next steps (when a project needs them)

- **Tracing/metrics:** OpenTelemetry SDK in the app → Elastic APM Server or the OTel Collector → same Elasticsearch. OTel is vendor-neutral, so the instrumentation survives a backend switch.
- **Retention:** an ILM policy so local indices don't grow forever (`delete` after 7 days is plenty for dev).
- **Alerts:** Kibana rules on error rate, available in the free tier.
