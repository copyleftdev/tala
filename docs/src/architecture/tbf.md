# The TBF Binary Format

TBF (TALA Binary Format) is the on-disk segment format for intent data. It is purpose-built for TALA's hybrid graph-plus-vector data model, co-designed for zero-copy memory-mapped access, SIMD-aligned embedding vectors, and graph adjacency traversal without deserialization.

TBF replaces general-purpose formats (Parquet, Arrow IPC, FlatBuffers) because none of them provide graph-native layout alongside SIMD-aligned vector storage. The cost is a format that only TALA reads and writes. The benefit is that every access pattern TALA cares about -- columnar field scans, embedding similarity, edge traversal, membership tests -- is optimized at the byte level.

## Segment Structure

A single `.tbf` segment file is laid out as follows:

```
Offset 0           ┌──────────────────────────────────┐
                    │        File Header (128B)        │
                    │                                  │
                    │  magic: "TALB"                   │
                    │  version, counts, region offsets │
Offset 128         ├──────────────────────────────────┤
                    │      Node Payload Region         │
                    │    (columnar, 64B-aligned)       │
                    │                                  │
                    │  [ids]  [timestamps]  [hashes]   │
                    │  [confidences]  [statuses]       │
                    ├──────────────────────────────────┤
                    │    Embedding Vector Region        │
                    │   (fixed-stride, 64B-aligned)    │
                    │                                  │
                    │  [vec_0] [pad] [vec_1] [pad] ... │
                    ├──────────────────────────────────┤
                    │        Edge Region (CSR)          │
                    │                                  │
                    │  [row_pointers]  [edge_entries]  │
                    ├──────────────────────────────────┤
                    │       Bloom Filter Block          │
                    │                                  │
                    │  [num_hashes] [num_bits] [bits]  │
                    └──────────────────────────────────┘
```

Every region boundary is padded to a 64-byte boundary. This guarantees that SIMD loads (`_mm512_load_ps` for AVX-512, `_mm256_load_ps` for AVX2) never straddle region or column boundaries.

## File Header

The header occupies the first 128 bytes and is laid out for direct `mmap` casting via `#[repr(C, packed)]`:

| Offset | Size | Field | Description |
|--------|------|-------|-------------|
| 0 | 4 | `magic` | `0x54414C42` ("TALB") |
| 4 | 2 | `version_major` | Format major version |
| 6 | 2 | `version_minor` | Format minor version |
| 8 | 8 | `segment_id` | Monotonic segment identifier |
| 16 | 8 | `created_at` | Unix timestamp (nanoseconds) |
| 24 | 8 | `node_count` | Number of intent nodes |
| 32 | 8 | `edge_count` | Number of edges |
| 40 | 4 | `embedding_dim` | Vector dimensionality |
| 44 | 1 | `embedding_dtype` | 0=f32, 1=f16, 2=bf16, 3=int8 |
| 45 | 1 | `compression` | 0=none, 1=lz4, 2=zstd |
| 46 | 2 | `flags` | Bit flags (see below) |
| 48 | 8 | `node_region_offset` | Byte offset to node payload region |
| 56 | 8 | `embed_region_offset` | Byte offset to embedding region |
| 64 | 8 | `edge_region_offset` | Byte offset to edge region |
| 72 | 8 | `string_table_offset` | Byte offset to string table |
| 80 | 8 | `bloom_offset` | Byte offset to bloom filter |
| 88 | 8 | `index_offset` | Byte offset to B+ tree index |
| 96 | 8 | `footer_offset` | Byte offset to footer |
| 104 | 4 | `checksum` | CRC32C of header bytes [0..104) |
| 108 | 20 | `reserved` | Zero-padded |

### Flags

| Bit | Name | Meaning |
|-----|------|---------|
| 0 | `has_outcomes` | Outcome fields are populated |
| 1 | `has_embeddings` | Embedding region is present |
| 2 | `is_compacted` | Segment is the result of compaction |
| 3 | `is_encrypted` | Payload regions are AES-256-GCM encrypted |
| 4 | `is_partial` | Segment was not cleanly closed (crash indicator) |

