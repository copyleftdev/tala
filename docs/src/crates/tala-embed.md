# tala-embed

SIMD-accelerated vector operations, quantization, and the HNSW approximate nearest neighbor index. This crate is the computational engine behind TALA's semantic search: it computes cosine similarity, dot products, and L2 distances using runtime-dispatched AVX2+FMA intrinsics on x86_64, with a scalar fallback that compiles everywhere. It also provides INT8 and FP16 quantization for storage compression and an HNSW index that delivers sub-millisecond top-k search over tens of thousands of vectors.

## Key Types

| Type | Description |
|------|-------------|
| `AlignedVec` | 64-byte aligned f32 vector for SIMD operations |
| `HnswIndex` | Hierarchical Navigable Small World approximate nearest neighbor index |

## Key Modules

| Module | Description |
|--------|-------------|
| `scalar` | Portable fallback: `dot_product`, `cosine_similarity`, `l2_distance_sq`, `norm_sq` |
| `avx2` | AVX2+FMA implementations (x86_64 only, `#[cfg(target_arch = "x86_64")]`) |
| `quantize` | Quantization: `f32_to_int8`, `int8_to_f32`, `f32_to_f16`, `f16_to_f32` |

## Dispatch Functions

| Function | Description |
|----------|-------------|
| `cosine_similarity(a, b)` | Cosine similarity with runtime ISA dispatch |
| `dot_product(a, b)` | Dot product with runtime ISA dispatch |
| `l2_distance_sq(a, b)` | L2 distance squared with runtime ISA dispatch |
| `batch_cosine(query, corpus, dim, results)` | Single-threaded batch cosine similarity |
| `batch_cosine_parallel(query, corpus, dim, results)` | Rayon-parallel batch cosine similarity |

---

## AlignedVec

A heap-allocated f32 vector guaranteed to begin at a 64-byte aligned address. This alignment is required for optimal AVX-512 loads and beneficial for AVX2. `AlignedVec` manages its own memory through the global allocator with `Layout::from_size_align(len * 4, 64)`.

```rust
pub struct AlignedVec {
    ptr: *mut f32,
    len: usize,
    cap: usize,
}
```

`AlignedVec` implements `Send`, `Sync`, `Clone`, `Deref<Target = [f32]>`, `DerefMut`, `From<Vec<f32>>`, and `From<&[f32]>`. It can be used anywhere a `&[f32]` is expected.

### Methods

```rust
impl AlignedVec {
    /// Allocate a zero-initialized vector of `len` elements at 64-byte alignment.
    pub fn new(len: usize) -> Self;

    /// View as a shared float slice.
    pub fn as_slice(&self) -> &[f32];

    /// View as a mutable float slice.
    pub fn as_mut_slice(&mut self) -> &mut [f32];

    /// Number of elements.
    pub fn len(&self) -> usize;

    /// True if the vector contains no elements.
    pub fn is_empty(&self) -> bool;
}
```

### Example

```rust
use tala_embed::AlignedVec;

let mut v = AlignedVec::new(384);
assert_eq!(v.len(), 384);
assert!(v.as_slice().as_ptr() as usize % 64 == 0);

// Populate from a Vec<f32>:
let data: Vec<f32> = (0..384).map(|i| i as f32).collect();
let aligned = AlignedVec::from(data);
assert_eq!(aligned[0], 0.0);
assert_eq!(aligned[383], 383.0);
```

---

## scalar Module

Portable implementations of the core vector operations. These are used on architectures without AVX2 support and serve as the reference implementation for correctness testing.

```rust
pub mod scalar {
    /// Dot product of two equal-length slices.
    #[inline]
    pub fn dot_product(a: &[f32], b: &[f32]) -> f32;

    /// Squared L2 norm of a vector: sum of x_i^2.
    #[inline]
    pub fn norm_sq(a: &[f32]) -> f32;

    /// Cosine similarity: dot(a,b) / (||a|| * ||b|| + epsilon).
    #[inline]
    pub fn cosine_similarity(a: &[f32], b: &[f32]) -> f32;

    /// Squared L2 distance: sum of (a_i - b_i)^2.
    #[inline]
    pub fn l2_distance_sq(a: &[f32], b: &[f32]) -> f32;
}
```

