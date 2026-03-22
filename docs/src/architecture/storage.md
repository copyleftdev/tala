# Storage Engine

The storage engine manages the lifecycle of intent data from initial capture through durable persistence. It is built on three layers, each optimized for a different phase of the data lifecycle, unified by the `StorageEngine` struct that implements the `IntentStore` trait.

## Three-Layer Architecture

```
┌────────────────────────────────────────────────────────┐
│                    StorageEngine                        │
│                                                         │
│  ┌──────────────────────────────────────────────────┐   │
│  │  WAL (Write-Ahead Log)                           │   │
│  │  Sequential append, fsync per batch              │   │
│  │  Crash recovery: replay into new segment         │   │
│  └──────────────────┬───────────────────────────────┘   │
│                     │                                    │
│  ┌──────────────────▼───────────────────────────────┐   │
│  │  Hot Buffer (in-memory accumulator)              │   │
│  │  Columnar layout, capacity-triggered flush       │   │
│  │  Serves real-time queries from memory            │   │
│  └──────────────────┬───────────────────────────────┘   │
│                     │ flush at capacity                  │
│  ┌──────────────────▼───────────────────────────────┐   │
│  │  Segments (immutable TBF files)                  │   │
│  │  Columnar nodes, aligned embeddings, CSR edges   │   │
│  │  Zero-copy mmap access, append-only              │   │
│  └──────────────────────────────────────────────────┘   │
│                                                         │
│  ┌──────────────────────────────────────────────────┐   │
│  │  HNSW Index (in-memory ANN)                      │   │
│  │  Semantic search across all stored intents       │   │
│  │  Parallel to WAL/Hot/Segment — always up to date │   │
│  └──────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────┘
```

## Write-Ahead Log

The WAL provides durability. Every intent is appended to the WAL before any other operation. If the process crashes mid-insert, the WAL can be replayed to recover all committed intents.

### Entry Format

Each WAL entry is a length-prefixed binary record:

| Field | Size | Description |
|-------|------|-------------|
| `payload_len` | 4 bytes | Total payload size (excludes this field) |
| `id` | 16 bytes | Intent UUID |
| `timestamp` | 8 bytes | Nanosecond epoch |
| `embed_len` | 4 bytes | Number of embedding dimensions |
| `embedding` | `embed_len * 4` bytes | f32 vector |
| `cmd_len` | 4 bytes | Command string length |
| `raw_command` | `cmd_len` bytes | UTF-8 command text |

The WAL uses a `BufWriter` with a 64KB buffer to amortize syscall overhead. After each append, `flush()` ensures data reaches the kernel buffer. The `fsync` policy is configurable: per-entry for maximum durability, or per-batch for throughput.

### Recovery

On startup, `replay_wal()` reads the WAL file sequentially, parsing entries until EOF or a truncated record (which indicates a crash mid-write). Truncated entries are discarded. All valid entries are re-ingested into the hot buffer and HNSW index.

```
Startup
    │
    ├─ WAL file exists?
    │     │
    │     ▼ yes
    │   replay_wal() → Vec<WalEntry>
    │     │
    │     ▼
    │   Re-insert each entry into HotBuffer + HNSW
    │     │
    │     ▼
    │   Truncate WAL (entries are now in hot buffer)
    │
    └─ no → fresh start
```

## Hot Buffer

The hot buffer is an in-memory accumulator that collects intents in columnar layout until flushed to a persistent TBF segment. It serves two purposes: batching writes for efficient segment construction, and providing immediate query access to recently ingested intents.

### Structure

Internally, the hot buffer stores each intent's fields decomposed into parallel vectors:

- `id: [u8; 16]` -- UUID bytes
- `timestamp: u64` -- nanosecond epoch
- `context_hash: u64` -- xxHash3 of execution context
- `confidence: f32` -- extraction confidence score
- `status: u8` -- outcome status enum
- `embedding: Vec<f32>` -- dense vector
- `parent_indices: Vec<usize>` -- indices of parent nodes (for edge construction)

This columnar decomposition mirrors the on-disk TBF layout and enables direct serialization without restructuring.

### Flush Cycle

The buffer has a configurable capacity (default: 64K nodes or 256MB). When a `push()` fills the buffer to capacity, it returns `true`, signaling the `StorageEngine` to initiate a flush.

The flush process:

1. A `SegmentWriter` is constructed with the buffer's embedding dimensionality.
2. Each buffered intent is pushed as a node, and parent edges are added.
3. `SegmentWriter.finish()` serializes all regions (columnar nodes, aligned embeddings, CSR edges, bloom filter) into a contiguous byte buffer.
4. The byte buffer is written to disk as a numbered `.tbf` file (e.g., `segment_000001.tbf`).
5. The hot buffer is cleared.

