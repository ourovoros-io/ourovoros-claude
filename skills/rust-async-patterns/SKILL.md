---
name: rust-async-patterns
description: Use when writing async Rust code with Tokio, debugging deadlocks, cancelled futures, or runtime panics. Use when choosing between spawn vs spawn_blocking, using select! or channels, or designing concurrent systems.
---

# Rust Async Patterns

## Overview

Write correct async Rust with Tokio. Avoid common pitfalls like blocking the runtime, cancellation issues, and deadlocks through proper patterns.

## When to Use

- Writing new async code
- Debugging hung or slow async operations
- Choosing between spawn, spawn_blocking, channels
- Designing concurrent task systems
- Handling graceful shutdown
- Async code panics or behaves unexpectedly

**Not for:** Sync concurrency with threads (use std threading), performance tuning (use `rust-performance`)

## Quick Reference

| Need | Pattern |
|------|---------|
| CPU-bound work | `spawn_blocking` |
| I/O-bound work | `spawn` |
| Shared state | `Arc<Mutex<T>>` or channels |
| Multiple tasks, first wins | `tokio::select!` |
| Multiple tasks, all complete | `join!` or `JoinSet` |
| Timeout | `tokio::time::timeout` |
| Graceful shutdown | `CancellationToken` |

## Core Patterns

### spawn vs spawn_blocking

```rust
// WRONG: Blocking code in async context
async fn process() {
    let result = expensive_computation();  // Blocks runtime!
    send(result).await;
}

// RIGHT: Use spawn_blocking for CPU work
async fn process() {
    let result = tokio::task::spawn_blocking(|| {
        expensive_computation()
    }).await?;
    send(result).await;
}

// RIGHT: spawn for I/O-bound async work
async fn fetch_all(urls: Vec<String>) -> Vec<Response> {
    let handles: Vec<_> = urls
        .into_iter()
        .map(|url| tokio::spawn(fetch(url)))
        .collect();

    futures::future::join_all(handles)
        .await
        .into_iter()
        .filter_map(Result::ok)
        .collect()
}
```

### select! for Racing

```rust
use tokio::select;
use tokio::time::{sleep, Duration};

async fn fetch_with_timeout(url: &str) -> Result<Response, Error> {
    select! {
        result = fetch(url) => result,
        _ = sleep(Duration::from_secs(10)) => Err(Error::Timeout),
    }
}

// Handle shutdown signal
async fn run_server(shutdown: CancellationToken) {
    loop {
        select! {
            conn = listener.accept() => {
                handle_connection(conn).await;
            }
            _ = shutdown.cancelled() => {
                println!("Shutting down");
                break;
            }
        }
    }
}
```

### Channels for Communication

```rust
use tokio::sync::mpsc;

async fn producer_consumer() {
    let (tx, mut rx) = mpsc::channel(100);

    // Producer
    tokio::spawn(async move {
        for i in 0..10 {
            tx.send(i).await.unwrap();
        }
    });

    // Consumer
    while let Some(value) = rx.recv().await {
        println!("Got: {value}");
    }
}

// Bounded for backpressure, unbounded for fire-and-forget
// mpsc: multi-producer, single-consumer
// broadcast: multi-producer, multi-consumer
// oneshot: single value, single consumer
// watch: single value, multi-consumer, latest only
```

### Shared State

```rust
use std::sync::Arc;
use tokio::sync::Mutex;

// Tokio Mutex for async-aware locking
struct SharedState {
    data: Arc<Mutex<HashMap<String, String>>>,
}

impl SharedState {
    async fn get(&self, key: &str) -> Option<String> {
        let guard = self.data.lock().await;
        guard.get(key).cloned()
    }

    async fn set(&self, key: String, value: String) {
        let mut guard = self.data.lock().await;
        guard.insert(key, value);
    }
}

// For read-heavy workloads
use tokio::sync::RwLock;
let data = Arc::new(RwLock::new(HashMap::new()));
let read_guard = data.read().await;  // Multiple readers OK
let write_guard = data.write().await;  // Exclusive write
```

### Cancellation Safety

```rust
use tokio_util::sync::CancellationToken;

async fn cancellable_work(token: CancellationToken) {
    loop {
        select! {
            _ = do_work() => {}
            _ = token.cancelled() => {
                // Cleanup before exit
                cleanup().await;
                return;
            }
        }
    }
}

// In main
let token = CancellationToken::new();
let task = tokio::spawn(cancellable_work(token.clone()));

// Later, to cancel:
token.cancel();
task.await?;
```

### JoinSet for Dynamic Tasks

```rust
use tokio::task::JoinSet;

async fn process_batch(items: Vec<Item>) -> Vec<Result<Output, Error>> {
    let mut set = JoinSet::new();

    for item in items {
        set.spawn(async move { process(item).await });
    }

    let mut results = Vec::new();
    while let Some(result) = set.join_next().await {
        results.push(result.unwrap());
    }
    results
}
```

## Common Mistakes

| Mistake | Issue | Fix |
|---------|-------|-----|
| `std::sync::Mutex` in async | Blocks runtime | Use `tokio::sync::Mutex` |
| `.await` in `spawn_blocking` | Can't await in sync context | Return value, await outside |
| Holding lock across await | Deadlock risk | Clone data, drop lock early |
| Ignoring `JoinHandle` | Task may be cancelled | `.await` or `abort()` explicitly |
| `block_on` inside async | Panic | Use `.await` or restructure |
| Unbounded channels | Memory leak | Use bounded with backpressure |

## Runtime Configuration

```rust
#[tokio::main]
async fn main() {
    // Default multi-threaded runtime
}

#[tokio::main(flavor = "current_thread")]
async fn main() {
    // Single-threaded, useful for testing
}

// Manual configuration
let runtime = tokio::runtime::Builder::new_multi_thread()
    .worker_threads(4)
    .enable_all()
    .build()?;
```

## Essential Crates

- `tokio` - Async runtime
- `tokio-util` - CancellationToken, codec utilities
- `futures` - Stream utilities, join_all
- `async-trait` - Async methods in traits
- `tracing` - Async-aware logging
