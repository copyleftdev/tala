# System Overview

TALA replaces the flat command log with a causality-aware, graph-structured narrative of intent. Where `history` records what was typed, TALA models *why* it was typed, what happened as a result, and how that result connects to everything before and after it.

The system is built from four subsystems, each responsible for a distinct phase of the intent lifecycle.

## Subsystems

**talad** (the daemon) is the central orchestrator. It captures raw commands from shell hooks and agent APIs, routes them through the intent extraction pipeline, manages storage, and serves queries over a Unix socket. Every other subsystem communicates through `talad`.

**weave** (the replay engine) reconstructs and re-executes narratives. Given a subgraph of the intent DAG, `weave` schedules commands in dependency order, adapts them to the current environment, detects idempotent operations, and handles failures with configurable recovery strategies (retry, skip, abort).

**kai** (the insight engine) operates over the accumulated narrative graph to detect patterns: recurring failure clusters, repeated command motifs, and predictive next-intent suggestions. It groups related intents by embedding-space proximity and surfaces structured reports.

**tala-cli** (the user interface) provides the command-line surface. Commands like `tala find`, `tala replay`, `tala diff`, `tala why`, and `tala stitch` translate user queries into requests against `talad` and format the results for human or machine consumption.

## Architecture

```
                         ┌─────────────────────┐
                         │  User / Agent Input  │
                         │  (shell hook, API)   │
                         └──────────┬───────────┘
                                    │
                                    ▼
                         ┌─────────────────────┐
                         │       talad          │
                         │  (tala-daemon)       │
                         │                      │
                         │  ┌───────────────┐   │
                         │  │    Intent      │   │
                         │  │ Normalization  │   │
                         │  │ (tala-intent)  │   │
                         │  └───────┬───────┘   │
                         │          │           │
                         │          ▼           │
                         │  ┌───────────────┐   │
                         │  │    Graph       │   │
                         │  │ Construction   │   │
                         │  │ (tala-graph)   │   │
                         │  └───────┬───────┘   │
                         │          │           │
                         │          ▼           │
                         │  ┌───────────────┐   │
                         │  │  Persistent    │   │
                         │  │    Store       │   │
                         │  │ (tala-store)   │   │
                         │  └───────────────┘   │
                         └───┬──────┬──────┬────┘
                             │      │      │
                    ┌────────┘      │      └────────┐
                    ▼               ▼               ▼
             ┌───────────┐  ┌───────────┐  ┌───────────┐
             │ tala-cli  │  │   weave   │  │    kai    │
             │ (query,   │  │ (replay,  │  │ (insight, │
             │  browse)  │  │  adapt)   │  │  predict) │
             └───────────┘  └───────────┘  └───────────┘
```

## Layered Design

The architecture separates concerns into three layers, each with distinct runtime characteristics.

### Capture Layer

Raw input enters through shell hooks (bash `PROMPT_COMMAND`, zsh `precmd`) or agent APIs. The capture layer is intentionally thin: it records the raw command string, execution context (working directory, environment hash, session ID), and timestamp, then forwards everything to `talad` over a Unix socket. No processing happens here. Latency budget: under 1 millisecond.

### Processing Layer

`talad` orchestrates three operations on every incoming command:

1. **Intent extraction** (`tala-intent`). The raw command is tokenized, classified into a category (build, deploy, debug, configure, query, navigate), and enriched with context metadata. An embedding model generates a dense vector representation. The output is a fully structured `Intent` node.

2. **Storage** (`tala-store`). The intent is durably written to the WAL, inserted into the HNSW index for semantic search, and buffered in the hot store. When the buffer reaches capacity, it flushes to an immutable TBF segment on disk.

3. **Graph construction** (`tala-graph`). HNSW approximate nearest-neighbor search identifies semantically related intents. Edge candidates are re-ranked by exact cosine similarity, and the top-k connections are formed as weighted causal edges in the narrative DAG.

### Query Layer

Consumers interact with the narrative graph through three interfaces:

- **tala-cli** issues semantic search (`tala find`), temporal range queries, graph traversals (`tala why`), and narrative diffs (`tala diff`).
- **weave** reads subgraphs and replays them, adapting commands to new environments and skipping idempotent steps.
- **kai** runs background analysis: clustering failures by embedding proximity, detecting recurring narrative motifs, and generating next-action predictions.

## Design Principles

The architecture enforces several invariants:

**Intent is a first-class primitive.** An intent is not a command wrapper. It is a structured representation of a desired outcome: `I = f(C, X, P)` where `C` is context, `X` is the action, and `P` is the expected postcondition. This representation survives across sessions, machines, and users.

**DAG, not tree.** The narrative graph is a directed acyclic graph with probabilistic edges. A single intent may have multiple causal parents (merge points) and multiple children (branch points). Tree structures cannot represent this.

**Binary-first storage.** TBF segments are columnar, SIMD-aligned, and zero-copy readable via `mmap`. No JSON serialization on the hot path.

**Trait boundaries between subsystems.** Every inter-crate interface is defined as a trait in `tala-core`. Concrete implementations live in their respective crates. This enforces that subsystems can be tested in isolation and swapped without cascading changes.

**Append-only durability.** Segments are immutable once flushed. The WAL provides crash recovery. No in-place mutation of on-disk data.