A reader encountering `is_partial = 1` knows the segment is incomplete. Recovery discards it and replays the WAL.

## Node Payload Region

Node fields are stored in columnar groups. Each column contains values for all N nodes, stored contiguously. This layout enables vectorized scans over a single field without pulling unrelated data into cache.

### Column Layout

| Column | Element Size | Content |
|--------|-------------|---------|
| `id` | 16 bytes | UUID (128-bit, big-endian) |
| `timestamp` | 8 bytes | u64 nanosecond epoch |
| `context_hash` | 8 bytes | u64 xxHash3 |
| `confidence` | 4 bytes | f32 extraction confidence |
| `outcome_status` | 1 byte | u8 enum: 0=pending, 1=success, 2=failure, 3=partial |

Each column start is padded to the next 64-byte boundary. For a segment with 100K nodes, the timestamp column occupies 800KB of contiguous memory -- a single field scan touches only those 800KB, not the full node data.

The `ColumnReader` provides zero-copy typed access. Given a column offset and a node index, it computes the byte position and interprets the value directly from the underlying buffer:

```rust
// Read the timestamp of node i
let ts = reader.read_u64(timestamp_col_offset, i);
// Computes: offset + i * 8, reads 8 bytes as little-endian u64
```

No allocation. No deserialization struct. Pointer arithmetic and a byte reinterpretation.

## Embedding Vector Region

The most performance-critical region. Embedding vectors are stored contiguously with fixed stride, aligned to 64-byte boundaries.

### Why 64-Byte Alignment

SIMD instruction sets impose alignment requirements on memory operands:

| ISA | Load Instruction | Required Alignment |
|-----|-----------------|-------------------|
| SSE | `_mm_load_ps` | 16 bytes |
| AVX2 | `_mm256_load_ps` | 32 bytes |
| AVX-512 | `_mm512_load_ps` | 64 bytes |
| NEON | `vld1q_f32` | 16 bytes (recommended) |

TALA standardizes on 64 bytes to satisfy all tiers. This means AVX-512 loads can use the aligned variant (`_mm512_load_ps`) instead of the unaligned variant (`_mm512_loadu_ps`), avoiding a potential penalty on some microarchitectures.

### Stride Calculation

The stride is the vector size rounded up to the next 64-byte boundary:

```
stride = align_up(dim * sizeof(dtype), 64)
```

For the default configuration (`dim=384`, `dtype=f32`):
- Vector size: 384 x 4 = 1,536 bytes
- 1,536 / 64 = 24 (already aligned)
- Stride: 1,536 bytes

For `dim=384`, `dtype=f16`:
- Vector size: 384 x 2 = 768 bytes
- 768 / 64 = 12 (already aligned)
- Stride: 768 bytes

### Access Pattern

Given node index `i`, the embedding is at byte offset:

```
embed_region_offset + (i * stride)
```

This is `O(1)` with zero deserialization. An `mmap`'d file pointer plus offset arithmetic is all that is needed. The `EmbeddingReader` wraps this:

```rust
pub fn get(&self, index: usize) -> &[f32] {
    let offset = index * self.stride;
    // SAFETY: offset is within bounds, alignment guaranteed by stride
    unsafe {
        std::slice::from_raw_parts(
            self.data[offset..].as_ptr() as *const f32,
            self.dim,
        )
    }
}
```

### Quantization

The `embedding_dtype` header field determines how bytes in the embedding region are interpreted:

| Value | Type | Bytes/Dim | Use Case |
|-------|------|-----------|----------|
| 0 | f32 | 4 | Hot store, real-time queries |
| 1 | f16 | 2 | Warm store, 2x compression |
| 2 | bf16 | 2 | Training-compatible |
| 3 | int8 | 1 | Cold store, 4x compression |

For int8, each vector carries an 8-byte header (`f32 scale`, `f32 zero_point`) for dequantization. Quality loss at int8 is minimal: Spearman rank correlation of similarity rankings versus f32 baseline is 0.990-0.995.

## Edge Region (CSR)

Edges use Compressed Sparse Row encoding, optimized for forward traversal: given a node, find all its outgoing edges.

### Structure

The edge region contains two arrays:

