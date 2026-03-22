# tala-wire

The TALA Binary Format (TBF) crate. Implements on-disk segment serialization with columnar node storage, CSR edge indexing, 64-byte aligned embedding regions, and bloom filter membership tests. Every data structure in this crate is designed for zero-copy read access and SIMD-friendly alignment.

## Key Types

| Type | Description |
|------|-------------|
| `BloomFilter` | Probabilistic membership test using FNV-1a double hashing |
| `CsrBuilder` | Builder for constructing a CSR edge index |
| `CsrIndex` | Compressed Sparse Row edge index (read-only) |
| `CsrEdge` | A single edge entry: target node, relation type, weight |
| `ColumnarBuffer` | In-memory columnar storage for intent node fields |
| `ColumnReader` | Zero-copy typed access to serialized column data |
| `EmbeddingWriter` | Writes 64-byte aligned embedding vectors |
| `EmbeddingReader` | Zero-copy read access to aligned embedding vectors |
| `SegmentWriter` | Orchestrates all regions into a complete TBF segment |
| `SegmentReader` | Parses and validates a TBF segment from bytes |
| `SegmentHeader` | Parsed header fields of a TBF segment |

## Constants

```rust
/// Magic bytes identifying a TBF segment: "TALB".
pub const MAGIC: [u8; 4] = *b"TALB";

/// Fixed header size in bytes.
pub const HEADER_SIZE: usize = 128;

/// Alignment boundary in bytes for all regions and columns.
pub const ALIGN: usize = 64;
```

## Utility Functions

```rust
/// Round `value` up to the next multiple of `alignment`.
#[inline]
pub fn align_up(value: usize, alignment: usize) -> usize;
```

---

## BloomFilter

A probabilistic set membership test backed by a bit vector of `u64` words. Uses FNV-1a double hashing to produce `num_hashes` independent bit positions per key. The filter is sized automatically from expected item count and desired false positive rate.

```rust
pub struct BloomFilter {
    pub bits: Vec<u64>,
    pub num_hashes: u32,
    pub num_bits: u64,
}
```

### Methods

```rust
impl BloomFilter {
    /// Create a new filter sized for `expected_items` with the target `fp_rate`.
    /// The number of bits and hash functions are computed from the optimal formulas.
    pub fn new(expected_items: usize, fp_rate: f64) -> Self;

    /// Insert a key into the filter.
    pub fn insert(&mut self, key: &[u8]);

    /// Test whether a key is possibly in the set. False positives are possible;
    /// false negatives are not.
    pub fn contains(&self, key: &[u8]) -> bool;

    /// Return the size of the bit vector in bytes.
    pub fn size_bytes(&self) -> usize;
}
```

### Example

```rust
use tala_wire::BloomFilter;

let mut bloom = BloomFilter::new(1000, 0.01);
bloom.insert(b"intent-abc");
assert!(bloom.contains(b"intent-abc"));
```

---

## CsrEdge

A single edge entry in the CSR index. Stored compactly with a 4-byte target node index, a 1-byte relation type tag, and a 4-byte weight. On disk each entry occupies 12 bytes (with 3 bytes padding).

```rust
#[derive(Clone, Debug)]
pub struct CsrEdge {
    pub target: u32,
    pub relation: u8,
    pub weight: f32,
}
```

---

## CsrBuilder and CsrIndex

The Compressed Sparse Row representation stores edges in a flat array with a row-offset array for O(1) adjacency lookup per node.

### CsrBuilder

```rust
pub struct CsrBuilder { /* private */ }

impl CsrBuilder {
    /// Create a builder for a graph with `node_count` nodes.
    pub fn new(node_count: usize) -> Self;

    /// Add a directed edge from node `from` to node `to`.
    pub fn add_edge(&mut self, from: usize, to: usize, relation: u8, weight: f32);

    /// Consume the builder and produce a read-only `CsrIndex`.
    pub fn build(self) -> CsrIndex;
}
```

### CsrIndex

```rust
pub struct CsrIndex { /* private */ }

impl CsrIndex {
    /// Return all edges originating from `node`. O(1) lookup via row offsets.
    pub fn edges_from(&self, node: usize) -> &[CsrEdge];

    /// Return the out-degree of `node`.
    pub fn degree(&self, node: usize) -> usize;

    /// Return the total number of edges in the index.
    pub fn edge_count(&self) -> usize;

    /// Return the number of nodes in the index.
    pub fn node_count(&self) -> usize;
}
```

### Example

