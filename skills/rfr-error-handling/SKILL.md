---
name: rfr-error-handling
description: Use when designing error types for Rust libraries or applications, choosing between thiserror and anyhow, structuring error enums, implementing the Error trait, or deciding when to panic vs return Result. Use when error handling is inconsistent or error types are poorly designed.
---

# Rust for Rustaceans — Ch 4: Error Handling

## Error Type Design

### Enumerated Errors (Libraries)
Each variant represents a specific, documented failure mode. Callers can match on variants.

```rust
#[derive(Debug, thiserror::Error)]
#[non_exhaustive]
pub enum ParseError {
    #[error("invalid header at byte {offset}")]
    InvalidHeader { offset: usize },

    #[error("unexpected EOF, expected {expected} more bytes")]
    UnexpectedEof { expected: usize },

    #[error("unsupported version {version}")]
    UnsupportedVersion { version: u8 },
}
```

### Opaque Errors (Applications)
When callers don't need to match on specific errors — just report/log them.

```rust
// Application code — use anyhow
use anyhow::{Context, Result};

fn load_config() -> Result<Config> {
    let text = std::fs::read_to_string("config.toml")
        .context("failed to read config file")?;
    let config: Config = toml::from_str(&text)
        .context("failed to parse config")?;
    Ok(config)
}
```

### Library vs Application

| Aspect | Library | Application |
|--------|---------|-------------|
| Error crate | `thiserror` | `anyhow` |
| Error type | Custom enum | `anyhow::Error` |
| Caller needs | Match on variants | Display/log message |
| `#[non_exhaustive]` | Yes — enables adding variants | N/A |
| Source chain | Via `#[source]` or `#[from]` | Via `.context()` |
| `Display` | Describe the error precisely | Include context |

## The `Error` Trait

```rust
pub trait Error: Debug + Display {
    fn source(&self) -> Option<&(dyn Error + 'static)> {
        None
    }
}
```

### Three Requirements
1. **`Debug`** — derive it
2. **`Display`** — describe what went wrong for humans
3. **`source()`** — return the underlying cause (if any)

### `thiserror` Does This For You

```rust
#[derive(Debug, thiserror::Error)]
pub enum Error {
    // #[error] generates Display
    // #[from] generates From<T> AND source()
    #[error("I/O error")]
    Io(#[from] std::io::Error),

    // #[source] marks the source without generating From
    #[error("connection failed to {addr}")]
    Connect {
        addr: SocketAddr,
        #[source]
        source: std::io::Error,
    },

    // No source — leaf error
    #[error("invalid port: {port}")]
    InvalidPort { port: u16 },
}
```

### `Display` Best Practices
- **Lowercase, no trailing period**: `"connection timed out"` not `"Connection timed out."`
- **Don't repeat the source's message**: The error chain handles that
- **Include relevant context**: what operation, what input
- **Be specific**: `"failed to parse header at byte 42"` not `"parse error"`

```rust
// ❌ Bad: repeats source message
#[error("I/O error: {0}")]
Io(#[from] std::io::Error),

// ✅ Good: describes what happened, source provides details
#[error("failed to read config file")]
ReadConfig(#[source] std::io::Error),
```

## The `?` Operator

### How It Works
1. Calls `From::from(error)` to convert the error type
2. Returns `Err(converted_error)` early

### `From` Conversions
- `#[from]` in thiserror generates `impl From<SourceError> for YourError`
- Only one `#[from]` per source type (otherwise ambiguous)
- For multiple errors of the same type, use `#[source]` + manual conversion

```rust
#[derive(Debug, thiserror::Error)]
pub enum Error {
    #[error("failed to read file")]
    ReadFile(#[from] std::io::Error),  // From<io::Error> generated

    #[error("failed to parse JSON")]
    ParseJson(#[from] serde_json::Error), // From<serde_json::Error>
}

fn load() -> Result<Data, Error> {
    let text = std::fs::read_to_string("data.json")?; // io::Error → Error
    let data = serde_json::from_str(&text)?;           // serde::Error → Error
    Ok(data)
}
```

