# tala-daemon

The top-level orchestrator tying all TALA subsystems together. Provides the `Daemon` facade with four primary operations -- ingest, query, replay, and insights -- backed by an `IngestPipeline` that coordinates intent extraction, storage, and graph edge formation. A `DaemonBuilder` configures and constructs `Daemon` instances for both in-memory and persistent modes.

This is a library crate. The binary wrapper, Unix socket listener, and TCP server are future work.

## Key Types

| Type | Description |
|------|-------------|
| `Daemon` | Top-level facade: ingest, query, replay, insights |
| `DaemonBuilder` | Builder pattern for configuring and constructing a `Daemon` |
| `DaemonMetrics` | Per-phase timing metrics for the daemon pipeline |

---

## Daemon

The unified interface to all TALA subsystems. Internally owns an `IngestPipeline` (which in turn owns the `IntentPipeline`, `StorageEngine`, and `NarrativeGraph`) and provides four public operations.

```rust
pub struct Daemon { /* private */ }
```

### ingest

```rust
impl Daemon {
    /// Ingest a raw command string with execution context.
    ///
    /// Pipeline:
    /// 1. Extract intent via IntentPipeline (tokenize, embed, classify)
    /// 2. Insert into StorageEngine (WAL, HNSW index, hot buffer)
    /// 3. Semantic search for edge candidates (top-5 nearest neighbors)
    /// 4. Form edges in the NarrativeGraph
    ///
    /// Returns the IntentId assigned to the new intent.
    pub fn ingest(&self, raw: &str, context: &Context) -> Result<IntentId, TalaError>;
}
```

### query

```rust
impl Daemon {
    /// Semantic search: find the `k` intents most similar to `embedding`.
    ///
    /// Delegates to StorageEngine::query_semantic. Returns
    /// (IntentId, cosine_similarity) pairs sorted by similarity descending.
    pub fn query(
        &self,
        embedding: &[f32],
        k: usize,
    ) -> Result<Vec<(IntentId, f32)>, TalaError>;
}
```

### replay

```rust
impl Daemon {
    /// Build a replay plan rooted at `root`, traversing up to `depth` hops
    /// forward in the narrative graph.
    ///
    /// Steps:
    /// 1. BFS forward from root to discover the reachable subgraph
    /// 2. Fetch commands from the store for each reachable node
    /// 3. Topologically sort via tala_weave::build_plan
    ///
    /// Returns TalaError::NodeNotFound if root is not in the graph.
    pub fn replay(
        &self,
        root: IntentId,
        depth: usize,
    ) -> Result<Vec<ReplayStep>, TalaError>;
}
```

### insights

```rust
impl Daemon {
    /// Generate insights from the current intent corpus.
    ///
    /// Analysis:
    /// 1. Collect all intents from the store, sorted by timestamp
    /// 2. Run pattern detection (n-gram frequency via InsightEngine)
    /// 3. Produce a narrative summary
    /// 4. Run k-means clustering with `k_clusters` clusters
    ///
    /// Returns a mix of RecurringPattern, Summary, and FailureCluster insights.
    pub fn insights(
        &self,
        k_clusters: usize,
    ) -> Result<Vec<Insight>, TalaError>;
}
```

### Accessors

```rust
impl Daemon {
    /// Access the underlying storage engine for direct queries.
    pub fn store(&self) -> &StorageEngine;

    /// Access the daemon-level metrics.
    pub fn daemon_metrics(&self) -> &Arc<DaemonMetrics>;
}
```

---

## DaemonBuilder

Builder for configuring and constructing a `Daemon`. Supports both in-memory and persistent modes.

```rust
pub struct DaemonBuilder { /* private */ }

impl DaemonBuilder {
    /// Create a new builder with default settings:
    /// - dim: 384
    /// - hot_capacity: 10,000
    pub fn new() -> Self;

    /// Set the embedding dimension.
    pub fn dim(self, dim: usize) -> Self;

    /// Set the hot buffer capacity (number of intents before segment flush).
    pub fn hot_capacity(self, hot_capacity: usize) -> Self;

    /// Build an in-memory daemon (no WAL, no persistence).
    pub fn build_in_memory(self) -> Daemon;

    /// Build a persistent daemon backed by the given directory.
    /// Creates WAL and segment files in `dir`.
    pub fn build(self, dir: impl AsRef<Path>) -> Result<Daemon, TalaError>;
}

impl Default for DaemonBuilder {
    fn default() -> Self { Self::new() }
}
```

### Example: In-Memory Daemon

```rust
use tala_core::Context;
use tala_daemon::DaemonBuilder;

let daemon = DaemonBuilder::new()
    .dim(384)
    .hot_capacity(1000)
    .build_in_memory();

let ctx = Context {
    cwd: "/home/user/project".to_string(),
    env_hash: 42,
    session_id: 1,
    shell: "zsh".to_string(),
    user: "ops".to_string(),
};

// Ingest
let id = daemon.ingest("cargo build --release", &ctx).unwrap();
assert!(daemon.store().get(id).unwrap().is_some());

// Query
let pipeline = tala_intent::IntentPipeline::new();
let query_emb = pipeline.embed("cargo build");
let results = daemon.query(&query_emb, 5).unwrap();
assert!(!results.is_empty());

// Replay
let plan = daemon.replay(id, 3).unwrap();
assert_eq!(plan[0].intent_id, id);

// Insights
let insights = daemon.insights(2).unwrap();
assert!(!insights.is_empty());
```

### Example: Persistent Daemon

```rust
use tala_daemon::DaemonBuilder;

let daemon = DaemonBuilder::new()
    .dim(384)
    .hot_capacity(100)
    .build("/tmp/tala-data")
    .unwrap();
```

---

## DaemonMetrics

Per-phase timing metrics for the daemon pipeline. All fields are `AtomicU64` with cumulative nanosecond timing and per-operation counts.

```rust
pub struct DaemonMetrics {
    /// Intent extraction time (cumulative nanoseconds).
    pub extract_ns: AtomicU64,
    pub extract_count: AtomicU64,

    /// Storage engine insert time (cumulative nanoseconds).
    pub store_insert_ns: AtomicU64,
    pub store_insert_count: AtomicU64,

    /// Edge formation time: semantic search + graph mutation (cumulative nanoseconds).
    pub edge_formation_ns: AtomicU64,
    pub edge_formation_count: AtomicU64,

    /// Semantic query time (cumulative nanoseconds).
    pub query_ns: AtomicU64,
    pub query_count: AtomicU64,

    /// Replay plan building time (cumulative nanoseconds).
    pub replay_ns: AtomicU64,
    pub replay_count: AtomicU64,

    /// Insight generation time (cumulative nanoseconds).
    pub insight_ns: AtomicU64,
    pub insight_count: AtomicU64,
}

impl DaemonMetrics {
    pub fn new() -> Self;
}

impl Default for DaemonMetrics {
    fn default() -> Self { Self::new() }
}
```

The daemon records timing for each phase of every operation. Access metrics via `daemon.daemon_metrics()`. The underlying storage engine also provides its own detailed metrics via `daemon.store().metrics()`.
