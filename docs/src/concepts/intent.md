# Intent as a Primitive

In traditional systems, the shell records commands. TALA records **intent**.

## The Difference

A command is a string: `kubectl rollout restart deployment/api-gateway -n production`. It tells you *what* was typed. It doesn't tell you *why*. Was this a routine deployment? A rollback after a failed canary? A panic restart during an incident at 2am? The string is identical in all three cases.

An intent is a structured representation of the desired outcome behind that command:

```rust
struct Intent {
    id: IntentId,
    timestamp: u64,           // nanosecond epoch
    raw_command: String,      // the original command
    embedding: Vec<f32>,      // 384-dim semantic vector
    context_hash: u64,        // hashed execution environment
    parent_ids: Vec<IntentId>,// causal predecessors
    outcome: Option<Outcome>, // what happened
    confidence: f32,          // system's confidence in classification
}
```

## The Formula

TALA models intent as a function:

```
I = f(C, X, P)
```

| Symbol | Meaning |
|--------|---------|
| **C** | The command or input signal |
| **X** | Context — working directory, environment, shell, user, session |
| **P** | Prior knowledge — historical embeddings from semantically similar past actions |

The same command in different contexts produces different intents. `systemctl restart nginx` during a deployment and during an incident are different intents with different causal chains, different expected outcomes, and different semantic neighborhoods.

## The Extraction Pipeline

When a raw command enters TALA, it passes through the intent extraction pipeline:

```
Raw command
  → Tokenization (command, args, flags, pipes, redirects)
  → Embedding (384-dimensional L2-normalized vector)
  → Classification (Build, Deploy, Debug, Configure, Query, Navigate)
  → Context attachment (cwd, user, shell, session, env hash)
  → Intent node creation
```

The embedding captures *meaning*, not syntax. Commands that do similar things — even with completely different syntax — land near each other in the embedding space. This is what enables semantic recall: searching by what you meant, not what you typed.

## Why This Matters

When intent is a first-class primitive:

- The system can **connect** actions that are causally related, even if they happened minutes or hours apart
- The system can **recall** past actions by meaning, not string matching
- The system can **replay** sequences of intent, adapting commands to changed context
- The system can **predict** what you're likely to do next based on the semantic trajectory of your session
- The system can **learn** — detecting patterns, clustering similar workflows, surfacing insights

None of this is possible with a flat list of strings.