**Row pointer array:** `(N+1)` entries of `u64`, where `N` is the node count. `row_pointers[i]` gives the index into the edge array where node `i`'s edges begin. `row_pointers[i+1] - row_pointers[i]` gives node `i`'s out-degree.

**Edge entry array:** `E` entries, where each entry is:

| Field | Size | Description |
|-------|------|-------------|
| `target` | 4 bytes | u32 index of destination node |
| `relation` | 1 byte | u8 enum: 0=causal, 1=temporal, 2=dependency, 3=retry, 4=branch |
| `weight` | 4 bytes | f32 confidence |
| `padding` | 3 bytes | Zero (alignment to 12 bytes) |

To find all edges from node `i`:

```rust
let start = row_pointers[i] as usize;
let end = row_pointers[i + 1] as usize;
let edges = &edge_array[start..end];
```

This is a single index lookup and a slice operation. No linked-list traversal, no hash table probe.

### Reverse Index

For backward traversal (find all nodes pointing to a given node), a CSC (Compressed Sparse Column) block can be appended after the CSR block, using the same format but transposed. Its presence is indicated by a flag in the edge region header.

## Bloom Filter Block

A bloom filter over node UUIDs provides fast negative lookups: "Is this intent definitely not in this segment?" This is critical for query routing across multiple segments. Before performing a full B+ tree lookup or column scan, the bloom filter rejects segments that cannot contain the target node.

### Layout

| Field | Size | Description |
|-------|------|-------------|
| `num_hashes` | 4 bytes | Number of hash functions |
| `num_bits` | 8 bytes | Total bit count |
| `bits` | `ceil(num_bits/64) * 8` bytes | Filter data as u64 words |

Target false positive rate: 1%. For 100K nodes, this requires approximately 120KB. The filter uses double hashing (FNV-1a based) with `k` hash functions derived from two base hashes.

## Why Binary-First

TALA stores intent data in a custom binary format rather than JSON, JSONL, or other text formats. The reasons are quantitative:

**Density.** A single intent with a 384-dimensional f32 embedding occupies 1,536 bytes for the embedding alone. In JSON, each float becomes 6-8 characters of text plus separators, inflating to approximately 4,000 bytes. Over millions of intents, this difference is the gap between fitting in memory and not.

**Zero-copy access.** An `mmap`'d TBF segment exposes node fields and embeddings directly as typed memory. A JSON file must be parsed, decoded, and allocated into structs before any field can be read.

**SIMD compatibility.** JSON floats are ASCII text. Computing cosine similarity over JSON-encoded vectors requires parsing every float before any math can happen. TBF embeddings are already in the format that SIMD instructions consume: contiguous `f32` arrays at 64-byte alignment.

**Graph traversal.** The CSR edge index enables `O(1)` lookup of a node's neighbors. In JSON, edges would be embedded in node objects or stored in a separate array requiring linear scan.

## Why Columnar

Row-oriented storage (one struct per intent, fields interleaved) is natural for single-record access but wasteful for analytical queries. When `tala find` needs to scan 100K timestamps to identify a time range, row-oriented layout forces it to pull every field of every intent into cache, even though only the 8-byte timestamp is needed.

Columnar layout stores each field as a contiguous array. A timestamp scan touches 800KB (100K x 8 bytes) instead of the full record size (potentially 2KB+ per intent including the embedding). This matches the access pattern that SIMD vectorization exploits: uniform arrays processed in bulk.

## Comparison with Existing Formats

| Property | TBF | Parquet | Arrow IPC | FlatBuffers |
|----------|-----|---------|-----------|-------------|
| Graph-native layout (CSR) | Yes | No | No | No |
| SIMD-aligned embeddings | Yes (64B) | No | Yes (64B) | No |
| Zero-copy mmap | Yes | Partial | Yes | Yes |
| Columnar node fields | Yes | Yes | Yes | No |
| Built-in bloom filter | Yes | Optional | No | No |
| Built-in graph index | Yes (B+ tree + CSR) | No | No | No |
| Append-only segments | Yes | Yes | No | No |
| Per-region compression | Yes | Per-column | No | No |
