# tala-store

The storage engine for TALA. Combines a write-ahead log (WAL) for durability, a hot buffer for batching writes into TBF segments, an HNSW index for semantic search, and a `HashMap`-backed intent store for point lookups. The `StorageEngine` type implements the `IntentStore` trait from `tala-core`, providing the unified storage abstraction used by the daemon and all higher-level subsystems.

All public types use interior mutability (`Mutex`, `RwLock`) so the engine is `Send + Sync` and can be shared across threads without external locking.

## Key Types

| Type | Description |
|------|-------------|
| `Wal` | Write-ahead log for crash recovery |
| `WalEntry` | A deserialized WAL entry |
| `HotBuffer` | In-memory intent accumulator with flush-to-segment |
| `QueryEngine` | HNSW-backed semantic search with cosine re-ranking |
| `StorageEngine` | Unified `IntentStore` implementation |
| `StorageMetrics` | Comprehensive per-lock and per-operation timing metrics |
| `LockStats` | Per-lock statistics: acquisition count, contention, wait/hold times |

---

## Wal

A file-backed write-ahead log. Every intent is appended to the WAL before it enters the hot buffer or HNSW index, ensuring durability across crashes. The WAL uses a simple binary format: `[4B payload_len][16B id][8B timestamp][4B embed_dim][embed_data][4B cmd_len][cmd_bytes]`.

```rust
pub struct Wal { /* private */ }

impl Wal {
    /// Create a new WAL file at the given path.
    pub fn create(path: impl AsRef<Path>) -> io::Result<Self>;

    /// Append an intent entry. Does not flush to disk automatically.
    pub fn append(
        &mut self,
        id: &[u8; 16],
        timestamp: u64,
        embedding: &[f32],
        raw_command: &str,
    ) -> io::Result<()>;

    /// Flush the internal buffer to disk.
    pub fn sync(&mut self) -> io::Result<()>;

    /// Return the number of entries written.
    pub fn entry_count(&self) -> u64;

    /// Return the WAL file path.
    pub fn path(&self) -> &Path;
}
```

### WalEntry and Replay

```rust
pub struct WalEntry {
    pub id: [u8; 16],
    pub timestamp: u64,
    pub embedding: Vec<f32>,
    pub raw_command: String,
}

/// Replay all entries from a WAL file.
pub fn replay_wal(path: impl AsRef<Path>) -> io::Result<Vec<WalEntry>>;
```

---

## HotBuffer

An in-memory accumulator that collects intents until it reaches capacity, then flushes them into a TBF segment via `tala_wire::SegmentWriter`. The flush operation is atomic: the buffer is cleared and a complete segment byte buffer is returned.

```rust
pub struct HotBuffer { /* private */ }

impl HotBuffer {
    /// Create a buffer for embeddings of `dim` dimensions with the given capacity.
    pub fn new(dim: usize, capacity: usize) -> Self;

    /// Push an intent. Returns `true` if the buffer has reached capacity
    /// and should be flushed.
    pub fn push(
        &mut self,
        id: [u8; 16],
        timestamp: u64,
        context_hash: u64,
        confidence: f32,
        status: u8,
        embedding: Vec<f32>,
        parent_indices: Vec<usize>,
    ) -> bool;

    /// Number of intents currently buffered.
    pub fn len(&self) -> usize;

    /// True if the buffer is empty.
    pub fn is_empty(&self) -> bool;

    /// Flush all buffered intents into a TBF segment byte buffer.
    /// Clears the buffer.
    pub fn flush(&mut self) -> Vec<u8>;
}
```

---

## QueryEngine

A standalone semantic search engine that pairs an HNSW index with a parallel intent metadata store. Used internally by `StorageEngine` but also available as a self-contained type.

```rust
pub struct QueryEngine { /* private */ }

impl QueryEngine {
    /// Create an engine for the given embedding dimensionality.
    /// Uses m=16, ef_construction=200 for the HNSW index.
    pub fn new(dim: usize) -> Self;

    /// Insert an intent's metadata and embedding into the search index.
    pub fn insert(
        &mut self,
        id: [u8; 16],
        timestamp: u64,
        raw_command: String,
        embedding: Vec<f32>,
    );

    /// Search for the `k` nearest intents by L2 distance.
    /// Returns (id, L2_distance) pairs.
    pub fn search(&mut self, query: &[f32], k: usize) -> Vec<([u8; 16], f32)>;

    /// Find edge candidates via HNSW approximate search + exact cosine re-rank.
    /// Returns top-k (id, cosine_similarity) pairs sorted descending.
    /// Searches 4*k candidates in HNSW, then re-ranks with exact cosine
    /// similarity for precision. Replaces O(n^2) brute-force edge formation.
    pub fn find_edge_candidates(
        &mut self,
        embedding: &[f32],
        k: usize,
    ) -> Vec<([u8; 16], f32)>;

    /// Number of intents in the index.
    pub fn len(&self) -> usize;

    /// True if the index is empty.
    pub fn is_empty(&self) -> bool;
}
```

---

## StorageEngine

The unified storage engine implementing `IntentStore`. Coordinates the WAL, hot buffer, HNSW index, and intent map. Every `insert` call follows this pipeline:

1. Append to WAL (if persistent mode)
2. Insert embedding into HNSW index
3. Push into hot buffer; if full, flush to a TBF segment file
4. Store the complete intent in the `HashMap` for point lookups

