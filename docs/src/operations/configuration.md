# Configuration

TALA simulators are configured entirely through environment variables. In the Docker Compose deployment, these are set in the `.env` file (copied from `deploy/.env.example`). When running `tala-sim` directly, export them in the shell.

## Environment Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `TALA_VERTICAL` | `medical` | Operational domain for the simulator. Supported values: `incident`, `deploy`, `observe`, `provision`. Legacy names (`medical`, `financial`, `ecommerce`, `gaming`) are mapped to these for backward compatibility. |
| `TALA_RATE_MS` | `500` | Milliseconds between intent ingestions. Lower values increase throughput. The Docker Compose file overrides this per-vertical via `INCIDENT_RATE_MS`, `DEPLOY_RATE_MS`, `OBSERVE_RATE_MS`, and `PROVISION_RATE_MS`. |
| `TALA_CHAOS_INTERVAL_S` | `60` | Seconds between chaos engine trigger checks. The engine sleeps for this duration between evaluations. |
| `TALA_CHAOS_PROBABILITY` | `0.15` | Probability (0.0 to 1.0) that a chaos event fires on each trigger check. At the default of 0.15 with a 60-second interval, expect roughly 9 events per hour. |
| `TALA_DATA_DIR` | `/data` | Directory for persistent storage (WAL, segments, HNSW index). Set to an empty string to run entirely in memory with no disk I/O. |
| `TALA_METRICS_PORT` | `9090` | TCP port for the Prometheus metrics HTTP server. Each simulator binds to `0.0.0.0:<port>` and serves `/metrics` and `/health` endpoints. |
| `TALA_DIM` | `384` | Embedding vector dimensionality. Must match the model used for intent extraction. The default of 384 corresponds to a lightweight sentence transformer. |
| `TALA_HOT_CAPACITY` | `10000` | Maximum number of intents held in the hot buffer before flushing to a TBF segment. Larger values reduce flush frequency but increase memory usage. |
| `TALA_QUERY_INTERVAL_S` | `10` | Seconds between semantic query probes. The query thread selects a random probe command from the vertical's vocabulary, embeds it, and searches the HNSW index for the top 10 matches. |
| `TALA_INSIGHT_INTERVAL_S` | `30` | Seconds between insight generation runs. Each run performs k-means clustering over recent embeddings with k=3 to detect intent patterns. |
| `TALA_REPLAY_INTERVAL_S` | `45` | Seconds between adaptive replay attempts. The replay thread picks a random recent intent and generates a replay plan of depth 3, traversing causal edges in the narrative graph. |
| `GRAFANA_PASSWORD` | `tala-demo` | Admin password for Grafana. Only used in the Docker Compose deployment. Change this before exposing Grafana to a network. |

## Docker Compose Overrides

The `docker-compose.yml` maps several top-level `.env` variables to per-service equivalents:

| `.env` Variable | Compose Mapping | Affected Services |
|----------------|-----------------|-------------------|
| `INCIDENT_RATE_MS` | `TALA_RATE_MS` | sim-incident |
| `DEPLOY_RATE_MS` | `TALA_RATE_MS` | sim-deploy |
| `OBSERVE_RATE_MS` | `TALA_RATE_MS` | sim-observe |
| `PROVISION_RATE_MS` | `TALA_RATE_MS` | sim-provision |
| `CHAOS_INTERVAL_S` | `TALA_CHAOS_INTERVAL_S` | all sim-* services |
| `CHAOS_PROBABILITY` | `TALA_CHAOS_PROBABILITY` | all sim-* services |
| `GRAFANA_PASSWORD` | `GF_SECURITY_ADMIN_PASSWORD` | grafana |

## Tuning Guidelines

### Ingest Rate

The ingest rate controls how fast intents flow into the system. At 200ms (the observe vertical default), the simulator produces 5 intents per second, or 300 per minute. At 400ms, it produces 2.5 per second.

The TALA ingest pipeline (extract, WAL append, HNSW insert, edge formation, hot buffer push) typically completes in 1-2ms, so rates down to about 5ms are sustainable on modern hardware without backpressure.

### Hot Buffer Capacity

The hot buffer accumulates intents in memory and flushes to a TBF segment when full. A capacity of 10,000 with 384-dimensional embeddings uses approximately:

- Intent metadata: ~200 bytes each = 2 MB
- Embeddings: 384 * 4 bytes * 10,000 = 15.4 MB
- Total: ~18 MB per simulator

Increase for higher throughput and fewer disk writes. Decrease for lower memory footprint and more frequent segment creation.

### Chaos Parameters

To disable chaos entirely, set `TALA_CHAOS_PROBABILITY=0.0`. To make chaos frequent for stress testing, increase probability and decrease interval:

```bash
CHAOS_INTERVAL_S=10
CHAOS_PROBABILITY=0.5
```

This fires a chaos event roughly every 20 seconds.

### In-Memory Mode

Set `TALA_DATA_DIR=""` (empty string) to bypass all disk I/O. The WAL is not created, segments are never flushed, and all data lives in the hot buffer until the process exits. Useful for benchmarking pure compute performance without filesystem variability.
