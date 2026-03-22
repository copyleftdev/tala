<p align="center">
  <img src="media/logo.png" alt="TALA" width="140">
</p>

<h1 align="center">TALA</h1>

<p align="center">
  <strong>Intent-Native Narrative Execution Layer</strong><br>
  <em>Remember the why.</em>
</p>

<p align="center">
  <img alt="Rust" src="https://img.shields.io/badge/rust-1.82%2B-b7410e?style=flat-square&logo=rust">
  <img alt="License" src="https://img.shields.io/badge/license-MIT%2FApache--2.0-blue?style=flat-square">
  <img alt="AI" src="https://img.shields.io/badge/AI-embeddings%20%C2%B7%20HNSW%20%C2%B7%20semantic%20search-6c3ecb?style=flat-square">
  <img alt="Linux" src="https://img.shields.io/badge/linux-systems%20layer-333?style=flat-square&logo=linux&logoColor=white">
</p>

<p align="center">
  <code>rust</code> <code>ai</code> <code>machine-learning</code> <code>embeddings</code> <code>vector-search</code> <code>semantic-search</code> <code>linux</code> <code>operating-systems</code> <code>observability</code> <code>chaos-engineering</code> <code>devops</code> <code>sre</code> <code>narrative-graph</code> <code>intent</code> <code>developer-tools</code> <code>systems-programming</code>
</p>

<p align="center">
  <a href="#what-is-tala">What is TALA</a> &middot;
  <a href="#architecture">Architecture</a> &middot;
  <a href="#quick-start">Quick Start</a> &middot;
  <a href="#crate-layout">Crate Layout</a> &middot;
  <a href="#performance">Performance</a> &middot;
  <a href="#license">License</a>
</p>

---

## What is TALA

Run `history` on any machine. You get a flat list of commands, numbered sequentially, stripped of all context. No record of what you were trying to accomplish. No link between the `grep` that found the bug and the `git commit` that fixed it. No memory of which deployment sequence worked last Thursday when the same thing broke.

Fifty years of Unix, and the best we have is an append-only text file. **Commands are captured. Intent is lost.**

TALA reimagines shell history as a **causality-aware, graph-structured narrative** of intent. Every action is captured not as a string, but as a structured node in a directed acyclic graph -- linked to what caused it, what it depended on, what it produced, and how confident the system is in the outcome.

**Design hypothesis:** Systems that model `intent + causality + outcome` outperform systems that model `commands + sequence`.

### What this enables

- **Semantic recall** -- search by meaning, not regex
- **Adaptive replay** -- re-execute workflows that adapt to changed context
- **Pattern detection** -- identify recurring failure clusters automatically
- **Prediction** -- anticipate the next action from historical embeddings
- **Narrative extraction** -- pull coherent stories from thousands of interactions

## Architecture

TALA models intent as a first-class OS primitive: `I = f(C, X, P)` where C is the command, X is context, and P is prior knowledge from historical embeddings.

```
                     tala-core
                    /    |    \
                   /     |     \
            tala-wire  tala-embed  (no inter-dep)
               |    \    /   |
               |  tala-store |
               |      |      |
          tala-graph   |  tala-intent
            /    \     |    /
     tala-weave  tala-kai
               \   |   /
             tala-net
                 |
           tala-daemon
                 |
            tala-cli
```

### Subsystems

| Component | Role |
|-----------|------|
| **talad** | Intent ingestion, normalization, graph construction |
| **weave** | Adaptive execution and replay engine |
| **kai** | Insight, inference, and summarization engine |

### Data flow

```
raw_command -> Intent extraction -> Embedding (384-dim)
            -> WAL append -> HNSW index -> Edge formation
            -> Narrative graph -> Segment flush (TBF binary format)
```

## Quick Start

### Prerequisites

- Rust 1.82+ (`rustup` recommended)
- Docker and Docker Compose (for the observatory demo)

### Build from source

```bash
git clone https://github.com/YOUR_ORG/tala.git
cd tala
cargo build --release
```

### Run benchmarks

```bash
cargo bench
```

### Launch the observatory demo

The demo runs 4 simulated Linux operations domains (Incident Response, Continuous Deployment, Observability, System Provisioning) with chaos injection, exporting metrics to Prometheus. A live topology dashboard visualizes how TALA structures operational intent into narrative.

```bash
cd deploy
cp .env.example .env    # adjust settings if desired
docker compose up -d
```

