---
name: rust-testing
description: Use when writing tests for Rust code, implementing TDD, or setting up test infrastructure with cargo test, proptest, mockall, or rstest. Use when tests are flaky, slow, or hard to maintain, or when unsure how to test async code or complex state.
---

# Rust Testing

## Overview

Write effective tests in Rust using cargo test, property-based testing, and mocking. Focus on tests that catch real bugs and remain maintainable.

## When to Use

- Writing new functionality (TDD)
- Adding tests to existing code
- Tests are flaky or slow
- Unsure how to test async code
- Need property-based testing
- Setting up test fixtures

**Not for:** Benchmarking (use `rust-performance`), integration/E2E testing infrastructure

## Quick Reference

| Need | Approach |
|------|----------|
| Unit test | `#[test]` in same file or `tests` module |
| Integration test | `tests/` directory |
| Async test | `#[tokio::test]` |
| Property test | `proptest!` macro |
| Mutation test | `cargo mutants` |
| Mock trait | `mockall` crate |
| Test fixtures | `rstest` crate |
| Snapshot test | `insta` crate |
| Expect panic | `#[should_panic]` |
| Skip slow test | `#[ignore]` |
| Float comparison | `approx` crate or epsilon check |
| UB detection | `cargo careful test` |

## Test Organization

```rust
// src/lib.rs or src/parser.rs
pub fn parse(input: &str) -> Result<Ast, ParseError> {
    // implementation
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn parse_empty_returns_empty_ast() {
        let result = parse("").unwrap();
        assert!(result.nodes.is_empty());
    }

    #[test]
    fn parse_invalid_returns_error() {
        let result = parse("{{{}}}");
        assert!(matches!(result, Err(ParseError::UnbalancedBraces)));
    }
}
```

## Testing Patterns

### Test Result Types

```rust
#[test]
fn test_with_result() -> Result<(), Box<dyn std::error::Error>> {
    let config = Config::load("test.toml")?;
    assert_eq!(config.name, "test");
    Ok(())
}
```

### Testing Panics and Ignoring Tests

```rust
#[test]
#[should_panic(expected = "division by zero")]
fn divide_by_zero_panics() {
    divide(10, 0);
}

#[test]
#[ignore]  // Run with: cargo test -- --ignored
fn slow_integration_test() {
    // Expensive test
}
```

### Floating-Point Comparisons

```rust
// BAD: Direct equality fails due to precision
assert_eq!(0.1 + 0.2, 0.3);  // FAILS!

// GOOD: Epsilon comparison
let result = 0.1 + 0.2;
assert!((result - 0.3).abs() < f64::EPSILON * 10.0);

// BETTER: approx crate
use approx::assert_relative_eq;
assert_relative_eq!(0.1 + 0.2, 0.3, epsilon = 1e-10);
```

### Async Tests

```rust
#[tokio::test]
async fn test_async_fetch() {
    let client = Client::new();
    let result = client.fetch("https://example.com").await;
    assert!(result.is_ok());
}

// Multi-threaded runtime for tests that need it
#[tokio::test(flavor = "multi_thread", worker_threads = 2)]
async fn test_concurrent_access() {
    // ...
}

// Timeout via tokio (not an attribute — wrap in the test body)
#[tokio::test]
async fn test_with_timeout() {
    let result = tokio::time::timeout(
        Duration::from_secs(5),
        slow_operation(),
    ).await;
    assert!(result.is_ok(), "operation timed out");
}
```

### Capturing tracing Output in Tests

```rust
// Use test_log crate to see tracing output during test failures
use test_log::test;

#[test(tokio::test)]
async fn test_with_logging() {
    tracing::info!("this shows on test failure");
    assert!(some_operation().await.is_ok());
}
```

### Property-Based Testing

```rust
use proptest::prelude::*;

proptest! {
    #[test]
    fn parse_roundtrip(input in "[a-z]{1,100}") {
        let parsed = parse(&input)?;
        let serialized = serialize(&parsed);
        prop_assert_eq!(input, serialized);
    }

    #[test]
    fn sort_maintains_length(mut vec in prop::collection::vec(any::<i32>(), 0..100)) {
        let original_len = vec.len();
        vec.sort();
        prop_assert_eq!(vec.len(), original_len);
    }
}
```

### Mutation Testing

Verify tests actually catch bugs — not just that they pass:

```bash
cargo install cargo-mutants
cargo mutants                    # Run all mutations
cargo mutants -p my_crate       # Specific crate
cargo mutants -- --test-threads=4  # Parallel
```

Mutation testing modifies your code (removing conditions, changing operators) and checks that at least one test fails. Surviving mutants = gaps in test coverage.

### Mocking with mockall

```rust
use mockall::{automock, predicate::*};

#[automock]
trait Database {
    fn get(&self, key: &str) -> Option<String>;
    fn set(&mut self, key: &str, value: &str) -> Result<(), Error>;
}

#[test]
fn test_with_mock_db() {
    let mut mock = MockDatabase::new();
    mock.expect_get()
        .with(eq("user:1"))
        .times(1)
        .returning(|_| Some("Alice".to_string()));

    let service = UserService::new(mock);
    assert_eq!(service.get_user_name("1"), Some("Alice".to_string()));
}
```

### Test Fixtures with rstest

```rust
use rstest::{rstest, fixture};

#[fixture]
fn test_db() -> Database {
    Database::in_memory()
}

#[rstest]
fn test_insert(test_db: Database) {
    test_db.insert("key", "value");
    assert_eq!(test_db.get("key"), Some("value"));
}

#[rstest]
#[case("hello", 5)]
#[case("", 0)]
#[case("rust", 4)]
fn test_length(#[case] input: &str, #[case] expected: usize) {
    assert_eq!(input.len(), expected);
}
```

## Verifying Test Quality

```bash
cargo careful test                  # Run with extra UB checks
cargo mutants                       # Mutation testing
cargo tarpaulin --out html          # Coverage report
```

Break the code intentionally, confirm a test fails, then fix. If no test fails, you have a gap.

## Common Commands

```bash
cargo test                          # Run all tests
cargo test test_name                # Run specific test
cargo test --lib                    # Only lib tests
cargo test --doc                    # Only doc tests
cargo test -- --nocapture           # Show println!/tracing output
cargo test -- --test-threads=1      # Sequential execution
RUST_BACKTRACE=1 cargo test         # With backtraces
cargo nextest run                   # Faster parallel runner
```

## Common Mistakes

| Mistake | Issue | Fix |
|---------|-------|-----|
| Testing private functions | Brittle tests | Test public API |
| Hardcoded paths | Fails on other machines | Use `tempfile` crate |
| Sleep in async tests | Flaky, slow | Use channels/notify/timeout |
| No error case tests | Miss failure modes | Test both Ok and Err |
| Giant test functions | Hard to debug | One assertion focus per test |
| Shared mutable state | Test pollution | Use fixtures, isolate state |
| Tests pass but don't catch bugs | False confidence | Mutation testing |

## Essential Crates

- `proptest` - Property-based testing
- `cargo-mutants` - Mutation testing
- `mockall` - Mock generation
- `rstest` - Fixtures and parametrized tests
- `insta` - Snapshot testing
- `tempfile` - Temporary files/directories
- `wiremock` - HTTP mocking
- `approx` - Floating-point comparisons
- `test-log` - Capture tracing in tests
- `cargo-nextest` - Faster test runner
- `cargo-tarpaulin` - Code coverage
