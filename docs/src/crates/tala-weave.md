# tala-weave

The adaptive replay engine. Takes a narrative graph (or subgraph) and produces a topologically sorted replay plan, applies environment variable substitutions, checks idempotency against completed steps, and orchestrates execution through a caller-supplied executor closure.

## Key Types

| Type | Description |
|------|-------------|
| `ReplayEngine` | Orchestrates the full replay pipeline: plan, transform, idempotency, execute |
| `ReplayConfig` | Configuration for a replay run: variables, completed set, dry-run flag |
| `Executor` | Type alias for the executor closure: `Box<dyn FnMut(&str) -> Outcome>` |

## Key Functions

| Function | Description |
|----------|-------------|
| `build_plan(graph, intent_ids, commands)` | Topological sort via Kahn's algorithm |
| `substitute_vars(input, vars)` | Replace `${VAR}` patterns in a command string |
| `filter_completed(steps, completed)` | Remove already-completed steps from a plan |

---

## build_plan

Produces a topologically sorted replay plan from a set of intent IDs and the edges between them in the narrative graph. Uses Kahn's algorithm. Returns `TalaError::CycleDetected` if the subgraph contains a cycle.

```rust
/// Build a topologically sorted replay plan.
///
/// Only considers edges between nodes in `intent_ids`. Each step's `deps`
/// field lists its predecessors within the subgraph.
pub fn build_plan(
    graph: &NarrativeGraph,
    intent_ids: &[IntentId],
    commands: &HashMap<IntentId, String>,
) -> Result<Vec<ReplayStep>, TalaError>;
```

The returned `Vec<ReplayStep>` is ordered such that no step appears before its dependencies. Each `ReplayStep` contains the intent ID, the command string, and a list of dependency IDs.

### Example

```rust
use std::collections::HashMap;
use tala_core::{IntentId, RelationType};
use tala_graph::NarrativeGraph;
use tala_weave::build_plan;

let mut graph = NarrativeGraph::new();
let a = IntentId::random();
let b = IntentId::random();
let c = IntentId::random();

graph.insert_node(a, 1, 1.0);
graph.insert_node(b, 2, 1.0);
graph.insert_node(c, 3, 1.0);
graph.add_edge(a, b, RelationType::Causal, 1.0);
graph.add_edge(b, c, RelationType::Causal, 1.0);

let mut commands = HashMap::new();
commands.insert(a, "echo A".to_string());
commands.insert(b, "echo B".to_string());
commands.insert(c, "echo C".to_string());

let plan = build_plan(&graph, &[a, b, c], &commands).unwrap();
assert_eq!(plan.len(), 3);
// A appears before B, B before C
let pos: HashMap<_, _> = plan.iter().enumerate()
    .map(|(i, s)| (s.intent_id, i)).collect();
assert!(pos[&a] < pos[&b]);
assert!(pos[&b] < pos[&c]);
```

### Cycle Detection

```rust
use tala_core::{IntentId, RelationType, TalaError};
use tala_graph::NarrativeGraph;

let mut graph = NarrativeGraph::new();
let a = IntentId::random();
let b = IntentId::random();
graph.insert_node(a, 1, 1.0);
graph.insert_node(b, 2, 1.0);
graph.add_edge(a, b, RelationType::Causal, 1.0);
graph.add_edge(b, a, RelationType::Causal, 1.0); // cycle

let result = build_plan(&graph, &[a, b], &HashMap::new());
assert!(matches!(result, Err(TalaError::CycleDetected)));
```

---

## substitute_vars

Replaces `${VAR}` patterns in a string with values from a lookup map. Unknown variables are left as-is. Handles edge cases: adjacent variables, unclosed braces, empty variable names.

```rust
/// Replace `${VAR}` patterns in `input` with values from `vars`.
/// Unknown variables are left intact.
pub fn substitute_vars(input: &str, vars: &HashMap<String, String>) -> String;
```

### Examples

```rust
use std::collections::HashMap;
use tala_weave::substitute_vars;

let mut vars = HashMap::new();
vars.insert("HOME".into(), "/home/user".into());
vars.insert("ENV".into(), "prod".into());

assert_eq!(
    substitute_vars("cd ${HOME} && deploy --env=${ENV}", &vars),
    "cd /home/user && deploy --env=prod"
);

// Unknown variables are left as-is
assert_eq!(
    substitute_vars("echo ${UNKNOWN}", &HashMap::new()),
    "echo ${UNKNOWN}"
);

// Adjacent variables
vars.insert("A".into(), "foo".into());
vars.insert("B".into(), "bar".into());
assert_eq!(substitute_vars("${A}${B}", &vars), "foobar");
```

---

## filter_completed