Open [http://localhost:8080](http://localhost:8080) to see the TALA Intent Observatory.

| Service | URL |
|---------|-----|
| Observatory Dashboard | [localhost:8080](http://localhost:8080) |
| Prometheus | [localhost:9090](http://localhost:9090) |
| Grafana | [localhost:3000](http://localhost:3000) |

To tear down:

```bash
docker compose down -v
```

## Crate Layout

```
crates/
  tala-core/      Foundation types + traits (zero dependencies except std)
  tala-wire/      TBF binary format: columnar storage, CSR edges, bloom filters
  tala-embed/     SIMD-accelerated vector ops, HNSW index, quantization
  tala-graph/     Narrative graph engine with BFS/DFS traversal
  tala-store/     WAL, hot buffer, storage engine, query engine
  tala-intent/    Tokenization, embedding, intent classification pipeline
  tala-weave/     Adaptive replay: plan building, variable substitution
  tala-kai/       Insight engine: k-means clustering, pattern detection, prediction
  tala-daemon/    Unified daemon orchestrating all subsystems
  tala-net/       Distributed TALA: node identity, consistent hashing, TLV codec, in-process transport
  tala-cli/       Command-line interface: ingest, query, replay, insights, narrative inspection
  tala-sim/       Multi-domain simulator with chaos injection
```

### Core types

| Type | Crate | Description |
|------|-------|-------------|
| `Intent` | tala-core | Structured intent node: id, timestamp, embedding, command, outcome |
| `Edge` | tala-core | Directed edge: from, to, relation type, weight |
| `NarrativeGraph` | tala-graph | DAG with adjacency lists, BFS/DFS, narrative extraction |
| `HnswIndex` | tala-embed | Hierarchical navigable small world ANN index |
| `StorageEngine` | tala-store | Unified storage: WAL + HNSW + hot buffer + segment flush |
| `Daemon` | tala-daemon | Top-level orchestrator: ingest, query, replay, insights |

### Design decisions

- **DAG, not tree** -- the narrative graph is a directed acyclic graph with probabilistic edges
- **Binary-first storage** -- TBF format, no JSON. Columnar + CSR hybrid
- **64-byte alignment** -- all embedding storage aligned for AVX-512
- **SIMD with runtime dispatch** -- compile all ISA variants, detect at startup
- **Append-only segments** -- segments are immutable once flushed. WAL provides durability
- **Trait boundaries between crates** -- inter-crate communication via traits in tala-core

## Performance

Benchmarked on x86-64 with Criterion. All times are median.

| Operation | Result |
|-----------|--------|
| Batch cosine similarity (1K, single thread) | 41.7 us |
| Batch cosine similarity (100K, parallel) | 3.41 ms |
| HNSW search (10K vectors, ef=50, top-10) | 139 us |
| Semantic query (10K corpus, top-10) | 151 us |
| Full segment write (1K nodes) | 209 us |
| WAL append (1K entries, dim=384) | 730 us |
| Full ingest pipeline (1K intents) | 1.10 ms |
| CSR traverse (10K lookups) | 21.7 us |
| Bloom lookup (1K queries) | 27 us |

## Observatory

The TALA Intent Observatory is a topology-first dashboard for observing the system in real time. Click any node in the live infrastructure graph to drill into its telemetry.

Features:
- **Living topology** with animated data flow between verticals and subsystems
- **Click any node** to open a detail drawer with deep metrics
- **Chaos mode indicator** that detects and displays the current fault injection state (Failure Injection, Latency Storm, Retry Cascade, Stampede, Mixed Chaos)
- **Per-node telemetry**: pipeline waterfall, HNSW capacity gauge, lock contention, edge relation breakdown, storage layer metrics

## Terminology

| Term | Meaning |
|------|---------|
| Intent | Structured representation of a desired outcome: `I = f(C, X, P)` |
| Narrative Graph | Directed acyclic probabilistic graph of intent nodes and causal edges |
| TBF | TALA Binary Format -- the on-disk segment format |
| CSR | Compressed Sparse Row -- edge storage layout |
| HNSW | Hierarchical Navigable Small World -- ANN index |
| talad | The TALA daemon -- orchestrates all subsystems |
| weave | Adaptive replay engine |
| kai | Insight and inference engine |

## Contributing

```bash
# Run checks before submitting
cargo check --workspace
cargo test --workspace
cargo bench
```

### Conventions

- `#[repr(C)]` on all structs that cross FFI or mmap boundaries
- `unsafe` blocks require a `// SAFETY:` comment
- SIMD intrinsics behind `#[cfg(target_arch)]` with scalar fallback
- No `unwrap()` in library code -- use `?` or explicit error handling
- Benchmarks in `crates/<name>/benches/`, not tests

## License

Licensed under either of

- [MIT license](https://opensource.org/licenses/MIT)
- [Apache License, Version 2.0](https://www.apache.org/licenses/LICENSE-2.0)

at your option.
