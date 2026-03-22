# Chaos Engineering

TALA includes a built-in chaos engine that injects faults into running simulators. The purpose is to validate that the narrative graph, HNSW index, WAL, and storage engine handle degraded conditions gracefully, and to generate realistic failure data for the Observatory dashboard.

## The ChaosEngine

The chaos engine runs in a dedicated thread per simulator. On each cycle it sleeps for `TALA_CHAOS_INTERVAL_S` seconds (default: 60), then rolls against `TALA_CHAOS_PROBABILITY` (default: 0.15). If the roll succeeds, a weighted random selection determines which fault to inject.

```
sleep(interval) -> roll < probability? -> select event -> execute
                         |
                         no -> loop
```

## Event Types and Weights

| Event | Weight | Description |
|-------|--------|-------------|
| ForcedFailure | 30% | Ingests a valid command, then attaches a forced failure outcome (exit code 137, latency 999ms). Tests that the narrative graph correctly records and connects failure nodes. |
| LatencySpike | 25% | Sleeps for `rate_ms * multiplier` (multiplier randomly chosen from 2-9). Simulates a stalled upstream dependency. The chaos thread itself blocks, modeling backpressure. |
| RetryStorm | 20% | Ingests the same command 3-14 times in rapid succession. Tests HNSW behavior under duplicate vectors and validates that the graph forms Retry-type edges between repeated intents. |
| IngestFailure | 10% | Attempts to ingest an empty command string, which is expected to fail. Validates that the ingest pipeline rejects malformed input without corrupting state. |
| DegenerateQuery | 10% | Queries the HNSW index with a zero vector (all dimensions 0.0). Tests that similarity search handles degenerate inputs without panicking or returning nonsensical results. |
| InsightStress | 5% | Runs insight generation with k=50 clusters (far above the normal k=3). Stress-tests the k-means implementation with an oversized k relative to the current corpus. |

## What Each Event Does

### ForcedFailure

The chaos thread generates a command from the vertical's workload generator, ingests it through the normal pipeline (extract, WAL, HNSW, edge formation, hot buffer), then overwrites the outcome with:

```rust
Outcome {
    status: Status::Failure,
    latency_ns: 999_999_999,
    exit_code: 137,
}
```

This creates a real intent node in the narrative graph with a failure outcome. The edge formation system will create causal edges from this node, and subsequent insight runs will detect it as part of a failure cluster.

### LatencySpike

The chaos thread sleeps for an extended duration, blocking itself. The multiplier is randomly chosen between 2 and 9, so with a default rate of 300ms, a spike lasts 600ms to 2.7 seconds. Since the chaos thread is separate from the ingest thread, normal ingestion continues unaffected. The spike is visible in the chaos latency counter (`tala_chaos_latency_spikes`).

### RetryStorm

The same command and context are ingested between 3 and 14 times in a tight loop with no sleep between iterations. This floods the HNSW index with near-identical vectors (the embedding of the same command produces the same vector each time) and tests:

- Hot buffer behavior under burst writes
- WAL throughput under sequential append pressure
- Edge formation's ability to detect and label retries

### IngestFailure

An empty command is submitted with a synthetic context (`cwd: /chaos`, `user: chaos`). The ingest pipeline should reject this at the extraction stage. The chaos engine expects and silently handles the error. This validates that error paths do not corrupt the WAL, HNSW index, or hot buffer.

### DegenerateQuery

A zero vector (all 0.0 values) is submitted as a semantic query for the top 10 results. A well-implemented cosine similarity function returns 0.0 for any comparison against the zero vector (since the norm is zero). HNSW should handle this without division-by-zero panics.

### InsightStress

The insight system runs k-means clustering with k=50 instead of the normal k=3. When the corpus has fewer than 50 intents, some clusters will be empty. When the corpus is large, this exercises the full k-means convergence loop and memory allocation for 50 centroids of dimension 384.

## Metrics

The chaos engine exposes the following Prometheus counters, all labeled by `vertical`:

| Metric | Description |
|--------|-------------|
| `tala_chaos_events_total` | Total chaos events triggered (all types) |
| `tala_chaos_failures_injected` | ForcedFailure events |
| `tala_chaos_latency_spikes` | LatencySpike events |
| `tala_chaos_retries_injected` | RetryStorm events |

IngestFailure, DegenerateQuery, and InsightStress increment only `tala_chaos_events_total`.

## Observatory Detection

The Observatory dashboard detects active chaos by watching the rate of `tala_chaos_events_total` per vertical. When the rate exceeds zero, it classifies the chaos mode based on which sub-counters are moving:

| Condition | Displayed Mode |
|-----------|----------------|
| Only `failures_injected` rising | Failure Injection |
| Only `latency_spikes` rising | Latency Storm |
| Only `retries_injected` rising | Retry Cascade |
| Multiple counters rising | Mixed Chaos |
| Very high total event rate | Stampede |

Affected domain nodes pulse visually and edges show distortion to make chaos immediately visible in the topology.

## Tuning Chaos

### Disable Entirely

```bash
TALA_CHAOS_PROBABILITY=0.0
```

The chaos thread still runs but never triggers. Negligible overhead.

### High-Frequency Stress

```bash
TALA_CHAOS_INTERVAL_S=5
TALA_CHAOS_PROBABILITY=0.8
```

Fires a chaos event roughly every 6 seconds. Useful for stress-testing the storage engine and observing how the narrative graph handles sustained fault injection.

### Failure-Heavy Profile

Chaos weights are compiled into the `ChaosEngine`. To change the distribution, modify the thresholds in `crates/tala-sim/src/chaos.rs` in the `maybe_trigger` method. The current boundaries:

```
0.00 - 0.30  ForcedFailure    (30%)
0.30 - 0.55  LatencySpike     (25%)
0.55 - 0.75  RetryStorm       (20%)
0.75 - 0.85  IngestFailure    (10%)
0.85 - 0.95  DegenerateQuery  (10%)
0.95 - 1.00  InsightStress    (5%)
```

Adjust the threshold constants and rebuild to shift the distribution.
