# PR #236 — visual verification: control-bound PromQL label matchers render with data

Addresses @shmsr's review note: *"please be very careful when working with `?param` … test
dashboards with data and see if they render and render fine."*

**Verdict: ✅ The migrated dashboard renders fine with live data, the dashboard control
binds `?instance` at render time, and changing the control re-filters the panels correctly.
No `Parameter [?instance] value not found`.**

## Environment

- Local OTLP lab (`infra/docker-compose.yml`): Grafana, Prometheus→OTel collector→Elasticsearch + Kibana.
- Kibana `9.5.0-SNAPSHOT` @ `http://localhost:15601`, ES @ `http://localhost:19200`.
- Live metric data: `~1.6M` docs in `.ds-metrics-prometheusreceiver.otel-default-*`,
  6 distinct `service.instance.id` values (cadvisor:8080, prometheus:9090, node-exporter:9100,
  nginx-exporter:9113, + 2 otelcol UUIDs).
- Branch under test: `fix/230-promql-command-control-params`.

## What was migrated

A focused fixture exercising exactly bug #230 — a `$instance` dashboard template variable
(`label_values(up, instance)`) used inside a PromQL label matcher:

| Panel | Source PromQL |
|---|---|
| Scrape duration by instance | `max(scrape_duration_seconds{instance=~"$instance"}) by (instance)` |
| Samples scraped by instance | `avg(scrape_samples_scraped{instance=~"$instance"}) by (instance)` |

Migrated + uploaded with `obs-migrate` (`--data-view metrics-* --upload --ensure-data-views`).
Dashboard id `a1428ba7-130e-b8b9-4a47-be63de1089f2`.

## 1. The fix routes the matcher to native ES|QL (visible `?instance`)

Both panels compiled to native ES|QL with the param surfaced in a bindable `WHERE … RLIKE ?instance`
clause, **not** trapped inside an opaque `PROMQL(...)` command. Migration report note:

> `Native PROMQL skipped: target does not support PromQL label matcher params yet`

```
TS metrics-*
| WHERE service.instance.id RLIKE ?instance
| WHERE scrape_duration_seconds IS NOT NULL
| STATS scrape_duration_seconds = MAX(scrape_duration_seconds) BY time_bucket = TBUCKET(5 minute), service.instance.id
| SORT time_bucket ASC
```

The migrated dashboard also wires an `esqlControl` named `instance` (VALUES_FROM_QUERY over
`service.instance.id`) that binds `?instance`.

## 2. Runtime proof — Kibana actually binds `?instance` (captured from `/internal/search/esql_async`)

The exact query + params Lens sent to ES (raw bodies: `panel-*.network-request`):

| State | `params` sent | ES response |
|---|---|---|
| Control = `.*` (default, all) | `[{"instance": ".*"}]` | success, all 6 instances returned |
| Control = `node-exporter:9100` | `[{"instance": "node-exporter:9100"}]` | success, `documents_found: 1635`, **only** node-exporter rows |

Both responses: `is_partial: false`, `_clusters.successful: 1`. No query error toast; the only
console error in the session is an unrelated `500` on `/internal/dashboard/user_activity/refresh`
(Kibana activity telemetry), not the ES|QL query.

## 3. Contrast — the old (broken) behavior

When `?instance` is referenced but cannot be bound (the pre-fix case where the param was trapped
inside the PROMQL command and the control couldn't see it), ES rejects the query — the #230 symptom:

```
line 1:48: Unknown query parameter [instance];
line 1:22: Invalid pattern for RLIKE [service.instance.id RLIKE ?instance]: expected string literal or parameter
```

## Screenshots

| File | Shows |
|---|---|
| `01-all-instances-control-star.png` | Control = `.*` → both panels render with all 6 instance series |
| `00-control-options.png` | The `instance` control's options (sourced from live `service.instance.id`) |
| `02-filtered-node-exporter.png` | Control = `node-exporter:9100` → both panels re-render to that single series |
| `03-panel-closeup-node-exporter.png` | Close-up of the filtered "Scrape duration" panel |

## Reproduce

```bash
# lab already up (infra/docker-compose.yml)
.venv/bin/python -m observability_migration.adapters.source.grafana.cli \
  --input-dir <dir-with-fixture> --output-dir /tmp/out \
  --assets dashboards --data-view "metrics-*" --esql-index "metrics-*" \
  --upload --kibana-url http://localhost:15601 --ensure-data-views
# open http://localhost:15601/app/dashboards#/view/<id>, toggle the instance control
```
