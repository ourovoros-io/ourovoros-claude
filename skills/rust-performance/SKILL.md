---
name: rust-performance
description: Use when optimizing Rust code performance, profiling with flamegraph, writing benchmarks with criterion, or when code is unexpectedly slow. Use when reducing allocations, choosing data structures, or tuning release profiles.
---

# Rust Performance

## Overview

Optimize Rust code systematically: measure first, identify bottlenecks, optimize with data. Avoid premature optimization — most code doesn't need tuning.

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
| UB/safety checks | `cargo careful test` |
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

fn bench_placement(c: &mut Criterion) {
    let mut group = c.benchmark_group("placement");

    group.bench_function("crush_v2", |b| {
        b.iter(|| calculate_placement(black_box(&cluster_map), black_box(file_hash)))
    });

    group.bench_function("crush_v1", |b| {
        b.iter(|| calculate_placement_v1(black_box(&cluster_map), black_box(file_hash)))
    });

    group.finish();
}

criterion_group!(benches, bench_placement);
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

### Zero-Copy with Bytes

```rust
use bytes::{Bytes, BytesMut, BufMut};

// BAD: Copies data through Vec<u8>
fn process_data(data: &[u8]) -> Vec<u8> {
    let mut output = data.to_vec();  // Allocation + copy
    transform(&mut output);
    output
}

// GOOD: Zero-copy with Bytes (reference-counted, cheaply cloneable)
fn process_data(data: Bytes) -> Bytes {
    // .clone() is O(1) — just increments refcount
    // .slice() is O(1) — returns a view
    data.slice(HEADER_LEN..)
}

// Building up data with BytesMut
fn encode_frame(payload: &[u8]) -> Bytes {
    let mut buf = BytesMut::with_capacity(4 + payload.len());
    buf.put_u32(payload.len() as u32);
    buf.put_slice(payload);
    buf.freeze()  // BytesMut -> Bytes (immutable, shareable)
}
```

Use `Bytes` when data is read-only and shared across tasks. Use `BytesMut` when building up data. Both avoid copies that `Vec<u8>` would require.

### Avoid Allocations

```rust
// SLOW: Allocates on each call
fn process(items: &[Item]) -> Vec<String> {
    let mut result = Vec::new();
    for item in items {
        result.push(item.name.clone());
    }
    result
}

// FAST: Reuse allocation
fn process_reuse(items: &[Item], output: &mut Vec<String>) {
    output.clear();
    for item in items {
        output.push(item.name.clone());
    }
}

// FAST: Pre-allocate
fn process_prealloc(items: &[Item]) -> Vec<String> {
    let mut result = Vec::with_capacity(items.len());
    for item in items {
        result.push(item.name.clone());
    }
    result
}

// FAST: Return references instead of cloning
fn names<'a>(items: &'a [Item]) -> Vec<&'a str> {
    let mut result = Vec::with_capacity(items.len());
    for item in items {
        result.push(item.name.as_str());
    }
    result
}
```

### Choose Right Data Structure

| Access Pattern | Data Structure | Notes |
|---------------|----------------|-------|
| Lookup-heavy | `HashMap` / `DashMap` | O(1) average |
| Ordered iteration | `BTreeMap` | O(log n) |
| Small collections (<100) | `Vec` | Cache-friendly linear scan |
| Stack-like | `Vec` with push/pop | O(1) amortized |
| Queue-like | `VecDeque` | O(1) push/pop both ends |
| Concurrent reads | `DashMap` | Lock-free sharded |
| Bounded cache | `quick_cache::Cache` | LRU with size limit |

### Reduce Cloning

```rust
// SLOW: Clone everything
fn process(data: &Data) {
    let name = data.name.clone();
    let items = data.items.clone();
}

// FAST: Borrow
fn process(data: &Data) {
    let name = &data.name;
}

// FAST: Move / destructure
fn process_owned(data: Data) {
    let Data { name, items, .. } = data;
}

// Cow for maybe-clone
use std::borrow::Cow;
fn normalize(input: &str) -> Cow<str> {
    if input.contains('\t') {
        Cow::Owned(input.replace('\t', "    "))
    } else {
        Cow::Borrowed(input)
    }
}
```

### Parallelism with Rayon

```rust
use rayon::prelude::*;

// Sequential
let mut results = Vec::new();
for item in &items {
    results.push(item.compute());
}

// Parallel (only when work > ~1μs per item and > 1000 items)
let results: Vec<_> = items.par_iter().map(|x| x.compute()).collect();
```

## Profiling Commands

```bash
# CPU flamegraph (Linux)
cargo flamegraph --bin myapp

# Memory profiling
heaptrack ./target/release/myapp
valgrind --tool=massif ./target/release/myapp

# UB and safety checks (slower but catches bugs)
cargo careful test

# Compile times
cargo build --timings

# Binary size analysis
cargo bloat --release
cargo bloat --release --crates

# Target-specific optimizations
RUSTFLAGS="-C target-cpu=native" cargo build --release
```

## Release Profile Tuning

```toml
# Cargo.toml
[profile.release]
lto = true           # Link-time optimization
codegen-units = 1    # Better optimization, slower compile
panic = "abort"      # Smaller binary, no unwinding
strip = true         # Strip debug symbols

[profile.release-fast]
inherits = "release"
opt-level = 3

[profile.release-small]
inherits = "release"
opt-level = "z"      # Optimize for size
```

## Common Mistakes

| Mistake | Issue | Fix |
|---------|-------|-----|
| Optimizing without measuring | Wasted effort | Profile first |
| Debug builds for perf | 10-100x slower | Use `--release` |
| `clone()` in hot loop | Allocation overhead | Borrow or restructure |
| `Vec` for FIFO | O(n) remove(0) | Use `VecDeque` |
| `HashMap` for tiny data | Hash overhead | `Vec` + linear search |
| `Box<dyn Trait>` in hot path | Vtable indirection | Use enum or generics |
| `Vec<u8>` for shared buffers | Copy on every pass | Use `Bytes` |
| Collecting then iterating | Extra allocation | Chain operations |
| `Arc<RwLock<HashMap>>` contention | Lock bouncing | `DashMap` |

## Essential Crates

- `criterion` - Statistical benchmarking
- `bytes` - Zero-copy byte buffers
- `rayon` - Data parallelism
- `dashmap` - Lock-free concurrent HashMap
- `quick_cache` - Bounded concurrent LRU cache
- `parking_lot` - Faster mutexes
- `smallvec` - Stack allocation for small vecs
- `compact_str` - Small string optimization
- `bumpalo` - Arena allocator
