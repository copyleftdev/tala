# Crate Dependency Graph

TALA is structured as a Cargo workspace of 12 crates. The dependency graph is a strict DAG: dependencies flow downward from foundation types to application binaries. No cycles exist. `cargo-deny` enforces this invariant via `[bans]` configuration.

## Dependency Hierarchy

```
                        tala-core
                       /    |    \
                      /     |     \
               tala-wire  tala-embed  (no inter-dependency)
                  |    \    /   |
                  |  tala-store |
                  |      |      |
             tala-graph  |  tala-intent
               /    \    |    /
        tala-weave  tala-kai
                  \   |   /
                tala-net
                    |
              tala-daemon
                    |
               tala-cli
```

## Governing Invariants

**No dependency cycles.** The workspace resolver enforces this structurally. A crate may only depend on crates above it in the hierarchy.

**Traits define boundaries.** Inter-crate communication happens through trait objects and generic bounds defined exclusively in `tala-core`. No concrete types from sibling crates appear in public APIs. `tala-store` implements `IntentStore`; `tala-embed` provides the similarity functions consumed through the `Embedder` trait; `tala-graph` implements `GraphEngine`. Each crate can be compiled, tested, and benchmarked in isolation.

**tala-core defines, others implement.** All shared vocabulary types (`Intent`, `Edge`, `IntentId`, `RelationType`, `TalaError`) and all trait definitions (`IntentStore`, `Embedder`, `GraphEngine`, `IntentExtractor`, `Transport`) live in `tala-core`. Implementation code lives in the crate that owns the concern.

## Crate Reference

| Crate | Layer | Description |
|-------|-------|-------------|
| `tala-core` | Foundation | Types, traits, error taxonomy. Zero dependencies beyond `std`. Every other crate imports this. |
| `tala-wire` | Storage | TBF binary format reader/writer: columnar serialization, CSR edges, bloom filters, B+ tree index. |
| `tala-embed` | Compute | SIMD-accelerated vector operations (AVX2, NEON), HNSW approximate nearest-neighbor index, quantization (f32/f16/int8). |
| `tala-store` | Storage | Storage engine: WAL, hot buffer, segment lifecycle, HNSW-backed semantic query. Implements `IntentStore`. |
| `tala-graph` | Compute | Narrative graph: adjacency-list DAG, BFS/DFS traversal, edge formation, narrative extraction and diffing. |
| `tala-intent` | Pipeline | Intent extraction: command tokenization, classification, context assembly, embedding generation. Implements `IntentExtractor`. |
| `tala-weave` | Execution | Replay engine: DAG-ordered scheduling, environment adaptation, idempotency detection, failure recovery. |
| `tala-kai` | Analysis | Insight engine: failure clustering, pattern detection, predictive suggestions, narrative summarization. |
| `tala-net` | Network | Distributed mode: QUIC transport, SWIM membership, Raft consensus, partition-aware replication. |
| `tala-daemon` | Application | `talad` binary: Unix socket server, ingest pipeline orchestration, event hooks, metrics, health checks. |
| `tala-cli` | Application | `tala` binary: user-facing CLI (`find`, `replay`, `diff`, `why`, `stitch`, `status`), output formatting. |
| `xtask` | Tooling | Custom build tasks: formatting, linting, benchmarking, coverage, release packaging. |

## Async Runtime Boundaries

Not every crate uses async. The workspace draws a clear line:

- **Sync crates:** `tala-core`, `tala-wire`, `tala-embed`, `tala-graph`. These are pure computation or I/O-free. They use `rayon` for parallelism where applicable, but no Tokio dependency.
- **Async crates:** `tala-daemon`, `tala-cli`, `tala-net`, `tala-weave`. These depend on Tokio for I/O multiplexing, network transport, and concurrent task management.
- **Bridging:** `tala-store` is structurally sync but used within `talad`'s async context. ONNX Runtime and CUDA calls in `tala-embed` are blocking and dispatched via `spawn_blocking`.

`tala-core` remains runtime-agnostic. Its trait definitions use `async fn` via `async-trait` where necessary, but the crate itself has no runtime dependency.

## Feature Flags

Feature flags control optional capabilities. Default features cover the common single-node case.

| Crate | Feature | Default | Controls |
|-------|---------|---------|----------|
| `tala-wire` | `mmap` | yes | Memory-mapped segment access |
| `tala-wire` | `compression` | yes | lz4 and zstd segment compression |
| `tala-wire` | `encryption` | no | AES-256-GCM encrypted segments |
| `tala-embed` | `simd` | yes | SIMD vector operations |
| `tala-embed` | `cuda` | no | NVIDIA GPU acceleration |
| `tala-embed` | `vulkan` | no | Vulkan compute acceleration |
| `tala-embed` | `onnx` | yes | Local ONNX model inference |
| `tala-embed` | `remote-embed` | no | Remote embedding API |
| `tala-net` | `cluster` | no | Distributed mode |

## Dependency Versions

All shared dependency versions are declared in the workspace `[workspace.dependencies]` table. Individual crates reference them with `{ workspace = true }`. This prevents version skew across the workspace and centralizes upgrade decisions.

Key external dependencies:

| Dependency | Version | Used By |
|------------|---------|---------|
| `tokio` | 1.x | `tala-daemon`, `tala-cli`, `tala-net`, `tala-weave` |
| `uuid` | 1.x (v7, serde) | `tala-core` |
| `thiserror` | 2.x | `tala-core` |
| `memmap2` | 0.9.x | `tala-wire` |
| `rayon` | 1.x | `tala-embed` |
| `quinn` | 0.11.x | `tala-net` |
| `criterion` | 0.5.x | Benchmarks across all crates |
| `proptest` | 1.x | Property tests |
