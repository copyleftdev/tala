# tala-kai

The insight engine. Provides analysis capabilities over intent histories: k-means clustering of intent embeddings, n-gram pattern detection in command sequences, frequency-based next-intent prediction, and narrative summarization. All analysis functions are stateless and operate on slices of data, making them safe for concurrent use. The `InsightEngine` orchestrator wraps these primitives and returns results as `Insight` values from `tala-core`.

## Key Types

| Type | Description |
|------|-------------|
| `ClusterResult` | Result of k-means clustering: assignments, centroids, convergence info |
| `Pattern` | A recurring command sequence with its frequency count |
| `NarrativeSummary` | Summary statistics for a set of intents |
| `InsightEngine` | Orchestrator that combines all analysis capabilities |

## Key Functions

| Function | Description |
|----------|-------------|
| `kmeans(embeddings, dim, k, max_iter, seed)` | Lloyd's k-means clustering |
| `detect_patterns(intents, min_count)` | N-gram frequency analysis (bigrams + trigrams) |
| `predict_next(history, corpus)` | Frequency-based next-command prediction |
| `summarize(intents)` | Narrative summary with statistics and top commands |

---

## kmeans

Runs Lloyd's k-means algorithm on a flat buffer of embedding vectors. Initializes centroids by sampling `k` random data points (using the provided seed for determinism), then iterates assignment and update steps until convergence or `max_iter` iterations.

```rust
pub fn kmeans(
    embeddings: &[f32],
    dim: usize,
    k: usize,
    max_iter: usize,
    seed: u64,
) -> Result<ClusterResult, TalaError>;
```

Returns `TalaError::DimensionMismatch` if:
- `dim` is zero
- `embeddings.len()` is not evenly divisible by `dim`
- `k` is zero or exceeds the number of data points

### ClusterResult

```rust
#[derive(Clone, Debug)]
pub struct ClusterResult {
    /// Cluster assignment for each input point (index into `centroids`).
    pub assignments: Vec<usize>,
    /// Final centroid vectors, shape: k x dim.
    pub centroids: Vec<Vec<f32>>,
    /// Number of iterations until convergence.
    pub iterations: usize,
    /// Whether the algorithm converged before hitting max_iter.
    pub converged: bool,
}
```

### Example

```rust
use tala_kai::kmeans;

// Two well-separated clusters in 2D
let embeddings = vec![
    0.0, 0.0,   // cluster A
    0.1, 0.1,   // cluster A
    10.0, 10.0, // cluster B
    9.9, 9.9,   // cluster B
];

let result = kmeans(&embeddings, 2, 2, 50, 42).unwrap();
assert_eq!(result.assignments.len(), 4);
assert_eq!(result.assignments[0], result.assignments[1]); // same cluster
assert_eq!(result.assignments[2], result.assignments[3]); // same cluster
assert_ne!(result.assignments[0], result.assignments[2]); // different clusters
assert!(result.converged);
```

---

## detect_patterns

Detects recurring command sequences (bigrams and trigrams) from a time-sorted list of intents. Returns patterns whose frequency meets or exceeds `min_count`, sorted by count descending, then by sequence length descending.

```rust
pub fn detect_patterns(intents: &[Intent], min_count: usize) -> Vec<Pattern>;
```

Intents MUST be pre-sorted by timestamp for meaningful n-gram extraction.

### Pattern

```rust
#[derive(Clone, Debug)]
pub struct Pattern {
    /// The command sequence (bigram or trigram).
    pub commands: Vec<String>,
    /// Number of occurrences in the corpus.
    pub count: usize,
}
```

### Example

```rust
use tala_kai::detect_patterns;

// Given intents with commands: [git status, git add, git commit] x 3
// detect_patterns finds the trigram with count=3 and bigrams with count>=3
let patterns = detect_patterns(&intents, 2);
assert!(!patterns.is_empty());

// Trigram "git status -> git add -> git commit" should have count 3
let trigram = patterns.iter()
    .find(|p| p.commands.len() == 3 && p.commands[0] == "git status")
    .unwrap();
assert_eq!(trigram.count, 3);
```

---

## predict_next

