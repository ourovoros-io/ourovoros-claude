---
name: rfr-unsafe-code
description: Use when writing or reviewing unsafe Rust code, raw pointer manipulation, MaybeUninit usage, UnsafeCell wrappers, or FFI safety contracts. Use when reasoning about soundness, validity invariants, or stacked borrows. Use when running Miri or sanitizers.
---

# Rust for Rustaceans — Ch 9: Unsafe Code

## What `unsafe` Allows (The Five Superpowers)

| Superpower | What it enables |
|-----------|----------------|
| Dereference raw pointers | `*const T` and `*mut T` access |
| Call unsafe functions | Functions with preconditions the compiler can't check |
| Access mutable statics | `static mut` read/write |
| Implement unsafe traits | `Send`, `Sync`, `GlobalAlloc` |
| Access fields of unions | Only one variant is valid at a time |

**`unsafe` does NOT disable the borrow checker.** References still have their normal rules. It only unlocks the five operations above.

## Safety vs Validity

### Validity Invariant
The invariant that must hold for a value's bit pattern to be valid for its type — enforced at ALL times, even in unsafe code.

| Type | Validity invariant |
|------|-------------------|
| `bool` | Must be `0` or `1` |
| `char` | Must be a valid Unicode scalar value |
| `&T` | Must be non-null, aligned, and point to valid `T` |
| `&mut T` | Same as `&T` plus exclusive access |
| `enum` | Discriminant must be a valid variant |
| `!` (never) | No valid value exists |

### Safety Invariant
Additional invariant that must hold for the type to behave correctly — may be temporarily violated in unsafe code, but must be restored before safe code observes the value.

```rust
// Vec's safety invariants:
// - ptr is a valid allocation of capacity bytes
// - len <= capacity
// - elements 0..len are initialized
// Validity: ptr is non-null, aligned
// Safety: elements must be initialized, len is accurate
```

### Soundness
A function is **sound** if no sequence of safe calls can cause undefined behavior. An unsafe block is sound if it correctly upholds all invariants.

```rust
// ✅ Sound: caller can't trigger UB
pub fn get(slice: &[u8], idx: usize) -> Option<&u8> {
    if idx < slice.len() {
        // SAFETY: idx is bounds-checked above
        Some(unsafe { slice.get_unchecked(idx) })
    } else {
        None
    }
}

// ❌ Unsound: safe caller can trigger UB
pub fn get_unchecked(slice: &[u8], idx: usize) -> &u8 {
    unsafe { slice.get_unchecked(idx) } // No bounds check!
}
```

## Raw Pointers

### `*const T` and `*mut T`
- Can be null, dangling, or unaligned
- No lifetime tracking — no borrow checker
- Creating a raw pointer is safe; dereferencing is unsafe

```rust
let x = 42;
let ptr: *const i32 = &x;           // Safe: creating pointer
let val = unsafe { *ptr };           // Unsafe: dereferencing

let mut y = 0;
let mut_ptr: *mut i32 = &mut y;
unsafe { *mut_ptr = 10; }           // Unsafe: writing through raw ptr
```

### Pointer Provenance
- A pointer is only valid if it was derived from a valid reference or allocation
- You can't manufacture pointers from integers (in general)
- `ptr::null()`, `ptr::null_mut()` — valid null pointers
- Use `NonNull<T>` for guaranteed non-null pointers (enables niche optimization)

### Common Raw Pointer Patterns
```rust
// Converting between references and pointers
let reference: &i32 = &42;
let ptr: *const i32 = reference;
let back: &i32 = unsafe { &*ptr };

// Pointer arithmetic
let arr = [10, 20, 30];
let ptr = arr.as_ptr();
let second = unsafe { *ptr.add(1) }; // 20

// Casting between pointer types
let byte_ptr: *const u8 = ptr.cast::<u8>();
```

## `MaybeUninit<T>`

**Safe wrapper for potentially uninitialized memory.**

```rust
use std::mem::MaybeUninit;

// Allocate uninitialized memory
let mut buf: MaybeUninit<[u8; 1024]> = MaybeUninit::uninit();

// Write to it (safe — writing to MaybeUninit is always OK)
let ptr = buf.as_mut_ptr() as *mut u8;
unsafe {
    std::ptr::write_bytes(ptr, 0, 1024); // Zero-fill
}

// Assume initialized (unsafe — you must ensure it's actually initialized)
let buf: [u8; 1024] = unsafe { buf.assume_init() };
```

### Rules
- Reading from `MaybeUninit` without initializing is UB
- `assume_init()` is the "trust me, it's initialized" assertion
- Use `MaybeUninit::zeroed()` for types where zero-bytes is valid
- For arrays: `MaybeUninit::uninit_array()` (nightly) or `[MaybeUninit::uninit(); N]`

