# Data Flow: Ingest to Narrative

This chapter walks through the three primary data paths in TALA: the ingest pipeline that converts raw commands into connected intent nodes, the query path that retrieves them, and the replay path that re-executes them.

## Ingest Pipeline

A raw command enters the system and emerges as a node in the narrative graph with causal edges to related intents. The pipeline has six stages.

```
Raw Command
    │
    ▼
┌──────────────────┐
│ IntentPipeline    │
│   .extract()      │  tala-intent
│                    │
│  tokenize          │
│  classify          │
│  embed             │
│  enrich            │
└────────┬───────────┘
         │
         ▼
    Intent { id, timestamp, embedding, raw_command, context_hash, ... }
         │
         ▼
┌──────────────────┐
│ StorageEngine     │
│   .insert()       │  tala-store
│                    │
│  ┌─ WAL.append()  │─────────▶ disk (durability)
│  │                 │
│  ├─ HNSW.insert() │─────────▶ in-memory ANN index
│  │                 │
│  └─ HotBuffer     │
│      .push()       │─────────▶ in-memory accumulator
│                    │
│  [if buffer full]  │
│  └─ flush()        │─────────▶ TBF segment on disk
└────────┬───────────┘
         │
         ▼
┌──────────────────┐
│ Edge Formation    │  tala-store + tala-graph
│                    │
│  HNSW.search()     │  find k*4 approximate neighbors
│      │             │
│      ▼             │
│  cosine re-rank    │  exact similarity on candidates
│      │             │
│      ▼             │
│  NarrativeGraph    │
│   .form_edges()    │  top-k become weighted causal edges
└────────────────────┘
```

### Stage 1: Intent Extraction

`IntentPipeline.extract()` receives the raw command string and the execution context (working directory, environment hash, session ID, shell, user). It performs four operations:

1. **Tokenization.** The command is split into structural tokens: binary name, flags, arguments, pipes, redirections.
2. **Classification.** A classifier assigns the intent to a category: Build, Deploy, Debug, Configure, Query, Navigate, or Other.
3. **Embedding.** The command text is passed to an embedding model (ONNX Runtime for local inference, or a remote API) to produce a dense `f32` vector of dimension 384. This vector encodes the semantic meaning of the command.
4. **Enrichment.** Metadata is attached: resource references (file paths, URLs, container names), estimated complexity, and a context hash (xxHash3 of the environment state).

The output is a fully populated `Intent` struct.

### Stage 2: WAL Append

`StorageEngine.insert()` begins by writing the intent to the write-ahead log. The WAL entry format is:

```
[4B payload_len][16B id][8B timestamp][4B embed_len][embed bytes][4B cmd_len][cmd bytes]
```

The WAL is `fsync`'d after each append (batched in practice). This guarantees that the intent survives a crash even if subsequent steps fail.

### Stage 3: HNSW Insert

The intent's embedding vector is inserted into the in-memory HNSW index. This makes it immediately discoverable by semantic search. The HNSW index maintains a multi-layer navigable small-world graph with parameters `M=16` (edges per node per layer) and `ef_construction=200` (search width during insertion).

An `index_map` (a `Vec<IntentId>` indexed by HNSW position) maintains the mapping between HNSW node indices and `IntentId` values.

### Stage 4: Hot Buffer Push

The intent's columnar fields (id, timestamp, context hash, confidence, status, embedding) are pushed into the `HotBuffer`. This is an in-memory accumulator that collects intents until it reaches its configured capacity.

### Stage 5: Segment Flush

When the hot buffer reaches capacity (default: 64K nodes or 256MB), it flushes to a TBF segment:

1. The `SegmentWriter` receives all buffered intents.
2. Node fields are serialized as columnar arrays, each 64-byte aligned.
3. Embeddings are written to the embedding region with 64-byte stride alignment.
4. Edges are encoded as a CSR index (row pointer array + edge entry array).
5. A bloom filter over node UUIDs is constructed (1% false positive rate).
6. The 128-byte TBF header is written with region offsets, counts, and flags.
7. The segment file is written to disk and `fsync`'d.
8. The hot buffer is cleared.

The segment is now immutable. It will never be modified in place.

### Stage 6: Edge Formation

After the intent is stored, the system identifies causal connections to existing intents. This is where the flat log becomes a graph.

