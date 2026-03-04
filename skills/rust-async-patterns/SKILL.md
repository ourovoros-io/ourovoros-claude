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
| Short critical section (no `.await` inside) | `std::sync::Mutex` or `parking_lot::Mutex` |
| Must hold lock across `.await` | `tokio::sync::Mutex` |
| Concurrent read-heavy map | `DashMap` |
| Multiple tasks, first wins | `tokio::select!` |
| Multiple tasks, all complete | `join!` or `JoinSet` |
| Many lightweight futures, single task | `FuturesUnordered` |
| Timeout | `tokio::time::timeout` |
| Graceful shutdown | `CancellationToken` + `TaskTracker` |
| Backpressure / rate limiting | `tokio::sync::Semaphore` |
| Signal one waiter | `tokio::sync::Notify` |
| Processing async sequences | `Stream` + `StreamExt` |
| Bridging sync caller to async | `Runtime::block_on` (outside runtime only) |
| Bridging async to sync work | `spawn_blocking` |

## Mutex Choice: std vs tokio

**Default to `std::sync::Mutex`** (or `parking_lot::Mutex`). Tokio's own documentation recommends this for most cases.

```rust
use std::sync::Mutex;

// CORRECT: std::sync::Mutex for short critical sections
async fn handle_request(state: Arc<Mutex<HashMap<String, String>>>) {
    let value = {
        let db = state.lock().expect("lock poisoned");
        db.get("key").cloned()
    }; // lock dropped BEFORE any .await

    if let Some(v) = value {
        send_response(v).await;
    }
}
```

**Use `tokio::sync::Mutex` only when you must hold the lock across an `.await`:**

```rust
use tokio::sync::Mutex;

// tokio::sync::Mutex -- ONLY when awaiting while locked
async fn send_message(stream: &Mutex<TcpStream>, msg: &[u8]) {
    let mut stream = stream.lock().await;
    stream.write_all(msg).await.expect("write failed");
    stream.flush().await.expect("flush failed");
}
```

**Key rule: never hold a `std::sync::Mutex` across an `.await` point.** The clippy lint `await_holding_lock` catches this.

**For high-contention short sections**, `parking_lot::Mutex` is faster than `std::sync::Mutex` (no poisoning, adaptive spinning).

## spawn vs spawn_blocking

```rust
// WRONG: Blocking code in async context
async fn process() {
    let hash = argon2::hash(password);  // Blocks runtime!
    send(hash).await;
}

// RIGHT: Use spawn_blocking for CPU work or sync I/O
async fn process() {
    let hash = tokio::task::spawn_blocking(move || {
        argon2::hash(password)
    }).await.expect("task panicked");
    send(hash).await;
}
```

**When to use `spawn_blocking`:**
- CPU-intensive work (hashing, compression, large serialization)
- Sync libraries doing I/O (rusqlite, filesystem ops)
- Any operation taking more than ~10-100 microseconds

**Note:** `tokio::fs` is just `spawn_blocking` internally. For high-throughput file I/O, consider `io_uring` via `tokio-uring`.

## Cancellation Safety

When `select!` resolves one branch, all other branches are **dropped**. If a dropped future was mid-operation, data may be lost.

**Cancellation-safe** (safe in `select!` loops):
- `mpsc::Receiver::recv()` -- message stays in channel
- `broadcast::Receiver::recv()`
- `TcpListener::accept()`
- `tokio::time::sleep()` / `interval::tick()`
- `AsyncReadExt::read()` -- reads only what's available

**NOT cancellation-safe** (data loss risk):
- `AsyncReadExt::read_exact()` -- partial bytes lost
- `AsyncBufReadExt::read_line()` -- partial line lost
- `AsyncReadExt::read_to_end()` / `read_to_string()`
- Any future doing multiple `.await`s with intermediate state

```rust
// WRONG: read_exact loses partial data on cancellation
loop {
    tokio::select! {
        result = reader.read_exact(&mut buf) => { process(&buf); }
        _ = shutdown.recv() => break,
    }
}

// RIGHT: use cancellation-safe read() with manual buffering
loop {
    tokio::select! {
        result = reader.read(&mut buf) => {
            match result {
                Ok(0) => break,
                Ok(n) => {
                    buffer.extend_from_slice(&buf[..n]);
                    // process complete messages from buffer
                }
                Err(e) => return Err(e.into()),
            }
        }
        _ = shutdown.recv() => break,
    }
}
```

