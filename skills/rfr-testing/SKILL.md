---
name: rfr-testing
description: Use when writing Rust tests, choosing test strategies, implementing property-based testing with proptest, fuzzing with cargo-fuzz, running Miri for undefined behavior detection, or benchmarking with criterion. Use when tests are flaky, slow, or insufficiently covering edge cases.
---

# Rust for Rustaceans — Ch 6: Testing

## Test Types

### Unit Tests
- Live in `mod tests` inside the source file
- Have access to private items
- Run with `cargo test`

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn parses_valid_input() {
        let result = parse("42").unwrap();
        assert_eq!(result, 42);
    }
}
```

### Integration Tests
- Live in `tests/` directory at crate root
- Can only access public API
- Each file is a separate crate (separate compilation)
- Share helpers via `tests/common/mod.rs` (not `tests/common.rs` — that would be a test file)

```
tests/
  common/
    mod.rs        # shared helpers (not compiled as a test)
  api_tests.rs    # integration test
  cli_tests.rs    # integration test
```

### Doc Tests
- Code in `///` doc comments is compiled and run
- Must compile and pass unless marked `no_run` or `ignore`
- Great for showing API usage and catching doc rot

```rust
/// Adds two numbers.
///
/// # Examples
///
/// ```
/// assert_eq!(my_crate::add(2, 3), 5);
/// ```
pub fn add(a: i32, b: i32) -> i32 { a + b }
```

### Test Attributes

| Attribute | Effect |
|-----------|--------|
| `#[test]` | Marks a test function |
| `#[should_panic]` | Passes if the test panics |
| `#[should_panic(expected = "msg")]` | Passes if panic message contains "msg" |
| `#[ignore]` | Skipped by default; run with `--ignored` |
| `#[cfg(test)]` | Only compiled during testing |

## What to Test

### Test Behavior, Not Implementation
```rust
// ❌ Tests implementation detail (internal data structure)
#[test]
fn internal_vec_has_three_items() {
    let cache = Cache::new();
    cache.insert("a", 1);
    assert_eq!(cache.items.len(), 1); // Coupling to internals
}

// ✅ Tests observable behavior
#[test]
fn inserted_item_is_retrievable() {
    let cache = Cache::new();
    cache.insert("a", 1);
    assert_eq!(cache.get("a"), Some(&1));
}
```

### Test Edges, Not Just Happy Path
- Empty inputs
- Boundary values (0, MAX, one-off)
- Malformed/corrupted data
- Missing resources
- Concurrent access
- Error paths (every `Err` variant should have a test that triggers it)

### Test Helpers
Factor out common setup into helper functions (not macros unless necessary):

```rust
#[cfg(test)]
mod tests {
    use super::*;

    fn test_server() -> Server {
        Server::builder()
            .port(0) // Random port
            .timeout(Duration::from_secs(1))
            .build()
    }

    fn sample_request() -> Request {
        Request::new("GET", "/health")
    }

    #[test]
    fn health_returns_200() {
        let server = test_server();
        let resp = server.handle(sample_request());
        assert_eq!(resp.status, 200);
    }
}
```

## Advanced Testing

### Property-Based Testing (proptest)
Instead of specific examples, declare properties that must hold for all inputs:

```rust
use proptest::prelude::*;

proptest! {
    #[test]
    fn encode_decode_roundtrip(msg in any::<u32>()) {
        let encoded = encode(msg);
        let decoded = decode(&encoded).unwrap();
        prop_assert_eq!(decoded, msg);
    }

    #[test]
    fn sort_preserves_length(mut v in prop::collection::vec(any::<i32>(), 0..100)) {
        let original_len = v.len();
        v.sort();
        prop_assert_eq!(v.len(), original_len);
    }
}
```

### When to Use Property Testing
- Serialization/deserialization roundtrips
- Parsers (parse → emit → re-parse should match)
- Sorting/searching algorithms
- Mathematical invariants
- State machines (sequence of operations → valid state)

### Fuzzing (cargo-fuzz)
Find crashes and panics with random input:

```rust
// fuzz/fuzz_targets/parse_input.rs
#![no_main]
use libfuzzer_sys::fuzz_target;

fuzz_target!(|data: &[u8]| {
    // Should not panic on any input
    let _ = my_crate::parse(data);
});
```

```bash
cargo +nightly fuzz run parse_input -- -max_len=1024
```

### When to Fuzz
- Parsers handling untrusted input
- Deserializers (network protocols, file formats)
- Any function that claims to handle arbitrary bytes
- Cryptographic implementations

### Miri — Undefined Behavior Detection
Detects UB that tests alone can't find:

```bash
# Install and run
rustup +nightly component add miri
cargo +nightly miri test
```

**Miri catches:**
- Use-after-free
- Out-of-bounds access
- Unaligned memory access
- Data races
- Invalid values in types with validity invariants
- Violations of stacked borrows

**Miri limitations:**
- Very slow (10-100x slower)
- Can't test all I/O operations
- Some FFI is unsupported
- Run on critical unsafe code, not the whole test suite

### cargo-careful
Enables stdlib debug assertions without the full Miri slowdown:

```bash
cargo install cargo-careful
cargo +nightly careful test
```

## Test Organization

### Naming Convention
Name tests by what they verify, not what they call:

```rust
// ❌ Describes the call
#[test] fn test_parse() { ... }

// ✅ Describes the behavior
#[test] fn parse_valid_input_returns_value() { ... }
#[test] fn parse_empty_string_returns_error() { ... }
#[test] fn parse_overflow_returns_too_large() { ... }
```

### Test Module Structure
```rust
#[cfg(test)]
mod tests {
    use super::*;

    // Constants for test data
    const VALID_KEY: [u8; 32] = [0xAA; 32];

    // Helper functions
    fn valid_input() -> Input { ... }

    // Group related tests logically
    // Happy path first, then edge cases, then error cases
    #[test] fn accepts_valid_input() { ... }
    #[test] fn handles_empty_input() { ... }
    #[test] fn rejects_malformed_input() { ... }
}
```

### Async Tests
```rust
#[tokio::test]
async fn connects_and_sends() {
    let server = spawn_test_server().await;
    let client = connect(server.addr()).await.unwrap();
    client.send(b"hello").await.unwrap();
}
```

## Benchmarking

### criterion
```rust
// benches/parsing.rs
use criterion::{black_box, criterion_group, criterion_main, Criterion};

fn bench_parse(c: &mut Criterion) {
    let input = include_bytes!("../testdata/sample.bin");
    c.bench_function("parse_sample", |b| {
        b.iter(|| parse(black_box(input)))
    });
}

criterion_group!(benches, bench_parse);
criterion_main!(benches);
```

### Benchmarking Rules
- Use `black_box()` to prevent the compiler from optimizing away the computation
- Benchmark realistic inputs, not trivial cases
- Run multiple times, compare distributions (criterion does this)
- Profile before optimizing — don't guess where time is spent
- Compare against a baseline: `cargo bench -- --save-baseline before`

## Linting

### Clippy
```bash
cargo clippy --all-targets --all-features -- -D warnings
```

- Run in CI as a gate (deny all warnings)
- Configure in `Cargo.toml` under `[lints.clippy]`
- Use `#[expect(clippy::lint_name)]` in test code for intentional violations (e.g., `unwrap_used`)
- Prefer `#[expect]` over `#[allow]` — `expect` warns if the lint no longer applies

## Common Mistakes

1. **Testing internals instead of behavior** — refactoring breaks tests that shouldn't break
2. **No edge case tests** — empty, zero, max, boundary values
3. **Ignoring error path testing** — every `Err` variant should have a test
4. **`tests/common.rs` instead of `tests/common/mod.rs`** — the former compiles as a test file
5. **Missing `#[cfg(test)]` on test helpers** — they end up in release builds
6. **Benchmarking without `black_box`** — compiler optimizes away the code
7. **Only happy-path property tests** — proptest strategies should cover edge cases too
8. **Skipping Miri on unsafe code** — tests alone can't find UB