## When to Panic

### Panic Is Appropriate When
- **Programming bug**: invariant violation that should never happen (index out of bounds in a data structure that guarantees bounds)
- **Unrecoverable state**: data corruption that makes continuing dangerous
- **Setup/initialization**: invalid configuration that prevents the program from starting at all

### Panic Is NOT Appropriate When
- User input is invalid → return `Result`
- External resource is unavailable → return `Result`
- A timeout occurs → return `Result`
- Basically: if it can happen in production, return `Result`

### Assert vs Debug Assert
- `assert!()` — always checked. Use for invariants that MUST hold.
- `debug_assert!()` — only in debug builds. Use for expensive checks.

```rust
// ✅ Appropriate panic: internal invariant
fn get_unchecked(&self, idx: usize) -> &T {
    debug_assert!(idx < self.len, "index out of bounds");
    unsafe { &*self.ptr.add(idx) }
}

// ❌ Don't panic on user input
fn parse_port(s: &str) -> u16 {
    s.parse().unwrap() // Will panic on invalid input!
}

// ✅ Return Result for fallible operations
fn parse_port(s: &str) -> Result<u16, ParseIntError> {
    s.parse()
}
```

## Error Design Patterns

### Boxed Error Fields
When an error variant holds a large payload, box it to keep the enum small:

```rust
#[derive(Debug, thiserror::Error)]
pub enum Error {
    #[error("peer key mismatch")]
    PeerKeyMismatch {
        expected: Box<[u8; 32]>, // 32 bytes on heap, not inline
    },
}
```

### Error Context Pattern
Add context at each layer without losing the source:

```rust
use anyhow::Context;

fn deploy() -> anyhow::Result<()> {
    build().context("build step failed")?;
    upload().context("upload step failed")?;
    notify().context("notification failed")?;
    Ok(())
}
// Error chain: "notification failed" → "SMTP connection refused"
```

### Downcasting
When you need to check for a specific error type in an opaque chain:

```rust
fn handle_error(err: &anyhow::Error) {
    if let Some(io_err) = err.downcast_ref::<std::io::Error>() {
        if io_err.kind() == std::io::ErrorKind::NotFound {
            // Handle file not found specifically
        }
    }
}
```

### Error Wrapping for Public APIs
Wrap internal errors so your public API doesn't leak implementation details:

```rust
// ❌ Leaks internal dependency
pub enum Error {
    Database(sqlx::Error), // Callers now depend on sqlx
}

// ✅ Opaque wrapper
pub enum Error {
    #[error("database error: {msg}")]
    Database { msg: String, #[source] source: Box<dyn std::error::Error + Send + Sync> },
}
```

## Quick Reference

| Guideline | Rule |
|-----------|------|
| Library errors | `thiserror` enum, `#[non_exhaustive]` |
| Application errors | `anyhow::Result` with `.context()` |
| Display format | Lowercase, no period, no source repetition |
| Source chain | `#[source]` or `#[from]`, implement `source()` |
| Panic | Only for bugs/invariants, never for user input |
| Large payloads | Box them inside error variants |
| `?` operator | Relies on `From` conversion |
| Public APIs | Don't leak internal dependency error types |

## Common Mistakes

1. **Using `unwrap()` in library code** — always return `Result`; let the caller decide
2. **Repeating source message in Display** — `#[error("io: {0}")]` with `#[from]` prints the source twice when the chain is displayed
3. **Exhaustive error enums in public APIs** — add `#[non_exhaustive]` to allow adding variants
4. **Using `anyhow` in libraries** — callers can't match on variants; use `thiserror`
5. **Swallowing errors with `let _ = ...`** — at minimum, log them
6. **Giant error enums** — split into domain-specific errors if you have >10 variants
7. **Missing `Send + Sync` on error types** — required for use across threads and with `?` in async code
