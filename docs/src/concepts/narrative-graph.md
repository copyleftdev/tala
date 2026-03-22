# The Narrative Graph

The narrative graph is TALA's central data structure. It's a directed acyclic probabilistic graph where nodes are intents and edges are causal relationships between them.

## Definition

```
G = (V, E, W)
```

| Symbol | Meaning |
|--------|---------|
| **V** | Intent nodes |
| **E** | Directed causal edges |
| **W** | Edge weights (probability / confidence) |

## Why a Graph

A flat list of commands loses structure. A tree is too rigid — real workflows branch, merge, and fork. A DAG captures the actual causal topology of operational work:

- An incident investigation spawns multiple diagnostic branches
- A deployment pipeline has linear stages but also parallel verification steps
- A provisioning sequence has dependencies that form a partial order, not a total order

The narrative graph preserves all of this.

## Narratives

A **narrative** is a coherent subgraph — a connected sequence of intents that tells a story. Given any intent node, TALA can extract its narrative by traversing the graph forward (consequences) and backward (causes).

```
N ⊆ G
```

For example, starting from a `systemctl restart nginx` node:

- **Backward traversal** reveals the chain that led to the restart: an alert fired, an engineer SSH'd in, checked logs, found an OOM event, resized memory limits
- **Forward traversal** reveals the consequences: a health check passed, the alert resolved, a post-incident summary was generated

This entire chain is the narrative. It's queryable, replayable, and durable.

## Graph Operations

| Operation | Method | Description |
|-----------|--------|-------------|
| Insert node | `insert_node(id, timestamp, confidence)` | Add an intent to the graph |
| Form edges | `form_edges(node, similarities, k)` | Connect to the top-K semantically similar nodes |
| Forward BFS | `bfs_forward(start, depth)` | Find all consequences of an intent |
| Backward BFS | `bfs_backward(start, depth)` | Find all causes of an intent (root-cause analysis) |
| Extract narrative | `extract_narrative(root, depth)` | Pull the connected subgraph as (nodes, edges) |

## Edge Formation

When a new intent node is created, TALA uses the HNSW semantic index to find the most similar existing intents. The top-K candidates become edge targets, weighted by cosine similarity. This means edges form based on *meaning*, not just temporal proximity.

The edge formation algorithm:

```
for each new node v:
    candidates = HNSW.search(v.embedding, k=10, ef=50)
    for (candidate, similarity) in candidates:
        if similarity > threshold:
            graph.add_edge(v, candidate, relation_type, weight=similarity)
```

Relation types are inferred from the semantic and temporal relationship between nodes. See [Edge Relations and Causality](./edges.md).