---

## avx2 Module

AVX2+FMA implementations available on x86_64. All functions are marked `#[target_feature(enable = "avx2,fma")]` and must be called through `unsafe` blocks. The dispatch functions handle this automatically.

```rust
#[cfg(target_arch = "x86_64")]
pub mod avx2 {
    /// Dot product -- 4-way unrolled (32 floats per iteration) to saturate
    /// FMA throughput. Four independent accumulator chains reduce the critical
    /// path 4x vs a single accumulator.
    #[target_feature(enable = "avx2,fma")]
    pub unsafe fn dot_product(a: &[f32], b: &[f32]) -> f32;

    /// Cosine similarity -- 2-way unrolled (6 accumulators: 2 dot + 2 norm_a
    /// + 2 norm_b). Fits within AVX2's 16 YMM register budget (10 live regs).
    #[target_feature(enable = "avx2,fma")]
    pub unsafe fn cosine_similarity(a: &[f32], b: &[f32]) -> f32;

    /// Squared L2 distance -- 4-way unrolled (sub then FMA).
    #[target_feature(enable = "avx2,fma")]
    pub unsafe fn l2_distance_sq(a: &[f32], b: &[f32]) -> f32;
}
```

---

## Dispatch Functions

These top-level functions detect AVX2+FMA at runtime via `is_x86_feature_detected!` and dispatch to the fastest available implementation. On non-x86_64 architectures, they fall through directly to the scalar path. Call these from application code; never call the `avx2` module functions directly.

```rust
/// Cosine similarity with automatic SIMD dispatch.
pub fn cosine_similarity(a: &[f32], b: &[f32]) -> f32;

/// Dot product with automatic SIMD dispatch.
pub fn dot_product(a: &[f32], b: &[f32]) -> f32;

/// L2 distance squared with automatic SIMD dispatch.
pub fn l2_distance_sq(a: &[f32], b: &[f32]) -> f32;
```

### Batch Operations

```rust
/// Compute cosine similarity of `query` against every vector in `corpus`.
/// `corpus` is a flat buffer: vector i = corpus[i*dim .. (i+1)*dim].
/// Results are written into `results[0..n]`.
pub fn batch_cosine(query: &[f32], corpus: &[f32], dim: usize, results: &mut [f32]);

/// Parallel batch cosine using Rayon. Processes corpus in chunks of 256
/// vectors per thread.
pub fn batch_cosine_parallel(query: &[f32], corpus: &[f32], dim: usize, results: &mut [f32]);
```

### Example

```rust
use tala_embed::{cosine_similarity, batch_cosine};

let a = vec![1.0, 0.0, 0.0, 0.0];
let b = vec![1.0, 0.0, 0.0, 0.0];
let sim = cosine_similarity(&a, &b);
assert!((sim - 1.0).abs() < 1e-5);

// Batch: query against 3 corpus vectors of dim 4
let query = vec![1.0, 0.0, 0.0, 0.0];
let corpus = vec![
    1.0, 0.0, 0.0, 0.0,  // identical to query
    0.0, 1.0, 0.0, 0.0,  // orthogonal
    0.5, 0.5, 0.0, 0.0,  // partially similar
];
let mut results = vec![0.0f32; 3];
batch_cosine(&query, &corpus, 4, &mut results);
assert!(results[0] > results[2]); // identical > partial
assert!(results[2] > results[1]); // partial > orthogonal
```

---

## quantize Module

Quantization routines for compressing f32 embeddings to INT8 or FP16. Useful for reducing storage footprint (4x for INT8, 2x for FP16) at the cost of some precision.

