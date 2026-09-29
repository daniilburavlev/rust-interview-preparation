# Advanced Rust: Senior Interview Questions

Questions that separate senior candidates: memory model, async internals, API design, performance and unsafe code. Answers are concise; expect the interviewer to dig into trade-offs.

## Contents

1. [Type system and API design](#1-type-system-and-api-design)
2. [Memory and performance](#2-memory-and-performance)
3. [Async internals](#3-async-internals)
4. [Concurrency and the memory model](#4-concurrency-and-the-memory-model)
5. [Unsafe, FFI and soundness](#5-unsafe-ffi-and-soundness)
6. [Production Rust services](#6-production-rust-services)

---

## 1. Type system and API design

### 1.1 What is variance and why does it matter?

Variance describes how subtyping of lifetimes propagates through a type. `&'a T` is covariant in `'a` (a longer lifetime can be used where a shorter one is expected); `&'a mut T` is covariant in `'a` but **invariant** in `T`; `Cell<T>` is invariant in `T`; `fn(T)` is contravariant in `T`. Getting variance wrong in unsafe code (e.g. a raw-pointer struct) can allow dangling references; `PhantomData` is used to declare the intended variance.

### 1.2 What is `PhantomData` used for?

A zero-sized marker telling the compiler a type logically owns or borrows a `T` it doesn't store: it controls variance, auto traits (`Send`/`Sync`) and drop check. Typical uses: raw-pointer containers, typed IDs (`Id<User>`), and the typestate pattern.

### 1.3 What are Higher-Ranked Trait Bounds (HRTB)?

`for<'a> F: Fn(&'a str) -> &'a str` means "for every lifetime `'a`". Needed when a closure must accept references with lifetimes chosen by the callee, not the caller. Usually inferred by elision in `Fn(&str)` bounds.

### 1.4 What are Generic Associated Types (GATs)?

Associated types with their own generic parameters, commonly lifetimes: `type Item<'a> where Self: 'a;`. They enable "lending iterators" that yield references into themselves, which plain `Iterator` can't express.

### 1.5 Explain the typestate and newtype patterns.

- **Newtype**: `struct UserId(u64);` gives a distinct type with zero runtime cost, preventing mix-ups and bypassing the orphan rule.
- **Typestate**: encode state in the type (`Connection<Closed>` → `Connection<Open>`), so invalid transitions don't compile. Methods consume `self` and return the new state.

### 1.6 What is the sealed trait pattern?

A public trait with a supertrait in a private module, so downstream crates can use it but not implement it. Lets a library add methods later without breaking changes.

### 1.7 How do you design a Rust library API that is easy to evolve?

Accept generic, borrowed inputs (`impl AsRef<Path>`, `&str`), return owned concrete types, use builders for many optional parameters, `#[non_exhaustive]` on public enums/structs, sealed traits, typed errors via `thiserror`, and follow semver strictly (adding a trait impl or a public field can be breaking). Tools: `cargo-semver-checks`.

### 1.8 `impl Trait` in argument vs return position?

In arguments it's sugar for an anonymous generic (the caller picks the type). In return position the callee picks one concrete, hidden type; you can't return two different types from branches (use `Box<dyn Trait>` or an enum). Since Rust 1.75, `async fn` and `-> impl Trait` are allowed in traits.

---

## 2. Memory and performance

### 2.1 How is a struct laid out in memory?

By default (`repr(Rust)`) the compiler may reorder fields to minimize padding; layout is unspecified. `#[repr(C)]` gives C-compatible order for FFI; `#[repr(transparent)]` guarantees a newtype has the same layout as its field; `#[repr(u8)]` fixes an enum's discriminant.

### 2.2 What is niche optimization?

The compiler uses invalid bit patterns of a type (null for references, values > 1 for `bool`, spare enum discriminants) to store an enum's tag. So `Option<&T>`, `Option<Box<T>>` and `Option<NonZeroU64>` cost nothing extra.

### 2.3 How do you reduce allocations in a hot path?

Reuse buffers (`clear()` instead of re-creating), `Vec::with_capacity`, `SmallVec`/`ArrayVec` for small collections, `Cow` to avoid cloning, `Bytes` for shared zero-copy buffers, arena allocators (`bumpalo`), interning strings, and a faster global allocator (`jemalloc`, `mimalloc`). Measure first with `dhat` or heaptrack.

### 2.4 How do you profile a Rust service?

CPU: `perf` + flamegraphs (`cargo flamegraph`), `samply`. Allocations: `dhat`, heaptrack. Async: `tokio-console` for task stalls and poll times. Microbenchmarks: `criterion` or `divan`. Always profile release builds with debug symbols (`debug = true` in the profile).

### 2.5 Which compiler and build settings affect performance?

`opt-level = 3`, `lto = "fat"` or `"thin"`, `codegen-units = 1`, `panic = "abort"`, profile-guided optimization (PGO), and `target-cpu=native` when the deploy hardware is known. Trade-off: slower builds.

### 2.6 What is drop check and when does it bite?

The compiler must ensure a value's `Drop` impl can't observe already-dropped borrowed data. Generic types with a `Drop` impl are assumed to access their `T` on drop, which can reject otherwise valid code; `#[may_dangle]` (unstable, unsafe) relaxes this in std collections.

### 2.7 What is false sharing and how do you avoid it in Rust?

Two threads writing different variables on the same cache line cause cache-line ping-pong. Pad hot per-thread data to cache-line size, e.g. `crossbeam_utils::CachePadded<T>` or `#[repr(align(64))]`.

---

## 3. Async internals

### 3.1 How does an `async fn` compile?

Into an anonymous enum-like state machine implementing `Future`. Each `.await` is a state; locals that live across `.await` are stored in the state. The future's size is the size of its largest state, so large locals across awaits bloat futures (box them or restructure).

### 3.2 Explain `poll`, `Waker` and `Context`.

The executor calls `Future::poll(self: Pin<&mut Self>, cx: &mut Context)`. It returns `Ready(v)` or `Pending`. Before returning `Pending`, the future must arrange for `cx.waker().wake()` to be called when progress is possible (e.g. registering with the I/O reactor). The executor then re-polls. A future that returns `Pending` without registering a waker hangs forever.

### 3.3 What is cancellation safety?

Dropping a future cancels it at its last `.await`. In `tokio::select!` the losing branches are dropped; if a branch had partially done work (read bytes into an internal buffer, taken a message), that work is lost. Cancel-safe methods (`mpsc::Receiver::recv`, `TcpListener::accept`) lose nothing; others (`read_exact`, `write_all`) may. Senior answer: design around it (keep buffers outside the future, use `select!` in loops carefully).

### 3.4 How does Tokio's scheduler work?

A multi-threaded work-stealing scheduler: each worker has a local run queue plus a global injection queue; idle workers steal tasks. There's a LIFO slot for the most recently woken task to improve locality, and cooperative budgeting (each task gets ~128 operations before being forced to yield) to prevent starvation.

### 3.5 `tokio::spawn` vs `join!` vs `select!`?

- `spawn`: runs a `'static + Send` future as an independent task, possibly in parallel on another thread; returns a `JoinHandle`.
- `join!`/`try_join!`: runs futures concurrently within the same task until all complete (no parallelism, no `'static` requirement).
- `select!`: waits for the first of several futures and drops the rest.

### 3.6 Why must spawned futures be `Send`, and how do you fix "future is not Send"?

The multi-threaded runtime may move a task between threads at any `.await`. Holding a non-`Send` value (`Rc`, `RefCell` borrow, `std::sync::MutexGuard`) across an `.await` makes the whole future non-`Send`. Fix: drop the value before the `.await` (scope it in a block), use `Arc`/`tokio::sync::Mutex`, or use `LocalSet`/`spawn_local`.

### 3.7 How do you implement backpressure in async Rust?

Bounded channels (`mpsc::channel(n)`: senders wait when full), `Semaphore` to cap concurrent work, `buffer_unordered(n)` on streams, and tower's `ConcurrencyLimit`/`RateLimit`/`LoadShed` layers. Unbounded channels hide overload until memory runs out.

### 3.8 How do you do graceful shutdown in a Tokio service?

Listen for SIGTERM/ctrl-c, broadcast a shutdown signal (`CancellationToken` from `tokio-util` or a `watch` channel), stop accepting new connections (`axum::serve(...).with_graceful_shutdown`), let in-flight requests finish with a timeout, flush producers and buffers, then exit. Use `TaskTracker` to wait for spawned tasks.

---

## 4. Concurrency and the memory model

### 4.1 Explain atomic memory orderings.

- `Relaxed`: atomicity only, no ordering with other memory ops. Fine for counters.
- `Release` (store) / `Acquire` (load): a release store "publishes" all prior writes to a thread that acquire-loads the same value. The basis of locks and message passing.
- `AcqRel`: both, for read-modify-write ops.
- `SeqCst`: plus a single global order of all SeqCst operations; the safest, rarely required.

### 4.2 How does `Arc` use orderings?

Increment uses `Relaxed` (you already have a reference, so nothing to synchronize). Decrement uses `Release`, and the thread that drops the count to zero does an `Acquire` fence before destroying the data, so all other threads' writes are visible before drop.

### 4.3 `Mutex` vs `RwLock` vs lock-free structures?

`Mutex` is simplest and often fastest under contention for short critical sections. `RwLock` helps when reads dominate and are long; it can starve writers and has higher overhead. Lock-free (`crossbeam` queues, `arc-swap`, `dashmap` sharding) reduces contention but is hard to get right. Senior answer: measure; often sharding or message passing beats clever locking.

### 4.4 What is `arc-swap` and when would you use it?

An atomically swappable `Arc<T>`: readers load a snapshot cheaply without locking, writers replace it wholesale. Ideal for read-mostly config or routing tables that are reloaded at runtime.

### 4.5 How can deadlocks happen in Rust and how do you prevent them?

Rust prevents data races, not deadlocks. Causes: inconsistent lock ordering, holding a lock while awaiting or calling back into user code, re-locking a non-reentrant mutex. Prevention: consistent lock ordering, short critical sections, never hold a lock across `.await` or I/O, prefer message passing.

---

## 5. Unsafe, FFI and soundness

### 5.1 What does "sound" mean for an unsafe abstraction?

No sequence of calls from **safe** code can cause undefined behavior. If safe code can trigger UB through your API, the API is unsound even if nobody has hit the bug yet.

### 5.2 Name common sources of undefined behavior.

Dereferencing dangling/unaligned/null pointers, creating two `&mut` to the same data (aliasing violations), creating invalid values (a `bool` that's 3, uninitialized memory read as an integer, invalid enum discriminant), data races, and violating `Send`/`Sync` contracts. `MaybeUninit` is the correct way to handle uninitialized memory.

### 5.3 What are Stacked Borrows / Tree Borrows and Miri?

Experimental models defining which pointer aliasing patterns are allowed. `cargo miri test` interprets code and detects UB under these models, including aliasing violations, use-after-free, and invalid values. It should run in CI for crates with unsafe code.

### 5.4 How do you call C from Rust and expose Rust to C?

Declare functions in `extern "C" { ... }` blocks (calls are `unsafe`), use `#[repr(C)]` types and `std::ffi::{CStr, CString}`. Expose Rust with `#[no_mangle] pub extern "C" fn`. Tools: `bindgen` (C → Rust) and `cbindgen` (Rust → C headers). Never let a panic unwind across an FFI boundary; catch it with `catch_unwind` or use `extern "C-unwind"` deliberately.

### 5.5 When is it correct to `unsafe impl Send/Sync`?

When your type contains raw pointers or other non-`Send`/`Sync` fields but you guarantee thread safety yourself (e.g. the pointee is only accessed under a lock). Document the reasoning in a `// SAFETY:` comment; a wrong impl is instant unsoundness.

---

## 6. Production Rust services

### 6.1 What does a typical production Rust web stack look like?

`tokio` runtime, `axum` or `actix-web` for HTTP, `tower` middleware (timeouts, retries, rate limiting), `tonic` for gRPC, `sqlx` or `diesel` for databases, `serde` for serialization, `tracing` + OpenTelemetry for observability, `rdkafka` for Kafka, and `config`/`figment` for configuration.

### 6.2 What is `tower::Service` and why is it important?

A trait `Service<Request>` with `poll_ready` (backpressure) and `call` (returns a future). Middleware are `Layer`s wrapping services, so timeouts, retries, auth and metrics compose uniformly across axum, tonic and hyper.

### 6.3 How do you structure observability in a Rust service?

`tracing` spans and structured events instead of `log`, `tracing-subscriber` with JSON output, `tracing-opentelemetry` for distributed traces, `metrics` or `prometheus` crates for RED metrics (rate, errors, duration), and propagate trace context through HTTP headers and Kafka message headers.

### 6.4 How do you handle panics in a server?

Tokio catches panics per task (the `JoinHandle` returns `Err`), and `tower-http`'s `CatchPanicLayer` turns handler panics into 500 responses. Still, treat panics as bugs: alert on them, and decide between `panic = "unwind"` (isolates failing requests) and `"abort"` (smaller, faster, relies on the orchestrator restarting the process).

### 6.5 How do you reduce Rust compile times in a large codebase?

Split into a workspace of smaller crates, cut heavy generic instantiation across crate boundaries, reduce proc-macro usage, use `sccache` or CI caching (`Swatinem/rust-cache`), a faster linker (`mold`, `lld`), `cargo check` in dev loops, and `cargo-hakari` to unify features. Measure with `cargo build --timings`.

### 6.6 How do you keep dependencies safe?

Commit `Cargo.lock` for binaries, run `cargo audit` (RustSec advisories) and `cargo deny` (licenses, duplicates, banned crates) in CI, use Dependabot/Renovate, and review new dependencies (`cargo vet`, `cargo crev`) for maintenance and unsafe usage.
