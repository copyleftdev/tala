# tala-intent

Converts raw shell command strings into structured `Intent` objects. The crate implements the `IntentExtractor` trait from `tala-core` through the `IntentPipeline`, which orchestrates four stages: tokenization, embedding generation, classification, and intent assembly.

## Key Types

| Type | Description |
|------|-------------|
| `Token` | A parsed token from a raw shell command |
| `IntentPipeline` | The full extraction pipeline (implements `IntentExtractor`) |

## Key Functions

| Function | Description |
|----------|-------------|
| `tokenize(raw)` | Parse a raw command string into structured tokens |
| `hash_context(ctx)` | FNV-1a hash of a `Context` for the intent's `context_hash` field |

---

## Token

An enum representing a single parsed element from a shell command. The tokenizer handles pipes, redirects, flags, quoted strings, and backslash escapes.

```rust
#[derive(Clone, Debug, PartialEq, Eq)]
pub enum Token {
    /// The command name (first word, or first word after a pipe).
    Command(String),
    /// A positional argument.
    Arg(String),
    /// A flag (short `-x` or long `--foo`).
    Flag(String),
    /// Pipe operator `|`.
    Pipe,
    /// Input redirect `<`.
    RedirectIn,
    /// Output redirect `>` or append `>>`.
    RedirectOut { append: bool },
}
```

---

## tokenize

```rust
/// Tokenize a raw shell command into structured tokens.
///
/// Handles:
/// - Pipes (`|`)
/// - Redirects (`<`, `>`, `>>`)
/// - Flags (`-x`, `--flag`)
/// - Quoted strings (single and double)
/// - Backslash escapes within double quotes
///
/// This is a simplified shell splitter, not a full POSIX parser.
pub fn tokenize(raw: &str) -> Vec<Token>;
```

### Examples

```rust
use tala_intent::{tokenize, Token};

// Simple command with flags and arguments
let tokens = tokenize("ls -la /tmp");
assert_eq!(tokens, vec![
    Token::Command("ls".into()),
    Token::Flag("-la".into()),
    Token::Arg("/tmp".into()),
]);

// Pipeline
let tokens = tokenize("cat file.txt | grep error");
assert_eq!(tokens, vec![
    Token::Command("cat".into()),
    Token::Arg("file.txt".into()),
    Token::Pipe,
    Token::Command("grep".into()),
    Token::Arg("error".into()),
]);

// Redirects
let tokens = tokenize("sort < input.txt >> output.txt");
assert_eq!(tokens, vec![
    Token::Command("sort".into()),
    Token::RedirectIn,
    Token::Arg("input.txt".into()),
    Token::RedirectOut { append: true },
    Token::Arg("output.txt".into()),
]);

// Quoted strings
let tokens = tokenize("echo \"hello world\"");
assert_eq!(tokens, vec![
    Token::Command("echo".into()),
    Token::Arg("hello world".into()),
]);

// Empty input yields no tokens
assert!(tokenize("").is_empty());
```

---

## IntentPipeline

The main extraction engine. Implements `IntentExtractor` from `tala-core`. Construction pre-computes exemplar embeddings for all intent categories, making classification a pure cosine-similarity lookup with no runtime model loading.

The pipeline is `Send + Sync` and can be shared across threads.

```rust
pub struct IntentPipeline { /* private */ }

impl IntentPipeline {
    /// Create a new pipeline. Pre-computes exemplar embeddings
    /// for intent classification.
    pub fn new() -> Self;

    /// Tokenize a raw command string.
    pub fn tokenize(&self, raw: &str) -> Vec<Token>;

    /// Generate a 384-dimensional embedding for a raw command.
    /// Uses a deterministic hash-based bag-of-characters approach
    /// with L2 normalization to unit length.
    pub fn embed(&self, raw: &str) -> Vec<f32>;

    /// Classify a command given its embedding.
    /// Compares against pre-computed exemplars using cosine similarity
    /// and returns the category with the highest average match.
    pub fn classify(&self, embedding: &[f32]) -> IntentCategory;
}

impl Default for IntentPipeline {
    fn default() -> Self { Self::new() }
}
```

### IntentExtractor Implementation

```rust
impl IntentExtractor for IntentPipeline {
    /// Extract a structured Intent from a raw command string and context.
    ///
    /// Pipeline:
    /// 1. Validate and tokenize the raw command
    /// 2. Generate a 384-dim embedding
    /// 3. Classify the intent category
    /// 4. Hash the execution context
    /// 5. Assemble the Intent with a random ID and current timestamp
    ///
    /// Returns `TalaError::ExtractionFailed` if the command is empty
    /// or produces no tokens.
    fn extract(&self, raw: &str, context: &Context) -> Result<Intent, TalaError>;
}
```

### Example

```rust
use tala_core::{Context, IntentExtractor};
use tala_intent::IntentPipeline;

let pipeline = IntentPipeline::new();

let ctx = Context {
    cwd: "/home/user/project".to_string(),
    env_hash: 42,
    session_id: 1,
    shell: "zsh".to_string(),
    user: "ops".to_string(),
};

let intent = pipeline.extract("cargo build --release", &ctx).unwrap();

assert_eq!(intent.raw_command, "cargo build --release");
assert_eq!(intent.embedding.len(), 384);
assert!(intent.context_hash != 0);
assert!(intent.outcome.is_none());
assert!((intent.confidence - 1.0).abs() < f32::EPSILON);
```

---

## IntentCategory

Classification of an intent's purpose. The classifier uses pre-computed exemplar embeddings for each category and finds the best match by average cosine similarity.

```rust
// Defined in tala-core, used by tala-intent
pub enum IntentCategory {
    Build,      // cargo build, make, gcc, npm run build
    Deploy,     // kubectl apply, docker push, terraform apply
    Debug,      // gdb, strace, perf record, valgrind
    Configure,  // vim ~/.bashrc, chmod, chown, git config
    Query,      // grep, find, ps aux, df -h, curl
    Navigate,   // cd, ls, pwd, tree
    Other(String),
}
```

The classifier falls back to `Other` if the best average similarity score is below 0.05.

---

## hash_context

```rust
/// Hash a Context into a u64 for the Intent's context_hash field.
///
/// Uses FNV-1a over all context fields: cwd, env_hash, session_id,
/// shell, and user. The hash is deterministic for the same input.
pub fn hash_context(ctx: &Context) -> u64;
```

### Example

```rust
use tala_core::Context;
use tala_intent::hash_context;

let ctx = Context {
    cwd: "/home/user".to_string(),
    env_hash: 0,
    session_id: 1,
    shell: "bash".to_string(),
    user: "ops".to_string(),
};

let h1 = hash_context(&ctx);
let h2 = hash_context(&ctx);
assert_eq!(h1, h2); // deterministic
```

---

## Embedding Details

The embedding generator produces 384-dimensional unit-length vectors using a deterministic hash-based approach. Each byte of the input command contributes to three positions in the vector via multiplicative hashing (FNV-like), with sign determined by hash bits and amplitude decayed by position (earlier characters contribute more). The result is L2-normalized.

This is a placeholder for real ML embeddings but preserves the property that similar command strings produce similar vectors, enabling meaningful cosine similarity comparisons.

| Property | Value |
|----------|-------|
| Dimensionality | 384 |
| Normalization | L2 (unit length) |
| Determinism | Same input always produces same output |
| Similarity preservation | Similar strings yield similar vectors |
