# Rust Basics: Questions Only

Questions without answers, for self-testing. Model answers are in [basics.md](basics.md).

## 1. Ownership and borrowing

- 1.1 What are the three ownership rules?
- 1.2 What is the difference between move, copy and clone?
- 1.3 What are the borrowing rules?
- 1.4 What are Non-Lexical Lifetimes (NLL)?
- 1.5 `String` vs `&str`?
- 1.6 Why can't you index a `String` by integer (`s[0]`)?

## 2. Lifetimes

- 2.1 What is a lifetime and why does Rust need annotations?
- 2.2 What are the lifetime elision rules?
- 2.3 What does `'static` mean?
- 2.4 How do you store a reference in a struct?

## 3. Types, memory and data layout

- 3.1 Stack vs heap in Rust?
- 3.2 What are `Sized` and dynamically sized types (DSTs)?
- 3.3 What is `Option<T>` and why no null?
- 3.4 What does `#[derive(...)]` do? Which traits are commonly derived?
- 3.5 `PartialEq` vs `Eq`, `PartialOrd` vs `Ord`?
- 3.6 What happens on integer overflow?

## 4. Traits and generics

- 4.1 What is a trait?
- 4.2 Static vs dynamic dispatch: `impl Trait`/generics vs `dyn Trait`?
- 4.3 What makes a trait dyn-compatible (object safe)?
- 4.4 Associated types vs generic type parameters on a trait?
- 4.5 What is the orphan rule?
- 4.6 What are `From`/`Into`, `AsRef`, `Deref`?
- 4.7 What are marker traits `Send` and `Sync`?

## 5. Smart pointers and interior mutability

- 5.1 When would you use `Box<T>`?
- 5.2 `Rc<T>` vs `Arc<T>`?
- 5.3 What is interior mutability? `Cell` vs `RefCell` vs `Mutex`/`RwLock`?
- 5.4 How do you avoid reference cycles with `Rc`?
- 5.5 What is `Cow<'a, T>`?

## 6. Error handling

- 6.1 `Result` vs `panic!`?
- 6.2 What does the `?` operator do?
- 6.3 `unwrap` vs `expect`? When are they acceptable?
- 6.4 `thiserror` vs `anyhow`?
- 6.5 How do you define a custom error type by hand?

## 7. Pattern matching and enums

- 7.1 How are Rust enums different from C enums?
- 7.2 What makes `match` powerful?
- 7.3 What is `#[non_exhaustive]`?

## 8. Closures and iterators

- 8.1 What are `Fn`, `FnMut` and `FnOnce`?
- 8.2 Why are iterators "zero-cost"?
- 8.3 `iter()` vs `iter_mut()` vs `into_iter()`?
- 8.4 What does `collect()` need to work?

## 9. Concurrency

- 9.1 How does Rust guarantee "fearless concurrency"?
- 9.2 How do you share state between threads?
- 9.3 What happens if a thread panics while holding a `Mutex`?

## 10. Async basics

- 10.1 What is a `Future` in Rust?
- 10.2 Why does Rust need an external runtime like Tokio?
- 10.3 What is `Pin` and why is it needed?
- 10.4 Why is blocking inside async code bad?
- 10.5 Threads vs async: when to use which?

## 11. Macros, modules and tooling

- 11.1 Declarative vs procedural macros?
- 11.2 How does the module system and visibility work?
- 11.3 What are Cargo features?
- 11.4 Which tools should every Rust developer know?
- 11.5 How are tests organized?

## 12. Unsafe Rust

- 12.1 What does `unsafe` allow you to do?
- 12.2 How should `unsafe` be used responsibly?
- 12.3 Is leaking memory considered unsafe?
