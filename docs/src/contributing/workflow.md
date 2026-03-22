# Development Workflow

TALA follows a spec-first, benchmark-driven development process. Code changes are validated against both compilation and performance. Spec changes are validated against structural rules and cross-reference integrity.

## Modifying Crate Code

Every code change follows this sequence:

1. **Read the relevant spec.** The crate-to-spec mapping:

   | Crate | Governing Spec |
   |-------|---------------|
   | tala-core | spec-03 (crate layout) |
   | tala-wire | spec-01 (binary format) |
   | tala-embed | spec-02 (embedding/SIMD) |
   | tala-graph | spec-03 (crate layout) |
   | tala-store | spec-03 (crate layout) |
   | tala-net | spec-04 (distributed) |

2. **Make the change.**

3. **Verify compilation across the workspace:**

   ```bash
   cargo check --workspace
   ```

   This catches type errors, missing imports, and dependency issues across all crates. It is significantly faster than `cargo build` because it skips code generation.

4. **Run tests:**

   ```bash
   cargo test --workspace
   ```

5. **Run the crate's benchmarks to check for regressions:**

   ```bash
   cargo bench --bench <crate>_bench
   ```

   Criterion will report whether performance has changed relative to the previous run. Any statistically significant regression on a passing target (see [Benchmark Targets](../performance/targets.md)) is a merge blocker.

6. **Run the full benchmark suite if the change touches hot-path code:**

   ```bash
   cargo bench
   ```

## Modifying a Spec

Specs have a dependency graph. Changes to a lower-numbered spec can break higher-numbered specs that reference it.

1. **Read the spec being modified** and all specs that depend on it:

   ```
   spec-01 (Binary Format)     <- standalone
   spec-02 (Embedding/SIMD)    <- standalone
   spec-03 (Crate Layout)      <- depends on spec-01, spec-02
   spec-04 (Distributed)       <- depends on spec-01, spec-03
   ```

   Modifying spec-01 requires reading spec-03 and spec-04. Modifying spec-02 requires reading spec-03. Modifying spec-03 requires reading spec-04.

2. **Make the change** following the spec writing rules:
   - RFC 2119 language for requirements (MUST, SHOULD, MAY)
   - Rust struct notation with `#[repr(C)]` where applicable
   - Quantitative performance targets with hardware assumptions
   - No TBD, TODO, or placeholders
   - Cross-references use `spec-NN` format

3. **Check downstream specs** for broken references. Search for `spec-NN` (where NN is the modified spec number) in all higher-numbered specs. Verify that every reference still points to a valid section.

4. **Verify the invariant**: lower-numbered specs MUST NOT reference higher-numbered specs. If you find yourself wanting spec-01 to reference spec-03, the dependency is inverted and the design needs rethinking.

## Adding a New Crate

1. **Identify where the crate sits in the dependency graph:**

   ```
                       tala-core
                      /    |    \
                tala-wire  tala-embed  (no inter-dep)
                   |    \    /   |
                   |  tala-store |
                   |      |      |
              tala-graph  |  tala-intent
                /    \    |    /
         tala-weave  tala-kai
                   \   |   /
                 tala-net
                     |
               tala-daemon
                     |
                tala-cli
   ```

2. **Create the crate:**

   ```bash
   cargo init --lib crates/tala-<name>
   ```

3. **Configure `Cargo.toml`:**

   ```toml
   [package]
   name = "tala-<name>"
   version.workspace = true
   edition.workspace = true
   license.workspace = true
   rust-version.workspace = true

   [dependencies]
   tala-core = { path = "../tala-core" }
   # Add other dependencies using workspace versions:
   # uuid = { workspace = true }
   ```

4. **Add to the workspace root `Cargo.toml`:**

   ```toml
   [workspace]
   members = [
       # ...existing crates...
       "crates/tala-<name>",
   ]
   ```

5. **Add a benchmark suite** if the crate contains performance-critical code:

   ```bash
   mkdir crates/tala-<name>/benches
   ```

   Create `crates/tala-<name>/benches/<name>_bench.rs` with Criterion benchmarks and register it in `Cargo.toml` with `harness = false`.

6. **Verify the workspace builds:**

   ```bash
   cargo check --workspace
   ```

## Adding a New Spec

1. Identify the gap: what concept is not yet covered by specs 01 through 04?
2. Determine which specs the new one depends on. The new spec number MUST be higher than all its dependencies.
3. Draft the spec following the structure: Overview, Data Structures, Requirements, Performance Targets.
4. Verify no circular references: lower specs MUST NOT reference higher specs.
5. Update `CLAUDE.md` with the new spec in the Spec Suite table and dependency graph.

## Review Checklist

Before merging any change:

- [ ] `cargo check --workspace` passes
- [ ] `cargo test --workspace` passes
- [ ] No performance regressions on passing benchmark targets
- [ ] Spec cross-references are valid (if specs were modified)
- [ ] No `unwrap()` or `expect()` in library code
- [ ] `unsafe` blocks have `// SAFETY:` comments
- [ ] New public types follow naming conventions (PascalCase types, snake_case fields)
