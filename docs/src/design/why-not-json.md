# Why Not JSON

TALA uses a custom binary format (TBF) instead of JSON, JSONL, or any text-based serialization. This is not a preference; it is a performance and architectural requirement.

## The Problem with JSON for TALA's Workload

TALA stores intent nodes that contain 384-dimensional f32 embedding vectors, graph edges with typed relations and weights, timestamps at nanosecond resolution, and outcome metadata. A typical intent node serialized as JSON looks like this:

```json
{
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "timestamp_ns": 1711234567890123456,
  "command": "kubectl rollout restart deployment/api",
  "embedding": [0.0234, -0.1567, 0.3891, ...],
  "outcome": {
    "status": "Success",
    "latency_ns": 234567,
    "exit_code": 0
  }
}
```

The embedding field alone -- 384 floats as decimal strings -- consumes approximately 3,500 bytes in JSON. The same data in binary is 1,536 bytes (384 * 4 bytes per f32). JSON spends 2.3x more bytes on the embedding field than the data itself occupies.

Multiply by 100,000 nodes in a segment, and the difference is 335 MB (JSON) vs. 147 MB (binary) for embeddings alone.

## Access Pattern Mismatch

TALA's query path has three distinct access patterns, none of which JSON serves well.

### Columnar Embedding Scan

Semantic search computes cosine similarity between a query vector and every embedding in a segment. This requires streaming through the embedding column contiguously. In TBF, the embedding region is a flat array of 64-byte-aligned f32 vectors, laid out consecutively:

```
[embedding_0][embedding_1][embedding_2]...
```

The CPU can prefetch the next cache line while processing the current one. SIMD instructions load 8 (AVX2) or 16 (AVX-512) floats at a time from aligned addresses.

In JSON, embeddings are interleaved with metadata fields, separated by field names, brackets, and commas. To scan embeddings, you must parse the entire document for every node, skipping fields you do not need. There is no way to jump to "the embedding of node N" without parsing everything before it.

### Graph Traversal

Traversing the narrative graph requires looking up all edges from a given source node. TBF uses Compressed Sparse Row (CSR) format, which stores edges sorted by source and provides an O(1) lookup to the start of any node's edge list via a row pointer array.

In JSON, edges would be either nested within node objects (requiring full document parse to extract) or stored as a separate array (requiring linear scan to find edges for a given source).

### Membership Testing

Before querying a segment, TALA checks whether a target intent ID exists in it using a Bloom filter. This is a constant-time probabilistic check that avoids scanning segments that cannot contain the target.

JSON has no equivalent. You would need to parse the entire file or maintain a separate index, which negates the simplicity argument for JSON.

## Zero-Copy Reads

TBF segments are designed for memory-mapped access. The `mmap` system call maps the file directly into the process address space. The embedding region, with its 64-byte alignment, can be accessed as a native `&[f32]` slice without any copying or parsing:

```rust
// Zero-copy: pointer into mmap'd file
let embedding: &[f32] = embedding_reader.get(node_index);
```

JSON requires allocating memory, parsing the text, converting decimal strings to binary floats, and storing the result. For a 100K-node segment with dim=384, this parse step alone takes tens of milliseconds. The TBF equivalent takes zero time -- the data is already in the correct binary layout in the kernel page cache.

## SIMD Alignment

AVX2 operates on 256-bit (32-byte) registers. AVX-512 operates on 512-bit (64-byte) registers. Aligned load instructions (`_mm256_load_ps`, `_mm512_load_ps`) require the source address to be aligned to the register width. Unaligned loads (`_mm256_loadu_ps`) work on any address but may incur a performance penalty when crossing cache line boundaries.

TBF guarantees 64-byte alignment for every embedding vector. This means aligned loads work for both AVX2 and AVX-512 without runtime checks.

JSON produces f32 arrays at whatever alignment `Vec<f32>::new()` provides (typically 8-byte on 64-bit systems). To use aligned SIMD loads, you must copy the data into an aligned buffer -- an extra allocation and memcpy for every access.

## The Cost of Parsing

The fundamental issue is that JSON encodes numbers as decimal text. Converting the string `"0.0234"` to the IEEE 754 float `0x3BBFCD36` requires:

1. Scanning for the decimal point
2. Parsing the integer part
3. Parsing the fractional part
4. Combining with appropriate scaling
5. Rounding to nearest representable float

This is approximately 20-50ns per float. For a 384-dimensional vector, that is 7-19us per embedding. For 100K embeddings, that is 700ms-1.9s just for float parsing.

TBF cost for the same operation: zero. The bytes in the file are already IEEE 754 floats. Memory mapping makes them available as native values.

## What JSON Gets Right

JSON is human-readable, widely supported, and trivially debuggable. For configuration files, API responses, and interchange formats, it is an excellent choice.

TALA does not need human readability in its storage path. No human reads 100,000 embedding vectors. The debugging tools (`tala-cli`) decode TBF segments into human-readable output when inspection is needed.

## Why Not Parquet / Arrow / FlatBuffers

- **Parquet**: designed for analytics (column scans, predicate pushdown, compression). Does not support graph adjacency structures. Adds a heavyweight dependency.
- **Arrow**: an in-memory columnar format with excellent SIMD support, but designed for tabular data. No native graph representation. The Arrow IPC format could carry embeddings but not CSR edges or Bloom filters.
- **FlatBuffers / Cap'n Proto**: zero-copy serialization frameworks with strong cross-language support. Could serve as the embedding and metadata format, but do not provide CSR graph indexing, Bloom filters, or the specific columnar layout TALA needs. Adopting them would mean building TALA's graph and index structures on top, gaining little.

TBF is purpose-built for TALA's specific combination of columnar embeddings, CSR graph edges, Bloom membership, and B+ tree indexing. It trades generality for a format that is exactly right for this workload.