```rust
use tala_wire::{CsrBuilder, CsrEdge};

let mut builder = CsrBuilder::new(4);
builder.add_edge(0, 1, 0, 1.0); // node 0 -> node 1, Causal
builder.add_edge(0, 2, 1, 0.5); // node 0 -> node 2, Temporal
builder.add_edge(2, 3, 0, 0.8); // node 2 -> node 3, Causal
let index = builder.build();

assert_eq!(index.node_count(), 4);
assert_eq!(index.edge_count(), 3);
assert_eq!(index.degree(0), 2);
assert_eq!(index.edges_from(0)[0].target, 1);
```

---

## ColumnarBuffer and ColumnReader

### ColumnarBuffer

In-memory columnar storage for intent node fields. Each field is stored in a separate vector, enabling efficient serialization where each column is 64-byte aligned for SIMD-friendly access.

```rust
pub struct ColumnarBuffer {
    pub ids: Vec<[u8; 16]>,
    pub timestamps: Vec<u64>,
    pub context_hashes: Vec<u64>,
    pub confidences: Vec<f32>,
    pub outcome_statuses: Vec<u8>,
}
```

#### Methods

```rust
impl ColumnarBuffer {
    /// Create an empty buffer.
    pub fn new() -> Self;

    /// Create an empty buffer with pre-allocated capacity.
    pub fn with_capacity(cap: usize) -> Self;

    /// Append a row.
    pub fn push(
        &mut self,
        id: &[u8; 16],
        timestamp: u64,
        context_hash: u64,
        confidence: f32,
        status: u8,
    );

    /// Return the number of rows.
    pub fn len(&self) -> usize;

    /// Return true if the buffer is empty.
    pub fn is_empty(&self) -> bool;

    /// Serialize all columns into a flat byte buffer.
    /// Returns (buffer, column_offsets) where offsets index:
    /// [0]=ids, [1]=timestamps, [2]=context_hashes, [3]=confidences, [4]=statuses.
    /// Each column begins at a 64-byte aligned offset.
    pub fn serialize(&self) -> (Vec<u8>, Vec<usize>);
}
```

### ColumnReader

Zero-copy typed access into a serialized column buffer. Each read method takes a column offset (from `ColumnarBuffer::serialize`) and a row index.

```rust
pub struct ColumnReader<'a> { /* private */ }

impl<'a> ColumnReader<'a> {
    /// Wrap a raw byte slice for typed column access.
    pub fn new(data: &'a [u8]) -> Self;

    /// Read a u64 value at the given column offset and row index.
    #[inline]
    pub fn read_u64(&self, col_offset: usize, index: usize) -> u64;

    /// Read an f32 value at the given column offset and row index.
    #[inline]
    pub fn read_f32(&self, col_offset: usize, index: usize) -> f32;

    /// Read a u8 value at the given column offset and row index.
    #[inline]
    pub fn read_u8(&self, col_offset: usize, index: usize) -> u8;

    /// Read a 16-byte ID at the given column offset and row index.
    #[inline]
    pub fn read_id(&self, col_offset: usize, index: usize) -> [u8; 16];
}
```

---

## EmbeddingWriter and EmbeddingReader

### EmbeddingWriter

Writes embedding vectors into a contiguous buffer with each vector padded to a 64-byte aligned stride. For `dim=384`, each vector occupies `align_up(384 * 4, 64) = 1536` bytes.

```rust
pub struct EmbeddingWriter { /* private */ }

impl EmbeddingWriter {
    /// Create a writer for embeddings of the given dimensionality.
    pub fn new(dim: usize) -> Self;

    /// Append an embedding vector. The slice length must equal `dim`.
    pub fn push(&mut self, embedding: &[f32]);

    /// Return the serialized byte buffer.
    pub fn as_bytes(&self) -> &[u8];

    /// Return the padded stride (bytes per embedding).
    pub fn stride(&self) -> usize;

    /// Return the number of embeddings written.
    pub fn count(&self) -> usize;
}
```

### EmbeddingReader

Zero-copy read access to aligned embedding vectors. Returns `&[f32]` slices directly from the underlying byte buffer without copying.

```rust
pub struct EmbeddingReader<'a> { /* private */ }

impl<'a> EmbeddingReader<'a> {
    /// Wrap a byte slice for zero-copy embedding access.
    /// `dim` must match the dimensionality used during writing.
    pub fn new(data: &'a [u8], dim: usize) -> Self;

    /// Return the embedding at the given index as a float slice.
    #[inline]
    pub fn get(&self, index: usize) -> &[f32];

    /// Return the number of embeddings in the buffer.
    pub fn count(&self) -> usize;
}
```

