# Advanced Rust: Questions Only

Questions without answers, for self-testing. Model answers are in [advanced.md](advanced.md).

## 1. Type system and API design

- 1.1 What is variance and why does it matter?
- 1.2 What is `PhantomData` used for?
- 1.3 What are Higher-Ranked Trait Bounds (HRTB)?
- 1.4 What are Generic Associated Types (GATs)?
- 1.5 Explain the typestate and newtype patterns.
- 1.6 What is the sealed trait pattern?
- 1.7 How do you design a Rust library API that is easy to evolve?
- 1.8 `impl Trait` in argument vs return position?

## 2. Memory and performance

- 2.1 How is a struct laid out in memory?
- 2.2 What is niche optimization?
- 2.3 How do you reduce allocations in a hot path?
- 2.4 How do you profile a Rust service?
- 2.5 Which compiler and build settings affect performance?
- 2.6 What is drop check and when does it bite?
- 2.7 What is false sharing and how do you avoid it in Rust?

## 3. Async internals

- 3.1 How does an `async fn` compile?
- 3.2 Explain `poll`, `Waker` and `Context`.
- 3.3 What is cancellation safety?
- 3.4 How does Tokio's scheduler work?
- 3.5 `tokio::spawn` vs `join!` vs `select!`?
- 3.6 Why must spawned futures be `Send`, and how do you fix "future is not Send"?
- 3.7 How do you implement backpressure in async Rust?
- 3.8 How do you do graceful shutdown in a Tokio service?

## 4. Concurrency and the memory model

- 4.1 Explain atomic memory orderings.
- 4.2 How does `Arc` use orderings?
- 4.3 `Mutex` vs `RwLock` vs lock-free structures?
- 4.4 What is `arc-swap` and when would you use it?
- 4.5 How can deadlocks happen in Rust and how do you prevent them?

## 5. Unsafe, FFI and soundness

- 5.1 What does "sound" mean for an unsafe abstraction?
- 5.2 Name common sources of undefined behavior.
- 5.3 What are Stacked Borrows / Tree Borrows and Miri?
- 5.4 How do you call C from Rust and expose Rust to C?
- 5.5 When is it correct to `unsafe impl Send/Sync`?

## 6. Production Rust services

- 6.1 What does a typical production Rust web stack look like?
- 6.2 What is `tower::Service` and why is it important?
- 6.3 How do you structure observability in a Rust service?
- 6.4 How do you handle panics in a server?
- 6.5 How do you reduce Rust compile times in a large codebase?
- 6.6 How do you keep dependencies safe?
