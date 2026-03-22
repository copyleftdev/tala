# Building from Source

## Toolchain

TALA pins its Rust toolchain via `rust-toolchain.toml` at the workspace root. Running any `cargo` command in the workspace will automatically install the correct version if you use `rustup`.

```bash
cat rust-toolchain.toml
```

The minimum supported Rust version is **1.82**.

## Workspace Build

```bash
cargo build --workspace
```

This builds all 12 crates in dependency order. For a release-optimized build:

```bash
cargo build --release
```

## Building Individual Crates

Each crate can be built independently:

```bash
cargo build -p tala-core
cargo build -p tala-embed
cargo build -p tala-sim
```

## The Simulator Binary

The `tala-sim` crate produces the only binary in the workspace:

```bash
cargo build --release --bin tala-sim
```

The binary is written to `target/release/tala-sim`.

## Feature Flags

Optional capabilities are gated behind feature flags:

| Flag | Crate | Description |
|------|-------|-------------|
| `cuda` | tala-embed | CUDA GPU acceleration for batch operations |
| `vulkan` | tala-embed | Vulkan compute shader support |
| `cluster` | tala-net | Distributed clustering features |
| `encryption` | tala-net | TLS encryption for mesh transport |

Default features cover the common single-node case. Enable extras with:

```bash
cargo build -p tala-embed --features cuda
```

## Docker Build

The `deploy/Dockerfile` produces a minimal container image:

```bash
docker build -t tala-sim -f deploy/Dockerfile .
```

The multi-stage build compiles in a full Rust image and copies only the binary into a slim Debian base.

## Verifying the Build

```bash
cargo test --workspace
cargo clippy --workspace -- -D warnings
```

All library code is warning-free under `clippy`. Tests validate serialization roundtrips, graph invariants, and storage durability.
