# tala-cli

The user-facing command-line interface for TALA. Provides command parsing, execution against a `Daemon`, and human-readable output formatting. This is a library crate; the binary wrapper (`main.rs`) is future work. The CLI supports five subcommands: `ingest`, `find`, `replay`, `status`, and `insights`.

## Key Types

| Type | Description |
|------|-------------|
| `Command` | Parsed CLI subcommand enum |
| `CommandParser` | Hand-written argument parser |
| `CommandRunner` | Executes parsed commands against a `Daemon` |
| `Output` | Structured output from command execution |
| `SearchResult` | A single semantic search result |
| `ReplayStepOutput` | A single replay plan step (display format) |
| `StatusOutput` | Daemon status information |
| `InsightOutput` | A single insight (display format) |

---

## Command

The parsed subcommand enum. Each variant carries the arguments extracted from the command line.

```rust
#[derive(Clone, Debug, PartialEq)]
pub enum Command {
    /// Ingest a raw shell command into the narrative graph.
    Ingest { raw_command: String },

    /// Semantic search for intents matching a query string.
    Find { query: String, k: usize },

    /// Build a replay plan from a root intent.
    Replay {
        root_id: String,
        depth: usize,
        dry_run: bool,
    },

    /// Show daemon status.
    Status,

    /// Run insight analysis (clustering + pattern detection).
    Insights { clusters: usize },
}
```

---

## CommandParser

A hand-written argument parser. Expects `args[0]` to be the binary name and `args[1]` to be the subcommand.

```rust
pub struct CommandParser;

impl CommandParser {
    /// Parse a slice of command-line arguments into a Command.
    ///
    /// Returns an error string for:
    /// - Missing subcommand
    /// - Unknown subcommand
    /// - Missing required arguments
    /// - Invalid flag values
    /// - Unknown flags
    pub fn parse(args: &[String]) -> Result<Command, String>;
}
```

### Subcommand Syntax

| Subcommand | Usage | Defaults |
|------------|-------|----------|
| `ingest` | `tala ingest <command>` | -- |
| `find` | `tala find <query> [--k N]` | k=10 |
| `replay` | `tala replay <uuid> [--depth N] [--dry-run]` | depth=3, dry_run=false |
| `status` | `tala status` | -- |
| `insights` | `tala insights [--clusters N]` | clusters=5 |

### Examples

```rust
use tala_cli::{CommandParser, Command};

fn args(strs: &[&str]) -> Vec<String> {
    strs.iter().map(|s| s.to_string()).collect()
}

// Ingest
let cmd = CommandParser::parse(&args(&["tala", "ingest", "kubectl apply -f deploy.yaml"])).unwrap();
assert_eq!(cmd, Command::Ingest { raw_command: "kubectl apply -f deploy.yaml".into() });

// Find with custom k
let cmd = CommandParser::parse(&args(&["tala", "find", "deploy", "--k", "20"])).unwrap();
assert_eq!(cmd, Command::Find { query: "deploy".into(), k: 20 });

// Replay with flags
let cmd = CommandParser::parse(&args(&[
    "tala", "replay", "550e8400-e29b-41d4-a716-446655440000",
    "--depth", "5", "--dry-run"
])).unwrap();
assert_eq!(cmd, Command::Replay {
    root_id: "550e8400-e29b-41d4-a716-446655440000".into(),
    depth: 5,
    dry_run: true,
});

// Status
let cmd = CommandParser::parse(&args(&["tala", "status"])).unwrap();
assert_eq!(cmd, Command::Status);

// Insights with custom cluster count
let cmd = CommandParser::parse(&args(&["tala", "insights", "--clusters", "8"])).unwrap();
assert_eq!(cmd, Command::Insights { clusters: 8 });
```

### Error Cases

