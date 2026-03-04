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
| Complex multi-layer app | `error-stack` with `change_context()` |
| Errors with many context fields | `snafu` or thiserror + `.map_err()` |
| Internal modules | `thiserror` or `anyhow` |
| FFI boundaries | Error codes + message |
| `no_std` library | `thiserror` v2 or hand-written `core::error::Error` |
| Simple one-off error struct | Hand-written `Error` impl |

## Library Errors with thiserror

```rust
use thiserror::Error;

#[derive(Error, Debug)]
pub enum ParseError {
    #[error("invalid syntax at line {line}: {message}")]
    InvalidSyntax { line: usize, message: String },

    #[error("unexpected token: expected {expected}, found {found}")]
    UnexpectedToken { expected: String, found: String },

    #[error("I/O error")]
    Io(#[from] std::io::Error),

    #[error("UTF-8 decoding error")]
    Utf8(#[from] std::string::FromUtf8Error),
}
```

## `#[source]` vs `#[from]`: Critical Distinction

`#[from]` implies `#[source]` AND generates `From<T>`. But it has limits:

```rust
// PROBLEM: Two operations produce io::Error but #[from] only works once
#[derive(Debug, Error)]
pub enum Error {
    #[error("failed to read config")]
    ReadConfig(#[from] io::Error),  // Gets the From impl

    #[error("failed to write output")]
    WriteOutput(io::Error),  // Does NOT get From
    // Now ? always maps io::Error to ReadConfig, even for writes!
}
```

**Use `#[source]` when the same error type appears in multiple variants or when variants carry context fields:**

```rust
#[derive(Debug, Error)]
pub enum Error {
    #[error("failed to read config from {path}")]
    ReadConfig {
        path: PathBuf,
        #[source] cause: io::Error,
    },

    #[error("failed to write output to {path}")]
    WriteOutput {
        path: PathBuf,
        #[source] cause: io::Error,
    },
}

// Construct explicitly with .map_err():
let data = fs::read_to_string(path)
    .map_err(|cause| Error::ReadConfig { path: path.to_owned(), cause })?;
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
```

### Downcasting anyhow errors

```rust
fn handle_error(err: anyhow::Error) {
    if let Some(io_err) = err.downcast_ref::<io::Error>() {
        if io_err.kind() == io::ErrorKind::NotFound {
            tracing::info!("file not found, creating default");
            return;
        }
    }
    tracing::error!(?err, "unexpected error");
}
```

`downcast_ref` checks the outermost error. Walk the chain with `err.chain()` to find inner causes.

## Error Reporting vs Error Handling

These are distinct concerns:

- **Error handling** -- code reacting programmatically (match on variants, retry, fallback)
- **Error reporting** -- humans reading logs or terminal output

**The source() rule:** An underlying error should be returned via `source()` OR included in `Display`, **never both**. Double-reporting produces garbled output.

```rust
// WRONG: Display includes source AND source() returns it
#[error("io error: {0}")]
Io(#[source] io::Error),  // prints "io error: No such file: No such file"

// CORRECT: Display describes the operation, source() provides the cause
#[error("failed to read config")]
Io(#[source] io::Error),
```

**Error messages** should be lowercase sentences without trailing punctuation: `"invalid digit found in string"`, not `"Invalid Digit Found In String."`.

## `#[error(transparent)]`

Forwards both `Display` and `source()` to the inner error. Two legitimate uses:

```rust
// 1. Catch-all for unexpected errors
#[derive(Debug, Error)]
pub enum MyError {
    #[error("specific problem: {0}")]
    Specific(String),

    #[error(transparent)]
    Other(#[from] anyhow::Error),
}

// 2. Opaque newtype for API stability
#[derive(Debug, Error)]
#[error(transparent)]
pub struct PublicError(#[from] InternalError);
```

## error-stack for Complex Applications

Attachment-based errors that maintain typed variants while providing rich context:

```rust
use error_stack::{Report, ResultExt};

#[derive(Debug, thiserror::Error)]
#[error("failed to parse config")]
struct ParseConfigError;

fn parse_config(path: &Path) -> Result<Config, Report<ParseConfigError>> {
    let content = std::fs::read_to_string(path)
        .change_context(ParseConfigError)?;

    toml::from_str(&content)
        .change_context(ParseConfigError)
        .attach_printable(format!("path: {}", path.display()))?
}
```

Use error-stack when: many layers with different error contexts, need structured diagnostic attachments, want typed errors in signatures with richer context than thiserror alone.

## Error Design Patterns

### Structured Fields Over Strings

```rust
// GOOD: Caller can match and recover
#[derive(Error, Debug)]
pub enum ValidationError {
    #[error("field {field} is required")]
    MissingField { field: &'static str },

    #[error("field {field} must be between {min} and {max}")]
    OutOfRange { field: &'static str, min: i64, max: i64 },
}
```

### Result Type Alias

```rust
pub type Result<T> = std::result::Result<T, MyError>;
```

### Error Logging with tracing

```rust
#[tracing::instrument(skip(data), err)]
fn process(data: &[u8]) -> Result<Output> {
    let parsed = parse(data)?;
    Ok(transform(parsed))
}
```

## `core::error::Error` (Rust 1.81+)

The `Error` trait is now in `core`, enabling `no_std` crates to implement standard error types. `thiserror` v2 supports `no_std`.

## Common Mistakes

| Mistake | Issue | Fix |
|---------|-------|-----|
| `String` as error type | No structure, can't match | Use enum with variants |
| `Box<dyn Error>` everywhere | Loses type info | Use `thiserror` enum |
| Swallowing errors | Silent failures | Propagate or log with tracing |
| `.unwrap()` in library | Panic in user code | Return Result |
| `anyhow` in library public API | Can't match on errors | Use `thiserror` |
| No context on `?` | Unclear error source | Add `.context()` |
| `#[from]` for same type in 2 variants | Wrong variant selected by `?` | Use `#[source]` + `.map_err()` |
| Source in Display AND source() | Double-printed error chain | Choose one, not both |
| `eprintln!` for errors | Unstructured, not captured | `tracing::error!` |

## Error Handling Checklist

1. **Library or Application?** Libraries use `thiserror`, apps use `anyhow`
2. **Can callers recover?** If yes, provide matchable variants
3. **Same error type in multiple variants?** Use `#[source]` not `#[from]`
4. **Is context preserved?** Use `#[from]`, `#[source]`, or `.context()`
5. **Are errors actionable?** Include relevant data in variants
6. **Are errors logged?** Use `tracing` with structured fields
7. **source() OR Display, never both?** Check for double-reporting

## Essential Crates

- `thiserror` -- Derive macro for library error types
- `anyhow` -- Flexible error handling for applications
- `error-stack` -- Attachment-based errors for complex apps
- `snafu` -- Context selectors for errors with many fields
- `miette` -- Fancy diagnostic errors with source snippets
- `tracing` -- Structured error logging with spans
