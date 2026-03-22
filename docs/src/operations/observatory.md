# The Observatory Dashboard

The TALA Intent Observatory is a topology-first dashboard that visualizes four operational domains, their TALA subsystems, and the narrative graph connecting them. It runs as a static HTML/CSS/JS application served by nginx, with Prometheus as the metrics backend.

## Architecture

```
sim-incident :9101    sim-deploy :9102    sim-observe :9103    sim-provision :9104
       |                    |                    |                     |
       +--------------------+--------------------+---------------------+
                                    |
                             Prometheus :9090
                                    |
                    +---------------+---------------+
                    |                               |
              Grafana :3000              Observatory (nginx) :8080
```

Each simulator exposes a `/metrics` endpoint in Prometheus exposition format. Prometheus scrapes all four every 5 seconds. The Observatory dashboard queries Prometheus through an nginx reverse proxy at `/api/`, which forwards requests to `http://prometheus:9090/api/`. This eliminates CORS issues and keeps the front-end purely static.

## Landing Page

The landing page presents the TALA concept: what it is, what problem it solves, why intent-native history matters. A live topology visualization in the background shows the four operational domains with animated edges, giving an immediate sense of the system's activity even before entering the dashboard.

Click **Enter Observatory** to transition to the full topology view.

## Topology View

The main dashboard is a full-viewport topology graph with three layers of nodes.

### Domain Nodes

Four outer nodes represent the operational verticals:

| Node | Domain | Metrics Shown |
|------|--------|---------------|
| Incident Response | `incident` | Intent count, ingest rate, patterns detected |
| Continuous Deployment | `deploy` | Intent count, ingest rate, patterns detected |
| Observability | `observe` | Intent count, ingest rate, patterns detected |
| Provisioning | `provision` | Intent count, ingest rate, patterns detected |

Each domain node displays live counters pulled from `tala_intents_ingested_total`, `tala_active_patterns`, and rate calculations over `tala_ingest_latency_us`.

### Capability Nodes

Four inner nodes represent the core TALA subsystems:

| Node | Subsystem | What It Shows |
|------|-----------|---------------|
| Extract | Intent extraction pipeline | Pipeline waterfall (extract, WAL, HNSW, edge, hot push latencies) |
| Remember | HNSW semantic index | Index size, average nodes visited per search, capacity gauge |
| Persist | WAL + segment storage | WAL entry count, segments flushed, bytes flushed, hot buffer fill ratio |
| Connect | Causal edge formation | Edge count, relation type breakdown (Causal, Temporal, Dependency, Retry, Branch) |

### Narrative Layer Hub

The central node represents the aggregate narrative graph. It shows total edge count across all domains, combined intelligence metrics (total patterns, clusters, replays, insights), and lock contention health drawn from `tala_lock_*` metrics.

### Animated Edges

Edges between nodes carry flowing particles that represent intent moving through the system. Particle speed and density reflect the current ingest rate. When chaos events fire, affected edges show visual disruption.

## Detail Panels

Click any node to open a detail drawer on the right side of the viewport.

### Domain Node Details

- **Narrative Structure**: graph nodes, causal edges, connectivity ratio
- **What TALA Learned**: patterns detected, clusters identified, replays generated, insights produced
- **Outcome Distribution**: success/failure/partial breakdown from `tala_intents_success_total`, `tala_intents_failure_total`, `tala_intents_partial_total`

### Capability Node Details

- **Extract**: pipeline waterfall showing per-stage latency (extract, WAL append, HNSW insert, edge search, hot push, segment flush)
- **Remember**: HNSW index size gauge, average search visited count, insert latency histogram
- **Persist**: WAL entries total, segments flushed, bytes flushed, hot buffer fill ratio gauge
- **Connect**: edge count, relation type breakdown, edge search latency

### Narrative Hub Details

Aggregate metrics across all domains, plus lock contention breakdown showing acquisitions, contentions, wait time, and hold time for each lock (intents, hnsw, index_map, wal, hot).

## Chaos Mode Indicator

When the chaos engine injects faults, a floating indicator appears at the bottom of the topology. It shows:

- **Current mode**: Failure Injection, Latency Storm, Retry Cascade, Mixed Chaos, or Stampede
- **Event rate**: chaos events per minute, calculated from `tala_chaos_events_total`
- **Affected domains**: which verticals have non-zero chaos counters
- **Visual disruption**: affected nodes pulse and edges distort

The chaos mode is inferred from which `tala_chaos_*` counters are incrementing. If only `tala_chaos_failures_injected` is rising, the mode is Failure Injection. If multiple counters are active simultaneously, the mode is Mixed Chaos.

See [Chaos Engineering](./chaos.md) for details on what each chaos event does and how to tune it.

## Prometheus Queries

The dashboard issues PromQL queries through the `/api/` proxy. Key queries:

| Metric | Query | Purpose |
|--------|-------|---------|
| Ingest rate | `rate(tala_intents_ingested_total[1m])` | Intents per second per vertical |
| Query latency p99 | `histogram_quantile(0.99, rate(tala_query_latency_us_bucket[5m]))` | Tail query latency |
| Chaos event rate | `rate(tala_chaos_events_total[5m])` | Chaos events per second |
| HNSW index size | `tala_hnsw_index_size` | Current vectors indexed |
| Hot buffer fill | `tala_hot_buffer_fill_ratio / 1000` | Fill ratio (0.0 to 1.0) |

## Accessing the Dashboard

After starting the Docker Compose stack:

```bash
# Observatory
open http://localhost:8080

# Grafana (for traditional time-series dashboards)
open http://localhost:3000

# Raw Prometheus UI
open http://localhost:9090
```
