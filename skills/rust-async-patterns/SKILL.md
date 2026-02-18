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
| Shared mutable state (concurrent) | `DashMap` or `Arc<RwLock<T>>` |
| Shared mutable state (simple) | `Arc<Mutex<T>>` or channels |
| Multiple tasks, first wins | `tokio::select!` |
| Multiple tasks, all complete | `join!` or `JoinSet` |
| Timeout | `tokio::time::timeout` |
| Graceful shutdown | `CancellationToken` |
| Backpressure / rate limiting | `tokio::sync::Semaphore` |
| Signal one waiter | `tokio::sync::Notify` |
| Structured concurrency | `TaskTracker` from `tokio-util` |
| Processing async sequences | `Stream` + `StreamExt` |

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
    let mut set = JoinSet::new();
    for url in urls {
        set.spawn(fetch(url));
    }

    let mut results = Vec::new();
    while let Some(result) = set.join_next().await {
        if let Ok(response) = result {
            results.push(response);
        }
    }
    results
}
```

### Semaphore for Backpressure

```rust
use std::sync::Arc;
use tokio::sync::Semaphore;

struct Gateway {
    upload_semaphore: Arc<Semaphore>,
    download_semaphore: Arc<Semaphore>,
}

impl Gateway {
    fn new() -> Self {
        Self {
            upload_semaphore: Arc::new(Semaphore::new(500)),
            download_semaphore: Arc::new(Semaphore::new(1000)),
        }
    }

    async fn handle_upload(&self, data: Bytes) -> Result<(), Error> {
        let _permit = self.upload_semaphore
            .acquire()
            .await
            .map_err(|_| Error::ShuttingDown)?;

        // Permit auto-drops when this scope exits
        do_upload(data).await
    }
}
```

### DashMap for Lock-Free Concurrent State

```rust
use dashmap::DashMap;

struct Registry {
    miners: DashMap<NodeId, MinerInfo>,
}

impl Registry {
    fn update_miner(&self, id: NodeId, info: MinerInfo) {
        self.miners.insert(id, info);
    }

    fn get_miner(&self, id: &NodeId) -> Option<MinerInfo> {
        self.miners.get(id).map(|r| r.value().clone())
    }

    fn remove_stale(&self, threshold: u64) {
        self.miners.retain(|_, v| v.last_seen > threshold);
    }
}
```

Prefer `DashMap` over `Arc<RwLock<HashMap>>` for concurrent read-heavy workloads. Use `Arc<RwLock<HashMap>>` when you need to lock multiple keys atomically.

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
                tracing::info!("shutting down");
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
    let (tx, mut rx) = mpsc::channel(100);  // Bounded for backpressure

    tokio::spawn(async move {
        for i in 0..10 {
            if tx.send(i).await.is_err() {
                break;  // Receiver dropped
            }
        }
    });

    while let Some(value) = rx.recv().await {
        tracing::debug!(value, "received");
    }
}
```

| Channel | Use Case |
|---------|----------|
| `mpsc` (bounded) | Multi-producer, single-consumer with backpressure |
| `mpsc` (unbounded) | Fire-and-forget (avoid in production — no backpressure) |
| `broadcast` | Multi-producer, multi-consumer, all get every message |
| `oneshot` | Single value, single consumer (request/response) |
| `watch` | Single value, multi-consumer, latest-only (config updates) |

### Notify for Signaling

```rust
use std::sync::Arc;
use tokio::sync::Notify;

struct WorkQueue {
    notify: Arc<Notify>,
    // ...
}

impl WorkQueue {
    async fn push(&self, item: Work) {
        // Add item to queue...
        self.notify.notify_one();  // Wake one waiter
    }

    async fn wait_for_work(&self) -> Work {
        loop {
            if let Some(item) = self.try_pop() {
                return item;
            }
            self.notify.notified().await;
        }
    }
}
```

### Shared State with RwLock

```rust
use std::sync::Arc;
use tokio::sync::RwLock;

struct SharedState {
    data: Arc<RwLock<HashMap<String, String>>>,
}

impl SharedState {
    async fn get(&self, key: &str) -> Option<String> {
        let guard = self.data.read().await;  // Multiple readers OK
        guard.get(key).cloned()
    }

    async fn set(&self, key: String, value: String) {
        let mut guard = self.data.write().await;  // Exclusive write
        guard.insert(key, value);
    }  // Lock dropped here — never hold across .await
}
```

### Cancellation Safety

```rust
use tokio_util::sync::CancellationToken;

async fn cancellable_work(token: CancellationToken) {
    loop {
        select! {
            _ = do_work() => {}
            _ = token.cancelled() => {
                cleanup().await;
                return;
            }
        }
    }
}

let token = CancellationToken::new();
let task = tokio::spawn(cancellable_work(token.clone()));
// Later:
token.cancel();
task.await?;
```

### TaskTracker for Structured Concurrency

```rust
use tokio_util::task::TaskTracker;

async fn serve(listener: TcpListener, shutdown: CancellationToken) {
    let tracker = TaskTracker::new();

    loop {
        select! {
            Ok((stream, _)) = listener.accept() => {
                let token = shutdown.clone();
                tracker.spawn(async move {
                    handle_connection(stream, token).await;
                });
            }
            _ = shutdown.cancelled() => break,
        }
    }

    tracker.close();
    tracker.wait().await;  // Wait for all connections to finish
    tracing::info!("all connections drained");
}
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
        match result {
            Ok(output) => results.push(output),
            Err(join_err) => tracing::error!(?join_err, "task panicked"),
        }
    }
    results
}
```

### Stream Processing

```rust
use tokio_stream::{StreamExt, wrappers::ReceiverStream};

async fn process_stream(rx: mpsc::Receiver<Event>) {
    let stream = ReceiverStream::new(rx);

    // Process with backpressure — only pulls when ready
    let mut stream = stream
        .filter(|e| e.is_relevant())
        .chunks_timeout(100, Duration::from_secs(1));

    while let Some(batch) = stream.next().await {
        process_batch(batch).await;
    }
}
```

## Common Mistakes

| Mistake | Issue | Fix |
|---------|-------|-----|
| `std::sync::Mutex` in async | Blocks runtime | `tokio::sync::Mutex` or `DashMap` |
| `.await` in `spawn_blocking` | Can't await in sync | Return value, await outside |
| Holding lock across `.await` | Deadlock risk | Clone data, drop lock before await |
| Ignoring `JoinHandle` | Task may be cancelled | `.await` or store in `JoinSet` |
| `block_on` inside async | Panic | Use `.await` or restructure |
| Unbounded channels | Memory leak under load | Bounded with backpressure |
| `Arc<RwLock<HashMap>>` for concurrent reads | Lock contention | `DashMap` |
| No backpressure on spawned tasks | OOM under load | `Semaphore` or bounded channel |

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
- `tokio-util` - CancellationToken, TaskTracker, codec utilities
- `tokio-stream` - Stream wrappers and combinators
- `dashmap` - Lock-free concurrent HashMap
- `futures` - Stream utilities, join_all
- `tracing` - Async-aware structured logging

**Note:** `async-trait` is largely unnecessary since Rust 1.75+ supports `async fn` in traits natively. Only needed for `dyn Trait` dispatch with async methods.
