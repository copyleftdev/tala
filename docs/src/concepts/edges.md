# Edge Relations and Causality

Edges in the narrative graph are typed and weighted. Each edge represents a specific kind of relationship between two intents.

## Relation Types

```rust
#[repr(u8)]
enum RelationType {
    Causal     = 0,  // A caused B
    Temporal   = 1,  // A happened before B in the same session
    Dependency = 2,  // B depends on A's output
    Retry      = 3,  // B is a retry of A after failure
    Branch     = 4,  // B is an alternative approach to the same goal as A
}
```

### Causal

The strongest relation. Intent A directly caused intent B. Example: an alert fires (A), triggering an SSH session to diagnose the issue (B).

### Temporal

Intent A preceded intent B within the same session. Weaker than causal — it captures sequence without asserting direct causation.

### Dependency

Intent B depends on the output or side effects of intent A. Example: a build step (A) must complete before a deploy step (B) can proceed.

### Retry

Intent B is a retry of intent A after A failed. TALA detects retries by identifying semantically similar commands with a preceding failure outcome.

### Branch

Intent B represents an alternative approach to the same goal as intent A. Detected when two semantically similar intents follow the same predecessor but take different paths.

## Edge Weights

Every edge carries a weight (`f32`, 0.0–1.0) representing the strength of the relationship. Weight is derived from:

- **Cosine similarity** between the embeddings of the connected intents
- **Temporal proximity** — closer in time means stronger weight
- **Confidence** of both nodes

High-weight edges indicate strong, confident causal relationships. Low-weight edges indicate weaker associations — possibly temporal coincidence rather than true causation.

## How Edges Form

Edge formation happens automatically during ingest. When a new intent is created:

1. The HNSW index finds the top-K semantically similar existing intents
2. Each candidate is evaluated for relation type based on temporal order, outcome status, and embedding distance
3. Edges are added with appropriate types and weights
4. The graph remains a DAG — cycle detection prevents invalid edges

This means the graph grows organically. You don't declare relationships. TALA infers them from the structure of your work.