After the flush, the segment is immutable. Subsequent reads of those intents come from the segment via `mmap` (in production) or from the in-memory intent store (in the current implementation).

### Metrics

The storage engine tracks detailed per-operation metrics via atomic counters:

| Metric | Description |
|--------|-------------|
| `wal_append_ns` | Cumulative WAL append time |
| `hnsw_insert_ns` | Cumulative HNSW insert time |
| `hot_push_ns` | Cumulative hot buffer push time |
| `segment_flush_ns` | Cumulative segment write time |
| `total_bytes_flushed` | Total bytes written to segments |
| `segments_flushed_count` | Number of segments produced |

Lock contention is monitored per lock (WAL, HNSW, hot buffer, intents, index map) with acquisition count, wait time, hold time, and worst-case values.

## Segments

Segments are immutable TBF binary files produced by flushing the hot buffer. See the [TBF Binary Format](tbf.md) chapter for the complete segment structure.

Key properties:

- **Immutable.** Once written, a segment is never modified. This eliminates write-write conflicts, simplifies concurrency, and enables lock-free reads via `mmap`.
- **Self-contained.** Each segment includes its own bloom filter (for fast negative lookups), B+ tree index (for point lookups), and CSR edge index (for graph traversal). No external index is required to read a segment.
- **Tiered.** Segments progress through temperature tiers: hot (in-memory buffer), warm (mmap'd on NVMe), cold (compressed, possibly quantized, on bulk storage).

### Compaction

Over time, many small segments accumulate. Compaction merges them:

1. Input segments are opened via `mmap`.
2. Nodes are merge-sorted by timestamp.
3. The CSR edge index is rebuilt from the merged node set.
4. The string table is deduplicated across segments.
5. Bloom filter and B+ tree index are reconstructed.
6. A new compacted segment is written with the `is_compacted` flag set.
7. Segment references are atomically swapped.
8. Input segments are unlinked.

Compaction also applies zstd compression to the string table. Embedding regions are never compressed because they require random access for SIMD similarity operations.

## The StorageEngine Struct

`StorageEngine` unifies all three layers behind the `IntentStore` trait. It uses interior mutability (`Mutex` and `RwLock`) to satisfy `Send + Sync` requirements for concurrent access from `talad`'s async runtime.

```rust
pub struct StorageEngine {
    dim: usize,
    intents: RwLock<HashMap<IntentId, Intent>>,  // concurrent reads
    hnsw: Mutex<HnswIndex>,                       // search mutates visit state
    index_map: RwLock<Vec<IntentId>>,              // HNSW index → IntentId
    wal: Option<Mutex<Wal>>,                       // durability (None = in-memory mode)
    hot: Mutex<HotBuffer>,                         // accumulator
    segment_dir: PathBuf,                          // segment output directory
    segment_seq: Mutex<u64>,                       // monotonic segment counter
    metrics: Arc<StorageMetrics>,                  // lock and pipeline metrics
}
```

The `insert()` path acquires locks in a fixed order to prevent deadlock: WAL, then HNSW, then index map, then hot buffer, then intent store. The intent store is updated last because it represents the point at which the insert becomes visible to queries -- durability (WAL) and indexing (HNSW) must complete first.

The engine can operate in two modes:

- **Persistent mode** (`StorageEngine::open(dim, dir, capacity)`): WAL and segments are written to the specified directory. Full crash recovery is available.
- **In-memory mode** (`StorageEngine::in_memory(dim, capacity)`): No WAL, no segment persistence. Segment flushes are discarded. Suitable for testing and benchmarking.

## Query Engine

The `QueryEngine` provides HNSW-backed semantic search with exact cosine re-ranking.

### Semantic Search

`query_semantic(embedding, k)` performs a two-phase search:

1. **HNSW approximate search.** The query embedding is searched against the HNSW index with `ef = max(k, 50)`. This returns `k` approximate nearest neighbors by L2 distance in `O(log n)` time.

2. **Cosine re-ranking.** Each candidate's stored embedding is retrieved from the HNSW node, and exact cosine similarity is computed using SIMD-dispatched operations. Results are returned as `(IntentId, similarity)` pairs.

For a 10K corpus with `ef=50` and `k=10`, end-to-end query latency is approximately 151 microseconds.

### Edge Candidate Search

`find_edge_candidates(embedding, k)` is a specialized search used during edge formation. It requests `k * 4` candidates from HNSW (to increase recall), computes exact cosine similarity on all candidates, sorts by descending similarity, and returns the top-k. This provides the high-quality similarity scores needed for edge weighting while avoiding the `O(n^2)` cost of comparing the new intent against every existing intent.

### Temporal Query

`query_temporal(TimeRange)` performs a full scan of the in-memory intent store, filtering by timestamp range and returning results sorted by time. This is acceptable for the current scale; at larger corpus sizes, a B+ tree index over timestamps within segments will replace the scan.
