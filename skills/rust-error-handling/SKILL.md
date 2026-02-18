---
name: rust-error-handling
description: Use when designing error types for a Rust library or application, when error handling is inconsistent, or when choosing between thiserror, anyhow, or custom error types.
---

# Rust Error Handling

## Overview

Design clear, usable error types in Rust. Libraries need structured errors for callers to match on. Applications need ergonomic error handling with context. Use `tracing` to log errors with structured fields.

## When to Use

- Creating a new library or module
- Error handling is ad-hoc or inconsistent
- Choosing between error handling crates
- Need to add context to errors
- Converting between error types
- Designing public API error types

**Not for:** Panic handling (that's a different concern), Result usage basics (see `rust-idiomatic-patterns`)

## Quick Reference

| Context | Recommended Approach |
|---------|---------------------|
| Library public API | `thiserror` with enum variants |
| Application code | `anyhow` with context |
| Internal modules | `thiserror` or `anyhow` |
| FFI boundaries | Error codes + message |
| Performance critical | Custom minimal types |

## Library Errors with thiserror

```rust
use thiserror::Error;

#[derive(Error, Debug)]
pub enum ParseError {
    #[error("invalid syntax at line {line}: {message}")]
    InvalidSyntax { line: usize, message: String },

    #[error("unexpected token: expected {expected}, found {found}")]
    UnexpectedToken { expected: String, found: String },

    #[error("file not found: {0}")]
    FileNotFound(String),

    #[error("I/O error")]
    Io(#[from] std::io::Error),

    #[error("UTF-8 decoding error")]
    Utf8(#[from] std::string::FromUtf8Error),
}

// Callers can match on variants
match parse(input) {
    Ok(ast) => process(ast),
    Err(ParseError::InvalidSyntax { line, .. }) => {
        tracing::error!(line, "syntax error in input");
    }
    Err(e) => return Err(e.into()),
}
```

## Application Errors with anyhow

```rust
use anyhow::{Context, Result, bail, ensure};

fn load_config(path: &str) -> Result<Config> {
    let content = std::fs::read_to_string(path)
        .with_context(|| format!("failed to read config from {path}"))?;

    let config: Config = toml::from_str(&content)
        .context("failed to parse config as TOML")?;

    ensure!(config.version >= 2, "config version must be >= 2");

    if config.name.is_empty() {
        bail!("config name cannot be empty");
    }

    Ok(config)
}

// Error output includes context chain:
// Error: failed to read config from app.toml
// Caused by: No such file or directory (os error 2)
```

## Error Design Patterns

### Structured Fields Over Strings

```rust
// BAD: Hard to handle programmatically
#[derive(Error, Debug)]
#[error("validation failed: {0}")]
pub struct ValidationError(String);

// GOOD: Caller can match and recover
#[derive(Error, Debug)]
pub enum ValidationError {
    #[error("field {field} is required")]
    MissingField { field: &'static str },

    #[error("field {field} must be between {min} and {max}")]
    OutOfRange { field: &'static str, min: i64, max: i64 },

    #[error("invalid email format: {value}")]
    InvalidEmail { value: String },
}
```

### Error Conversion

```rust
#[derive(Error, Debug)]
pub enum ServiceError {
    #[error("database error")]
    Database(#[from] DatabaseError),

    #[error("network error")]
    Network(#[from] reqwest::Error),

    #[error("invalid input: {0}")]
    InvalidInput(String),
}

// Manual From impl for complex conversions
impl From<serde_json::Error> for ServiceError {
    fn from(err: serde_json::Error) -> Self {
        ServiceError::InvalidInput(err.to_string())
    }
}
```

### Result Type Alias

```rust
// Define once in lib.rs or error.rs
pub type Result<T> = std::result::Result<T, MyError>;

// Use throughout crate — no need to specify error type
pub fn process(input: &str) -> Result<Output> {
    // ...
}
```

### Combining thiserror and anyhow

```rust
// In library: thiserror for public errors
#[derive(Error, Debug)]
pub enum LibError {
    #[error("parse error")]
    Parse(#[from] ParseError),
}

// In application: anyhow for context
use anyhow::{Context, Result};

fn run() -> Result<()> {
    let data = mylib::parse(input)
        .context("failed to parse input file")?;
    Ok(())
}
```

### Error Logging with tracing

```rust
// Log errors with structured context
fn handle_request(req: Request) -> Result<Response> {
    let result = process(&req).map_err(|e| {
        tracing::error!(
            request_id = %req.id,
            path = %req.path,
            ?e,
            "request processing failed"
        );
        e
    })?;
    Ok(result)
}

// Or use tracing's instrument for automatic context
#[tracing::instrument(skip(data), err)]
fn process(data: &[u8]) -> Result<Output> {
    // Errors auto-logged with function args as span fields
    let parsed = parse(data)?;
    Ok(transform(parsed))
}
```

## Common Mistakes

| Mistake | Issue | Fix |
|---------|-------|-----|
| `String` as error type | No structure, can't match | Use enum with variants |
| `Box<dyn Error>` everywhere | Loses type info | Use `thiserror` enum |
| Swallowing errors | Silent failures | Propagate or log with tracing |
| `.unwrap()` in library | Panic in user code | Return Result |
| `anyhow` in library public API | Can't match on errors | Use `thiserror` |
| No context on `?` | Unclear error source | Add `.context()` |
| Giant error enums | Exposes internals | Group by domain |
| `eprintln!` for errors | Unstructured, not captured | `tracing::error!` |

## Error Handling Checklist

1. **Library or Application?** Libraries use `thiserror`, apps use `anyhow`
2. **Can callers recover?** If yes, provide matchable variants
3. **Is context preserved?** Use `#[from]` or `.context()`
4. **Are errors actionable?** Include relevant data in variants
5. **Is the error type public?** Only expose what callers need
6. **Are errors logged?** Use `tracing` with structured fields

## Essential Crates

- `thiserror` - Derive macro for error types
- `anyhow` - Flexible error handling for applications
- `miette` - Fancy diagnostic errors with source snippets
- `color-eyre` - Colorful error reports
- `tracing` - Structured error logging with spans
