# otel stack

Grafana observability stack for traces, metrics, and logs via OpenTelemetry.

## Services

| Service | Port | Purpose |
|---|---|---|
| Grafana | 3000 | Dashboards and visualization |
| Prometheus | 9090 | Metrics storage (30d / 2 GB) |
| Loki | 3100 | Log storage (30d) |
| Tempo | 3200 | Trace storage (30d) |
| OTel Collector | 4317 (gRPC), 4318 (HTTP) | OTLP ingestion — single entry point |
| Alertmanager | 9093 | Alert routing |
| Node Exporter | 9100 | Host system metrics |

## Architecture

```
Your apps / other stacks
        │
        ▼  OTLP gRPC :4317 / HTTP :4318
   otelcol (collector)
   ┌──────┼──────┐
   ▼      ▼      ▼
 Tempo  Prom   Loki
   └──────┴──────┘
              │
              ▼
           Grafana :3000
```

Tempo's metrics generator also pushes service-graph and span metrics into
Prometheus automatically, enabling RED metrics from traces without any extra
instrumentation.

## Retention and storage

| Backend | Time | Size cap |
|---|---|---|
| Prometheus | 30 days | 2 GB (enforced by `--storage.tsdb.retention.size`) |
| Loki | 30 days | ~2 GB (enforced by retention period + ingestion rate limits) |
| Tempo | 30 days | ~1 GB (time-based only; no hard size cap) |

Data is persisted to `/mnt/docker/otel/` on the host.

## Setup

```bash
# Create host data directories
sudo mkdir -p /mnt/docker/otel/{grafana,prometheus,loki,tempo,alertmanager}
sudo chown -R $USER:$USER /mnt/docker/otel

# Copy and fill in env
cp .env.example .env
$EDITOR .env

docker compose up -d
```

Grafana is available at http://localhost:3000. Default credentials are
`admin` / the password set in `.env`.

## Sending telemetry from other stacks

The collector is reachable two ways:

**Option 1 — host network** (easiest, no network config needed):
```
OTEL_EXPORTER_OTLP_ENDPOINT=http://host.docker.internal:4317
```

**Option 2 — join the `monitoring` network** (cleaner, uses DNS):
Add to your other `docker-compose.yml`:
```yaml
services:
  your-service:
    networks:
      - monitoring

networks:
  monitoring:
    external: true
    name: monitoring
```
Then set:
```
OTEL_EXPORTER_OTLP_ENDPOINT=http://otelcol:4317
```

## Alertmanager

Alerts are routed to a `null` receiver by default (silently dropped). Edit
`config/alertmanager/alertmanager.yml` to wire up real receivers (Slack,
PagerDuty, email, etc.) and reload:

```bash
docker compose kill -s SIGHUP alertmanager
```

## Grafana dashboards

Drop dashboard JSON files into `config/grafana/provisioning/dashboards/` and
they will be auto-loaded within 10 seconds. Community dashboards for this stack:

- Node Exporter Full: `1860`
- Loki dashboard: `13639`
- Tempo / tracing: `16310`

Import by ID via Grafana UI → Dashboards → Import.

### Provisioned dashboards

| File | Description |
|---|---|
| `errors.json` | Error rate and span failure tracking |
| `loki-logs.json` | Raw log explorer |
| `node-exporter.json` | Host system metrics |
| `ai-briefing-feature-health.json` | AI briefing pipeline feature health (trend detection, source credibility, link enrichment) |

### SQLite datasources

The `frser-sqlite-datasource` plugin is auto-installed via `GF_INSTALL_PLUGINS`.
Two SQLite datasources are provisioned for the ai-briefing pipeline:

| Datasource | UID | File |
|---|---|---|
| SQLite — Trends | `sqlite-trends` | `/home/alexander/ai-briefing/trends.db` |
| SQLite — Credibility | `sqlite-credibility` | `/home/alexander/ai-briefing/credibility.db` |

The ai-briefing directory is mounted read-only into the Grafana container at `/data/ai-briefing`.

## Environment variables

| Variable | Default | Description |
|---|---|---|
| `GRAFANA_ADMIN_PASSWORD` | `admin` | Grafana admin password |

See `.env.example`.
