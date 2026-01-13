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
| Async test | `#[tokio::test]` or `#[async_std::test]` |
| Property test | `proptest!` macro |
| Mock trait | `mockall` crate |
| Test fixtures | `rstest` crate |
| Snapshot test | `insta` crate |
| Expect panic | `#[should_panic]` |
| Skip slow test | `#[ignore]` |
| Float comparison | `approx` crate or epsilon check |

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
// Return Result for cleaner assertions
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
    divide(10, 0);  // Should panic
}

#[test]
#[ignore]  // Skip by default, run with: cargo test -- --ignored
fn slow_integration_test() {
    // Expensive test
}
```

### Floating-Point Comparisons

```rust
// BAD: Direct equality fails due to precision
#[test]
fn test_float_bad() {
    assert_eq!(0.1 + 0.2, 0.3);  // FAILS!
}

// GOOD: Use epsilon comparison
#[test]
fn test_float_epsilon() {
    let result = 0.1 + 0.2;
    let expected = 0.3;
    assert!((result - expected).abs() < f64::EPSILON * 10.0);
}

// BETTER: Use approx crate
use approx::assert_relative_eq;

#[test]
fn test_float_approx() {
    assert_relative_eq!(0.1 + 0.2, 0.3, epsilon = 1e-10);
}
```

### Async Tests

```rust
#[tokio::test]
async fn test_async_fetch() {
    let client = Client::new();
    let result = client.fetch("https://example.com").await;
    assert!(result.is_ok());
}

// With timeout
#[tokio::test(flavor = "multi_thread")]
#[timeout(Duration::from_secs(5))]
async fn test_with_timeout() {
    // ...
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

## Common Commands

```bash
cargo test                    # Run all tests
cargo test test_name          # Run specific test
cargo test --lib              # Only lib tests
cargo test --doc              # Only doc tests
cargo test -- --nocapture     # Show println! output
cargo test -- --test-threads=1  # Sequential execution
RUST_BACKTRACE=1 cargo test   # With backtraces
```

## Common Mistakes

| Mistake | Issue | Fix |
|---------|-------|-----|
| Testing private functions | Brittle tests | Test public API |
| Hardcoded paths | Fails on other machines | Use `tempfile` crate |
| Sleep in async tests | Flaky, slow | Use channels/conditions |
| No error case tests | Miss failure modes | Test both Ok and Err |
| Giant test functions | Hard to debug | One assertion focus per test |
| Shared mutable state | Test pollution | Use fixtures, isolate state |

## Essential Crates

- `proptest` - Property-based testing
- `mockall` - Mock generation
- `rstest` - Fixtures and parametrized tests
- `insta` - Snapshot testing
- `tempfile` - Temporary files/directories
- `fake` - Fake data generation
- `wiremock` - HTTP mocking
- `approx` - Floating-point comparisons
- `cargo-tarpaulin` - Code coverage
- `cargo-nextest` - Faster test runner
