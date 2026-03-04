---
name: rfr-async-programming
description: Use when writing async Rust code, understanding Futures, Pin, Waker mechanics, async/await desugaring, or async runtime internals. Use when debugging cancelled futures, pinning errors, blocking in async, or choosing between spawn and spawn_blocking.
---

# Rust for Rustaceans — Ch 8: Asynchronous Programming

## The Async Model

### What `async` Actually Does
- `async fn` returns an `impl Future<Output = T>` — it does NOT run the body
- The body executes only when the future is `.await`ed or polled
- Each `.await` point is a yield point — the future can be suspended and resumed
- The compiler transforms async fn into a state machine (enum with one variant per `.await` point)

```rust
// This:
async fn fetch(url: &str) -> String {
    let response = reqwest::get(url).await.unwrap();
    response.text().await.unwrap()
}

// Becomes roughly:
enum FetchFuture<'a> {
    Start { url: &'a str },
    WaitingForGet { fut: reqwest::ResponseFuture },
    WaitingForText { fut: reqwest::TextFuture },
    Done,
}
// + impl Future for FetchFuture
```

### The `Future` Trait
```rust
pub trait Future {
    type Output;
    fn poll(
        self: Pin<&mut Self>,
        cx: &mut Context<'_>,
    ) -> Poll<Self::Output>;
}

pub enum Poll<T> {
    Ready(T),
    Pending,
}
```

- `poll()` is called by the runtime to drive the future
- Returns `Ready(value)` when done, `Pending` when waiting
- Must NOT be called again after returning `Ready` (fused)
- The future must register a waker via `cx.waker()` before returning `Pending`

### Waker
- **Purpose**: tells the runtime "this future is ready to make progress, poll me again"
- When an I/O operation completes, the waker is invoked
- The runtime then re-polls the future
- Without calling the waker, the future will never be polled again

```rust
impl Future for MyFuture {
    type Output = i32;
    fn poll(
        self: Pin<&mut Self>,
        cx: &mut Context<'_>,
    ) -> Poll<i32> {
        if self.is_ready() {
            Poll::Ready(self.value)
        } else {
            // Register waker so we'll be polled again
            self.register_waker(cx.waker().clone());
            Poll::Pending
        }
    }
}
```

## Pin

### Why Pin Exists
- Async state machines may contain self-references (a field referencing another field)
- Moving a self-referential struct invalidates the internal pointers
- `Pin<&mut T>` guarantees the value will not be moved in memory

### `Pin` and `Unpin`

| Type | `Unpin`? | `Pin` behavior |
|------|----------|----------------|
| `i32`, `String`, most types | Yes | `Pin` is a no-op, freely movable |
| `async {}` blocks/futures | No | Must be pinned, cannot be moved after first poll |
| `PhantomPinned` | No | Explicitly opts out of `Unpin` |

### Pinning in Practice
```rust
use std::pin::Pin;
use tokio::pin;

// Stack pinning with tokio::pin!
let fut = async { 42 };
tokio::pin!(fut);
let result = fut.await;

// Heap pinning with Box::pin
let fut: Pin<Box<dyn Future<Output = i32>>> = Box::pin(async { 42 });

// Pin projection — accessing fields of a pinned struct
// Use the `pin-project` crate for safe projections
```

### Pin Rules
1. Once pinned, a `!Unpin` value must never be moved
2. `Pin<&mut T>` prevents `mem::swap`, `mem::replace`, `std::mem::take`
3. `Unpin` types can always be safely unpinned (most types)
4. Use `Box::pin()` for heap pinning, `tokio::pin!()` for stack pinning
5. `pin-project` crate handles pin projection safely

## Async Patterns

### Spawning Tasks
```rust
// Spawn a concurrent task (runs on the runtime's thread pool)
let handle = tokio::spawn(async move {
    expensive_work().await
});
let result = handle.await?;

// spawn_blocking for CPU-bound or blocking I/O
let result = tokio::task::spawn_blocking(|| {
    std::fs::read_to_string("large_file.txt")
}).await?;
```

### `spawn` vs `spawn_blocking`

| | `tokio::spawn` | `spawn_blocking` |
|---|---|---|
| For | Async I/O-bound work | CPU-bound or blocking I/O |
| Runs on | Async thread pool | Dedicated blocking pool |
| Blocks runtime? | YES if you do sync I/O | No |
| Requires `Send` | Yes | Yes |

### `select!` — Race Multiple Futures
```rust
use tokio::select;

select! {
    result = fetch_data() => {
        handle_data(result);
    }
    _ = tokio::time::sleep(Duration::from_secs(5)) => {
        handle_timeout();
    }
}
```