Removes steps whose `intent_id` appears in the `completed` set. Used for idempotent replay: if a step has already been executed successfully, it is skipped.

```rust
/// Filter out steps whose `intent_id` appears in `completed`.
pub fn filter_completed(
    steps: Vec<ReplayStep>,
    completed: &HashSet<IntentId>,
) -> Vec<ReplayStep>;
```

---

## Executor

The type alias for executor closures. An executor receives a transformed command string and returns an `Outcome`.

```rust
pub type Executor = Box<dyn FnMut(&str) -> Outcome>;
```

---

## ReplayConfig

Configuration for a replay run.

```rust
pub struct ReplayConfig {
    /// Environment variable substitutions to apply to commands.
    pub vars: HashMap<String, String>,
    /// Set of intent IDs already completed (for idempotency).
    pub completed: HashSet<IntentId>,
    /// If true, skip execution and return the plan with Pending outcomes.
    pub dry_run: bool,
}

impl Default for ReplayConfig {
    fn default() -> Self {
        Self {
            vars: HashMap::new(),
            completed: HashSet::new(),
            dry_run: true,
        }
    }
}
```

---

## ReplayEngine

The replay orchestrator. Combines plan building, variable substitution, idempotency checking, and execution into a single engine.

```rust
pub struct ReplayEngine { /* private */ }

impl ReplayEngine {
    /// Create a new replay engine with the given configuration.
    pub fn new(config: ReplayConfig) -> Self;

    /// Set the executor closure for live replay. Consumes and returns self
    /// for builder-style chaining.
    pub fn with_executor(self, executor: Executor) -> Self;

    /// Produce a dry-run plan: ordered, transformed steps with completed
    /// steps removed. Does not execute anything.
    pub fn dry_run(
        &self,
        graph: &NarrativeGraph,
        intent_ids: &[IntentId],
        commands: &HashMap<IntentId, String>,
    ) -> Result<Vec<ReplayStep>, TalaError>;

    /// Execute the replay. Returns a `ReplayResult` for every step in the
    /// plan (including skipped ones).
    ///
    /// Behavior per step:
    /// - If the step's ID is in `completed`: skipped=true, Status::Success, latency=0
    /// - If `dry_run` is set: skipped=false, Status::Pending, latency=0
    /// - Otherwise: applies variable substitution, calls the executor, returns outcome
    pub fn execute(
        &mut self,
        graph: &NarrativeGraph,
        intent_ids: &[IntentId],
        commands: &HashMap<IntentId, String>,
    ) -> Result<Vec<ReplayResult>, TalaError>;
}
```

### Example: Dry Run

```rust
use std::collections::{HashMap, HashSet};
use tala_core::{IntentId, RelationType, Status};
use tala_graph::NarrativeGraph;
use tala_weave::{ReplayConfig, ReplayEngine};

let mut graph = NarrativeGraph::new();
let a = IntentId::random();
graph.insert_node(a, 1, 1.0);

let mut commands = HashMap::new();
commands.insert(a, "deploy --env=${ENV}".to_string());

let mut vars = HashMap::new();
vars.insert("ENV".to_string(), "staging".to_string());

let config = ReplayConfig {
    vars,
    completed: HashSet::new(),
    dry_run: true,
};

let engine = ReplayEngine::new(config);
let plan = engine.dry_run(&graph, &[a], &commands).unwrap();

assert_eq!(plan.len(), 1);
assert_eq!(plan[0].command, "deploy --env=staging");
```

### Example: Live Execution

```rust
use std::collections::{HashMap, HashSet};
use tala_core::{IntentId, Outcome, RelationType, Status};
use tala_graph::NarrativeGraph;
use tala_weave::{Executor, ReplayConfig, ReplayEngine};

let mut graph = NarrativeGraph::new();
let a = IntentId::random();
let b = IntentId::random();
graph.insert_node(a, 1, 1.0);
graph.insert_node(b, 2, 1.0);
graph.add_edge(a, b, RelationType::Causal, 1.0);

let mut commands = HashMap::new();
commands.insert(a, "echo A".to_string());
commands.insert(b, "echo B".to_string());

let config = ReplayConfig {
    vars: HashMap::new(),
    completed: HashSet::new(),
    dry_run: false,
};

let executor: Executor = Box::new(|_cmd| Outcome {
    status: Status::Success,
    latency_ns: 100,
    exit_code: 0,
});

let mut engine = ReplayEngine::new(config).with_executor(executor);
let results = engine.execute(&graph, &[a, b], &commands).unwrap();

assert_eq!(results.len(), 2);
assert!(results.iter().all(|r| !r.skipped));
assert!(results.iter().all(|r| r.outcome.status == Status::Success));
```
