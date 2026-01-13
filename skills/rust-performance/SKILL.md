---
name: rust-performance
description: Use when optimizing Rust code performance, profiling with flamegraph, writing benchmarks with criterion, or when code is unexpectedly slow. Use when reducing allocations, choosing data structures, or tuning release profiles.
---

# Rust Performance

## Overview

Optimize Rust code systematically: measure first, identify bottlenecks, optimize with data. Avoid premature optimization - most code doesn't need tuning.

## When to Use

- Code is measurably too slow
- Writing performance-critical paths
- Choosing between data structures
- Setting up benchmarks
- Profiling CPU or memory usage
- Unexpected performance regression

**Not for:** General code improvement (use `rust-code-clarity`), first implementation (optimize later)

## Quick Reference

| Goal | Tool/Approach |
|------|---------------|
| Benchmark code | `criterion` crate |
| CPU profiling | `perf`, `flamegraph` |
| Memory profiling | `heaptrack`, `valgrind` |
| Allocation tracking | `dhat`, `#[global_allocator]` |
| Compile-time | `cargo build --timings` |
| Binary size | `cargo bloat`, `twiggy` |

## Performance Workflow

```
1. Measure (don't guess)
   ↓
2. Identify bottleneck
   ↓
3. Hypothesize improvement
   ↓
4. Implement change
   ↓
5. Measure again
   ↓
6. Keep or revert
```

## Benchmarking with Criterion

```rust
// benches/my_benchmark.rs
use criterion::{black_box, criterion_group, criterion_main, Criterion};

fn fibonacci(n: u64) -> u64 {
    match n {
        0 => 0,
        1 => 1,
        _ => fibonacci(n - 1) + fibonacci(n - 2),
    }
}

fn bench_fibonacci(c: &mut Criterion) {
    c.bench_function("fib 20", |b| {
        b.iter(|| fibonacci(black_box(20)))
    });
}

// Compare implementations
fn bench_compare(c: &mut Criterion) {
    let mut group = c.benchmark_group("string-concat");

    group.bench_function("format", |b| {
        b.iter(|| format!("{}{}", black_box("hello"), black_box("world")))
    });

    group.bench_function("push_str", |b| {
        b.iter(|| {
            let mut s = String::from(black_box("hello"));
            s.push_str(black_box("world"));
            s
        })
    });

    group.finish();
}

criterion_group!(benches, bench_fibonacci, bench_compare);
criterion_main!(benches);
```

```toml
# Cargo.toml
[dev-dependencies]
criterion = "0.5"

[[bench]]
name = "my_benchmark"
harness = false
```

## Common Optimizations

### Avoid Allocations

```rust
// SLOW: Allocates on each call
fn process(items: &[Item]) -> Vec<String> {
    items.iter().map(|i| i.name.clone()).collect()
}

// FAST: Reuse allocation
fn process_reuse(items: &[Item], output: &mut Vec<String>) {
    output.clear();
    output.extend(items.iter().map(|i| i.name.clone()));
}

// FAST: Return iterator instead of Vec
fn process_iter(items: &[Item]) -> impl Iterator<Item = &str> {
    items.iter().map(|i| i.name.as_str())
}
```

### Choose Right Data Structure

```rust
// Lookup heavy → HashMap/HashSet (O(1) average)
// Ordered iteration → BTreeMap/BTreeSet (O(log n))
// Small collections (<100) → Vec (cache friendly)
// Stack-like → Vec with push/pop
// Queue-like → VecDeque
// Frequent insert/remove middle → LinkedList (rare!)

// Small string optimization
use compact_str::CompactString;  // Inline <=24 bytes
use smol_str::SmolStr;  // Inline <=22 bytes, immutable
```

### Reduce Cloning

```rust
// SLOW: Clone everything
fn process(data: Data) -> Result<Output, Error> {
    let name = data.name.clone();
    let items = data.items.clone();
    // ...
}

// FAST: Borrow or move
fn process(data: &Data) -> Result<Output, Error> {
    let name = &data.name;  // Borrow
    // ...
}

fn process_owned(data: Data) -> Result<Output, Error> {
    let Data { name, items, .. } = data;  // Move/destructure
    // ...
}

// Use Cow for maybe-clone
use std::borrow::Cow;
fn process_cow(input: &str) -> Cow<str> {
    if needs_modification(input) {
        Cow::Owned(modify(input))
    } else {
        Cow::Borrowed(input)
    }
}
```

### Parallelism with Rayon

```rust
use rayon::prelude::*;

// Sequential
let sum: i64 = items.iter().map(|x| x.compute()).sum();

// Parallel (drop-in replacement)
let sum: i64 = items.par_iter().map(|x| x.compute()).sum();

// Only parallelize if work is substantial
// Rule of thumb: >1μs per item, >1000 items
```

## Profiling Commands

```bash
# CPU flamegraph (Linux)
cargo install flamegraph
cargo flamegraph --bin myapp

# Memory profiling
valgrind --tool=massif ./target/release/myapp
heaptrack ./target/release/myapp

# Compile times
cargo build --timings
cargo +nightly build -Z timings

# Binary size analysis
cargo install cargo-bloat
cargo bloat --release
cargo bloat --release --crates
```

## Common Mistakes

| Mistake | Issue | Fix |
|---------|-------|-----|
| Optimizing without measuring | Wasted effort | Profile first |
| Debug builds for perf | 10-100x slower | Use `--release` |
| `clone()` in hot loop | Allocation overhead | Borrow or restructure |
| `Vec` for FIFO | O(n) remove(0) | Use `VecDeque` |
| `HashMap` for tiny data | Hash overhead | Use `Vec` + linear search |
| Box<dyn Trait> in hot path | Vtable indirection | Use enum or generics |
| Collecting to Vec then iterating | Extra allocation | Chain iterators |

## Release Profile Tuning

```toml
# Cargo.toml
[profile.release]
lto = true           # Link-time optimization
codegen-units = 1    # Better optimization, slower compile
panic = "abort"      # Smaller binary, no unwinding

[profile.release-fast]
inherits = "release"
opt-level = 3

[profile.release-small]
inherits = "release"
opt-level = "z"      # Optimize for size
strip = true
```

## Essential Crates

- `criterion` - Statistical benchmarking
- `rayon` - Data parallelism
- `parking_lot` - Faster mutexes
- `hashbrown` - Faster HashMap (now in std)
- `smallvec` - Stack allocation for small vecs
- `compact_str` - Small string optimization
- `bumpalo` - Arena allocator
