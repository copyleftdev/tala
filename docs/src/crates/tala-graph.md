# tala-graph

In-memory narrative graph engine backed by dual adjacency lists. The graph stores intent nodes with their metadata and directed, typed, weighted edges. It supports forward and backward BFS traversal, automatic edge formation from similarity scores, and narrative extraction for replay planning.

The graph is a directed graph (not necessarily acyclic at this layer; cycle detection is enforced by the replay planner in `tala-weave`). Each edge carries a `RelationType` from `tala-core` and a floating-point weight.

## Key Types

| Type | Description |
|------|-------------|
| `NarrativeGraph` | In-memory graph with dual adjacency lists, BFS/DFS, and edge formation |

---

## NarrativeGraph

The core graph type. Internally maintains two `HashMap`-backed adjacency lists (forward and backward) and a node metadata map. All operations are O(1) amortized for node/edge insertion and adjacency lookup.

```rust
pub struct NarrativeGraph { /* private */ }
```

The internal representation stores:
- `forward: HashMap<IntentId, Vec<(IntentId, RelationType, f32)>>` -- outgoing edges
- `backward: HashMap<IntentId, Vec<(IntentId, RelationType, f32)>>` -- incoming edges
- `nodes: HashMap<IntentId, NodeData>` -- per-node metadata (timestamp, confidence)

### Construction

```rust
impl NarrativeGraph {
    /// Create an empty graph.
    pub fn new() -> Self;
}

impl Default for NarrativeGraph {
    fn default() -> Self { Self::new() }
}
```

### Node Operations

```rust
impl NarrativeGraph {
    /// Insert a node with its timestamp and confidence score.
    /// Initializes empty adjacency entries in both forward and backward maps.
    pub fn insert_node(&mut self, id: IntentId, timestamp: u64, confidence: f32);

    /// Return the total number of nodes in the graph.
    pub fn node_count(&self) -> usize;

    /// Return true if the graph contains a node with the given ID.
    pub fn contains_node(&self, id: IntentId) -> bool;

    /// Return all node IDs in the graph. Order is not guaranteed.
    pub fn node_ids(&self) -> Vec<IntentId>;
}
```

### Edge Operations

```rust
impl NarrativeGraph {
    /// Add a directed edge from `from` to `to` with the given relation type and weight.
    /// Updates both forward and backward adjacency lists.
    pub fn add_edge(
        &mut self,
        from: IntentId,
        to: IntentId,
        relation: RelationType,
        weight: f32,
    );

    /// Return the total number of edges (sum of all forward adjacency list lengths).
    pub fn edge_count(&self) -> usize;
}
```

### Edge Formation

```rust
impl NarrativeGraph {
    /// Automatically form edges between `new_node` and existing nodes based
    /// on similarity scores.
    ///
    /// Sorts `similarities` by score descending, then connects the top `k`
    /// existing nodes to `new_node` with `RelationType::Causal` edges.
    /// The similarity score is used as the edge weight.
    pub fn form_edges(
        &mut self,
        new_node: IntentId,
        similarities: &mut [(IntentId, f32)],
        k: usize,
    );
}
```

### Traversal

```rust
impl NarrativeGraph {
    /// BFS forward from `start`, visiting up to `max_depth` hops along
    /// outgoing edges. Returns all visited node IDs in BFS order.
    pub fn bfs_forward(&self, start: IntentId, max_depth: usize) -> Vec<IntentId>;

    /// BFS backward from `start`, following incoming edges.
    /// Useful for root-cause analysis.
    pub fn bfs_backward(&self, start: IntentId, max_depth: usize) -> Vec<IntentId>;

    /// Extract a narrative subgraph: bidirectional BFS from `root` up to
    /// `max_depth` hops in both directions.
    /// Returns (visited_nodes, edges) where edges are (from, to) pairs.
    pub fn extract_narrative(
        &self,
        root: IntentId,
        max_depth: usize,
    ) -> (Vec<IntentId>, Vec<(IntentId, IntentId)>);
}
```

### Adjacency Access

```rust
impl NarrativeGraph {
    /// Return the forward neighbors (successors) of a node.
    /// Each entry is (neighbor_id, relation_type, weight).
    /// Returns an empty slice if the node has no outgoing edges or does not exist.
    pub fn successors(&self, id: IntentId) -> &[(IntentId, RelationType, f32)];

    /// Return the backward neighbors (predecessors) of a node.
    /// Each entry is (neighbor_id, relation_type, weight).
    /// Returns an empty slice if the node has no incoming edges or does not exist.
    pub fn predecessors(&self, id: IntentId) -> &[(IntentId, RelationType, f32)];
}
```

### Example

```rust
use tala_core::{IntentId, RelationType};
use tala_graph::NarrativeGraph;

let mut graph = NarrativeGraph::new();

let a = IntentId::random();
let b = IntentId::random();
let c = IntentId::random();

graph.insert_node(a, 1000, 0.95);
graph.insert_node(b, 2000, 0.90);
graph.insert_node(c, 3000, 0.85);

graph.add_edge(a, b, RelationType::Causal, 0.9);
graph.add_edge(b, c, RelationType::Temporal, 0.7);

assert_eq!(graph.node_count(), 3);
assert_eq!(graph.edge_count(), 2);

// Forward traversal from a: reaches a, b, c
let forward = graph.bfs_forward(a, 10);
assert_eq!(forward.len(), 3);

// Backward traversal from c: reaches c, b, a
let backward = graph.bfs_backward(c, 10);
assert_eq!(backward.len(), 3);

// Successors of a: [(b, Causal, 0.9)]
let succs = graph.successors(a);
assert_eq!(succs.len(), 1);
assert_eq!(succs[0].0, b);

// Predecessors of c: [(b, Temporal, 0.7)]
let preds = graph.predecessors(c);
assert_eq!(preds.len(), 1);
assert_eq!(preds[0].0, b);
```

### Edge Formation Example

```rust
use tala_core::IntentId;
use tala_graph::NarrativeGraph;

let mut graph = NarrativeGraph::new();

// Existing nodes
let n1 = IntentId::random();
let n2 = IntentId::random();
let n3 = IntentId::random();
graph.insert_node(n1, 100, 1.0);
graph.insert_node(n2, 200, 1.0);
graph.insert_node(n3, 300, 1.0);

// New node arrives with similarity scores against existing nodes
let new = IntentId::random();
graph.insert_node(new, 400, 1.0);

let mut similarities = vec![
    (n1, 0.3),
    (n2, 0.9),
    (n3, 0.7),
];

// Connect to top-2 by similarity
graph.form_edges(new, &mut similarities, 2);

// n2 and n3 should now have edges to `new`
assert_eq!(graph.edge_count(), 2);
assert_eq!(graph.predecessors(new).len(), 2);
```

### Narrative Extraction Example

```rust
use tala_core::{IntentId, RelationType};
use tala_graph::NarrativeGraph;

let mut graph = NarrativeGraph::new();
let a = IntentId::random();
let b = IntentId::random();
let c = IntentId::random();

graph.insert_node(a, 1, 1.0);
graph.insert_node(b, 2, 1.0);
graph.insert_node(c, 3, 1.0);
graph.add_edge(a, b, RelationType::Causal, 1.0);
graph.add_edge(b, c, RelationType::Causal, 1.0);

// Extract narrative from b: discovers a (backward) and c (forward)
let (nodes, edges) = graph.extract_narrative(b, 5);
assert_eq!(nodes.len(), 3);
assert_eq!(edges.len(), 2);
```
