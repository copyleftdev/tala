# Adaptive Replay

Adaptive replay is TALA's ability to re-execute a sequence of intents, adapted to the current context.

## Beyond Scripts

A shell script is a static sequence of commands. If the environment changes — different hostnames, different paths, different versions — the script breaks. You edit it manually and try again.

TALA's replay engine works differently. It operates on *intents*, not commands. It understands the dependency structure, can substitute variables, and can skip steps that have already been completed.

## How Replay Works

1. **Select a narrative** — choose a root intent and extract its forward narrative from the graph
2. **Build a plan** — topologically sort the intents by their dependency edges (Kahn's algorithm)
3. **Substitute variables** — apply context-specific variable replacements (`${HOST}`, `${VERSION}`, etc.)
4. **Filter completed** — skip intents whose outcomes are already satisfied in the current state
5. **Execute** — run each step in dependency order, recording new outcomes

```rust
// Build a replay plan
let plan = build_plan(&graph, &intent_ids, &commands)?;

// Configure replay
let config = ReplayConfig {
    vars: HashMap::from([("HOST", "web-04.prod"), ("VERSION", "2.1.0")]),
    completed: HashSet::new(),
    dry_run: false,
};

// Execute
let results = engine.execute(&graph, &intent_ids, &commands)?;
```

## Dry Run

Before executing, you can preview the plan:

```rust
let steps = engine.dry_run(&graph, &intent_ids, &commands)?;
for step in &steps {
    println!("{}: {} (deps: {:?})", step.intent_id, step.command, step.deps);
}
```

This shows the exact sequence that would be executed, with dependencies resolved and variables substituted, without running anything.

## Idempotency

The replay engine supports idempotent re-execution. If you provide a set of already-completed intent IDs, those steps are skipped:

```rust
let remaining = filter_completed(plan, &completed_ids);
```

This makes replay safe to re-run. If a replay fails halfway through, you can re-run it and only the incomplete steps will execute.

## Why This Matters

Traditional approaches to repeatable operations — scripts, runbooks, playbooks — are static artifacts that drift from reality. TALA's replay is derived from *what actually worked*, adapted to *what's true now*. The replay plan is a living thing, generated from the narrative graph, not a file someone wrote six months ago.