---

## SegmentWriter

Orchestrates the serialization of a complete TBF segment. Accepts nodes (with their columnar fields and embeddings) and edges, then produces a single contiguous byte buffer containing: header, node columns, embedding region, CSR edge index, and bloom filter.

```rust
pub struct SegmentWriter { /* private */ }

impl SegmentWriter {
    /// Create a writer for segments with the given embedding dimensionality.
    pub fn new(dim: usize) -> Self;

    /// Add a node with all its fields and embedding.
    pub fn push_node(
        &mut self,
        id: &[u8; 16],
        timestamp: u64,
        context_hash: u64,
        confidence: f32,
        status: u8,
        embedding: &[f32],
    );

    /// Add a directed edge between two nodes (by insertion-order index).
    pub fn add_edge(&mut self, from: usize, to: usize, relation: u8, weight: f32);

    /// Finalize the segment into a contiguous byte buffer with a valid TBF header.
    pub fn finish(self) -> Vec<u8>;
}
```

### Example

```rust
use tala_wire::SegmentWriter;

let dim = 8;
let mut writer = SegmentWriter::new(dim);

let id = [0u8; 16];
let embedding = vec![0.1f32; dim];
writer.push_node(&id, 1000, 42, 0.95, 1, &embedding);
writer.add_edge(0, 0, 0, 1.0); // self-edge for demonstration

let segment_bytes = writer.finish();
assert!(segment_bytes.len() > 128); // at least header + data
```

---

## SegmentHeader

The parsed fields from a TBF segment's 128-byte header.

```rust
pub struct SegmentHeader {
    pub version_minor: u16,
    pub version_major: u16,
    pub node_count: u64,
    pub edge_count: u64,
    pub dim: u32,
    pub node_region_offset: u64,
    pub embed_region_offset: u64,
    pub edge_region_offset: u64,
    pub bloom_offset: u64,
}
```

---

## SegmentReader

Parses and validates a TBF segment from a byte buffer. Provides typed access to every region: columnar node fields, embeddings, CSR edges, and bloom filter.

```rust
pub struct SegmentReader<'a> { /* private */ }

impl<'a> SegmentReader<'a> {
    /// Parse a TBF segment from raw bytes. Returns an error string on failure
    /// (invalid magic, truncated header).
    pub fn open(data: &'a [u8]) -> Result<Self, &'static str>;

    /// Access the parsed header.
    pub fn header(&self) -> &SegmentHeader;

    /// Number of intent nodes in the segment.
    pub fn node_count(&self) -> usize;

    /// Number of edges in the segment.
    pub fn edge_count(&self) -> usize;

    /// Embedding dimensionality.
    pub fn dim(&self) -> usize;

    /// Return a ColumnReader over the raw segment bytes.
    pub fn column_reader(&self) -> ColumnReader<'a>;

    /// Read a node ID by row index.
    #[inline]
    pub fn read_id(&self, index: usize) -> [u8; 16];

    /// Read a timestamp by row index.
    #[inline]
    pub fn read_timestamp(&self, index: usize) -> u64;

    /// Read a context hash by row index.
    #[inline]
    pub fn read_context_hash(&self, index: usize) -> u64;

    /// Read a confidence score by row index.
    #[inline]
    pub fn read_confidence(&self, index: usize) -> f32;

    /// Read an outcome status byte by row index.
    #[inline]
    pub fn read_status(&self, index: usize) -> u8;

    /// Zero-copy embedding reader for the embedding region.
    pub fn embedding_reader(&self) -> EmbeddingReader<'a>;

    /// Reconstruct the CSR edge index from the edge region.
    pub fn csr_index(&self) -> CsrIndex;

    /// Reconstruct the bloom filter from the bloom region.
    pub fn bloom_filter(&self) -> BloomFilter;

    /// Return the raw segment bytes.
    pub fn as_bytes(&self) -> &[u8];
}
```

### Roundtrip Example

```rust
use tala_wire::{SegmentWriter, SegmentReader};

let dim = 8;
let mut writer = SegmentWriter::new(dim);

let id = [0xABu8; 16];
let embedding = vec![1.0f32; dim];
writer.push_node(&id, 42, 7, 0.9, 1, &embedding);

let bytes = writer.finish();
let reader = SegmentReader::open(&bytes).expect("valid segment");

assert_eq!(reader.node_count(), 1);
assert_eq!(reader.read_id(0), id);
assert_eq!(reader.read_timestamp(0), 42);

let emb = reader.embedding_reader();
assert_eq!(emb.get(0).len(), dim);
```
