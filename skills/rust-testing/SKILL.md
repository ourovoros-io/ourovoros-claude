---
name: rust-testing
description: Use when writing tests for Rust code, implementing TDD, or setting up test infrastructure with cargo test, proptest, mockall, or rstest. Use when tests are flaky, slow, or hard to maintain, or when unsure how to test async code or complex state.
---

# Rust Testing

## Overview

Write effective tests in Rust using cargo-nextest, property-based testing, and mocking. Focus on tests that catch real bugs and remain maintainable.

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
| Test runner | `cargo nextest run` (process-per-test isolation) |
| Unit test | `#[test]` in same file or `tests` module |
| Integration test | `tests/` directory |
| Async test | `#[tokio::test]` |
| Time-dependent async | `#[tokio::test(start_paused = true)]` |
| Property test | `proptest!` macro |
| Mutation test | `cargo mutants` |
| Fuzz test | `cargo fuzz run target` (nightly) |
| Concurrency test | `loom` (exhaustive interleaving) |
| Mock trait | `mockall` crate |
| Test fixtures | `rstest` crate |
| Snapshot test | `insta` crate |
| CLI test | `assert_cmd` + `predicates` |
| Container test | `testcontainers` |
| Expect panic | `#[should_panic]` |
| Skip slow test | `#[ignore]` |
| Log assertions | `tracing-test` |
| Log visibility | `test-log` |
| UB detection | `cargo +nightly miri test` |

## Test Organization

```rust
// src/parser.rs
pub fn parse(input: &str) -> Result<Ast, ParseError> {
    // implementation
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn parse_empty_returns_empty_ast() {
        let result = parse("").expect("should parse empty input");
        assert!(result.nodes.is_empty());
    }

    #[test]
    fn parse_invalid_returns_error() {
        let result = parse("{{{}}}");
        assert!(matches!(result, Err(ParseError::UnbalancedBraces)));
    }
}
```

**Integration tests** go in `tests/` directory. Shared helpers use `tests/common/mod.rs` (not `tests/common.rs`).

## Async Tests

```rust
#[tokio::test]
async fn test_async_fetch() {
    let client = Client::new();
    let result = client.fetch("https://example.com").await;
    assert!(result.is_ok());
}

// Multi-threaded runtime for tests that need true concurrency
#[tokio::test(flavor = "multi_thread", worker_threads = 2)]
async fn test_concurrent_access() {
    let handle = tokio::spawn(async { compute().await });
    handle.await.expect("task panicked");
}
```

### Time-dependent tests with `start_paused`

```rust
// Time advances instantly when awaited -- no real waiting
#[tokio::test(start_paused = true)]
async fn test_timeout_behavior() {
    tokio::time::sleep(Duration::from_secs(3600)).await;
    // Runs immediately, not after an hour
}
```

### Common async test mistakes

1. **Wrong runtime flavor:** `#[tokio::test]` defaults to `current_thread`. Spawned tasks may not progress without `multi_thread`.
2. **Real sleeps:** Without `start_paused`, `tokio::time::sleep` actually sleeps.
3. **Blocking the runtime:** Sync I/O on the async runtime starves other tasks -- use `spawn_blocking`.
4. **Not testing cancellation safety:** Drop futures mid-execution and verify state consistency.

## Property-Based Testing with proptest

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

Use `prop_assert!` / `prop_assert_eq!` inside proptest blocks (not `assert!`).

### Custom strategies with prop_compose!

```rust
prop_compose! {
    fn arb_order()(
        id in 1u64..1_000_000,
        quantity in 1u32..1000,
        price_cents in 1u64..100_000_00
    ) -> Order {
        Order { id, quantity, price_cents }
    }
}

proptest! {
    #[test]
    fn order_total_positive(order in arb_order()) {
        let total = u64::from(order.quantity) * order.price_cents;
        prop_assert!(total > 0);
    }
}
```

**Failure persistence:** proptest writes failing cases to `proptest-regressions/` files. Commit these to version control.

## Fuzz Testing with cargo-fuzz

For parsers, deserializers, and any code handling untrusted input:

```rust
// fuzz/fuzz_targets/parse_input.rs
#![no_main]
use libfuzzer_sys::fuzz_target;

fuzz_target!(|data: &[u8]| {
    if let Ok(s) = std::str::from_utf8(data) {
        let _ = my_crate::parse(s);
    }
});
```

```bash
cargo +nightly fuzz run parse_input -- -max_total_time=300
cargo +nightly fuzz tmin parse_input crash-*  # minimize crash
```

Convert minimized crash inputs into unit tests for regression.

## Concurrency Testing with loom

Exhaustive exploration of thread interleavings:

```rust
#[cfg(loom)]
use loom::sync::{Arc, atomic::{AtomicUsize, Ordering}};
#[cfg(not(loom))]
use std::sync::{Arc, atomic::{AtomicUsize, Ordering}};

#[cfg(loom)]
#[test]
fn test_concurrent_counter() {
    loom::model(|| {
        let counter = Arc::new(AtomicUsize::new(0));
        let threads: Vec<_> = (0..2).map(|_| {
            let counter = counter.clone();
            loom::thread::spawn(move || {
                counter.fetch_add(1, Ordering::SeqCst);
            })
        }).collect();
        for t in threads { t.join().expect("thread panicked"); }
        assert_eq!(counter.load(Ordering::SeqCst), 2);
    });
}
```

Run with: `RUSTFLAGS="--cfg loom" cargo test --release --lib`

## Snapshot Testing with insta

For complex outputs where manual expected values are error-prone:

```rust
use insta::assert_json_snapshot;

#[test]
fn test_api_response() {
    let response = build_response();
    assert_json_snapshot!(response, {
        ".timestamp" => "[timestamp]",  // redact dynamic values
        ".request_id" => "[uuid]",
    });
}
```

Review snapshots: `cargo insta review`

## CLI Testing with assert_cmd

```rust
use assert_cmd::Command;
use predicates::prelude::*;

#[test]
fn test_cli_success() {
    Command::cargo_bin("my-tool")
        .expect("binary not found")
        .arg("--version")
        .assert()
        .success()
        .stdout(predicate::str::contains("1.0"));
}
```

## Integration Tests with testcontainers

```rust
use testcontainers::{core::WaitFor, runners::AsyncRunner, GenericImage, ImageExt};

#[tokio::test]
async fn test_with_postgres() {
    let container = GenericImage::new("postgres", "16-alpine")
        .with_exposed_port(5432.tcp())
        .with_wait_for(WaitFor::message_on_stdout("ready to accept connections"))
        .with_env_var("POSTGRES_PASSWORD", "test")
        .start()
        .await
        .expect("failed to start container");

    let port = container.get_host_port_ipv4(5432).await.expect("port");
    let conn = format!("postgres://postgres:test@localhost:{port}/postgres");
    // Use conn for integration testing...
}
```

## Mocking with mockall

```rust
use mockall::{automock, predicate::*};

#[automock]
trait Database {
    fn get(&self, key: &str) -> Option<String>;
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

## Mutation Testing

Verify tests catch real bugs -- not just that they pass:

```bash
cargo mutants                    # run all mutations
cargo mutants -p my_crate       # specific crate
```

Surviving mutants = test coverage gaps.

## Capturing tracing Output

**`tracing-test`** -- when you need to **assert** on log content:

```rust
use tracing_test::traced_test;

#[traced_test]
#[test]
fn test_logging() {
    tracing::info!("processing request");
    assert!(logs_contain("processing request"));
}
```

**`test-log`** -- when you want **visibility** during debugging:

```rust
use test_log::test;

#[test(tokio::test)]
async fn async_test_with_logging() {
    tracing::debug!("visible with RUST_LOG=debug");
}
```

## Common Commands

```bash
cargo nextest run                   # Run all tests (recommended)
cargo nextest run test_name         # Run specific test
cargo nextest run --profile ci      # CI profile (retries, no fail-fast)
cargo test --doc                    # Doc tests (nextest doesn't support these)
cargo test -- --nocapture           # Show output
cargo +nightly miri test            # UB detection
cargo +nightly careful test         # Extra runtime checks
RUST_BACKTRACE=1 cargo test         # With backtraces
```

## Common Mistakes

| Mistake | Issue | Fix |
|---------|-------|-----|
| Testing private functions | Brittle tests | Test public API |
| Hardcoded paths | Fails on other machines | Use `tempfile` crate |
| Sleep in async tests | Flaky, slow | Use `start_paused` / channels / timeout |
| No error case tests | Miss failure modes | Test both Ok and Err |
| Giant test functions | Hard to debug | One assertion focus per test |
| Shared mutable state | Test pollution | Use fixtures, isolate state |
| Tests pass but miss bugs | False confidence | Mutation testing, property testing |
| `current_thread` runtime for spawned tasks | Tasks don't progress | Use `multi_thread` flavor |

## Essential Crates

- `cargo-nextest` -- Process-per-test runner (faster, isolated)
- `proptest` -- Property-based testing with automatic shrinking
- `cargo-mutants` -- Mutation testing
- `cargo-fuzz` -- Fuzz testing (nightly)
- `loom` -- Exhaustive concurrency testing
- `mockall` -- Mock generation
- `rstest` -- Fixtures and parametrized tests
- `insta` -- Snapshot testing
- `assert_cmd` -- CLI testing
- `testcontainers` -- Docker-based integration tests
- `tracing-test` -- Assert on log output
- `test-log` -- See logs during test failures
- `tempfile` -- Temporary files/directories
- `wiremock` -- HTTP mocking
- `approx` -- Floating-point comparisons
