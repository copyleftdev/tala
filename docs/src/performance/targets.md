# Benchmark Targets

TALA defines quantitative performance targets in spec-01 (binary format) and spec-02 (embedding acceleration). These are validated by Criterion benchmark suites in each crate. Targets that pass are guarded against regression; targets that do not yet pass have identified remediation paths.

## Results Summary

All measurements taken on x86-64 with AVX2+FMA, 384-dimensional f32 embeddings unless otherwise noted.

| Operation | Spec Target | Measured | Status |
|-----------|-------------|----------|--------|
| Cosine similarity (dim=384, AVX2) | < 20ns | 39.6ns | Needs AVX-512 |
| Batch cosine (1K, single thread) | < 50us | 41.7us | **PASS** |
| Batch cosine (100K, parallel) | < 5ms | 3.41ms | **PASS** |
| HNSW search (10K, ef=50, top-10) | < 1ms | 139us | **PASS** |
| Columnar scan (100K timestamps) | -- | 48us | Baseline |
| CSR traverse (10K lookups) | -- | 21.7us | Baseline |
| Bloom lookup (1K queries) | -- | 27us | Baseline |
| Full segment write (1K nodes) | -- | 209us | Baseline |
| WAL append (1K entries, dim=384) | -- | 730us | Baseline |
| Semantic query (10K corpus, top-10) | < 50ms | 151us | **PASS** |
| Full ingest pipeline (1K) | -- | 1.10ms | Baseline |

## Status Definitions

- **PASS**: measured value meets or exceeds the spec target. Regressions on passing targets are merge blockers.
- **Baseline**: no spec target defined. The measured value establishes a baseline for regression detection.
- **Needs AVX-512**: the target was set assuming AVX-512 SIMD width. Current implementation uses AVX2 (256-bit), which processes 8 floats per cycle instead of 16. The 2x gap is expected.

## Detailed Breakdown

### Cosine Similarity (Single Pair)

Measures the wall-clock time to compute cosine similarity between two 384-dimensional f32 vectors using the AVX2+FMA kernel.

- **Target**: < 20ns (spec-02, assuming AVX-512 at 512-bit width)
- **Measured**: 39.6ns on AVX2 (256-bit width)
- **Analysis**: The AVX2 inner loop processes 8 floats per iteration (48 iterations for dim=384). AVX-512 would process 16 floats per iteration (24 iterations), roughly halving cycle count. The 39.6ns / 20ns ratio aligns with the 2x throughput difference.

### Batch Cosine (1K Vectors, Single Thread)

Computes cosine similarity between one query vector and 1,000 corpus vectors sequentially on a single core.

- **Target**: < 50us
- **Measured**: 41.7us
- **Analysis**: 41.7ns per vector pair, consistent with single-pair measurements. Memory-bandwidth limited at this scale since the corpus fits in L2 cache.

### Batch Cosine (100K Vectors, Parallel)

Computes cosine similarity between one query vector and 100,000 corpus vectors using Rayon parallel iterators across all available cores.

- **Target**: < 5ms
- **Measured**: 3.41ms
- **Analysis**: Linear scaling from the single-thread case would predict ~4.17ms. The measured value of 3.41ms shows effective parallelization with minimal scheduling overhead.

### HNSW Search

Searches a 10,000-vector HNSW index for the top 10 nearest neighbors with ef=50 (search beam width).

- **Target**: < 1ms
- **Measured**: 139us
- **Analysis**: 7x headroom below target. The HNSW implementation uses M=16 (connections per layer) and ef_construction=200. Average nodes visited per search is tracked via `tala_hnsw_avg_search_visited` in the metrics.

### Columnar Scan (100K Timestamps)

Scans the timestamp column of a 100K-node columnar buffer to filter by time range.

- **Measured**: 48us
- **Analysis**: Sequential scan over contiguous u64 values. Benefits from hardware prefetch and cache-line alignment. Establishes the baseline for time-range queries over TBF segments.

### CSR Traverse (10K Lookups)

Performs 10,000 adjacency lookups in a Compressed Sparse Row index, retrieving the edge list for each source node.

- **Measured**: 21.7us
- **Analysis**: ~2.17ns per lookup. CSR provides O(1) access to the start of each node's edge list via the row pointer array, then sequential scan over edges.

### Bloom Lookup (1K Queries)

Tests 1,000 UUIDs against a Bloom filter for membership.

- **Measured**: 27us
- **Analysis**: ~27ns per query. Multiple hash functions (k=7) are computed per probe. False positive rate depends on filter sizing relative to the number of inserted elements.

### Full Segment Write (1K Nodes)

Serializes 1,000 intent nodes with embeddings, edges, and metadata into a complete TBF segment via `SegmentWriter`.

- **Measured**: 209us
- **Analysis**: Includes columnar encoding of node payloads, 64-byte-aligned embedding packing, CSR edge construction, Bloom filter population, and CRC32C checksum computation.

### WAL Append (1K Entries)

Appends 1,000 intent entries (each with a 384-dimensional embedding) to the write-ahead log.

- **Measured**: 730us
- **Analysis**: ~730ns per entry. Each append serializes the intent, writes to the log file with fsync semantics, and updates the entry counter.

### Semantic Query (10K Corpus, Top-10)

End-to-end semantic search: given a query embedding, search the HNSW index over 10K stored intents and return the top 10 results with metadata.

- **Target**: < 50ms
- **Measured**: 151us
- **Analysis**: 330x headroom. The query path is HNSW search (139us) plus metadata lookup for the 10 result IDs (~12us).

### Full Ingest Pipeline (1K Intents)

End-to-end ingestion of 1,000 intents through the complete pipeline: extraction, WAL append, HNSW insert, edge formation, and hot buffer push.

- **Measured**: 1.10ms
- **Analysis**: ~1.1us per intent across all pipeline stages. Edge formation dominates at scale due to O(n^2) nearest-neighbor search for edge candidates.

## Regression Policy

Performance regressions on passing targets are merge blockers. Run `cargo bench` before merging any change that touches hot-path code in `tala-embed`, `tala-wire`, `tala-store`, or `tala-graph`.

Baseline measurements should not regress by more than 20% without investigation. Criterion's built-in comparison detects statistically significant changes.