## select! Patterns

### Biased mode for priority ordering

```rust
loop {
    tokio::select! {
        biased;
        // Check shutdown FIRST -- always honored promptly
        _ = shutdown.cancelled() => break,
        // High-priority work next
        msg = priority_rx.recv() => { handle_priority(msg).await; }
        // Normal work last
        msg = normal_rx.recv() => { handle_normal(msg).await; }
    }
}
```

Default `select!` randomizes branch order (prevents starvation). Use `biased;` when:
- Shutdown signals must be checked before more work
- Priority queues where ordering matters
- **Pitfall:** if the first branch is always ready, lower branches starve

### Borrowing in select!

```rust
// WRONG: can't borrow buf mutably in two branches
tokio::select! {
    n = reader1.read(&mut buf) => {}  // buf borrowed here
    n = reader2.read(&mut buf) => {}  // ERROR: already borrowed
}

// RIGHT: separate buffers
let (mut buf1, mut buf2) = ([0u8; 1024], [0u8; 1024]);
tokio::select! {
    n = reader1.read(&mut buf1) => { /* use buf1 */ }
    n = reader2.read(&mut buf2) => { /* use buf2 */ }
}
```

## Channels

| Channel | Use Case |
|---------|----------|
| `mpsc` (bounded) | Multi-producer, single-consumer with backpressure |
| `mpsc` (unbounded) | Fire-and-forget (avoid in production -- no backpressure) |
| `broadcast` | Multi-producer, multi-consumer, all get every message |
| `oneshot` | Single value, single consumer (request/response) |
| `watch` | Single value, multi-consumer, latest-only (config updates) |

```rust
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
```

## Graceful Shutdown

### CancellationToken + TaskTracker (recommended)

```rust
use tokio_util::sync::CancellationToken;
use tokio_util::task::TaskTracker;

async fn serve(listener: TcpListener, token: CancellationToken) {
    let tracker = TaskTracker::new();

    loop {
        tokio::select! {
            biased;
            _ = token.cancelled() => break,
            Ok((stream, addr)) = listener.accept() => {
                let child_token = token.child_token();
                tracker.spawn(async move {
                    handle_connection(stream, addr, child_token).await;
                });
            }
        }
    }

    // Phase 1: stop accepting (done -- broke the loop)
    // Phase 2: drain in-flight with timeout
    tracker.close();
    match tokio::time::timeout(Duration::from_secs(30), tracker.wait()).await {
        Ok(()) => tracing::info!("all connections drained"),
        Err(_) => tracing::warn!("shutdown timed out"),
    }
}
```

`CancellationToken` advantages over broadcast channels:
- Hierarchical (child tokens cancel when parent cancels)
- `cancelled()` is cancellation-safe
- Can check synchronously with `is_cancelled()`

## JoinSet vs FuturesUnordered

| | `JoinSet` | `FuturesUnordered` |
|---|-----------|-------------------|
| Execution | Spawns tasks on runtime (multi-threaded) | Runs on current task (single-threaded) |
| Overhead | Per-task spawn cost | Minimal (no spawn) |
| Cancellation | Tasks cancelled on `JoinSet` drop | Cancelled with parent future |
| Best for | I/O-bound concurrent work | Many lightweight transforms |

```rust
// JoinSet -- for spawned concurrent work
let mut set = JoinSet::new();
for url in urls {
    set.spawn(async move { fetch(url).await });
}
while let Some(result) = set.join_next().await {
    match result {
        Ok(response) => results.push(response),
        Err(join_err) => tracing::error!(?join_err, "task panicked"),
    }
}

// FuturesUnordered -- for lightweight futures without spawn
use futures::stream::{FuturesUnordered, StreamExt};
let mut futs = FuturesUnordered::new();
for item in items {
    futs.push(async move { transform(item).await });
}
while let Some(result) = futs.next().await {
    process(result);
}
```

## Bridging Sync and Async

### Calling async from sync (outside runtime)

