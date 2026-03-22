# Intent vs. Command

Shell history records commands. TALA records intents. This distinction is not cosmetic -- it changes what the system can reason about.

## What a Command Captures

A command is a string submitted to a shell:

```
kubectl rollout restart deployment/api
```

This is what traditional `history` records. The information content is:

- The executable invoked (`kubectl`)
- The arguments passed (`rollout restart deployment/api`)
- The timestamp (if `HISTTIMEFORMAT` is set)

That is all. Everything else -- why the command was run, what state the system was in, what happened next, whether it worked -- is lost.

## What an Intent Captures

TALA records the same action as a structured intent node:

```rust
Intent {
    id: IntentId("a3f1c9..."),
    timestamp_ns: 1711234567890123456,
    command: "kubectl rollout restart deployment/api",
    context: Context {
        cwd: "/home/ops/infra",
        shell: "zsh",
        user: "ops",
        env_hash: 0x4a2b...,
        session_id: 7,
    },
    embedding: [0.0234, -0.1567, 0.3891, ...],  // 384 dimensions
    outcome: Outcome {
        status: Success,
        latency_ns: 2_340_000_000,
        exit_code: 0,
    },
}
```

Plus edges connecting this intent to other intents:

```rust
Edge { from: "a3f1c9...", to: "7b2e4a...", relation: Causal, weight: 0.87 }
Edge { from: "a3f1c9...", to: "d4f1a2...", relation: Temporal, weight: 1.0 }
Edge { from: "a3f1c9...", to: "e8c3b1...", relation: Retry, weight: 0.92 }
```

The information content is fundamentally richer:

- **Semantic embedding**: a vector representation of the command's meaning, enabling similarity search
- **Execution context**: working directory, shell, user, environment state
- **Outcome**: whether it succeeded, how long it took, what exit code it produced
- **Causal links**: what caused this action and what it caused
- **Relation types**: whether an edge represents causation, temporal sequence, dependency, retry, or branching

## The Same Command, Different Intents

A critical insight: the same command string can represent completely different intents depending on context. Consider `kubectl rollout restart deployment/api` in three scenarios.

### Scenario 1: Routine Deployment

The deployment pipeline runs on schedule. The restart is part of a normal rollout after a new image was pushed.

- **Context**: cwd is `/ci/deploy`, user is `ci-bot`, session is a CI pipeline
- **Causal parent**: `docker push api:v2.3.1` (the image build that triggered deployment)
- **Outcome**: Success, 12s latency
- **Embedding**: clusters with other routine deployment commands

### Scenario 2: Incident Response

An engineer restarts the API deployment at 3am because pods are OOMKilling.

- **Context**: cwd is `/home/ops`, user is `oncall-eng`, session is an SSH session
- **Causal parent**: `kubectl get pods -n production` (the diagnostic command that revealed the problem)
- **Outcome**: Success, 45s latency (cluster under memory pressure)
- **Embedding**: clusters with other incident response commands

### Scenario 3: Failed Retry

The same restart is attempted after a first attempt timed out. The cluster's API server is unreachable.

- **Context**: cwd is `/home/ops`, user is `oncall-eng`, same session as Scenario 2
- **Causal parent**: the previous failed restart attempt
- **Edge type to parent**: Retry (not Causal)
- **Outcome**: Failure, 30s latency, exit code 1
- **Embedding**: same vector as Scenario 2, but the intent graph structure differs

Traditional history records three identical lines. TALA records three distinct intent nodes with different contexts, different causal chains, different outcomes, and different positions in the narrative graph. The semantic recall system can distinguish between "routine deploys" and "emergency restarts" because the embeddings cluster differently based on surrounding context.

## What This Enables

### Semantic Recall

Search by meaning instead of string matching. "What did I do last time the API was OOMKilling?" retrieves the Scenario 2 intent cluster, not the Scenario 1 routine deploys, because the query embedding is closer to incident-response intents than to CI pipeline intents.

With `history | grep kubectl`, you get all three scenarios mixed together with no way to distinguish them.

### Adaptive Replay

Given a past workflow (a narrative subgraph), replay it in the current context. The replay engine traverses the causal edges from a root intent, finds all downstream intents, and generates a plan adjusted for the current state. This is only possible because the graph encodes which actions caused which other actions.

With history, you would need to manually identify which commands were related, determine their order, and hope the context has not changed.

### Pattern Detection

K-means clustering over intent embeddings reveals recurring patterns: "every Thursday at 2pm, there is a cluster of incident-response intents targeting the payment service." This detection operates over the semantic content (embeddings) and temporal structure (timestamps and causal edges), both of which are absent from traditional history.

### Failure Correlation

When an intent has a failure outcome, TALA can traverse its causal ancestors to identify what led to the failure, and traverse its causal descendants to identify what was affected by it. This is graph traversal over typed edges -- a fundamentally different operation than searching a flat log.

## The Representation Cost

Capturing intents instead of commands costs more storage per record. A command string is 50-200 bytes. An intent with a 384-dimensional embedding is approximately 1,700 bytes (1,536 for the embedding, plus metadata). This is a 10-30x increase per record.

The trade-off is justified for two reasons:

1. **Human interaction rates are low.** A busy engineer generates perhaps 500 commands per day. At 1,700 bytes each, that is 850 KB/day, 310 MB/year. Storage is not a constraint.

2. **The value of structured data compounds.** A flat history of 10,000 commands is marginally more useful than a history of 1,000 commands -- you can grep for more things. A narrative graph of 10,000 intents is qualitatively more useful than 1,000 -- the pattern detection, causal analysis, and replay capabilities improve as the graph grows and more connections emerge.

## Summary

| Property | Command (history) | Intent (TALA) |
|----------|-------------------|---------------|
| Data model | String | Structured node in a DAG |
| Context | None | Working directory, shell, user, environment |
| Meaning | Literal text | Embedding vector in semantic space |
| Outcome | Not recorded | Status, latency, exit code |
| Relations | Sequential order | Causal, temporal, dependency, retry, branch |
| Search | Regex over strings | Nearest-neighbor in embedding space |
| Replay | Manual copy-paste | Automated, context-aware |
| Pattern detection | Not possible | Clustering over embeddings and graph structure |