**Cancellation safety**: When one branch completes, others are dropped. If a future is not cancellation-safe, work done before the last `.await` is lost.

### `join!` — Run Concurrently, Wait for All
```rust
use tokio::join;

let (users, posts, comments) = join!(
    fetch_users(),
    fetch_posts(),
    fetch_comments(),
);
```

### Streams (Async Iterators)
```rust
use tokio_stream::StreamExt;

let mut stream = tokio_stream::iter(vec![1, 2, 3]);
while let Some(value) = stream.next().await {
    process(value);
}
```

## Async Pitfalls

### 1. Blocking in Async Context
**Problem**: Calling blocking functions (file I/O, `thread::sleep`, heavy computation) in async code blocks the entire runtime thread.

```rust
// ❌ Blocks the async runtime thread
async fn bad() {
    std::thread::sleep(Duration::from_secs(1)); // Blocks!
    std::fs::read_to_string("file.txt");        // Blocks!
}

// ✅ Use async equivalents or spawn_blocking
async fn good() {
    tokio::time::sleep(Duration::from_secs(1)).await;
    tokio::fs::read_to_string("file.txt").await;
}
```

### 2. Holding Locks Across `.await`
**Problem**: `MutexGuard` held across await points blocks other tasks.

```rust
// ❌ Guard held across .await
let guard = mutex.lock().await;
do_async_work().await; // Other tasks can't lock!
drop(guard);

// ✅ Lock, copy/clone what you need, release, then await
let data = {
    let guard = mutex.lock().await;
    guard.clone()
};
do_async_work_with(data).await;
```

### 3. Forgetting `Send` Bounds
- `tokio::spawn` requires `Future + Send`
- A future is `Send` if all data held across `.await` points is `Send`
- `Rc`, `RefCell`, `MutexGuard` (std) are NOT `Send`

### 4. Future Size
- Each `.await` point adds to the future's size (it stores the state machine)
- Large futures increase stack usage and memory pressure
- Use `Box::pin()` for large futures, especially recursive ones

```rust
// ❌ Recursive async — infinite type size
async fn traverse(node: Node) {
    traverse(node.left).await;  // Infinite recursion in type
}

// ✅ Box the recursive future
fn traverse(node: Node) -> Pin<Box<dyn Future<Output = ()>>> {
    Box::pin(async move {
        traverse(node.left).await;
    })
}
```

### 5. Cancellation
- Dropping a future cancels it — work done since the last `.await` may be lost
- `select!` drops the losing branches
- Use `tokio::sync::oneshot` or `CancellationToken` for graceful cancellation
- Design futures to be cancellation-safe: don't do critical work between `.await` points unless it's idempotent

### 6. `async` Trait Methods
```rust
// Since Rust 1.75: native async in traits
trait Service {
    async fn call(&self, req: Request) -> Response;
}

// For object safety (dyn Trait), use async-trait crate:
#[async_trait::async_trait]
trait Service: Send + Sync {
    async fn call(&self, req: Request) -> Response;
}
```

## Structured Concurrency

### Task Hierarchy
- Prefer `JoinSet` or structured spawning over fire-and-forget `tokio::spawn`
- Ensure all spawned tasks are awaited before the parent returns
- Propagate errors upward, don't swallow them

```rust
use tokio::task::JoinSet;

let mut set = JoinSet::new();
for url in urls {
    set.spawn(async move { fetch(url).await });
}

while let Some(result) = set.join_next().await {
    handle(result??);
}
```

## Quick Reference

| Concept | Key Point |
|---------|-----------|
| `async fn` | Returns a Future, doesn't execute |
| `.await` | Suspends until future completes |
| `Poll::Pending` | Must register waker before returning |
| `Pin` | Prevents moving self-referential futures |
| `Unpin` | Opts out of pin restrictions (most types) |
| `spawn` | For async I/O work |
| `spawn_blocking` | For CPU/blocking work |
| `select!` | Race futures, cancel losers |
| `join!` | Run concurrently, await all |
| Cancellation | Dropping = cancelling |

## Common Mistakes

1. **Blocking in async** — use `spawn_blocking` or async equivalents
2. **Lock across `.await`** — release locks before awaiting
3. **Fire-and-forget spawn** — always join/await spawned tasks
4. **Recursive async without boxing** — infinite type size
5. **Ignoring cancellation safety** — `select!` drops futures mid-execution
6. **Forgetting `.await`** — the future does nothing without it
7. **`std::sync::Mutex` in async** — use `tokio::sync::Mutex` when held across await
8. **Huge futures on the stack** — box them with `Box::pin()`