```rust
// Missing subcommand
assert!(CommandParser::parse(&args(&["tala"])).is_err());

// Unknown subcommand
let err = CommandParser::parse(&args(&["tala", "frobnicate"])).unwrap_err();
assert!(err.contains("unknown subcommand"));

// Missing required argument
assert!(CommandParser::parse(&args(&["tala", "ingest"])).is_err());

// Invalid flag value
let err = CommandParser::parse(&args(&["tala", "find", "query", "--k", "abc"])).unwrap_err();
assert!(err.contains("invalid value for --k"));

// Unknown flag
let err = CommandParser::parse(&args(&["tala", "find", "query", "--verbose"])).unwrap_err();
assert!(err.contains("unknown flag"));
```

---

## CommandRunner

Executes parsed `Command` values against a `Daemon` instance, producing structured `Output`.

```rust
pub struct CommandRunner;

impl CommandRunner {
    /// Run a parsed command against the given daemon.
    ///
    /// - Ingest: extracts intent with default Context, returns Ingested
    /// - Find: embeds query via IntentPipeline, searches store, returns SearchResults
    /// - Replay: parses UUID, calls daemon.replay(), returns ReplayPlan
    /// - Status: returns StatusOutput (currently placeholder values)
    /// - Insights: calls daemon.insights(), returns Insights
    pub fn run(daemon: &Daemon, cmd: Command) -> Result<Output, TalaError>;
}
```

### Example

```rust
use tala_cli::{Command, CommandRunner, Output};
use tala_daemon::DaemonBuilder;

let daemon = DaemonBuilder::new().dim(384).build_in_memory();

let output = CommandRunner::run(
    &daemon,
    Command::Ingest { raw_command: "ls -la".into() },
).unwrap();

match output {
    Output::Ingested { id } => {
        assert!(!id.is_empty());
        println!("Ingested intent: {id}");
    }
    _ => panic!("expected Ingested"),
}
```

---

## Output

Structured output from command execution. Implements `Display` for human-readable formatting.

```rust
#[derive(Clone, Debug)]
pub enum Output {
    /// Result of an ingest command.
    Ingested { id: String },
    /// Results of a semantic search.
    SearchResults(Vec<SearchResult>),
    /// Replay plan steps.
    ReplayPlan(Vec<ReplayStepOutput>),
    /// Daemon status.
    Status(StatusOutput),
    /// Insight analysis results.
    Insights(Vec<InsightOutput>),
}
```

### Display Formatting

Each variant formats itself for terminal output:

```
// Ingested
Ingested: 550e8400-e29b-41d4-a716-446655440000

// SearchResults
Search results (3 found):
  [0] 550e8400-...  (similarity: 0.9500)
  [1] 661f9511-...  (similarity: 0.8200)
  [2] 772a0622-...  (similarity: 0.7100)

// ReplayPlan
Replay plan (2 steps):
  [0] echo hello  (id: 550e8400-..., deps: 0)
  [1] echo world  (id: 661f9511-..., deps: 1)

// Status
TALA daemon status:
  Nodes:    42
  Edges:    100
  Commands: 42
  Dim:      384

// Insights
Insights (2 found):
  [0] [pattern] Recurring sequence  (confidence: 0.85)
  [1] [summary] 42 intents over 3.0s...  (confidence: 1.00)
```

---

## Supporting Output Types

### SearchResult

```rust
#[derive(Clone, Debug)]
pub struct SearchResult {
    pub id: String,
    pub similarity: f32,
}
```

### ReplayStepOutput

```rust
#[derive(Clone, Debug)]
pub struct ReplayStepOutput {
    pub id: String,
    pub command: String,
    pub dep_count: usize,
}
```

### StatusOutput

```rust
#[derive(Clone, Debug)]
pub struct StatusOutput {
    pub node_count: usize,
    pub edge_count: usize,
    pub command_count: usize,
    pub dim: usize,
}
```

### InsightOutput

```rust
#[derive(Clone, Debug)]
pub struct InsightOutput {
    pub kind: String,
    pub description: String,
    pub confidence: f32,
}
```

The `kind` field maps from `InsightKind` as follows:

| InsightKind | String |
|-------------|--------|
| `RecurringPattern` | `"pattern"` |
| `FailureCluster` | `"failure"` |
| `Prediction` | `"prediction"` |
| `Summary` | `"summary"` |
