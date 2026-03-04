---
name: rfr-concurrency
description: Use when designing concurrent or parallel Rust code, choosing between channels and shared state, understanding atomic orderings, preventing deadlocks, or reasoning about Send and Sync bounds. Use when code has data races, lock contention, or thread safety issues.
---

# Rust for Rustaceans — Ch 10: Concurrency (and Parallelism)

## The Concurrency Landscape

### Concurrency vs Parallelism
- **Concurrency**: structuring code to handle multiple tasks (may interleave on one core)
- **Parallelism**: actually running multiple tasks simultaneously (multiple cores)
- Rust's ownership system prevents data races at compile time

### Concurrency Models

| Model | Mechanism | Trade-offs |
|-------|-----------|------------|
| Shared state | `Arc<Mutex<T>>` | Simple for small scopes; contention at scale |
| Message passing | Channels (`mpsc`, `crossbeam`) | Decoupled; may need serialization |
| Lock-free | Atomics | Highest performance; hardest to get right |
| Actor model | Tasks + channels | Clean architecture; overhead per message |

## Send and Sync

### Definitions
- `T: Send` — safe to **move** `T` to another thread
- `T: Sync` — safe to **share** `&T` across threads (i.e., `&T: Send`)

### Key Relationships
```
T: Sync  ⟺  &T: Send
```

### Common Types

| Type | Send | Sync | Why |
|------|------|------|-----|
| `i32`, `String`, `Vec<T>` | Yes | Yes | No shared mutable state |
| `Rc<T>` | **No** | **No** | Non-atomic reference count |
| `Arc<T>` (T: Send+Sync) | Yes | Yes | Atomic reference count |
| `Cell<T>`, `RefCell<T>` | Yes | **No** | Interior mutability, not thread-safe |
| `Mutex<T>` (T: Send) | Yes | Yes | Lock provides exclusivity |
| `RwLock<T>` (T: Send+Sync) | Yes | Yes | Lock provides exclusivity |
| `*const T`, `*mut T` | **No** | **No** | Raw pointers: no safety guarantees |
| `MutexGuard<T>` | **No** | Yes | Must unlock on same thread (usually) |

### Manually Implementing Send/Sync
```rust
struct MyPointerWrapper(*mut u8);

// SAFETY: The pointed-to data is only accessed through
// this wrapper, which provides exclusive access via &mut self.
unsafe impl Send for MyPointerWrapper {}
unsafe impl Sync for MyPointerWrapper {}
```

**Only impl these if you can PROVE the invariants hold.**

## Shared State

### `Arc<Mutex<T>>` Pattern
```rust
use std::sync::{Arc, Mutex};
use std::thread;

let counter = Arc::new(Mutex::new(0));

let handles: Vec<_> = (0..10).map(|_| {
    let counter = Arc::clone(&counter);
    thread::spawn(move || {
        let mut num = counter.lock().unwrap();
        *num += 1;
    })
}).collect();

for handle in handles {
    handle.join().unwrap();
}
```

### `Mutex` vs `RwLock`

| | `Mutex<T>` | `RwLock<T>` |
|---|---|---|
| Readers | One at a time | Many concurrent |
| Writers | One at a time | One at a time, exclusive |
| Overhead | Lower | Higher (reader tracking) |
| Use when | Writes are frequent | Reads dominate, writes are rare |

### Avoiding Deadlocks

1. **Lock ordering**: always acquire locks in the same order
2. **Minimize lock scope**: hold locks for the shortest time possible
3. **Avoid nested locks**: if you must, document the ordering
4. **Use `try_lock()`**: non-blocking attempt, returns immediately
5. **Prefer channels**: no locks = no deadlocks

```rust
// ❌ Deadlock risk: A locks mutex1 then mutex2, B locks mutex2 then mutex1
// ✅ Always lock in consistent order

// ❌ Lock held too long
let guard = data.lock().unwrap();
expensive_computation(&guard); // Blocks everyone
drop(guard);

// ✅ Copy what you need, release lock
let snapshot = {
    let guard = data.lock().unwrap();
    guard.clone()
};
expensive_computation(&snapshot);
```

### Poisoned Mutexes
- If a thread panics while holding a `Mutex`, it becomes "poisoned"
- `.lock()` returns `Err(PoisonError)`
- `.lock().unwrap()` will panic on poisoned mutex
- Use `.lock().unwrap_or_else(|e| e.into_inner())` to recover

## Message Passing

### `std::sync::mpsc`
```rust
use std::sync::mpsc;

let (tx, rx) = mpsc::channel();

// Clone tx for multiple producers
let tx2 = tx.clone();

thread::spawn(move || { tx.send(1).unwrap(); });
thread::spawn(move || { tx2.send(2).unwrap(); });

// Receive messages
while let Ok(msg) = rx.recv() {
    println!("got: {msg}");
}
```

### Channel Types

