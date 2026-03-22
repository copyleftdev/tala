# Design Decisions

These ten decisions are settled. They define the architectural boundaries of TALA and are not subject to re-evaluation.

## 1. Intent Is a First-Class Primitive

**Decision**: An intent is not a wrapper around a command string. It is a structured representation of a desired outcome: `I = f(C, X, P)`, where C is the command, X is the execution context, and P is prior knowledge from historical embeddings.

**Reasoning**: Command strings are lossy. `kubectl rollout restart deployment/api` carries no information about why the restart was performed, what state the cluster was in, or what outcome was expected. By modeling intent as a composite of command, context, embedding, and outcome, TALA captures the full semantic surface. This enables semantic recall, pattern detection, and adaptive replay -- none of which are possible over raw strings.

**Alternative rejected**: Wrapping commands with metadata annotations (like shell plugins that tag history entries). This approach is incremental but does not change the data model. You still have a flat list with decorations. TALA requires a graph-native data model where intent is the node, not the string.

## 2. DAG, Not Tree

**Decision**: The narrative graph is a directed acyclic graph with probabilistic edges, not a tree.

**Reasoning**: Real workflows are not hierarchical. A single intent can have multiple causes (a failed deploy and an alert firing together trigger an incident response). A single intent can feed multiple downstream actions (a successful build triggers both a deploy and a notification). Trees cannot represent shared causality or fan-out without duplication. DAGs can.

**Alternative rejected**: Tree-structured history (like process trees in an OS). Trees force a single-parent constraint that does not reflect how human workflows actually branch and merge. Undo trees (like vim's undo) capture branching but not convergence.

## 3. Binary-First Storage

**Decision**: TALA uses a custom binary format (TBF) for on-disk segments. No JSON, JSONL, CSV, or text-based serialization anywhere in the storage path.

**Reasoning**: See [Why Not JSON](./why-not-json.md) for the full argument. The short version: JSON requires parsing on every read, cannot be memory-mapped for zero-copy access, wastes bytes on field names and delimiters, and cannot be aligned for SIMD operations. TBF is a columnar + CSR hybrid designed for TALA's specific access patterns.

**Alternative rejected**: JSON Lines for human readability, Parquet for columnar analytics, FlatBuffers/Cap'n Proto for zero-copy. Each of these serves a different primary use case. None provides the combination of columnar embedding access, CSR graph traversal, Bloom membership testing, and 64-byte alignment that TALA requires.

## 4. 64-Byte Alignment

**Decision**: All embedding storage is aligned to 64-byte boundaries, both in memory (`AlignedVec`) and on disk (TBF embedding region).

**Reasoning**: 64 bytes is the cache line size on x86-64 and the natural alignment for AVX-512 (512-bit = 64-byte vectors). Even when running AVX2 (256-bit = 32-byte), 64-byte alignment ensures that loads never split across cache lines and that the transition to AVX-512 requires no storage format changes.

**Alternative rejected**: 32-byte alignment (sufficient for AVX2) or no alignment (rely on unaligned loads). Unaligned loads incur a penalty on some microarchitectures and prevent the use of aligned load instructions. 32-byte alignment would require a format version bump when AVX-512 is adopted.

## 5. SIMD with Runtime Dispatch

**Decision**: Compile all ISA variants (scalar, SSE4.1, AVX2+FMA, AVX-512, NEON, SVE) and select at startup based on `cpuid` / feature detection.

**Reasoning**: A single TALA binary must run on any x86-64 machine from 2008 (SSE4.1) to current (AVX-512). Compile-time selection via `target-cpu=native` produces binaries that crash on older hardware. Runtime dispatch via function pointers initialized once at startup adds zero per-call overhead (the pointer is resolved once, then called directly).

**Alternative rejected**: Compile-time CPU targeting (`RUSTFLAGS="-C target-cpu=native"`). This produces the fastest binary for one specific machine but requires separate builds per microarchitecture. Unacceptable for distributed deployment.

## 6. Trait Boundaries Between Crates

**Decision**: Inter-crate communication happens through traits defined in `tala-core`. No crate imports concrete types from a sibling crate in its public API.

**Reasoning**: Trait boundaries enforce the dependency graph at the type level. `tala-store` depends on `tala-core::IntentStore` (trait), not on `tala-graph::NarrativeGraph` (concrete type). This means `tala-store` can be compiled and tested without `tala-graph`. It means alternative implementations can be swapped in for testing (mock stores, deterministic clocks). It means the crate graph remains a DAG even as the system grows.

**Alternative rejected**: Direct struct imports between sibling crates. This creates tight coupling and eventually produces dependency cycles that Cargo rejects. Trait boundaries prevent this structurally.

## 7. Append-Only Segments

**Decision**: TBF segments are immutable once flushed from the hot buffer. No in-place mutation. The WAL provides durability for data not yet flushed.

**Reasoning**: Immutable segments enable lock-free reads via memory mapping. Multiple readers can mmap the same segment file without coordination. Compaction (merging small segments into larger ones) produces new segments and deletes old ones atomically. This is the same model used by LSM-tree storage engines (RocksDB, LevelDB, Cassandra) for good reason: it separates the write path (WAL + hot buffer) from the read path (immutable segments) and eliminates read-write contention.

**Alternative rejected**: Mutable segments with in-place updates. This requires page-level locking, complicates crash recovery (partial writes), and prevents safe concurrent mmap reads.

## 8. Tokio for Async

**Decision**: The daemon (`tala-daemon`) and network (`tala-net`) crates use the Tokio async runtime. Pure compute crates (`tala-embed`, `tala-wire`, `tala-graph`) are synchronous.

**Reasoning**: The daemon is I/O-bound (accepting connections, reading WAL, flushing segments, network replication). Tokio is the standard Rust async runtime for I/O-heavy workloads. Compute crates are CPU-bound (SIMD similarity, graph traversal, serialization). Making them async would add unnecessary overhead from task scheduling and `.await` points in tight loops. The boundary is clean: compute crates expose synchronous APIs, and the daemon wraps them in `spawn_blocking` or dedicated threads.

**Alternative rejected**: Fully synchronous design (thread-per-connection). This limits concurrency under high connection counts. Also rejected: fully async design (async everywhere). This forces compute-heavy code onto the async executor, starving I/O tasks.

## 9. Benchmark-Driven Development

**Decision**: Algorithms are proven in Criterion benchmarks before being integrated into the production code path.

**Reasoning**: TALA's value proposition depends on performance. If HNSW search takes 10ms instead of 139us, semantic recall becomes impractical for interactive use. By benchmarking first, we establish that an algorithm meets its target before building the system around it. Regressions on passing benchmarks are merge blockers, enforcing a performance floor that cannot silently degrade.

**Alternative rejected**: Implement first, optimize later. This approach accumulates performance debt and makes it difficult to identify which change caused a regression. Benchmark-first development catches problems at the point of introduction.

## 10. Spec-First

**Decision**: Specifications precede implementation. Every subsystem has a spec that defines its data structures, requirements (using RFC 2119 language), and performance targets before code is written.

**Reasoning**: Specs force design decisions to be made explicitly and documented permanently. They create a shared vocabulary between contributors. They make it possible to review a design without reading code. The spec dependency graph (spec-01 and spec-02 are standalone; spec-03 references both; spec-04 references spec-01 and spec-03) mirrors the crate dependency graph, ensuring architectural consistency.

**Alternative rejected**: Code-first development with documentation after the fact. This produces implementations that encode implicit design decisions, making it difficult for new contributors to understand why things are the way they are.