## `UnsafeCell<T>`

**The fundamental building block of interior mutability.**

- All interior mutability types (`Cell`, `RefCell`, `Mutex`, `RwLock`, `Atomic*`) contain `UnsafeCell` at their core
- `UnsafeCell<T>` is the ONLY way to get `&mut T` from `&T` without UB
- Makes the containing type invariant over `T`
- Makes the containing type `!Sync` (unless you manually impl Sync)

```rust
use std::cell::UnsafeCell;

struct MyCell<T> {
    value: UnsafeCell<T>,
}

impl<T> MyCell<T> {
    fn new(value: T) -> Self {
        Self { value: UnsafeCell::new(value) }
    }

    fn set(&self, val: T) {
        // SAFETY: single-threaded access guaranteed
        // (MyCell is !Sync so can't be shared across threads)
        unsafe { *self.value.get() = val; }
    }

    fn get(&self) -> T where T: Copy {
        // SAFETY: UnsafeCell::get returns *mut T
        unsafe { *self.value.get() }
    }
}
```

## Writing Sound Unsafe Code

### The SAFETY Comment
Every `unsafe` block must have a `// SAFETY:` comment explaining why the preconditions are met:

```rust
// SAFETY: `idx` has been bounds-checked by the caller
// via the `if idx < self.len` guard above.
unsafe { self.data.get_unchecked(idx) }
```

### Checklist for Unsafe Code

1. **Document preconditions** on every `unsafe fn` with `# Safety` doc section
2. **SAFETY comment** on every `unsafe {}` block
3. **Minimize unsafe scope** — only the smallest necessary expression
4. **Maintain invariants** — restore safety invariants before returning to safe code
5. **No aliased mutability** — `&T` and `&mut T` must not coexist for the same data
6. **Alignment** — all references must be aligned for their type
7. **Initialization** — all reads must be from initialized memory
8. **Lifetime correctness** — returned references must outlive their origin

### Unsafe Traits
```rust
// Declaring an unsafe trait
/// # Safety
/// Implementors must ensure that `data_ptr` returns a valid,
/// aligned pointer to `len()` initialized elements.
unsafe trait ContiguousBuffer {
    fn data_ptr(&self) -> *const u8;
    fn len(&self) -> usize;
}

// Implementing an unsafe trait
unsafe impl ContiguousBuffer for Vec<u8> {
    fn data_ptr(&self) -> *const u8 { self.as_ptr() }
    fn len(&self) -> usize { self.len() }
}
```

## Verification Tools

### Miri
```bash
rustup +nightly component add miri
cargo +nightly miri test
```

**Detects:**
- Use-after-free, double-free
- Out-of-bounds memory access
- Unaligned pointer access
- Use of uninitialized memory
- Data races
- Stacked borrows violations
- Invalid enum discriminants

### Sanitizers
```bash
# AddressSanitizer
RUSTFLAGS="-Z sanitizer=address" cargo +nightly test

# ThreadSanitizer (data race detection)
RUSTFLAGS="-Z sanitizer=thread" cargo +nightly test

# MemorySanitizer (uninitialized reads)
RUSTFLAGS="-Z sanitizer=memory" cargo +nightly test
```

### cargo-careful
```bash
cargo install cargo-careful
cargo +nightly careful test
```

Enables stdlib debug assertions — catches many issues without full Miri slowdown.

## Common Undefined Behaviors

| UB | Example |
|----|---------|
| Null pointer dereference | `unsafe { *std::ptr::null::<i32>() }` |
| Dangling reference | Reference to dropped value |
| Unaligned access | `*(ptr as *const u64)` where ptr isn't 8-aligned |
| Data race | Two threads writing without synchronization |
| Invalid bool | `std::mem::transmute::<u8, bool>(2)` |
| Aliased `&mut` | Two `&mut` to same data simultaneously |
| Uninitialized read | Reading `MaybeUninit` before init |
| Infinite loop without side effects | `loop {}` (technically) |

## Common Mistakes

1. **`unsafe` block too large** — wrap only the minimal unsafe operation, not the whole function
2. **Missing SAFETY comment** — every unsafe block needs justification
3. **`transmute` instead of `from_ne_bytes`/`to_ne_bytes`** — use safe alternatives when they exist
4. **Assuming alignment** — casts like `ptr as *const u64` require alignment proof
5. **Not running Miri** — tests pass ≠ no UB; always run Miri on unsafe code
6. **Exposing unsafe fn publicly** — wrap in a safe API that upholds invariants
7. **`mem::forget` for cleanup** — forgetting is safe but may leak; don't rely on Drop for safety
8. **Raw pointer arithmetic without `add`/`sub`/`offset`** — manual arithmetic can overflow; use pointer methods