```rust
pub mod quantize {
    /// Symmetric f32 -> INT8 quantization.
    /// Returns (quantized_bytes, scale_factor). Dequantize with: f32 = i8 * scale.
    pub fn f32_to_int8(src: &[f32]) -> (Vec<i8>, f32);

    /// INT8 -> f32 dequantization.
    pub fn int8_to_f32(src: &[i8], scale: f32) -> Vec<f32>;

    /// f32 -> IEEE 754 half-precision (FP16), stored as u16.
    pub fn f32_to_f16(src: &[f32]) -> Vec<u16>;

    /// FP16 (u16) -> f32 conversion.
    pub fn f16_to_f32(src: &[u16]) -> Vec<f32>;
}
```

### Example

```rust
use tala_embed::quantize::{f32_to_int8, int8_to_f32};

let original = vec![0.5, -0.3, 0.9, 0.0];
let (quantized, scale) = f32_to_int8(&original);
let recovered = int8_to_f32(&quantized, scale);

for (o, r) in original.iter().zip(recovered.iter()) {
    assert!((o - r).abs() < 0.01);
}
```

---

## HnswIndex

A Hierarchical Navigable Small World index for approximate nearest neighbor search. Uses L2 distance (via cached norms: `||a||^2 + ||b||^2 - 2*dot(a,b)`) internally and returns results sorted by L2 distance. All stored vectors are kept in `AlignedVec` for SIMD-friendly access.

The implementation uses generation-based visited tracking (a `Vec<u32>` with a monotonic generation counter) to avoid per-search `HashSet` allocation. Visited state resets in O(1) by incrementing the generation.

```rust
pub struct HnswIndex { /* private */ }

impl HnswIndex {
    /// Create a new index with the given dimensionality, connectivity `m`,
    /// and construction beam width `ef_construction`. Uses seed 42 for the RNG.
    pub fn new(dim: usize, m: usize, ef_construction: usize) -> Self;

    /// Create an index with an explicit RNG seed for deterministic builds.
    pub fn with_seed(dim: usize, m: usize, ef_construction: usize, seed: u64) -> Self;

    /// Insert a vector into the index. Returns the vector's internal index.
    pub fn insert(&mut self, vector: Vec<f32>) -> usize;

    /// Search for the `k` nearest neighbors of `query`.
    /// `ef` controls the search beam width (higher = more accurate, slower).
    /// Returns (internal_index, L2_distance) pairs sorted by distance ascending.
    pub fn search(&mut self, query: &[f32], k: usize, ef: usize) -> Vec<(usize, f32)>;

    /// Access a stored vector by its internal index.
    #[inline]
    pub fn get_vector(&self, idx: usize) -> &[f32];

    /// Number of vectors in the index.
    pub fn len(&self) -> usize;

    /// True if the index contains no vectors.
    pub fn is_empty(&self) -> bool;
}
```

### Parameters

| Parameter | Typical Value | Effect |
|-----------|--------------|--------|
| `m` | 16 | Maximum connections per node per layer. Higher values improve recall at the cost of memory and insert time. |
| `ef_construction` | 200 | Beam width during insertion. Higher values build a better graph but slow down construction. |
| `ef` (search) | 50 | Beam width during search. Must be >= `k`. Higher values improve recall at the cost of latency. |

### Example

```rust
use tala_embed::HnswIndex;

let dim = 4;
let mut index = HnswIndex::new(dim, 16, 200);

// Insert 100 vectors
for i in 0..100 {
    let v = vec![i as f32; dim];
    index.insert(v);
}

// Search for the 5 nearest neighbors of a query
let query = vec![50.0; dim];
let results = index.search(&query, 5, 50);

assert_eq!(results.len(), 5);
// Results are sorted by L2 distance ascending
for window in results.windows(2) {
    assert!(window[0].1 <= window[1].1);
}
```

### Performance

The HNSW index delivers sub-millisecond search latency at 10K vectors with `ef=50`. Measured benchmarks:

| Operation | Corpus Size | Time |
|-----------|-------------|------|
| Search (top-10, ef=50) | 10K vectors, dim=384 | 139 us |
| Semantic query (full pipeline) | 10K vectors, dim=384 | 151 us |
