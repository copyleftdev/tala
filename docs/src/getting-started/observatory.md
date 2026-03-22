# Running the Observatory

The TALA Intent Observatory is a topology-first dashboard that visualizes the system in real time. It ships as a static HTML/CSS/JS application served by nginx, with Prometheus as the metrics backend.

## Architecture

```
┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐
│ sim-incident │  │ sim-deploy  │  │ sim-observe  │  │sim-provision│
│   :9101      │  │   :9102     │  │   :9103      │  │   :9104     │
└──────┬───────┘  └──────┬──────┘  └──────┬───────┘  └──────┬──────┘
       │                 │                │                  │
       └────────┬────────┴────────┬───────┴──────────┬───────┘
                │                 │                   │
         ┌──────▼──────┐   ┌─────▼─────┐   ┌────────▼────────┐
         │ Prometheus   │   │  Grafana  │   │   Observatory   │
         │   :9090      │   │  :3000    │   │   (nginx) :8080 │
         └─────────────┘   └───────────┘   └─────────────────┘
```

## Starting

```bash
cd deploy
cp .env.example .env
docker compose up -d
```

## What You'll See

### Landing Page

The landing page tells the TALA story — what it is, what problem it solves, and why it exists. A live topology visualization shows the four operational domains with real-time metrics flowing, even before you enter the dashboard.

### Dashboard Topology

After clicking **Enter Observatory**, you see a full-viewport topology graph:

- **Four domain nodes** — Incident Response, Continuous Deployment, Observability, Provisioning — each showing live intent counts, ingest rates, and patterns detected
- **Four capability nodes** — Extract (command to intent), Remember (HNSW semantic index), Persist (WAL durable log), Connect (causal edge formation)
- **Central narrative layer** — the TALA hub showing the total edge count of the growing narrative graph
- **Animated edges** with flowing particles representing intent being structured in real time

### Detail Panels

Click any node to open a detail drawer with deep telemetry:

- **Domain nodes** show narrative structure (graph nodes, causal edges, connectivity), what TALA learned (patterns, clusters, replays, insights), and outcome distribution
- **Capability nodes** show subsystem-specific metrics: pipeline waterfall, HNSW capacity gauge, WAL/hot buffer/segment stats, or graph relation breakdown
- **The narrative layer hub** shows aggregate intelligence across all domains plus lock contention health

### Chaos Mode Indicator

When the chaos engine injects faults, a floating indicator appears at the bottom of the topology showing:

- Current mode: **Failure Injection**, **Latency Storm**, **Retry Cascade**, **Mixed Chaos**, or **Stampede**
- Event rate in events/minute
- Which domains are affected
- Visual disruption on affected edges and nodes

## Configuration

See [Configuration](../operations/configuration.md) for tuning ingest rates, chaos probability, and other parameters.
