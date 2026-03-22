# tala-core

Foundation types and traits for the intent-native narrative execution layer. Every crate in the TALA workspace depends on `tala-core`; it defines the vocabulary that all subsystems share. The crate carries no dependencies beyond `uuid` and `thiserror`, making it safe to link from any context -- SIMD kernels, storage engines, network codecs, and CLI tools alike.

## Key Types

| Type | Description |
|------|-------------|
| `IntentId` | UUID wrapper identifying a single intent node |
| `Intent` | Full intent node: identity, timestamp, embedding, command, outcome |
| `Edge` | Directed edge between two intent nodes |
| `RelationType` | Edge classification enum (Causal, Temporal, Dependency, Retry, Branch) |
| `Outcome` | Execution result: status, latency, exit code |
| `Status` | Outcome status enum (Pending, Success, Failure, Partial) |
| `Context` | Execution environment captured at intent time |
| `TimeRange` | Inclusive-start, exclusive-end nanosecond time window |
| `ReplayStep` | A single step in a replay plan |
| `ReplayResult` | Result of executing a replay step |
| `IntentCategory` | Classification of an intent's purpose |
| `Insight` | An observation produced by the analysis engine |
| `InsightKind` | Classification of insight types |
| `TalaError` | Unified error taxonomy for the workspace |
| `IntentStore` | Trait: storage abstraction for intents |
| `IntentExtractor` | Trait: raw command to structured intent conversion |

---

## IntentId

A newtype over `uuid::Uuid` providing a unique, opaque handle for every intent node in the system.

```rust
#[derive(Clone, Copy, PartialEq, Eq, Hash, Debug)]
pub struct IntentId(pub Uuid);
```

### Methods

```rust
impl IntentId {
    /// Generate a random v4 UUID.
    pub fn random() -> Self;

    /// Access the raw 16-byte representation.
    pub fn as_bytes(&self) -> &[u8; 16];
}
```

`IntentId` implements `Default` by generating a random identifier. Two separately constructed `IntentId` values are virtually guaranteed to be distinct.

```rust
let id = IntentId::random();
assert_eq!(id.as_bytes().len(), 16);
```

---

## Intent

The central data structure of the entire system. An `Intent` captures everything known about a single user action: its identity, when it occurred, the raw command text, a dense embedding vector for semantic search, and the optional execution outcome.

```rust
#[derive(Clone, Debug)]
pub struct Intent {
    pub id: IntentId,
    pub timestamp: u64,
    pub raw_command: String,
    pub embedding: Vec<f32>,
    pub context_hash: u64,
    pub parent_ids: Vec<IntentId>,
    pub outcome: Option<Outcome>,
    pub confidence: f32,
}
```

| Field | Description |
|-------|-------------|
| `id` | Unique identifier assigned at creation |
| `timestamp` | Nanosecond epoch when the intent was captured |
| `raw_command` | The original shell command string |
| `embedding` | Dense f32 vector (typically dim=384) for semantic similarity |
| `context_hash` | FNV-1a hash of the execution context |
| `parent_ids` | Predecessor intent IDs forming causal chains |
| `outcome` | Execution result, attached asynchronously |
| `confidence` | Extraction confidence score in `[0.0, 1.0]` |

---

## Edge

A directed, weighted edge connecting two intent nodes. Edges carry a `RelationType` that classifies the nature of the relationship and a floating-point weight representing connection strength.

```rust
#[derive(Clone, Debug)]
pub struct Edge {
    pub from: IntentId,
    pub to: IntentId,
    pub relation: RelationType,
    pub weight: f32,
}
```

---

## RelationType

Classifies how two intent nodes relate. Stored as a `#[repr(u8)]` enum so it can be serialized into a single byte in the TBF binary format and CSR edge entries.

```rust
#[repr(u8)]
#[derive(Clone, Copy, Debug, PartialEq, Eq)]
pub enum RelationType {
    Causal     = 0,
    Temporal   = 1,
    Dependency = 2,
    Retry      = 3,
    Branch     = 4,
}
```

| Variant | Meaning |
|---------|---------|
| `Causal` | The source intent directly caused the target |
| `Temporal` | The two intents are temporally adjacent |
| `Dependency` | The target depends on the source's output |
| `Retry` | The target is a retry of the source |
| `Branch` | The target is an alternative path from the source |

---

## Outcome and Status

An `Outcome` records the result of executing an intent. It is attached to an `Intent` asynchronously after execution completes.

```rust
#[derive(Clone, Debug)]
pub struct Outcome {
    pub status: Status,
    pub latency_ns: u64,
    pub exit_code: i32,
}
```

