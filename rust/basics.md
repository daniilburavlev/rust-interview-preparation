# Rust Basics: Most Frequently Asked Interview Questions

Core questions that come up in almost every Rust interview, from screening calls to senior loops. Each answer is a concise model answer; at senior level, expect follow-ups asking *why* the language is designed this way.

## Contents

1. [Ownership and borrowing](#1-ownership-and-borrowing)
2. [Lifetimes](#2-lifetimes)
3. [Types, memory and data layout](#3-types-memory-and-data-layout)
4. [Traits and generics](#4-traits-and-generics)
5. [Smart pointers and interior mutability](#5-smart-pointers-and-interior-mutability)
6. [Error handling](#6-error-handling)
7. [Pattern matching and enums](#7-pattern-matching-and-enums)
8. [Closures and iterators](#8-closures-and-iterators)
9. [Concurrency](#9-concurrency)
10. [Async basics](#10-async-basics)
11. [Macros, modules and tooling](#11-macros-modules-and-tooling)
12. [Unsafe Rust](#12-unsafe-rust)

---

## 1. Ownership and borrowing

### 1.1 What are the three ownership rules?

1. Every value has exactly one owner (a variable or a field).
2. There can be only one owner at a time; assignment or passing by value **moves** ownership.
3. When the owner goes out of scope, the value is dropped (its `Drop` runs and memory is freed).

This gives deterministic resource cleanup (RAII) without a garbage collector.

### 1.2 What is the difference between move, copy and clone?

- **Move**: the default for non-`Copy` types. Bits are copied shallowly and the source becomes unusable.
- **Copy**: an implicit bitwise copy for types implementing `Copy` (integers, `bool`, `char`, `&T`, tuples/arrays of `Copy` types). The source stays valid. A type can be `Copy` only if it has no `Drop` impl and all fields are `Copy`.
- **Clone**: an explicit, possibly expensive deep copy via `.clone()`.

```rust
let a = String::from("hi");
let b = a;            // move: `a` is no longer usable
let c = b.clone();    // deep copy: both `b` and `c` are valid
let x = 5; let y = x; // copy: both valid
```

### 1.3 What are the borrowing rules?

At any time you may have **either** any number of shared references (`&T`) **or** exactly one mutable reference (`&mut T`), and references must never outlive the value they point to. This "aliasing XOR mutability" rule prevents data races and iterator invalidation at compile time.

### 1.4 What are Non-Lexical Lifetimes (NLL)?

Since Rust 2018, a borrow lasts until its **last use**, not until the end of the lexical scope. So this compiles:

```rust
let mut v = vec![1, 2, 3];
let first = &v[0];
println!("{first}");  // last use of `first`
v.push(4);            // OK: shared borrow already ended
```

### 1.5 `String` vs `&str`?

- `String` is an owned, growable, heap-allocated UTF-8 buffer (`ptr`, `len`, `capacity`).
- `&str` is a borrowed string slice (`ptr`, `len`) pointing into a `String`, a static literal, or any UTF-8 bytes.

Accept `&str` in function parameters (a `&String` derefs to it); return `String` when you create new data. The same pattern applies to `Vec<T>` vs `&[T]` and `PathBuf` vs `&Path`.

### 1.6 Why can't you index a `String` by integer (`s[0]`)?

Strings are UTF-8, so a character can take 1 to 4 bytes and indexing by byte position could split a character. Use `s.chars()`, `s.bytes()`, or byte-range slicing `&s[0..3]` (which panics if the range is not on a char boundary).

---

## 2. Lifetimes

### 2.1 What is a lifetime and why does Rust need annotations?

A lifetime is the region of code in which a reference is valid. The compiler infers most of them; annotations like `'a` are needed when a function returns a reference and the compiler cannot tell which input it is tied to. Annotations **describe** relationships; they never change how long values live.

```rust
fn longest<'a>(x: &'a str, y: &'a str) -> &'a str {
    if x.len() > y.len() { x } else { y }
}
```

### 2.2 What are the lifetime elision rules?

1. Each reference parameter gets its own lifetime.
2. If there is exactly one input lifetime, it is assigned to all output lifetimes.
3. If there is a `&self` or `&mut self`, its lifetime is assigned to all output lifetimes.

If these rules don't determine the output lifetime, you must annotate.

### 2.3 What does `'static` mean?

- `&'static T`: a reference valid for the whole program (e.g. string literals, leaked data).
- `T: 'static` as a bound: `T` contains no non-static borrows. An owned `String` satisfies `T: 'static` even though it can be dropped any time. This is why `std::thread::spawn` requires `F: 'static`: the closure must not borrow from the spawning stack frame.

### 2.4 How do you store a reference in a struct?

The struct needs a lifetime parameter, and an instance can't outlive the borrowed data:

```rust
struct Parser<'a> { input: &'a str }
```

---

## 3. Types, memory and data layout

### 3.1 Stack vs heap in Rust?

Values live on the stack by default (fixed size known at compile time). Heap allocation is explicit via `Box`, `Vec`, `String`, `Rc`, `Arc`, etc. The owning handle sits on the stack and frees the heap memory on drop.

### 3.2 What are `Sized` and dynamically sized types (DSTs)?

`Sized` types have a size known at compile time; generic parameters are implicitly `T: Sized`. DSTs like `str`, `[T]` and `dyn Trait` can only be used behind a pointer, which becomes a **fat pointer** (data pointer + length or vtable pointer). Use `T: ?Sized` to accept DSTs.

### 3.3 What is `Option<T>` and why no null?

`Option<T>` is `Some(T)` or `None`; the compiler forces you to handle absence. Thanks to niche optimization, `Option<&T>`, `Option<Box<T>>` and `Option<NonZeroU32>` have the same size as the inner type, so there is zero overhead compared to a nullable pointer.

### 3.4 What does `#[derive(...)]` do? Which traits are commonly derived?

It auto-generates trait implementations: `Debug`, `Clone`, `Copy`, `PartialEq`, `Eq`, `PartialOrd`, `Ord`, `Hash`, `Default`. `serde` adds `Serialize`/`Deserialize` via procedural macros.

### 3.5 `PartialEq` vs `Eq`, `PartialOrd` vs `Ord`?

`Eq` and `Ord` promise a total relation (reflexive: `a == a` always). Floats implement only `PartialEq`/`PartialOrd` because `NaN != NaN`. That's why `f64` can't be a `HashMap` key or be sorted with `.sort()` directly (use `sort_by(|a, b| a.total_cmp(b))`).

### 3.6 What happens on integer overflow?

In debug builds it panics; in release builds it wraps (two's complement) unless `overflow-checks` is enabled. Use explicit methods to state intent: `checked_add`, `wrapping_add`, `saturating_add`, `overflowing_add`.

---

## 4. Traits and generics

### 4.1 What is a trait?

A set of methods (and associated types/constants) a type can implement, similar to interfaces. Traits can have default method implementations and are used both as generic bounds and as trait objects.

### 4.2 Static vs dynamic dispatch: `impl Trait`/generics vs `dyn Trait`?

- **Generics / `impl Trait`**: monomorphized, a separate copy of the code per concrete type. Calls are direct and inlinable; cost is larger binaries and compile time.
- **`dyn Trait`**: a single copy, calls go through a vtable (fat pointer). Allows heterogeneous collections like `Vec<Box<dyn Shape>>`; cost is an indirect call and no inlining.

### 4.3 What makes a trait dyn-compatible (object safe)?

Roughly: methods must not be generic over types, must not return `Self`, and must take `self` via a pointer type (`&self`, `&mut self`, `Box<Self>`...). Methods that break this can be excluded with `where Self: Sized`.

### 4.4 Associated types vs generic type parameters on a trait?

Use an **associated type** when each implementing type has exactly one natural choice (`Iterator::Item`). Use a **generic parameter** when a type may implement the trait many times for different types (`From<T>`, `Add<Rhs>`).

### 4.5 What is the orphan rule?

You can implement a trait for a type only if the trait or the type is defined in your crate. It keeps implementations coherent (no two crates can provide conflicting impls). Workaround: the **newtype** pattern `struct MyVec(Vec<u8>);`.

### 4.6 What are `From`/`Into`, `AsRef`, `Deref`?

- `From<T>` for conversions; implementing `From` gives you `Into` for free. `?` uses `From` to convert errors.
- `AsRef<T>` for cheap reference-to-reference conversion in generic APIs (`fn open<P: AsRef<Path>>(p: P)`).
- `Deref` lets smart pointers behave like references and enables **deref coercion** (`&String` → `&str`, `&Box<T>` → `&T`). Don't use `Deref` to emulate inheritance.

### 4.7 What are marker traits `Send` and `Sync`?

- `Send`: ownership of the value can be transferred to another thread.
- `Sync`: `&T` can be shared between threads (`T: Sync` iff `&T: Send`).

They are auto traits, implemented automatically when all fields are `Send`/`Sync`. `Rc<T>` is neither; `Cell`/`RefCell` are `Send` but not `Sync`; `Arc<T>` is `Send + Sync` when `T: Send + Sync`.

---

## 5. Smart pointers and interior mutability

### 5.1 When would you use `Box<T>`?

To put data on the heap: recursive types (`enum List { Cons(i32, Box<List>), Nil }`), large values you want to move cheaply, and trait objects (`Box<dyn Error>`).

### 5.2 `Rc<T>` vs `Arc<T>`?

Both are reference-counted shared ownership. `Rc` uses non-atomic counters (single thread, cheaper); `Arc` uses atomic counters (thread-safe). Both give only shared (`&T`) access.

### 5.3 What is interior mutability? `Cell` vs `RefCell` vs `Mutex`/`RwLock`?

Mutating data through a shared reference, with the safety check moved from compile time to runtime or restricted by API:

- `Cell<T>`: get/set by value, no borrowing; for `Copy`-like small data; single-threaded.
- `RefCell<T>`: runtime-checked borrows (`borrow`, `borrow_mut`); **panics** on violation; single-threaded.
- `Mutex<T>` / `RwLock<T>`: thread-safe equivalents that block instead of panicking.
- Atomics (`AtomicUsize`, ...) for lock-free primitives.

Common combos: `Rc<RefCell<T>>` single-threaded, `Arc<Mutex<T>>` multi-threaded.

### 5.4 How do you avoid reference cycles with `Rc`?

Cycles of `Rc` never reach a count of zero and leak. Use `Weak<T>` (`Rc::downgrade`) for back-pointers such as child → parent in a tree; `upgrade()` returns `Option<Rc<T>>`.

### 5.5 What is `Cow<'a, T>`?

"Clone on write": holds either a borrowed or an owned value, allocating only when mutation is needed. Useful for functions that usually return the input unchanged but sometimes need to modify it.

---

## 6. Error handling

### 6.1 `Result` vs `panic!`?

- `Result<T, E>` for **recoverable**, expected failures (I/O, parsing, network).
- `panic!` for **bugs** and broken invariants. A panic unwinds the thread (or aborts with `panic = "abort"`).

Library code should almost never panic on bad input.

### 6.2 What does the `?` operator do?

On `Err(e)` it returns early with `Err(From::from(e))`; on `Ok(v)` it evaluates to `v`. It also works with `Option` (returns `None`). The function must return a compatible type.

### 6.3 `unwrap` vs `expect`? When are they acceptable?

Both panic on `None`/`Err`; `expect` adds a message. Acceptable in tests, prototypes, and when an invariant guarantees success (document it with `expect("reason")`). Avoid them in production paths that can legitimately fail.

### 6.4 `thiserror` vs `anyhow`?

- `thiserror`: derive typed error enums for **libraries**, so callers can match on variants.
- `anyhow`: a single opaque `anyhow::Error` with context (`.context("...")`) for **applications**, where you mostly log or report errors.

### 6.5 How do you define a custom error type by hand?

Implement `Debug`, `Display` and `std::error::Error` (optionally `source()`), plus `From` conversions for underlying errors so `?` works.

---

## 7. Pattern matching and enums

### 7.1 How are Rust enums different from C enums?

They are algebraic data types (tagged unions): each variant can carry different data. Combined with exhaustive `match`, the compiler ensures every case is handled.

```rust
enum Message { Quit, Move { x: i32, y: i32 }, Write(String) }
```

### 7.2 What makes `match` powerful?

Exhaustiveness checking, destructuring, guards (`if` conditions), bindings (`id @ 1..=9`), ranges, and `_` wildcards. `if let`, `while let` and `let ... else` are shorthand for single-pattern matches.

### 7.3 What is `#[non_exhaustive]`?

It forces downstream crates to include a wildcard arm when matching, so the library can add variants or fields later without a breaking change.

---

## 8. Closures and iterators

### 8.1 What are `Fn`, `FnMut` and `FnOnce`?

Traits closures implement based on how they use captured variables:

- `FnOnce`: may consume captured values; callable at least once. Every closure implements it.
- `FnMut`: mutates captured values; callable many times.
- `Fn`: only reads captured values; callable many times, even concurrently.

`Fn: FnMut: FnOnce`. The `move` keyword forces captures by value (needed for threads and async tasks), but doesn't by itself decide which trait is implemented.

### 8.2 Why are iterators "zero-cost"?

Adapters like `map`, `filter`, `zip` are lazy structs that are monomorphized and inlined; the compiled loop is typically as fast as a hand-written one, often with bounds checks eliminated.

### 8.3 `iter()` vs `iter_mut()` vs `into_iter()`?

They yield `&T`, `&mut T` and `T` (consuming the collection) respectively. `for x in &v` calls `iter()`, `for x in v` calls `into_iter()`.

### 8.4 What does `collect()` need to work?

The target type must implement `FromIterator`, and it's usually specified with a type annotation or turbofish: `collect::<Vec<_>>()`. Collecting an iterator of `Result<T, E>` into `Result<Vec<T>, E>` stops at the first error.

---

## 9. Concurrency

### 9.1 How does Rust guarantee "fearless concurrency"?

Ownership plus `Send`/`Sync`. Data races are compile-time errors: you can't share `&mut` across threads, and non-thread-safe types (`Rc`, `RefCell`) can't cross thread boundaries. Deadlocks and logical race conditions are still possible.

### 9.2 How do you share state between threads?

`Arc<Mutex<T>>` or `Arc<RwLock<T>>` for shared mutable state, atomics for counters/flags, or message passing with channels (`std::sync::mpsc`, `crossbeam`, `tokio::sync::mpsc`). `std::thread::scope` lets threads borrow local data without `Arc` because they're guaranteed to join before the scope ends.

### 9.3 What happens if a thread panics while holding a `Mutex`?

The mutex becomes **poisoned**; later `lock()` calls return `Err(PoisonError)`, which you can still recover from with `into_inner()`. (`parking_lot::Mutex` has no poisoning.)

---

## 10. Async basics

### 10.1 What is a `Future` in Rust?

A value representing a computation that may not be finished. Futures are **lazy**: nothing happens until they're polled. `async fn` compiles into a state machine implementing `Future`; each `.await` is a potential suspension point.

### 10.2 Why does Rust need an external runtime like Tokio?

The standard library defines `Future` but no executor. A runtime (Tokio, async-std, smol) provides the executor that polls tasks, a reactor (epoll/kqueue/IOCP) that wakes them on I/O readiness, timers, and async I/O types.

### 10.3 What is `Pin` and why is it needed?

Async state machines can hold references to their own fields (self-referential). `Pin<P>` guarantees the pointee won't move in memory after being pinned, keeping those internal references valid. Most types are `Unpin` and unaffected.

### 10.4 Why is blocking inside async code bad?

A runtime has a few worker threads; a blocking call (heavy CPU work, `std::thread::sleep`, synchronous I/O, holding a `std::sync::Mutex` across `.await`) stalls every task on that thread. Use `tokio::task::spawn_blocking`, async equivalents, or `tokio::sync::Mutex` when a lock must be held across `.await`.

### 10.5 Threads vs async: when to use which?

Async suits many concurrent I/O-bound tasks (thousands of connections) with low per-task overhead. Threads (or `rayon`) suit CPU-bound parallel work. Mixing them is normal: async for I/O, a thread pool for compute.

---

## 11. Macros, modules and tooling

### 11.1 Declarative vs procedural macros?

- **Declarative** (`macro_rules!`): pattern matching on token trees, defined in the same crate. E.g. `vec!`.
- **Procedural**: functions that take and return a `TokenStream`, in a separate `proc-macro` crate. Three kinds: derive (`#[derive(Serialize)]`), attribute (`#[tokio::main]`), function-like (`sql!(...)`).

Macros operate before type checking, so they can generate code that functions can't (variadic arguments, new items).

### 11.2 How does the module system and visibility work?

Items are private by default. `pub` makes them public, `pub(crate)` visible within the crate, `pub(super)` within the parent module. A crate is the compilation unit (a library or binary); a package (`Cargo.toml`) contains one or more crates; a workspace groups packages sharing `Cargo.lock` and `target/`.

### 11.3 What are Cargo features?

Conditional compilation flags declared in `Cargo.toml` and used via `#[cfg(feature = "x")]`. Features must be **additive**, since Cargo unifies features across the dependency graph.

### 11.4 Which tools should every Rust developer know?

`cargo build/test/run`, `cargo fmt` (rustfmt), `cargo clippy` (lints), `cargo doc`, `cargo bench`/criterion, `cargo audit` / `cargo deny` (vulnerabilities and licenses), and `miri` for detecting undefined behavior in unsafe code.

### 11.5 How are tests organized?

Unit tests in a `#[cfg(test)] mod tests` inside the source file (can test private items); integration tests in `tests/` (only the public API); doc tests in `///` examples, which are compiled and run by `cargo test`.

---

## 12. Unsafe Rust

### 12.1 What does `unsafe` allow you to do?

Five extra abilities: dereference raw pointers, call `unsafe` functions (including FFI), access or modify mutable statics, implement `unsafe` traits (e.g. `Send`/`Sync` manually), and access union fields. It does **not** turn off the borrow checker.

### 12.2 How should `unsafe` be used responsibly?

Keep unsafe blocks small, wrap them in safe abstractions that uphold invariants, document the invariants with `// SAFETY:` comments, and test with `miri`. The standard library itself (`Vec`, `Arc`) is built this way.

### 12.3 Is leaking memory considered unsafe?

No. Leaks are memory-safe: `std::mem::forget` and `Box::leak` are safe functions, and `Rc` cycles leak without `unsafe`. This is why code can't rely on `Drop` running for soundness.