| Type | Bounded | Backpressure | Blocking |
|------|---------|--------------|----------|
| `mpsc::channel()` | No (unbounded) | No | `recv()` blocks |
| `mpsc::sync_channel(n)` | Yes | `send()` blocks when full | Both block |
| `crossbeam::channel::bounded(n)` | Yes | Yes | Configurable |
| `crossbeam::channel::unbounded()` | No | No | `recv()` blocks |
| `tokio::sync::mpsc` | Yes | `send().await` when full | Async |
| `tokio::sync::oneshot` | 1 item | N/A | One-shot response |
| `tokio::sync::broadcast` | Yes | Slow receivers lose msgs | Multi-consumer |
| `tokio::sync::watch` | 1 (latest) | Overwrites | Latest value |

### Channel Selection
```rust
// crossbeam select! for multiple channels
use crossbeam::channel::{select, bounded};

let (tx1, rx1) = bounded(10);
let (tx2, rx2) = bounded(10);

select! {
    recv(rx1) -> msg => handle_msg(msg.unwrap()),
    recv(rx2) -> msg => handle_msg(msg.unwrap()),
    default(Duration::from_secs(1)) => handle_timeout(),
}
```

## Atomics

### Atomic Types
`AtomicBool`, `AtomicI32`, `AtomicU64`, `AtomicUsize`, `AtomicPtr<T>`

### Memory Ordering

| Ordering | Guarantee | Use case |
|----------|-----------|----------|
| `Relaxed` | Only atomicity, no ordering | Counters, statistics |
| `Acquire` | All writes before the corresponding Release are visible | Load side of lock |
| `Release` | All prior writes become visible to Acquire loads | Store side of lock |
| `AcqRel` | Both Acquire and Release | Read-modify-write (CAS) |
| `SeqCst` | Total global ordering | Default, simplest to reason about |

### Ordering Rules of Thumb
- **Start with `SeqCst`** — correctness first
- **`Relaxed` for counters** — just need atomicity, not ordering
- **`Acquire`/`Release` pairs** — for synchronizing data access through a flag
- **Never use `Relaxed` for synchronization** — it only guarantees the atomic itself

```rust
use std::sync::atomic::{AtomicBool, AtomicUsize, Ordering};

// Simple counter (Relaxed is fine)
static REQUEST_COUNT: AtomicUsize = AtomicUsize::new(0);
REQUEST_COUNT.fetch_add(1, Ordering::Relaxed);

// Flag for "data is ready" (needs Acquire/Release)
static DATA_READY: AtomicBool = AtomicBool::new(false);
static mut DATA: u64 = 0;

// Writer thread:
unsafe { DATA = 42; }
DATA_READY.store(true, Ordering::Release);

// Reader thread:
if DATA_READY.load(Ordering::Acquire) {
    let val = unsafe { DATA }; // Guaranteed to see 42
}
```

### Compare-and-Swap (CAS)
```rust
use std::sync::atomic::{AtomicU64, Ordering};

let counter = AtomicU64::new(0);

// Atomic increment without lock
loop {
    let current = counter.load(Ordering::Relaxed);
    match counter.compare_exchange(
        current,
        current + 1,
        Ordering::AcqRel,
        Ordering::Relaxed,
    ) {
        Ok(_) => break,
        Err(_) => continue, // Another thread changed it, retry
    }
}
// Or just use fetch_add which does this internally
```

## Thread Pools (Rayon)

```rust
use rayon::prelude::*;

// Parallel iteration
let sum: i64 = (0..1_000_000)
    .into_par_iter()
    .map(|i| i * i)
    .sum();

// Parallel sort
let mut data = vec![5, 3, 1, 4, 2];
data.par_sort();

// Fork-join parallelism
let (left, right) = rayon::join(
    || compute_left(),
    || compute_right(),
);
```

### When to Use Rayon
- CPU-bound work that can be split into independent chunks
- Data parallelism (map/filter/reduce over collections)
- NOT for I/O-bound work (use async instead)
- Minimum ~1000 items or ~1ms per item to overcome overhead

## Scoped Threads

```rust
// Scoped threads can borrow from the parent stack
std::thread::scope(|s| {
    let data = vec![1, 2, 3];

    s.spawn(|| {
        println!("{data:?}"); // Borrows data — no Arc needed!
    });

    s.spawn(|| {
        println!("len: {}", data.len()); // Also borrows
    });
}); // All scoped threads are joined here
```

**Scoped threads vs regular threads:**
- Regular: must `'static` or move — can't borrow from parent
- Scoped: can borrow from parent — joined before scope ends

## Common Mistakes

1. **`Rc` across threads** — use `Arc` (atomic reference count)
2. **`RefCell` across threads** — use `Mutex` or `RwLock`
3. **Holding locks across expensive work** — clone data, drop lock, then process
4. **Wrong atomic ordering** — start with `SeqCst`, optimize only after profiling
5. **Unbounded channels with fast producers** — causes unbounded memory growth
6. **Lock ordering inconsistency** — document and enforce lock acquisition order
7. **`thread::spawn` for I/O-bound work** — use async tasks instead
8. **Ignoring poisoned mutexes** — decide on a recovery strategy or crash early