```rust
pub struct StorageEngine { /* private */ }

impl StorageEngine {
    /// Create a persistent storage engine backed by the given directory.
    /// Creates WAL and segment files in `dir`.
    pub fn open(
        dim: usize,
        dir: impl AsRef<Path>,
        hot_capacity: usize,
    ) -> Result<Self, TalaError>;

    /// Create an in-memory storage engine (no WAL, no segment persistence).
    pub fn in_memory(dim: usize, hot_capacity: usize) -> Self;

    /// Access the storage metrics.
    pub fn metrics(&self) -> &Arc<StorageMetrics>;
}
```

### IntentStore Implementation

`StorageEngine` implements all five methods of the `IntentStore` trait:

```rust
impl IntentStore for StorageEngine {
    /// Insert an intent. Enforces dimension match. Pipeline: WAL -> HNSW -> hot buffer -> store.
    fn insert(&self, intent: Intent) -> Result<IntentId, TalaError>;

    /// Retrieve an intent by ID. Returns `None` if not found.
    fn get(&self, id: IntentId) -> Result<Option<Intent>, TalaError>;

    /// Semantic search: find the `k` nearest intents by cosine similarity.
    /// Returns (IntentId, cosine_similarity) pairs.
    fn query_semantic(
        &self,
        embedding: &[f32],
        k: usize,
    ) -> Result<Vec<(IntentId, f32)>, TalaError>;

    /// Temporal query: return all intents in the time range, sorted by timestamp.
    fn query_temporal(&self, range: TimeRange) -> Result<Vec<Intent>, TalaError>;

    /// Attach an outcome to an existing intent. Returns NodeNotFound if the ID
    /// does not exist.
    fn attach_outcome(&self, id: IntentId, outcome: Outcome) -> Result<(), TalaError>;
}
```

### Example

```rust
use tala_core::{Intent, IntentId, IntentStore, Outcome, Status, TimeRange};
use tala_store::StorageEngine;

let engine = StorageEngine::in_memory(8, 1000);

let intent = Intent {
    id: IntentId::random(),
    timestamp: 42,
    raw_command: "cargo build".to_string(),
    embedding: vec![0.1; 8],
    context_hash: 7,
    parent_ids: vec![],
    outcome: None,
    confidence: 0.95,
};

let id = engine.insert(intent).unwrap();
let retrieved = engine.get(id).unwrap().unwrap();
assert_eq!(retrieved.raw_command, "cargo build");

// Attach an outcome
engine.attach_outcome(id, Outcome {
    status: Status::Success,
    latency_ns: 1000,
    exit_code: 0,
}).unwrap();

// Temporal query
let results = engine.query_temporal(TimeRange { start: 0, end: 100 }).unwrap();
assert_eq!(results.len(), 1);
```

---

## StorageMetrics

Comprehensive metrics for diagnosing storage engine performance. Contains per-lock statistics for all five internal locks and cumulative timing for each pipeline stage. All counters are `AtomicU64` with `Relaxed` ordering.

```rust
pub struct StorageMetrics {
    // Per-lock stats
    pub intents_lock: LockStats,
    pub hnsw_lock: LockStats,
    pub index_map_lock: LockStats,
    pub wal_lock: LockStats,
    pub hot_lock: LockStats,

    // Pipeline sub-operation timing (cumulative nanoseconds)
    pub wal_append_ns: AtomicU64,
    pub wal_append_count: AtomicU64,
    pub hnsw_insert_ns: AtomicU64,
    pub hnsw_insert_count: AtomicU64,
    pub hot_push_ns: AtomicU64,
    pub hot_push_count: AtomicU64,
    pub segment_flush_ns: AtomicU64,
    pub segment_flush_count: AtomicU64,
    pub edge_search_ns: AtomicU64,
    pub edge_search_count: AtomicU64,

    // HNSW internals
    pub hnsw_search_visited: AtomicU64,
    pub hnsw_search_count: AtomicU64,

    // Store state
    pub hot_buffer_len: AtomicU64,
    pub hot_buffer_capacity: AtomicU64,
    pub wal_entry_count: AtomicU64,
    pub total_bytes_flushed: AtomicU64,
    pub segments_flushed_count: AtomicU64,
}
```

---

## LockStats

Per-lock statistics tracked via atomics. Every lock acquisition and release in the storage engine records timing information here.

```rust
pub struct LockStats {
    /// Total number of lock acquisitions.
    pub acquisitions: AtomicU64,
    /// Number of acquisitions where wait exceeded 1 microsecond (contention proxy).
    pub contentions: AtomicU64,
    /// Cumulative time spent waiting to acquire the lock (nanoseconds).
    pub total_wait_ns: AtomicU64,
    /// Cumulative time spent holding the lock (nanoseconds).
    pub total_hold_ns: AtomicU64,
    /// Worst-case wait time observed (nanoseconds).
    pub max_wait_ns: AtomicU64,
    /// Worst-case hold time observed (nanoseconds).
    pub max_hold_ns: AtomicU64,
}

impl LockStats {
    pub fn new() -> Self;

    /// Record the wait time for a lock acquisition.
    /// Increments the contention counter if wait exceeds 1 microsecond.
    pub fn record_acquisition(&self, wait_ns: u64);

    /// Record the hold duration when a lock is released.
    pub fn record_release(&self, hold_ns: u64);
}
```
