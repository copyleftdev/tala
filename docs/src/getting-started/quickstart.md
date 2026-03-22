# Quick Start

Get TALA running in under five minutes.

## Prerequisites

- [Rust 1.82+](https://rustup.rs/) (recommended via `rustup`)
- [Docker](https://docs.docker.com/get-docker/) and Docker Compose (for the observatory demo)

## Clone and Build

```bash
git clone https://github.com/copyleftdev/tala.git
cd tala
cargo build --release
```

The workspace compiles 12 crates. The first build downloads dependencies and takes a few minutes.

## Run the Test Suite

```bash
cargo test --workspace
```

## Run Benchmarks

```bash
cargo bench
```

Criterion benchmarks live in `crates/<name>/benches/`. Results are written to `target/criterion/` with HTML reports.

## Launch the Observatory Demo

The fastest way to see TALA in action is the Docker Compose demo. It runs four simulated Linux operations domains — Incident Response, Continuous Deployment, Observability, and System Provisioning — each generating realistic shell commands, structuring them into intent narratives, and exporting metrics to Prometheus.

```bash
cd deploy
cp .env.example .env     # adjust settings if desired
docker compose up -d
```

Open your browser:

| Service | URL |
|---------|-----|
| **Observatory Dashboard** | [http://localhost:8080](http://localhost:8080) |
| **Prometheus** | [http://localhost:9090](http://localhost:9090) |
| **Grafana** | [http://localhost:3000](http://localhost:3000) |

The observatory landing page tells the TALA story, then lets you enter a live topology view of the system. Click any node to drill into its telemetry.

## Tear Down

```bash
docker compose down -v
```

The `-v` flag removes named volumes. Omit it to preserve data between restarts.