```rust
struct SyncClient {
    rt: tokio::runtime::Runtime,
    inner: AsyncClient,
}

impl SyncClient {
    pub fn new() -> Result<Self, Error> {
        let rt = tokio::runtime::Runtime::new()?;
        let inner = rt.block_on(AsyncClient::connect())?;
        Ok(Self { rt, inner })
    }

    pub fn query(&self, sql: &str) -> Result<Rows, Error> {
        self.rt.block_on(self.inner.query(sql))
    }
}
```

**Never call `block_on` from inside a Tokio worker thread** -- it panics.

## Cooperative Scheduling and Starvation

Tokio uses cooperative scheduling -- tasks must yield at `.await` points. CPU work between awaits starves other tasks.

```rust
// WRONG: CPU work with no yield points
async fn process_large_file(data: Vec<u8>) -> Stats {
    let mut stats = Stats::default();
    for chunk in data.chunks(1024) {
        stats.update(expensive_computation(chunk)); // starves runtime
    }
    stats
}

// RIGHT: use spawn_blocking for bulk CPU work
async fn process_large_file(data: Vec<u8>) -> Stats {
    tokio::task::spawn_blocking(move || {
        let mut stats = Stats::default();
        for chunk in data.chunks(1024) {
            stats.update(expensive_computation(chunk));
        }
        stats
    }).await.expect("task panicked")
}
```

For mixed async+CPU work, insert `tokio::task::yield_now().await` periodically. Use `tokio-console` to diagnose task starvation (long poll times).

## async fn in Traits (Rust 1.75+)

Native async trait methods work for static dispatch:

```rust
trait Storage {
    async fn get(&self, key: &str) -> Option<Vec<u8>>;
    async fn put(&self, key: &str, value: &[u8]) -> Result<(), Error>;
}
```

**Limitations:**
- **Not object-safe:** Cannot use `dyn Storage` -- use `async_trait` crate or `trait_variant::make` for dynamic dispatch
- **No implicit `Send` bound:** Returned futures are not `Send` by default -- use `trait_variant::make(SendStorage: Send)` or manual desugaring for `tokio::spawn`

## Async Drop Does Not Exist

The `Drop` trait is synchronous. Use explicit shutdown methods:

```rust
impl Connection {
    /// Must be called before dropping for clean shutdown.
    async fn close(self) -> Result<(), Error> {
        self.inner.shutdown().await
    }
}

impl Drop for Connection {
    fn drop(&mut self) {
        if !self.closed {
            tracing::warn!("Connection dropped without calling close()");
        }
    }
}
```

## When NOT to Use Async

- **CPU-bound work** -- async adds state machine overhead for no benefit
- **Simple CLI tools** -- `ureq` (sync) is simpler than `reqwest` + tokio
- **Hard real-time** -- cooperative scheduling cannot guarantee latency
- **Sync-only ecosystem** -- wrapping everything in `spawn_blocking` adds complexity

Honest costs: larger binaries (~2-4 MB from tokio), slower compilation, harder debugging, more complex error messages.

## Common Mistakes

| Mistake | Issue | Fix |
|---------|-------|-----|
| `std::sync::Mutex` held across `.await` | Blocks runtime thread | Scope the guard, or use `tokio::sync::Mutex` |
| `tokio::sync::Mutex` for short sections | Unnecessary overhead | Use `std::sync::Mutex` or `parking_lot` |
| `.await` in `spawn_blocking` | Can't await in sync | Return value, await outside |
| Ignoring `JoinHandle` | Task may be cancelled | `.await` or store in `JoinSet` |
| `block_on` inside async | Panic | Use `.await` or restructure |
| Unbounded channels | Memory leak under load | Bounded with backpressure |
| No backpressure on spawned tasks | OOM under load | `Semaphore` or bounded channel |
| Non-cancellation-safe methods in `select!` | Data loss | Use `read()` not `read_exact()` |
| CPU work without yield points | Starves other tasks | `spawn_blocking` or `yield_now()` |

## Essential Crates

- `tokio` -- Async runtime
- `tokio-util` -- CancellationToken, TaskTracker, codec utilities
- `tokio-stream` -- Stream wrappers and combinators
- `dashmap` -- Lock-free concurrent HashMap
- `parking_lot` -- Faster synchronous mutexes
- `futures` -- Stream utilities, FuturesUnordered
- `tracing` -- Async-aware structured logging
- `tokio-console` -- Runtime debugger (task starvation, waker counts)
