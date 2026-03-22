# Rust Conventions

These conventions apply to all Rust code in the `crates/` directory. They are enforced by review and, where possible, by compiler checks.

## Workspace Dependencies

All shared dependency versions are declared in the workspace root `Cargo.toml` under `[workspace.dependencies]`:

```toml
[workspace.dependencies]
uuid = { version = "1", features = ["v4"] }
thiserror = "2"
rand = { version = "0.8", features = ["small_rng"] }
rayon = "1"
criterion = { version = "0.5", features = ["html_reports"] }
```

Individual crates reference these with `{ workspace = true }`:

```toml
[dependencies]
uuid = { workspace = true }

[dev-dependencies]
criterion = { workspace = true }
```

Never specify a version directly in a crate's `Cargo.toml` if the dependency exists in the workspace table. This prevents version skew across crates.

## `#[repr(C)]` Rules

Apply `#[repr(C)]` to every struct that crosses an FFI boundary, is memory-mapped from disk, or is serialized by reinterpreting raw bytes:

```rust
#[repr(C)]
pub struct TbfHeader {
    pub magic: u32,
    pub version_major: u16,
    pub version_minor: u16,
    pub segment_id: u64,
    // ...
}
```

Use `#[repr(C, packed)]` only when padding must be eliminated (e.g., wire protocol headers where every byte position is specified). Packed structs require unaligned access, so reads must go through `ptr::read_unaligned`.

Do not use `#[repr(C)]` on purely internal structs that never leave Rust's type system. Let the compiler optimize their layout.

## Unsafe Code and SAFETY Comments

Every `unsafe` block requires a `// SAFETY:` comment on the line immediately above it. The comment must explain the invariant that makes the code sound -- not what the code does, but why it is safe to do it:

```rust
// SAFETY: `AlignedVec` guarantees 64-byte alignment and `a.len() == b.len()`
// is checked by the caller. The pointer arithmetic stays within the allocation.
unsafe {
    let va = _mm256_load_ps(a.as_ptr().add(i));
    let vb = _mm256_load_ps(b.as_ptr().add(i));
    // ...
}
```

Bad SAFETY comments (do not do this):

```rust
// SAFETY: we need to call this intrinsic
unsafe { ... }
```

If you cannot articulate why an `unsafe` block is sound, it probably is not.

## SIMD Architecture Gating

SIMD code lives in architecture-gated modules:

```rust
#[cfg(target_arch = "x86_64")]
pub mod avx2 {
    use std::arch::x86_64::*;

    #[target_feature(enable = "avx2,fma")]
    pub unsafe fn cosine_similarity(a: &[f32], b: &[f32]) -> f32 {
        // ...
    }
}

#[cfg(target_arch = "aarch64")]
pub mod neon {
    // ...
}

pub mod scalar {
    pub fn cosine_similarity(a: &[f32], b: &[f32]) -> f32 {
        // Always available, always correct
    }
}
```

A scalar fallback MUST exist for every SIMD function. The scalar implementation is the correctness reference -- SIMD variants are tested against it.

Use `#[target_feature(enable = "avx2,fma")]` on SIMD functions. This allows the compiler to emit AVX2+FMA instructions within the function body even when the global target does not include them. The function must be called through a function pointer or via `is_x86_feature_detected!` guard.

Use `_mm256_loadu_ps` (unaligned load) by default. Use `_mm256_load_ps` (aligned load) only when alignment is guaranteed by `AlignedVec` or TBF segment layout.

## Inlining

Apply `#[inline]` to functions called in hot loops:

- Cosine similarity and other vector operations
- Columnar accessor functions (timestamp lookup, embedding read)
- Hash functions in Bloom filter probes
- CSR row pointer lookups

Do not apply `#[inline]` to cold paths (error handling, configuration parsing, initialization). Unnecessary inlining increases binary size and instruction cache pressure.

Use `#[inline(always)]` only when benchmarks prove it matters. In most cases, `#[inline]` (which is a hint, not a directive) is sufficient.

## No `unwrap()` in Library Code

Library code (`src/lib.rs` and all modules it includes) must not use `unwrap()` or `expect()`. Use the `?` operator to propagate errors, or return `Result`:

```rust
// Good
pub fn get_intent(&self, id: IntentId) -> Result<&Intent, TalaError> {
    self.index.get(&id).ok_or(TalaError::NotFound(id))
}

// Bad -- panics on missing intent
pub fn get_intent(&self, id: IntentId) -> &Intent {
    self.index.get(&id).unwrap()
}
```

`unwrap()` and `expect()` are acceptable in:

- Benchmark code (`benches/*.rs`)
- Test code (`#[cfg(test)]` modules, `tests/*.rs`)
- Static initialization where failure means the program cannot run (e.g., regex compilation of a constant pattern)

## Error Handling

All errors flow through the `TalaError` enum defined in `tala-core`. Crate-specific error types use `#[from]` with `thiserror` for automatic conversion:

```rust
#[derive(Debug, thiserror::Error)]
pub enum TalaError {
    #[error("intent not found: {0}")]
    NotFound(IntentId),

    #[error("WAL write failed: {0}")]
    WalError(#[from] std::io::Error),

    #[error("segment corrupt: {0}")]
    SegmentCorrupt(String),
    // ...
}
```

Do not define standalone error enums in individual crates. Add variants to `TalaError` or use `#[from]` to convert crate-local errors into `TalaError`.

## Trait Definitions

Trait definitions live in `tala-core` under `src/traits/`. Implementations live in the crate that owns the concern:

```
tala-core/src/traits/store.rs    -> defines IntentStore trait
tala-store/src/lib.rs            -> implements IntentStore for StorageEngine
```

Public APIs between sibling crates use trait bounds, never concrete types from the sibling:

```rust
// Good: depends on trait from tala-core
pub fn query<S: IntentStore>(store: &S, embedding: &[f32]) -> Vec<IntentId> { ... }

// Bad: depends on concrete type from tala-store
pub fn query(store: &tala_store::StorageEngine, embedding: &[f32]) -> Vec<IntentId> { ... }
```

## Benchmarks

Benchmark files live in `crates/<name>/benches/`, not in `tests/`. Use Criterion with `harness = false`:

```toml
[[bench]]
name = "embed_bench"
harness = false
```

Benchmark groups should be named descriptively (`cosine_similarity`, `hnsw_search`, `wal_append`) and use `BenchmarkId` for parameterized variants:

```rust
group.bench_with_input(BenchmarkId::new("avx2", dim), &dim, |b, _| {
    b.iter(|| unsafe { avx2::cosine_similarity(&a, &b) })
});
```

## Feature Flags

Optional capabilities use feature flags:

| Feature | Purpose |
|---------|---------|
| `cuda` | CUDA GPU acceleration for batch similarity |
| `vulkan` | Vulkan compute shader acceleration |
| `cluster` | Distributed mode (Raft, SWIM, QUIC) |
| `encryption` | AES-256-GCM encryption for TBF segments |

Default features cover the common single-node case. GPU and cluster features are opt-in:

```toml
[features]
default = []
cuda = ["dep:cuda-runtime-sys"]
cluster = ["dep:tala-net"]
```

## Naming Conventions

- Type names: PascalCase (`IntentId`, `BloomFilter`, `CsrIndex`)
- Field names: snake_case (`node_count`, `embedding_dim`, `created_at`)
- Function names: snake_case (`cosine_similarity`, `flush_segment`)
- Constants: SCREAMING_SNAKE_CASE (`BUCKET_BOUNDS`, `TBF_MAGIC`)
- Module names: snake_case (`src/traits/store.rs`, `src/scalar.rs`)