Predicts the most likely next command given recent history and a corpus of past intents. Uses a frequency model that checks trigram, bigram, and unigram contexts in order of preference. Longer context matches are preferred because they carry more signal.

```rust
/// Returns `None` if no match is found, history is empty,
/// or the corpus has fewer than 2 intents.
pub fn predict_next(history: &[String], corpus: &[Intent]) -> Option<String>;
```

### Example

```rust
use tala_kai::predict_next;

// Corpus: [cd src, cargo build, cargo test] repeated
let history = vec!["cd src".to_string(), "cargo build".to_string()];
let prediction = predict_next(&history, &corpus);
assert_eq!(prediction, Some("cargo test".to_string()));
```

---

## summarize

Produces summary statistics for a set of intents: counts of successes, failures, and pending outcomes, time span, most frequent commands, and a human-readable text summary.

```rust
pub fn summarize(intents: &[Intent]) -> NarrativeSummary;
```

### NarrativeSummary

```rust
#[derive(Clone, Debug)]
pub struct NarrativeSummary {
    /// Total number of intents.
    pub total: usize,
    /// Number with Success outcome.
    pub successes: usize,
    /// Number with Failure outcome.
    pub failures: usize,
    /// Number with no outcome attached (or Pending status).
    pub pending: usize,
    /// Earliest timestamp (nanosecond epoch).
    pub time_start: u64,
    /// Latest timestamp (nanosecond epoch).
    pub time_end: u64,
    /// Most common commands, sorted by frequency descending (up to 10).
    pub top_commands: Vec<(String, usize)>,
    /// Human-readable summary text.
    pub text: String,
}
```

### Example

```rust
use tala_kai::summarize;

let summary = summarize(&intents);
assert_eq!(summary.total, intents.len());
assert!(summary.text.contains("intents"));
assert!(summary.text.contains("Success rate"));
```

---

## InsightEngine

The orchestrator that wraps all analysis capabilities and returns results as `Insight` values from `tala-core`. Configurable thresholds allow tuning pattern detection and clustering behavior.

```rust
pub struct InsightEngine {
    /// Minimum n-gram count threshold for pattern detection.
    pub pattern_threshold: usize,
    /// K-means seed for deterministic clustering.
    pub seed: u64,
    /// Maximum k-means iterations.
    pub max_kmeans_iter: usize,
}

impl Default for InsightEngine {
    fn default() -> Self {
        Self {
            pattern_threshold: 2,
            seed: 42,
            max_kmeans_iter: 100,
        }
    }
}
```

### Methods

```rust
impl InsightEngine {
    /// Create a new engine with default settings.
    pub fn new() -> Self;

    /// Cluster intent embeddings into `k` groups.
    /// Delegates to `kmeans()` with the engine's seed and max iterations.
    pub fn analyze_clusters(
        &self,
        embeddings: &[f32],
        dim: usize,
        k: usize,
    ) -> Result<ClusterResult, TalaError>;

    /// Detect recurring patterns in a sorted list of intents.
    /// Returns each pattern as an `Insight` with kind `RecurringPattern`.
    /// Confidence is the pattern frequency divided by corpus length.
    pub fn detect_patterns(&self, intents: &[Intent]) -> Vec<Insight>;

    /// Predict the next command given recent history and a corpus.
    /// Returns an `Insight` with kind `Prediction` if a match is found.
    pub fn predict_next(
        &self,
        history: &[String],
        corpus: &[Intent],
    ) -> Option<Insight>;

    /// Summarize a set of intents into an `Insight` with kind `Summary`.
    /// Always returns an insight with confidence 1.0.
    pub fn summarize(&self, intents: &[Intent]) -> Insight;
}
```

### Example

```rust
use tala_core::InsightKind;
use tala_kai::InsightEngine;

let engine = InsightEngine::new();

// Pattern detection
let insights = engine.detect_patterns(&intents);
assert!(insights.iter().all(|i| i.kind == InsightKind::RecurringPattern));

// Prediction
if let Some(prediction) = engine.predict_next(&history, &corpus) {
    assert_eq!(prediction.kind, InsightKind::Prediction);
}

// Summary
let summary = engine.summarize(&intents);
assert_eq!(summary.kind, InsightKind::Summary);
assert_eq!(summary.confidence, 1.0);
```