```rust
#[repr(u8)]
#[derive(Clone, Copy, Debug, PartialEq, Eq)]
pub enum Status {
    Pending = 0,
    Success = 1,
    Failure = 2,
    Partial = 3,
}
```

---

## Context

Captures the execution environment at intent time. Used to compute the `context_hash` field on `Intent`, enabling queries that group intents by environment.

```rust
#[derive(Clone, Debug, Default)]
pub struct Context {
    pub cwd: String,
    pub env_hash: u64,
    pub session_id: u64,
    pub shell: String,
    pub user: String,
}
```

---

## TimeRange

A half-open time interval `[start, end)` expressed in nanosecond epoch timestamps. Used by `IntentStore::query_temporal` to retrieve intents within a window.

```rust
#[derive(Clone, Copy, Debug)]
pub struct TimeRange {
    pub start: u64,
    pub end: u64,
}
```

---

## ReplayStep and ReplayResult

Types used by the replay engine (`tala-weave`) to represent plan steps and their outcomes.

```rust
#[derive(Clone, Debug)]
pub struct ReplayStep {
    pub intent_id: IntentId,
    pub command: String,
    pub deps: Vec<IntentId>,
}

#[derive(Clone, Debug)]
pub struct ReplayResult {
    pub step: ReplayStep,
    pub outcome: Outcome,
    pub skipped: bool,
}
```

`ReplayStep` is a single entry in a topologically sorted replay plan. `deps` lists the IDs of steps that must complete before this step may execute. `ReplayResult` wraps a step with its execution outcome and a flag indicating whether the step was skipped due to idempotency.

---

## IntentCategory and Insight Types

Classification types for the intent pipeline and insight engine.

```rust
#[derive(Clone, Debug, PartialEq, Eq, Hash)]
pub enum IntentCategory {
    Build,
    Deploy,
    Debug,
    Configure,
    Query,
    Navigate,
    Other(String),
}
```

```rust
#[derive(Clone, Debug)]
pub struct Insight {
    pub kind: InsightKind,
    pub description: String,
    pub intent_ids: Vec<IntentId>,
    pub confidence: f32,
}

#[derive(Clone, Debug, PartialEq, Eq)]
pub enum InsightKind {
    RecurringPattern,
    FailureCluster,
    Prediction,
    Summary,
}
```

---

## TalaError

A unified error taxonomy for the entire workspace. Crate-specific errors in downstream crates derive `From` conversions into `TalaError` via `thiserror`.

```rust
#[derive(Debug, thiserror::Error)]
pub enum TalaError {
    #[error("segment not found: {0:?}")]
    SegmentNotFound(SegmentId),

    #[error("segment corrupted: {0}")]
    SegmentCorrupted(String),

    #[error("node not found: {0:?}")]
    NodeNotFound(IntentId),

    #[error("dimension mismatch: expected {expected}, got {got}")]
    DimensionMismatch { expected: usize, got: usize },

    #[error("extraction failed: {0}")]
    ExtractionFailed(String),

    #[error("cycle detected in narrative graph")]
    CycleDetected,

    #[error(transparent)]
    Io(#[from] std::io::Error),
}
```

The `SegmentId` type referenced by `SegmentNotFound` is a monotonic segment identifier:

```rust
#[derive(Clone, Copy, Debug, PartialEq, Eq, Hash)]
pub struct SegmentId(pub u64);
```

---

## IntentStore Trait

The core storage abstraction. Implemented by `tala_store::StorageEngine`. All methods accept `&self` and use interior mutability (locks) so the store can be shared across threads.

```rust
pub trait IntentStore: Send + Sync {
    /// Insert a fully formed intent. Returns its ID.
    fn insert(&self, intent: Intent) -> Result<IntentId, TalaError>;

    /// Retrieve an intent by ID. Returns `None` if not found.
    fn get(&self, id: IntentId) -> Result<Option<Intent>, TalaError>;

    /// Semantic search: find the `k` nearest intents by embedding similarity.
    /// Returns (IntentId, cosine_similarity) pairs.
    fn query_semantic(
        &self,
        embedding: &[f32],
        k: usize,
    ) -> Result<Vec<(IntentId, f32)>, TalaError>;

    /// Temporal query: return all intents within a time range, sorted by timestamp.
    fn query_temporal(&self, range: TimeRange) -> Result<Vec<Intent>, TalaError>;

    /// Attach an outcome to an existing intent.
    fn attach_outcome(&self, id: IntentId, outcome: Outcome) -> Result<(), TalaError>;
}
```

---

## IntentExtractor Trait

Converts a raw command string and execution context into a structured `Intent`. Implemented by `tala_intent::IntentPipeline`.

```rust
pub trait IntentExtractor: Send + Sync {
    fn extract(&self, raw: &str, context: &Context) -> Result<Intent, TalaError>;
}
```
