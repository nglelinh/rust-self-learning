---
layout: post
title: contents
chapter: home
order: 1
owner: Nguyen Le Linh
---

A deep-dive course on Rust systems programming — from ownership fundamentals to concurrency, async, and unsafe code — designed for developers who already know at least one language and want to internalize Rust's safety guarantees.

The **English** track covers every chapter below. The **Vietnamese** track is in progress: Chapter 1 is available; later chapters are planned. See Chapter 00 on the Vietnamese sidebar for the current status.

# Course Objectives

- Master Rust's ownership, borrowing, and lifetime system so you can reason about memory correctness without a garbage collector.

- Build fluency with Rust's type system — traits, generics, enums, and pattern matching — and apply them to write expressive, zero-cost abstractions.

- Write safe concurrent and asynchronous programs using threads, channels, `Arc<Mutex<T>>`, and `async`/`await` with Tokio.

- Understand when and how to use `unsafe` Rust responsibly, and how to interoperate with C/C++ via FFI.

- Develop production-quality Rust code: structured error handling, comprehensive testing, benchmarking, and macro authoring.

## Course Outline

### Chapter 1: Foundations
- Setup, Cargo, and the Rust toolchain
- Your first Rust program
- Primitive types and operators
- Control flow and pattern matching basics
- Modules, crates, and package layout
- Modern applications — sparse crates.io, rust-analyzer, WASI 0.2, Edition 2024 *(optional; EN+VI)*

### Chapter 2: Ownership and Core Types
- Ownership fundamentals — move semantics and the stack/heap model
- Borrowing rules and the borrow checker
- Slices and string types
- Structs, methods, and associated functions
- Enums, `Option<T>`, and `Result<T, E>`
- Modern applications — Binder, Linux 6.1+ Rust, CISA memory-safe roadmaps *(optional)*

### Chapter 3: Traits and Robustness
- Error handling with `?`, `thiserror`, and `anyhow`
- Advanced error propagation patterns
- Collections, iterators, and iterator adaptors
- Traits I — defining and implementing traits
- Traits II — trait objects and dynamic dispatch
- Modern applications — RFC 3185 async traits, hashbrown, anyhow/thiserror at scale *(optional)*

### Chapter 4: Advanced Ownership
- Lifetimes I — annotating references
- Lifetimes II — lifetime elision and complex scenarios
- Designing ownership-friendly APIs
- Smart pointers — `Box`, `Rc`, `Arc`
- Interior mutability — `Cell`, `RefCell`, `Mutex`
- Modern applications — Tokio `Arc`, rustc arenas, Wasmtime stores, GATs *(optional)*

### Chapter 5: Concurrency and Async
- Concurrency I — threads and `std::sync`
- Concurrency II — message passing and shared state
- Async I — `async`/`await` fundamentals
- Async II — Tokio and async I/O
- Testing I — unit and integration tests
- Modern applications — Pingora, Tokio/`hyper`/`axum`, tokio-console *(optional)*

### Chapter 6: Systems and Advanced Topics
- Testing II — property-based and fuzz testing
- Macros — declarative and procedural
- Unsafe Rust — raw pointers and invariants
- FFI and C interoperability
- Capstone project
- Modern applications — Cranelift/Wasmtime, PyO3/`cxx`, Miri, cargo-fuzz *(optional)*

### Chapter 7: Desktop App Programming
- GUI landscape — framework survey, rendering models compared, why GPUI
- GPUI — App, Window, View, Render; Model entities and subscriptions; element API; async tasks
- Layout systems — box model, constraint trees, flex layout, spacing and alignment (no CSS)
- OS concepts — processes vs threads, event loop anatomy, file system, timers
- Networking fundamentals — TCP, WebSocket, HTTP vs persistent connections, Serde, connection lifecycle
- App architecture — events → state → render pipeline, Tokio channels, practical checklist
- Modern applications — Zed/GPUI 2, Tauri 2.0, COSMIC/iced *(optional)*

### Chapter 8: Operating Systems Programming
- Processes and signals — spawning processes, pipes, Unix signal model, safe signal handling
- Files, file descriptors, and low-level I/O — VFS, `std::fs`, raw `File`, `mmap`
- Memory management — virtual memory, stack vs heap, custom allocators, arenas
- Inter-process communication — anonymous pipes, FIFOs, Unix sockets, shared memory
- System calls, libc, and the nix crate — syscall mechanism, POSIX API, `epoll`, `strace`
- Rust in the kernel and on bare metal — `#![no_std]`, MMIO, Linux kernel modules, `embedded-hal` *(optional)*
- Modern applications — Embassy, Ferrocene, Rust Binder, WASI 0.2 *(optional)*

## Main Textbooks

- Steve Klabnik and Carol Nichols, *The Rust Programming Language*, No Starch Press, 2023. (Available free at [doc.rust-lang.org/book](https://doc.rust-lang.org/book/))

- Jon Gjengset, *Rust for Rustaceans*, No Starch Press, 2021.

## References

- *The Rustonomicon* — guide to unsafe Rust: [doc.rust-lang.org/nomicon](https://doc.rust-lang.org/nomicon/)

- *Rust Reference*: [doc.rust-lang.org/reference](https://doc.rust-lang.org/reference/)

- *Rust API Guidelines*: [rust-lang.github.io/api-guidelines](https://rust-lang.github.io/api-guidelines/)

## Additional Resources

- [Rustlings](https://github.com/rust-lang/rustlings) — small exercises to get used to reading and writing Rust
- [Exercism Rust track](https://exercism.org/tracks/rust) — practice problems with community mentoring
- [Tokio documentation](https://tokio.rs/) — async runtime used in Chapter 5