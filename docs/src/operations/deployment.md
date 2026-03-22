# Deployment

TALA ships a Docker Compose stack that runs four simulation verticals, a Prometheus metrics backend, Grafana dashboards, and the Observatory front-end. Everything starts from a single `docker compose up`.

## Prerequisites

- Docker Engine 24+ with Compose v2
- 4 GB RAM minimum (each simulator uses ~200 MB, Prometheus and Grafana add ~500 MB)
- Ports 3000, 8080, 9090, 9101-9104 available on the host

## Quick Start

```bash
cd deploy
cp .env.example .env   # adjust values as needed
docker compose up -d
```

The stack builds the `tala-sim` binary from source using the workspace Dockerfile, then launches all services. First build takes a few minutes; subsequent starts use cached layers.

## Services

### Simulation Verticals

Four instances of `tala-sim` run independently, each modeling a different operational domain. Each exposes Prometheus metrics on its container's port 9090.

| Service | Container | Domain | Host Port | Default Rate |
|---------|-----------|--------|-----------|-------------|
| sim-incident | `sim-incident` | Incident Response | 9101 | 400ms |
| sim-deploy | `sim-deploy` | Continuous Deployment | 9102 | 300ms |
| sim-observe | `sim-observe` | Observability | 9103 | 200ms |
| sim-provision | `sim-provision` | Provisioning | 9104 | 350ms |

Each simulator runs five concurrent threads: ingest (generates intents at the configured rate), query (semantic search every `TALA_QUERY_INTERVAL_S`), insight (pattern detection every `TALA_INSIGHT_INTERVAL_S`), replay (adaptive replay every `TALA_REPLAY_INTERVAL_S`), and chaos (fault injection every `TALA_CHAOS_INTERVAL_S`). A sixth gauge-updater thread samples internal metrics every 5 seconds.

### Prometheus

```
prom/prometheus:v2.51.0 on port 9090
```

Scrapes all four simulators every 5 seconds via the `tala-sim` job. Retains 30 days of TSDB data. Lifecycle API is enabled for configuration reload without restart.

- Config: `deploy/prometheus/prometheus.yml` (mounted read-only)
- Data: `prometheus-data` named volume at `/prometheus`

### Grafana

```
grafana/grafana:10.4.0 on port 3000
```

Pre-provisioned with a Prometheus data source and the TALA Overview dashboard. Default credentials are `admin` / the value of `GRAFANA_PASSWORD` (defaults to `tala-demo`).

- Provisioning: `deploy/grafana/provisioning/` (datasources and dashboard loader, read-only)
- Dashboards: `deploy/grafana/dashboards/` (JSON dashboard definitions, read-only)
- Data: `grafana-data` named volume at `/var/lib/grafana`

### Observatory Dashboard (nginx)

```
nginx:1.25-alpine on port 8080
```

Serves the static Observatory front-end and proxies `/api/` requests to Prometheus, allowing the browser to query metrics without CORS issues.

- Static files: `deploy/dashboard/` mounted at `/usr/share/nginx/html`
- Config: `deploy/dashboard/nginx.conf` mounted at `/etc/nginx/conf.d/default.conf`

## Port Summary

| Port | Service | Protocol |
|------|---------|----------|
| 3000 | Grafana | HTTP |
| 8080 | Observatory Dashboard | HTTP |
| 9090 | Prometheus | HTTP |
| 9101 | sim-incident metrics | HTTP |
| 9102 | sim-deploy metrics | HTTP |
| 9103 | sim-observe metrics | HTTP |
| 9104 | sim-provision metrics | HTTP |

## Volume Mounts

The stack defines six named volumes:

| Volume | Used By | Mount Point | Purpose |
|--------|---------|-------------|---------|
| `incident-data` | sim-incident | `/data` | WAL, segments, HNSW index |
| `deploy-data` | sim-deploy | `/data` | WAL, segments, HNSW index |
| `observe-data` | sim-observe | `/data` | WAL, segments, HNSW index |
| `provision-data` | sim-provision | `/data` | WAL, segments, HNSW index |
| `prometheus-data` | prometheus | `/prometheus` | TSDB storage |
| `grafana-data` | grafana | `/var/lib/grafana` | Grafana state and plugins |

All simulation data volumes are independent. Removing one does not affect others.

## Networking

All services join a single bridge network (`tala-net`). Simulators are addressable by container name from Prometheus (e.g., `sim-incident:9090`). The nginx proxy reaches Prometheus at `prometheus:9090`.

## Environment Configuration

Copy `.env.example` to `.env` in the `deploy/` directory and adjust values before starting. See [Configuration](./configuration.md) for the full variable reference.

```bash
cp .env.example .env
# Edit .env to set ingest rates, chaos parameters, Grafana password
```

## Scaling

To adjust ingest throughput, modify the `*_RATE_MS` variables in `.env`. Lower values produce more intents per second. Each simulator runs independently, so you can also scale by adding additional verticals to `docker-compose.yml` following the existing pattern.

To add a fifth simulator:

1. Add a new service block in `docker-compose.yml` using the `*sim-defaults` anchor
2. Set `TALA_VERTICAL` to the domain name
3. Map a new host port (e.g., `9105:9090`)
4. Add a corresponding volume
5. Add the scrape target to `prometheus/prometheus.yml`

## Restart and Recovery

All services use `restart: unless-stopped`. If a simulator crashes, Docker restarts it automatically. The WAL ensures durability across restarts when persistent storage is configured.

To restart a single service:

```bash
docker compose restart sim-incident
```

To restart the entire stack:

```bash
docker compose restart
```

## Tear Down

Stop and remove containers, keeping volumes:

```bash
docker compose down
```

Stop, remove containers, and delete all data:

```bash
docker compose down -v
```

This destroys all WAL data, segment files, Prometheus TSDB, and Grafana state. Use only when you want a clean start.

## Viewing Logs

```bash
# All services
docker compose logs -f

# Single service
docker compose logs -f sim-incident

# Last 100 lines
docker compose logs --tail=100 sim-deploy
```

Simulators log to stderr. Key log lines include the startup banner (configuration summary), periodic progress reports (every 100 intents), and chaos event announcements.
