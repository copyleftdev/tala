# Outcomes and Confidence

Every intent in TALA has an **outcome** — what actually happened when the action was executed.

## The Outcome Triple

```rust
struct Outcome {
    status: Status,      // Success, Failure, Partial, Pending
    latency_ns: u64,     // execution time in nanoseconds
    exit_code: i32,      // process exit code
}
```

Outcomes are attached asynchronously. An intent is created when the command is issued; the outcome is recorded when execution completes. This separation is important — it allows TALA to capture the intent even if the command never finishes.

## Status

| Status | Meaning |
|--------|---------|
| `Pending` | Command issued, outcome not yet recorded |
| `Success` | Completed with exit code 0 |
| `Failure` | Completed with non-zero exit code |
| `Partial` | Completed but with incomplete or degraded results |

## Confidence

Each intent carries a **confidence score** (`f32`, 0.0–1.0) representing the system's confidence in its classification and embedding. Factors that affect confidence:

- **Command complexity** — simple commands like `ls` have high confidence; ambiguous multi-pipe chains may have lower confidence
- **Context richness** — commands with clear context (known cwd, established session) score higher
- **Embedding quality** — commands with close semantic neighbors in the index score higher than novel, unseen patterns

Confidence propagates through the graph. An edge from a high-confidence node carries more weight than one from a low-confidence node.

## Why Outcomes Matter

Outcomes close the loop between intent and result. Without them, TALA would capture what you *tried* to do but not what *happened*. With outcomes:

- **Pattern detection** can identify commands that frequently fail in specific contexts
- **Adaptive replay** can skip steps that are known to succeed in the current state
- **Narrative summarization** can report success rates across a workflow
- **Root-cause analysis** can trace backward from a failure to the chain of intents that led to it