1. **Approximate search.** `QueryEngine.find_edge_candidates()` queries the HNSW index with the new intent's embedding, requesting `k * 4` approximate nearest neighbors. HNSW returns candidates in `O(log n)` time, replacing what would otherwise be an `O(n^2)` brute-force scan.

2. **Exact re-ranking.** Each candidate's stored embedding is retrieved, and exact cosine similarity is computed using SIMD-accelerated operations. The candidates are sorted by descending similarity.

3. **Edge creation.** The top-k candidates (by exact cosine similarity) become edges in the `NarrativeGraph`. Each edge is typed as `Causal` and weighted by the similarity score. `NarrativeGraph.form_edges()` inserts these as directed edges from the existing node to the new node, maintaining both forward and backward adjacency lists.

## Query Path

Queries enter through `tala-cli` and are dispatched to `StorageEngine` methods based on query type.

### Semantic Search (`tala find`)

```
Query embedding
    │
    ▼
StorageEngine.query_semantic(embedding, k)
    │
    ├─ HNSW.search(embedding, k, ef=50)
    │     └─ returns [(index, l2_distance)]
    │
    ├─ index_map lookup: index → IntentId
    │
    └─ cosine re-rank: exact similarity on HNSW candidates
         └─ returns [(IntentId, similarity)]
```

The two-phase approach (approximate HNSW search followed by exact re-ranking) balances speed and accuracy. HNSW search at `ef=50` over a 10K corpus completes in approximately 139 microseconds. The re-ranking step is negligible because it only operates on `k` candidates.

### Temporal Range Query (`tala status`, time-based filters)

```
StorageEngine.query_temporal(TimeRange { start, end })
    │
    └─ Full scan of in-memory intent HashMap
         └─ Filter by timestamp range
              └─ Sort by timestamp ascending
```

### Graph Traversal (`tala why`)

```
NarrativeGraph.bfs_backward(intent_id, max_depth)
    │
    └─ BFS over backward adjacency lists
         └─ Returns causal predecessors up to max_depth hops
```

Backward traversal from a failure event surfaces its causal chain: what preceded it, what triggered it, and which earlier intents contributed to the state that caused it to fail.

### Narrative Extraction (`tala diff`, `tala stitch`)

```
NarrativeGraph.extract_narrative(root, max_depth)
    │
    └─ Bidirectional BFS from root
         └─ Returns (visited_nodes, edges) — the connected subgraph
```

This extracts a coherent narrative: the subgraph of all intents reachable from a root node in both forward and backward directions, bounded by depth.

## Replay Path

Replay reads a narrative subgraph and re-executes it in a new context. This is the `tala replay` path through `tala-weave`.

```
Narrative subgraph (nodes + edges)
    │
    ▼
┌──────────────────┐
│ Planner           │
│                    │
│  Topological sort  │  Respect edge dependencies
│  of intent nodes   │
└────────┬───────────┘
         │
         ▼
┌──────────────────┐
│ Transform         │
│                    │
│  Adapt commands    │  Apply environment deltas
│  to current state  │  (different cwd, env vars, paths)
└────────┬───────────┘
         │
         ▼
┌──────────────────┐
│ Executor          │
│                    │
│  For each step:    │
│  ┌─ Idempotency   │  Skip if already satisfied
│  │   check         │
│  ├─ Execute        │  Sandboxed, with timeout
│  └─ Record outcome │  Attach to intent node
│                    │
│  On failure:       │
│  ├─ Retry          │  Configurable attempts
│  ├─ Skip           │  Mark and continue
│  └─ Abort          │  Halt replay
└────────────────────┘
```

The planner performs a topological sort of the narrative subgraph to determine execution order. Edges encode dependencies: if intent A has a causal edge to intent B, A must complete before B begins.

The transform step adapts commands to the current environment. If the original narrative was recorded in `/home/alice/project` but is being replayed in `/home/bob/project`, path references are rewritten. Environment variable differences are detected via context hash comparison.

The executor runs each command in a sandboxed subprocess with a configurable timeout. Before execution, it checks whether the command's postcondition is already satisfied (idempotency detection). If so, the step is skipped. After execution, the outcome (status, latency, exit code) is attached back to the intent node via `StorageEngine.attach_outcome()`.
