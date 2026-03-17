# Rustonomicon Deep Reference

Complete detailed reference companion to the SKILL.md. Consult for in-depth explanations, edge cases, and extended examples.

## Chapter 1: Meet Safe and Unsafe

### The Dual Language Model

Rust contains two programming languages: Safe Rust and Unsafe Rust. Safe Rust guarantees type-safety and memory-safety — no dangling pointers, no use-after-free, no UB. Unsafe Rust is exactly like Safe Rust with the same rules and semantics, but unlocks 5 extra capabilities.

### The Three Options Problem

When implementation details matter in a safe language:
1. Fiddle with code to encourage optimizations
2. Adopt cumbersome design for desired implementation
3. Rewrite in a language that exposes those details (typically C)

Rust eliminates option 3 by embedding unsafe capabilities within the safe language.

### The Fundamental Property

**No matter what, Safe Rust can't cause Undefined Behavior.**

### Trust Asymmetry

- Safe Rust inherently trusts Unsafe Rust (assumes it's written correctly)
- Unsafe Rust **cannot** blindly trust Safe Rust
- Example: `BTreeMap` requires `Ord` but must be robust against incorrect implementations — it won't cause UB even with a broken `Ord`, though it may behave erratically

### When to Use `unsafe` Traits

- Safe trait: Easier to implement, but unsafe code must defend against incorrect impls
- Unsafe trait: Shifts responsibility to implementor
- Mark traits `unsafe` when unsafe code can't reasonably defend against broken impls
- `Send`/`Sync` are unsafe because thread safety is fundamental — can't defend against it
- `GlobalAlloc` is unsafe because everything depends on correct memory management

### Automatic Derivation of Send/Sync

Types composed entirely of `Send` types are automatically `Send`. Same for `Sync`. This minimizes the pervasive unsafety of making them unsafe traits.

---

## Chapter 2: Data Representation

### repr(Rust) Details

- No guarantees on field ordering between different type definitions
- Two instances of the same type DO have identical layout
- Compiler may reorder fields to minimize padding
- Different monomorphizations of the same generic type may have different layouts
- Enum tag size/position is unspecified
- Null pointer optimization: `Option<&T>` = pointer-sized (None = null)

### repr(C) Details

- Exact C layout: fields in declaration order with C padding rules
- Required for FFI boundary types
- ZSTs remain zero-sized (unlike C++ empty types which take 1 byte)
- DST pointers and tuples are NOT FFI-safe
- `Option<&T>` with repr(C) type still gets null pointer optimization for FFI-safe pointer types
- Fieldless enums: equivalent to `repr(u*)` with platform-default size

### repr(transparent)

- Only for struct or single-variant enum with exactly one non-ZST field
- Layout and ABI identical to that field
- Can transmute between wrapper and inner type
- Can pass through FFI where inner type is expected
- Only part of public ABI if the single field is pub or documented

### repr(packed) Dangers

- Forces maximum alignment of 1 (or N for `packed(N)`)
- Unaligned loads may be penalized (x86) or fault (ARM)
- Taking a reference to a packed field can cause UB
- Should almost never be used without `repr(C)` for FFI

### repr(align(N))

- Forces minimum alignment of N
- Useful for cache-line separation in concurrent code
- Incompatible with `repr(packed)`

### DST Details

- Trait objects (`dyn Trait`): wide pointer = ptr + vtable pointer
- Slices (`[T]`, `str`): wide pointer = ptr + length
- Structs with DST last field become DSTs themselves
- Custom DSTs via generic type + unsizing coercion:
  ```rust
  struct MySuperSliceable<T: ?Sized> { info: u32, data: T }
  let sized: MySuperSliceable<[u8; 8]> = /* ... */;
  let dynamic: &MySuperSliceable<[u8]> = &sized; // unsizing coercion
  ```

### ZST Safety

- Safe code doesn't need to worry about ZSTs
- Unsafe code must handle: pointer offsets are no-ops, allocators need non-zero size
- References to ZSTs must still be non-null and suitably aligned
- Loading/storing through a null pointer to ZST is NOT UB (exception to normal rules)

### Empty Types

- `enum Void {}` — cannot be instantiated
- `Result<T, Void>` optimized to just `T`
- `let Ok(num) = res;` is irrefutable when `Err` type is empty
- **Never** use `*const Void` for C's `void*` — dereferencing is UB, creating `&Void` is UB
- Use `*const ()` instead — can be made into a reference safely, reads/writes are no-ops

---

## Chapter 3: Ownership and Lifetimes (Full Detail)

### Why Ownership Exists

Problem: ensuring pointers are always valid is much more complicated than verifying references don't escape their referent's scope.

Two critical failure modes:
1. **Reference escaping scope**: returning `&local_var` — caught by simple scope analysis
2. **Invalidation through mutation**: holding `&vec[0]` while `vec.push()` may reallocate — requires understanding of aliasing

Rust's solution: references **freeze** the referent and its owners while active.

### Reference Rules

1. A reference cannot outlive its referent
2. A mutable reference cannot be aliased

### Aliasing Deep Dive

Variables and pointers alias if they refer to overlapping regions of memory. Aliasing matters for compiler optimizations.

**Key insight: writes are the primary hazard for optimizations.**

Example optimization enabled by no-alias guarantee:
```rust
fn compute(input: &u32, output: &mut u32) {
    // Compiler can cache *input because &mut output can't alias &input
    let cached = *input;
    if cached > 10 { *output = 2; }
    else if cached > 5 { *output *= 2; }
}
```
In C/C++, `input` and `output` could alias, preventing this optimization.

### Lifetime Mechanics (Detailed)

Lifetimes are regions of code, not scopes. They can have:
- **Holes**: invalidate and reinitialize a reference
- **Pauses**: same variable, two distinct borrows tied to it
- **Branches**: different last-use points in different branches

The borrow checker minimizes lifetime extent — a reference is alive from creation to last use.

**Exception for Drop**: if a type has a destructor, the destructor is considered a use at end of scope:
```rust
struct X<'a>(&'a i32);
impl Drop for X<'_> { fn drop(&mut self) {} }

let mut data = vec![1, 2, 3];
let x = X(&data[0]);
println!("{:?}", x);
data.push(4); // ERROR: x's destructor keeps borrow alive
// Fix: drop(x) before push, or restructure
```

### Lifetime Elision Rules (Complete)

For `fn` definitions and `fn` types:
1. Each elided input lifetime → distinct parameter
2. Single input lifetime → assigned to all elided outputs
3. Multiple inputs but one is `&self`/`&mut self` → self's lifetime for all elided outputs
4. Otherwise → compile error, must be explicit

For `impl` headers: all types are input positions.

### Unbounded Lifetimes

Sources:
1. Dereferencing raw pointers: `unsafe { &*raw_ptr }`
2. `transmute` and `transmute_copy`

Properties:
- Become as big as context demands
- Stronger than `'static` (can mold into `&'a &'a T`)
- Almost always wrong

**Fix**: Use lifetime elision at function boundaries. Output lifetime bound by input lifetime:
```rust
// BAD: unbounded — output doesn't derive from input
fn get_str<'a>(s: *const String) -> &'a str { unsafe { &*s } }

// GOOD: output bounded by input via elision
fn get_str(s: &String) -> &str { s.as_str() }
```

### HRTB Details

Problem: can't name the lifetime needed inside a closure bound until entering the function body.

```rust
// Can't write: where F: Fn(&'??? (u8, u16)) -> &'??? u8
// Solution:
where for<'a> F: Fn(&'a (u8, u16)) -> &'a u8
```

`for<'a>` = "for all choices of `'a`" — produces infinite trait bounds that F must satisfy.

### Borrow Checker Limitations

**Mutate-and-share problem:**
```rust
impl Foo {
    fn mutate_and_share(&mut self) -> &Self { &*self }
    fn share(&self) {}
}
let mut foo = Foo;
let loan = foo.mutate_and_share(); // mutable borrow with lifetime of loan
foo.share(); // ERROR: can't immutably borrow, still mutably borrowed
println!("{:?}", loan);
```
Correct by reference semantics but rejected by lifetime system.

**Get-default problem:**
```rust
fn get_default<'m, K, V>(map: &'m mut HashMap<K, V>, key: K) -> &'m mut V {
    match map.get_mut(&key) {
        Some(value) => value, // borrows map for 'm
        None => {
            map.insert(key, V::default()); // ERROR: second mutable borrow
            map.get_mut(&key).unwrap()
        }
    }
}
```
First borrow extends across match arms due to lifetime constraints.

---

## Chapter 4: Subtyping and Variance (Full Detail)

### Subtyping Definition

`Sub <: Super` means Sub satisfies all requirements of Super (and may have more).

For lifetimes: `'long <: 'short` if `'long` completely contains `'short`.

### Why &mut T Must Be Invariant

```rust
fn assign<T>(input: &mut T, val: T) { *input = val; }

let mut hello: &'static str = "hello";
{
    let world = String::from("world");
    assign(&mut hello, &world); // If allowed: hello = &world
}
println!("{hello}"); // use-after-free! world is dropped
```

If `&mut T` were covariant over `T`, the compiler would allow `&mut &'static str` to be treated as `&mut &'short str`, enabling writing a short-lived reference where a `'static` one is expected.

### Why fn(T) Is Contravariant

```rust
fn store(input: &'static str) { GLOBAL_VEC.push(input); }
fn demo<'a>(input: &'a str, f: fn(&'a str)) { f(input); }

demo("hello", store); // OK: "hello" is 'static
demo(&short_lived, store); // BAD: would push non-static into static vec
```

`fn(&'static str)` cannot be a subtype of `fn(&'a str)` because the latter accepts any lifetime, while the former requires `'static`. Contravariance inverts: `fn(&'a str) <: fn(&'static str)`.

### User-Defined Type Variance

Struct inherits variance from fields:
- All uses covariant → struct covariant
- All uses contravariant → struct contravariant
- Mixed or any invariant → struct invariant
- **Invariance wins all conflicts**

---

## Chapter 5: Drop Check and PhantomData (Full Detail)

### Why Drop Check Exists

Without drop check, a generic type's destructor could access already-freed borrowed data:

```rust
struct Inspector<'a>(&'a u8);
impl Drop for Inspector<'_> {
    fn drop(&mut self) { println!("{}", self.0); } // reads borrowed data!
}

let mut world = World { inspector: None, days: Box::new(1) };
world.inspector = Some(Inspector(&world.days));
// If days drops first, Inspector reads freed memory in its destructor!
```

### #[may_dangle] (Unstable)

Assert that Drop impl doesn't access the marked parameter:
```rust
unsafe impl<#[may_dangle] 'a> Drop for Inspector<'a> {
    fn drop(&mut self) {
        // Must NOT access self.0 (the &'a u8)
        println!("done");
    }
}
```

**Danger**: Indirect access via callbacks or trait methods can still access dangling data. The compiler doesn't check this.

### PhantomData for Standard Library Types

`Vec<T>` uses `PhantomData<T>` + `#[may_dangle]` pattern:
1. `#[may_dangle]` tells drop checker Vec doesn't care about `T`'s lifetime
2. `PhantomData<T>` tells drop checker Vec **owns** `T` values (re-enables protection)
3. Result: Vec can be dropped when `T`'s references are dangling, but only because Vec's Drop carefully drops `T` values before deallocating

### Drop Flags Implementation

- Tracked on stack (not in type, as in old Rust)
- Static when initialization state known at compile time (no runtime cost)
- Dynamic (boolean flag) when conditional initialization makes state ambiguous

---

## Chapter 6: Splitting Borrows (Full Detail)

### Struct Fields

Borrow checker understands disjoint field access:
```rust
let a = &mut x.a; // OK
let b = &mut x.b; // OK — different field
*a = 10; *b = 20;
```

### Slices — Requires Unsafe

`split_at_mut` implementation:
```rust
pub fn split_at_mut(&mut self, mid: usize) -> (&mut [T], &mut [T]) {
    let len = self.len();
    let ptr = self.as_mut_ptr();
    unsafe {
        assert!(mid <= len);
        (from_raw_parts_mut(ptr, mid),
         from_raw_parts_mut(ptr.add(mid), len - mid))
    }
}
```

### Safe Mutable Iterator Patterns

**Key insight**: `Iterator::next(&mut self) -> Option<Self::Item>` — `Item` has no connection to `self`, so multiple results can coexist.

Pattern for linked lists:
```rust
impl<'a, T> Iterator for IterMut<'a, T> {
    type Item = &'a mut T;
    fn next(&mut self) -> Option<Self::Item> {
        self.0.take().map(|node| {
            self.0 = node.next.as_mut().map(|n| &mut **n);
            &mut node.elem
        })
    }
}
```

Pattern for slices using `mem::take`:
```rust
fn next(&mut self) -> Option<Self::Item> {
    let slice = mem::take(&mut self.0);
    if slice.is_empty() { return None; }
    let (l, r) = slice.split_at_mut(1);
    self.0 = r;
    l.get_mut(0)
}
```

---

## Chapter 7: Type Conversions (Full Detail)

### Coercion Rules

Coercions are implicit type weakening. They do NOT apply in trait matching:
```rust
trait Trait {}
impl<'a> Trait for &'a i32 {}
fn foo<X: Trait>(t: X) {}
let t: &mut i32 = &mut 0;
foo(t); // ERROR: &mut i32 doesn't implement Trait
        // even though &mut i32 coerces to &i32
```

### Dot Operator Resolution

1. Try `T::foo(value)` (by value)
2. Try `<&T>::foo(value)` then `<&mut T>::foo(value)` (autoref)
3. Deref via `Deref` trait, retry from step 1
4. Unsize (e.g., `[T; N]` → `[T]`), retry from step 1

**Clone gotcha without bound:**
```rust
fn do_stuff<T: Clone>(value: &T) {
    let cloned = value.clone(); // cloned: T (calls T::clone)
}
fn do_stuff<T>(value: &T) {
    let cloned = value.clone(); // cloned: &T (calls <&T>::clone, just copies ref!)
}
```

### Cast Rules

- Every coercion can be done via `as`
- Infallible at runtime but can silently truncate
- Raw slice casts don't adjust length: `*const [u16] as *const [u8]` creates half-length slice
- Not transitive: `e as U1 as U2` valid doesn't mean `e as U2` valid

### Transmute Details

**Absolutely forbidden:**
- `&T` to `&mut T` — always UB, no exceptions, optimizer assumes shared refs are immutable
- Creating values with invalid bit patterns (e.g., `transmute::<u8, bool>(3)`)

**Dangerous but sometimes necessary:**
- Different compound types: must ensure same layout (use `repr(C)`)
- Different generic instantiations: `repr(Rust)` gives NO layout guarantees even for same generic type
- References: produces unbounded lifetime

**`transmute_copy<T, U>`**: Even more dangerous — no size check. UB if `size_of::<U>() > size_of::<T>()`.

---

## Chapter 8: Uninitialized Memory (Full Detail)

### Safe Handling: Checked Initialization

Rust statically prevents reading uninitialized variables:
- Basic branch analysis: every branch must assign before use
- Delayed initialization allowed if all branches assign exactly once
- Move semantics: moving a non-Copy value uninitializes the source
- Reassignment after move requires `mut`

### Unsafe Handling: MaybeUninit

Three-step process:
1. Create `[const { MaybeUninit::uninit() }; SIZE]`
2. Initialize each element (assigning to MaybeUninit is safe — dropping MaybeUninit is no-op)
3. Transmute to initialized type (only valid for arrays, NOT arbitrary Container<MaybeUninit<T>>)

**Wrong way to initialize:**
```rust
*x[i].as_mut_ptr() = value; // WRONG: tries to drop the old (uninit) value
```

**Right way:**
```rust
x[i] = MaybeUninit::new(value); // OK: MaybeUninit drop is no-op
// OR
x[i].as_mut_ptr().write(value); // OK: ptr::write doesn't drop old value
```

### Obtaining Pointers to Uninit Data

Never create references to uninitialized data. Use raw pointer arithmetic:
```rust
// For struct fields:
let f1_ptr = unsafe { &raw mut (*uninit.as_mut_ptr()).field };
unsafe { f1_ptr.write(value); }
```

---

## Chapter 9: OBRM, Constructors, Destructors, Leaking (Full Detail)

### Constructors

- One true constructor: name type + initialize all fields
- No implicit Copy/Move/Default/Assignment constructors
- Move = memcpy (types can't care about memory location)
- `Clone::clone()` must be explicit (never implicit)
- `Copy` = implicit clone via bitwise copy (subset of Clone)
- `Default` trait: rarely used outside generic programming
- Convention: `Type::new()` for "default" constructor

### Destructors — Recursive Drop

After `Drop::drop(&mut self)` runs, Rust recursively drops ALL fields. No stable way to prevent this.

**SuperBox problem:**
```rust
impl Drop for SuperBox<T> {
    fn drop(&mut self) {
        // Deallocate box's contents manually
        unsafe { Global.deallocate(self.my_box.ptr.cast(), Layout::new::<T>()); }
        // THEN Rust drops self.my_box, which tries to deallocate AGAIN!
        // Double-free!
    }
}
```

**Fix with Option:**
```rust
impl Drop for SuperBox<T> {
    fn drop(&mut self) {
        let my_box = self.my_box.take().unwrap(); // take() sets field to None
        unsafe { Global.deallocate(my_box.ptr.cast(), Layout::new::<T>()); }
        mem::forget(my_box); // don't run Box's destructor
    }
}
```

### Leaking — Complete Analysis

**Drain leak:**
```rust
let mut vec = vec![Box::new(0); 4];
let mut drainer = vec.drain(..);
drainer.next(); drainer.next(); // two elements moved out
mem::forget(drainer); // destructor doesn't run
println!("{}", vec[0]); // UB: reading freed memory!
```

**Solution — Leak amplification:** Set `vec.len = 0` before starting drain. If drainer is leaked, more data leaks (but no UB). Destructor restores correct len.

**Rc overflow:**
```rust
// mem::forget enough Rc clones to overflow ref_count back to 0
// Then remaining Rc's drop triggers use-after-free
// Solution: std Rc checks for overflow and aborts
```

**thread::scoped (removed from std):**
```rust
let guard = thread::scoped(|| { *data += 1; });
mem::forget(guard); // Skips join! Thread may outlive data.
// Fundamental design flaw: safety depended on destructor running
```

---

## Chapter 10: Unwinding and Exception Safety (Full Detail)

### Rust's Error Hierarchy

1. `Option` — something reasonably absent
2. `Result` — something went wrong, can be handled
3. `panic!` — something went wrong, cannot be handled (unwinds stack)
4. `abort` — catastrophic failure

### Unwinding Implementation

- Optimized for "doesn't unwind" case (zero cost when no panic)
- Actually unwinding is expensive (more than Java)
- `catch_unwind` catches panics without spawning threads (but use sparingly)
- `panic=abort` skips unwinding entirely

### Vec::push_all Exception Safety Bug

```rust
impl<T: Clone> Vec<T> {
    fn push_all(&mut self, to_push: &[T]) {
        self.reserve(to_push.len());
        unsafe {
            self.set_len(self.len() + to_push.len()); // BUG: set len before init
            for (i, x) in to_push.iter().enumerate() {
                self.ptr().add(i).write(x.clone()); // clone() can panic!
            }
        }
    }
}
```
If `clone()` panics, Vec has wrong len → reads uninitialized memory on drop.

**Fix:** Set `len` after the loop (or increment each iteration).

### BinaryHeap Sift-Up — Guard Pattern

```rust
struct Hole<'a, T: 'a> {
    data: &'a mut [T],
    elt: Option<T>,  // the removed element
    pos: usize,      // current hole position
}

impl<T> Drop for Hole<'_, T> {
    fn drop(&mut self) {
        // ALWAYS fill the hole, whether we panic or not
        unsafe {
            let pos = self.pos;
            ptr::write(&mut self.data[pos], self.elt.take().unwrap());
        }
    }
}
```

This is the definitive pattern for exception safety with unsafe state transitions.

### Poisoning

`Mutex` poisons itself if a `MutexGuard` is dropped during panic:
- Future `lock()` calls return `Err` (or panic)
- Data may be in inconsistent state (not UB, but logically wrong)
- Can still force access via `PoisonError::into_inner()`
- Safety guard, not memory safety mechanism

---

## Chapter 11: Concurrency (Full Detail)

### Send and Sync — Deep Analysis

**Key insight:** `T: Sync` ⟺ `&T: Send`

**Raw pointers** are marked `!Send + !Sync` as a lint:
- Actually could be safely sent in many cases
- But prevents types containing raw pointers from auto-deriving thread safety
- Types with raw pointers have untracked ownership — likely not thread-safe by default

**Implementing Send/Sync:**
```rust
// Must verify: exclusive ownership, no shared mutable state without sync
unsafe impl<T: Send> Send for Carton<T> {}
// Must verify: &Carton<T> provides no unsynchronized interior mutability
unsafe impl<T: Sync> Sync for Carton<T> {}
```

**MutexGuard is !Send but Sync:**
- !Send: must unlock on same thread that locked (library requirement)
- Sync: sharing &MutexGuard between threads is fine (dropping a reference does nothing)

### Atomics — Complete Model

**Compiler reordering:** Compilers transform code assuming single-threaded execution. Multi-threaded programs need atomics to prevent reordering.

**Hardware reordering:** CPUs use local caches. Writes may become visible to other threads in different order than executed.

**Hardware categories:**
- Strongly-ordered (x86/64): Most accesses have acquire-release semantics for free
- Weakly-ordered (ARM): Must explicitly request ordering. Test on weak hardware!

**Data accesses vs atomic accesses:**
- Data accesses: freely reordered, propagated lazily — how data races occur
- Atomic accesses: tell hardware/compiler about multi-threading

**Ordering details:**

*SeqCst:*
- Cannot be reordered relative to ANY other access
- All threads agree on a single global execution order
- Still emits memory fences even on strongly-ordered platforms
- Rarely necessary; easy to downgrade later

*Acquire-Release:*
- Acquire: nothing after moves before. Things before CAN move after.
- Release: nothing before moves after. Things after CAN move before.
- Creates happens-before relationship between threads on SAME location
- No causality with other threads or different locations
- Free on strongly-ordered platforms

*Relaxed:*
- Only guarantee: operation is atomic (no torn reads/writes)
- Can be freely reordered
- No happens-before relationship
- Good for: counters, statistics, flags where ordering doesn't matter

### Data Races vs Race Conditions

**Data race (UB):** Two+ threads access same memory, at least one writes, at least one unsynchronized. Prevented by ownership + Send/Sync.

**Race condition (logic bug):** Correct behavior depends on thread scheduling. Cannot be prevented by type system. Safe Rust can have race conditions.

**Combined with unsafe = dangerous:**
```rust
if idx.load(Ordering::SeqCst) < data.len() {
    unsafe {
        // BUG: idx could have changed between check and use!
        println!("{}", data.get_unchecked(idx.load(Ordering::SeqCst)));
    }
}
```

---

## Chapter 12: FFI (Full Detail)

### Complete FFI Workflow

1. **Build script** (`build.rs`): Link against native library
2. **Declare** foreign functions in `unsafe extern "C"` block
3. **Create safe wrapper** that handles pointer/buffer management
4. **Test** with unit tests

### Safe Wrapper Pattern

```rust
pub fn validate_compressed_buffer(src: &[u8]) -> bool {
    unsafe {
        snappy_validate_compressed_buffer(src.as_ptr(), src.len()) == 0
    }
}
```

Key: the wrapper is NOT marked `unsafe` — it guarantees safety for all inputs.

### Buffer Management Pattern

```rust
pub fn compress(src: &[u8]) -> Vec<u8> {
    unsafe {
        let srclen = src.len();
        let mut dstlen = snappy_max_compressed_length(srclen);
        let mut dst = Vec::with_capacity(dstlen);
        snappy_compress(src.as_ptr(), srclen, dst.as_mut_ptr(), &mut dstlen);
        dst.set_len(dstlen); // Set actual length after C function writes
        dst
    }
}
```

### Callbacks

**Simple callback:** `extern "C" fn callback(a: i32)` — pass as function pointer

**Object-targeted callback:** Pass `*mut RustObject` as context, cast back in callback:
```rust
unsafe extern "C" fn callback(target: *mut RustObject, a: i32) {
    unsafe { (*target).a = a; }
}
```

**Async callbacks from C threads:** Require synchronization. Use `mpsc::channel` to forward data to Rust thread. Deregister callback in destructor.

### Opaque Types

```rust
#[repr(C)]
pub struct OpaqueHandle {
    _data: (),
    _marker: PhantomData<(*mut u8, PhantomPinned)>,
}
```
- `()` prevents instantiation outside module
- `*mut u8` in PhantomData makes it `!Send + !Sync`
- `PhantomPinned` makes it `!Unpin`
- `#[repr(C)]` for FFI compatibility
- **Never** use empty enum for opaque FFI types — compiler assumes uninhabited

### Nullable Pointer Optimization for FFI

`Option<extern "C" fn(c_int) -> c_int>` has same layout as C function pointer. `None` = null. No transmute needed.

### FFI and Unwinding

| ABI | Rust panic behavior | Foreign exception behavior |
|-----|--------------------|-----------------------------|
| `"C"` | Safely aborts | UB |
| `"C-unwind"` | Unwinds across boundary | Unwinds across boundary |
| `"Rust"` | Always permits unwinding | N/A |

Use `catch_unwind` for Rust code called from C:
```rust
#[unsafe(no_mangle)]
pub extern "C" fn safe_from_c() -> i32 {
    match std::panic::catch_unwind(|| { /* ... */ }) {
        Ok(_) => 0,
        Err(_) => 1,
    }
}
```

### Foreign Globals

```rust
unsafe extern {
    static rl_readline_version: libc::c_int;      // read-only
    static mut rl_prompt: *const libc::c_char;     // mutable (all access unsafe)
}
```

### Calling Conventions

`stdcall`, `cdecl`, `fastcall`, `system` (platform default), `C`, `win64`, `sysv64`, `aapcs`, `thiscall`, `vectorcall` (unstable)

`"system"` = `stdcall` on win32/x86, `C` elsewhere. Best for Windows API.

### Interoperability Notes

- `Box<T>` uses non-nullable pointers but is managed by internal allocators — don't manually create
- `Vec`/`String` are contiguous memory but NOT null-terminated
- Use `CString`/`CStr` for null-terminated C strings
- Rust links against `libc` and `libm` by default

---

## Chapter 12.1: Beneath std (#![no_std])

### Requirements for `#![no_std]` Executables

1. `#![no_std]` attribute
2. `#![no_main]` attribute (no Rust-generated entry point)
3. `#[panic_handler]` function: `fn(&PanicInfo) -> !`
4. Entry point symbol (platform-specific: `main`, `_start`, etc.)
5. `eh_personality` lang item (some platforms, nightly only)

### Panic Handler

```rust
#[panic_handler]
fn panic(info: &PanicInfo) -> ! {
    // Option A: halt
    loop {}
    // Option B: log to semihosting
    writeln!(stderr, "{}", info).ok();
    loop {}
    // Option C: abort
    core::intrinsics::abort()
}
```

Panic crates allow swapping behavior via dependency:
```rust
#[cfg(debug_assertions)]
extern crate panic_semihosting;  // dev: log to host
#[cfg(not(debug_assertions))]
extern crate panic_halt;         // release: halt silently
```

### libc in no_std

```toml
[dependencies]
libc = { version = "0.2", default-features = false }
```
Must disable default features (which include `std`).

### compiler_builtins

Needed when building `core` from source and linker reports missing symbols like `__aeabi_memcpy`.
