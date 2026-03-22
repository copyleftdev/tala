# Running Benchmarks

TALA uses [Criterion](https://bheisler.github.io/criterion.rs/book/) for all benchmarks. Each crate that contains performance-critical code has a benchmark suite in its `benches/` directory. Criterion produces stable, statistically rigorous measurements with HTML reports.

## Prerequisites

- Rust toolchain as pinned in `rust-toolchain.toml`
- A machine with AVX2 support for SIMD benchmarks (most x86-64 CPUs from 2013+)
- Close other workloads to reduce measurement noise

## Running All Benchmarks

From the workspace root:

```bash
cargo bench
```

This runs every benchmark suite across all crates. Expect 2-5 minutes depending on hardware. Results are printed to the terminal and saved as HTML reports.

## Per-Crate Benchmarks

Run benchmarks for a single crate:

```bash
# Embedding engine (cosine, batch, HNSW)
cargo bench --bench embed_bench

# Wire format (columnar, CSR, Bloom, segment)
cargo bench --bench wire_bench

# Graph engine
cargo bench --bench graph_bench

# Storage engine (WAL, hot buffer, query, ingest pipeline)
cargo bench --bench store_bench
```

## Filtering Benchmarks

Criterion accepts a filter argument to run a subset of benchmarks by name:

```bash
# Only cosine similarity benchmarks
cargo bench --bench embed_bench -- cosine

# Only HNSW benchmarks
cargo bench --bench embed_bench -- hnsw

# Only WAL benchmarks
cargo bench --bench store_bench -- wal
```

## HTML Reports

Criterion generates HTML reports in `target/criterion/`. After running benchmarks:

```bash
# Open the report index
open target/criterion/report/index.html
```

Each benchmark group has its own page with:

- Time distribution plot (violin or PDF)
- Iteration time scatter plot
- Comparison against the previous run (if available)
- Statistical summary (mean, median, standard deviation, confidence interval)

## Interpreting Results

### Terminal Output

Criterion prints a summary for each benchmark:

```
cosine_similarity/avx2/384
                        time:   [39.2 ns 39.6 ns 40.1 ns]
                        change: [-0.5% +0.3% +1.1%] (p = 0.42 > 0.05)
                        No change in performance detected.
```

- The three values in brackets are the lower bound, point estimate, and upper bound of the 95% confidence interval.
- The `change` line compares against the previous run. If the change is statistically significant (p < 0.05), Criterion reports "Performance has regressed" or "Performance has improved."

### Comparing Against Targets

After running benchmarks, compare measured values against the targets in [Benchmark Targets](./targets.md):

| Operation | Target | What to Check |
|-----------|--------|---------------|
| Cosine similarity | < 20ns | `cosine_similarity/avx2/384` |
| Batch cosine 1K | < 50us | `batch_cosine/1000` |
| Batch cosine 100K | < 5ms | `batch_cosine_parallel/100000` |
| HNSW search | < 1ms | `hnsw_search/10000` |
| Semantic query | < 50ms | `semantic_query/10000` |

Any passing target that regresses beyond its threshold is a merge blocker.

## Stable Measurements

For reliable results:

1. **Disable CPU frequency scaling**: set the governor to `performance` mode.
   ```bash
   sudo cpupower frequency-set -g performance
   ```

2. **Pin to a single NUMA node** (multi-socket systems):
   ```bash
   numactl --cpunodebind=0 --membind=0 cargo bench
   ```

3. **Warm up**: Criterion automatically runs warmup iterations. The default configuration is sufficient for most benchmarks.

4. **Multiple runs**: if a result is borderline, run the benchmark three times and take the median.

## Adding New Benchmarks

Benchmarks live in `crates/<name>/benches/<name>_bench.rs`. Follow this structure:

```rust
use criterion::{criterion_group, criterion_main, Criterion};

fn bench_operation(c: &mut Criterion) {
    let mut group = c.benchmark_group("operation_name");
    // setup
    group.bench_function("variant", |b| {
        b.iter(|| {
            // code under measurement
        })
    });
    group.finish();
}

criterion_group!(benches, bench_operation);
criterion_main!(benches);
```

Register the benchmark in the crate's `Cargo.toml`:

```toml
[[bench]]
name = "<name>_bench"
harness = false
```

Use `harness = false` to let Criterion control the benchmark harness. Use `Throughput` annotations for operations measured in elements per second.
