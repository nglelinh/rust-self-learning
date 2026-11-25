---
layout: post
title: "04-21 Modern Applications — Lifetimes and Smart Pointers in Real Runtimes"
chapter: "04"
order: 21
owner: "OpenCode"
lang: en
categories:
  - chapter04
lesson_type: optional
---

Once you learn lifetimes, `Box`, `Rc`/`Arc`, and `RefCell`, a natural question arises: *how do real flagship Rust systems—like the Tokio async runtime, the `rustc` compiler itself, and the Wasmtime WebAssembly engine—actually wire these tools together at scale?* You will rarely see messy `Rc<RefCell<T>>` spaghetti in those codebases. Instead, you see deliberate architectural choices: `'static` ownership across Tokio task boundaries, arena allocation (`bumpalo`/`typed-arena`) inside `rustc`, and **Generic Associated Types (GATs)** (stabilized in Rust 1.65) unlocking lending iterators that borrow directly from internal buffers!

This optional capstone connects Chapter 4’s ownership primitives to 2022–2026 production runtimes—showing why `'static` beats lifetime gymnastics across async boundaries and how GATs solve an abstraction problem Rust couldn’t even express before 1.65.

---

## Key ideas: Ownership Architecture in Production Runtimes

### 1. `'static` Bounds & Shared Ownership in Async Runtimes (Tokio)
When you call `tokio::spawn(async move { ... })`, the spawned task may outlive the stack frame that spawned it and migrate across worker threads. Therefore, Tokio requires `Future + Send + 'static`—meaning **no borrowed stack references**! Real services share immutable config or connection pools via `Arc<State>` rather than fighting impossible lifetimes.
- **Core questions:** Why does `T: 'static` mean “owns all its data (no short-lived borrows),” NOT “lives forever in global memory”?
- **Real-world reach:** `tokio`, `axum`, `hyper`, and multi-threaded microservices.

### 2. Arena Allocation & Bulk Lifetimes (`rustc`, Wasmtime)
Instead of allocating and freeing 500,000 tiny AST nodes individually with `Box` or `Rc`, compilers allocate all nodes contiguously inside a **bump arena** tied to one single lifetime `'tcx`. Every node borrows `&'tcx Node` freely, cache locality skyrockets, and the entire arena frees in **O(1)** at the end of compilation!
- **Core questions:** How does tying a whole phase’s memory to one arena lifetime `'a` eliminate reference counting?
- **Real-world reach:** `rustc` (`TyCtxt<'tcx>`), game-frame scratch allocators, and request-scoped web arenas.

### 3. Generic Associated Types (GATs) & Lending Iterators (Rust 1.65+)
Standard `Iterator::Item` cannot borrow from `&mut self` (you can `.collect()` all items simultaneously!). **GATs** (`type Item<'a> where Self: 'a;`) let a **lending iterator** reuse a single internal buffer across calls—handing out `&'a [u8]` windows one line at a time with **zero per-line heap allocation**!
- **Core questions:** Why couldn’t `std::iter::Iterator` express a zero-copy line reader reusing one `String` buffer?
- **Real-world reach:** Zero-copy file/socket streaming, database row cursors, and custom async traits.

## 60-minute teaching plan

- 0 to 10 min: Lifetimes as a runtime budget (request, task, compilation session).
- 10 to 25 min: `Arc` as Tokio’s social contract; when `Arc<Mutex<T>>` is a smell.
- 25 to 40 min: rustc arenas and interning — alternatives to `Rc` at compiler scale.
- 40 to 50 min: GATs (1.65) as the feature that unlocked async traits and lending iterators.
- 50 to 60 min: Interior mutability at the edge: `parking_lot`, `OnceLock`, lock poisoning.

## Objectives

You will be able to look at a production type — `Arc<Handle>`, a rustc `Ty<'tcx>`, a Wasmtime `Store<T>` — and say whether the lifetime is a *session*, a *task*, or a *lie you replaced with an id*. You will treat `Arc` as a concurrency tax you can count, not a default. The mental-model shift is from “I annotated `'a` until it compiled” to “I picked the owner of this byte for the next ten milliseconds of a live system.”

## Prerequisites

Required Chapter 4: lifetime mental model, borrowed structs, ownership-friendly APIs, smart pointers, interior mutability. You should be able to write `struct View<'a>(&'a str)` and explain why `Rc<RefCell<T>>` exists. Tokio and rustc internals are used as *examples*, not as homework prerequisites.

## Introduction

A lifetime in rustc is not a teaching example. It is `'tcx`: the compiler session. Almost every type in the type checker is `Ty<'tcx>`, interned in an arena the session owns. Drop the session and every borrow dies together. That is Chapter 4’s “borrowed struct” lecture implemented as a 400-crate workspace.

A lifetime in Tokio is often *refused*. Tasks outlive the stack frame that spawned them, so the runtime clones `Arc` handles instead of trying to name a borrow across `.await`. The 2022–2026 Tokio ecosystem (`tracing`, `tower`, `hyper` 1.0, `axum`) is an encyclopedia of that choice.

A lifetime in Wasmtime is a `Store<T>`: guest linear memory and host state that must not be aliased the way a C embedder would alias a `char*`. The Component Model (WASI 0.2, January 2024) doubled down on this — no exporting raw linear memory as the IPC fabric.

This lesson walks those three runtimes as design reviews. We will write small stand-ins, not fork rustc.

## Key Concepts

### Session lifetimes versus task lifetimes

If every interesting value dies together, give them one lifetime and an arena. rustc does this. If values must move to other threads or outlive the current future, *do not borrow* — own, or share with `Arc`. Tokio does this.

Students who try to store `&'a Client` inside a spawned task meet the compiler immediately. The production translation is `Arc<Client>` or “pass the id, look it up later.” Both are Chapter 4 API design.

### `Arc` is a protocol

`Arc<T>` says: many tasks may *observe*, and if anyone mutates, they do it through a lock or a channel. Tokio’s `Handle`, `Client` types in `hyper`/`reqwest`, and shared `Client` state in Pingora-style proxies are `Arc` because the alternative is a lifetime that crosses a scheduler.

Count the clones. An `Arc` clone is an atomic increment. In a per-request path that is fine; in a per-byte path it is a bug. rustc almost never puts `Arc` on types that appear in the inner type-checker loop — it uses arenas and copyable interned ids instead.

### GATs made “lending” APIs real

Generic associated types stabilized in Rust 1.65 (November 2022). They are why `async fn` in traits could desugar to an associated future type, and why lending iterators (`fn next(&mut self) -> Option<&T>` as a trait) stopped being a research poster. When you read a 2023–2025 trait that returns something borrowing `self`, you are reading a GAT even if the author wrote `async fn` sugar.

### Interior mutability is a runtime, not a field

`Mutex`, `RwLock`, `OnceLock` (stabilized and then used everywhere in 2023–2025 std code), and `parking_lot` are how processes cache config and intern strings. Poisoning (`std::sync::Mutex`) versus non-poisoning (`parking_lot`) is an operational choice: do you want a panic in one thread to brick the process, or keep going?

## Code Walkthroughs

### Naive: borrow across a spawn

```rust
struct Client {
    token: String,
}

fn naive(client: &Client) {
    std::thread::spawn(|| {
        println!("{}", client.token); // error: may outlive borrowed value
    });
}
```

The compiler is describing Tokio’s scheduler. A thread (or a task) does not have a slot for your `'a`.

### First fix that does not scale: clone the world

```rust
fn clone_world(client: &Client) {
    let token = client.token.clone();
    std::thread::spawn(move || println!("{token}"));
}
```

Fine for a `String`. Catastrophic for a 200 MB policy table. This is the “clone to silence rustc” habit Chapter 2 warned about, now on a thread boundary.

### Idiomatic: share the owner, borrow inside the task

```rust
use std::sync::Arc;

struct Client {
    token: String,
    policy: Vec<u8>,
}

fn share(client: Arc<Client>) {
    let c = Arc::clone(&client);
    std::thread::spawn(move || {
        let token: &str = &c.token; // borrow is *inside* the task
        println!("{token} ({} bytes of policy)", c.policy.len());
    });
}
```

The lifetime of `token` is the task body, not the caller. That is the Tokio pattern: `Arc` at the boundary, ordinary `&` inside.

### Session style: one lifetime, many borrows

```rust
struct Session {
    intern: Vec<String>,
}

#[derive(Clone, Copy)]
struct Sym(usize);

impl Session {
    fn intern(&mut self, s: &str) -> Sym {
        if let Some(i) = self.intern.iter().position(|x| x == s) {
            return Sym(i);
        }
        self.intern.push(s.to_string());
        Sym(self.intern.len() - 1)
    }

    fn get(&self, s: Sym) -> &str {
        &self.intern[s.0]
    }
}

fn check<'s>(sess: &'s Session, a: Sym, b: Sym) -> bool {
    sess.get(a) == sess.get(b)
}
```

`Sym` is copyable. `&str` is borrowed from `Session`. rustc’s `Ty<'tcx>` is this idea with more types. You do not `Arc` every interned string; you keep the arena alive for the session.

Trying to store `sess.get(a)` in a struct that outlives `sess` is the Chapter 4 error. rustc’s answer is “don’t store the `&str`; store `Ty` / `Sym`.”

## Examples

### Example 1 — Tokio and the `Arc` tax

`tokio::sync::Mutex` versus `std::sync::Mutex`, `Arc<Notify>`, and `JoinHandle` are documented around the same question: what lives until the task completes? The Tokio tutorial’s shared-state chapter is the official reading. Pingora’s 2022 blog describes a multithreaded async proxy — many workers, shared configuration, no request-scoped borrow of the process config. That is `Arc<Config>` plus per-request owned buffers.

### Example 2 — rustc arenas after NLL and GATs

The compiler session (`TyCtxt`) owns arenas. Query results are interned so that `Ty<'tcx>` is a pointer-sized copyable value. Adding a `'tcx` to a new rustc type is a design review, not a syntax chore. When students struggle with “I need the struct to hold a reference,” rustc’s answer is usually “hold an interned id.”

GATs (1.65) and RPITIT/AFIT (1.75) then let rustc and the ecosystem *write traits* over those session types without boxing every future.

### Example 3 — Wasmtime stores and WASI 0.2

Wasmtime’s `Store<T>` is the owner of guest memory. Host functions borrow the store for a call and must not alias it after return. WASI 0.2’s Component Model (Bytecode Alliance, January 2024) refuses the old C habit of “here is a pointer into the other module’s memory.” The lifetime is the call; the ownership is the store. That is Chapter 4 as an ABI.

## Applications in Systems Programming

**Request workers.** Own the request `Bytes`, share the `Arc<AppState>`, never put a `&Request` in a `'static` task.

**Compilers and analyzers.** rust-analyzer mirrors rustc: a database owns the world, ids are copy, borrows are short.

**Caches.** `OnceLock<Config>` for process-global immutable config; `RwLock<HashMap<..>>` for a mutable cache; interned keys if the cache is on the type-checker path.

**Embedders.** Game engines and language runtimes (Wasmtime, Deno’s V8 bindings) pick one owner for the isolate/store and treat everything else as a handle.

## Challenges and Extensions

`Arc<Mutex<T>>` soup is the failure mode of “it compiled.” If every field is behind a lock, you no longer have an ownership story; you have a deadlock generator. Prefer channels (Chapter 5) or shard the map.

Self-referential structs still do not become easy. Production code uses `ouroboros` / `yoke` rarely and arenas often.

Lending iterators and GATs have syntax cost. Teams document the desugaring (`type Item<'a> = &'a T`) so the next reader does not think it is magic.

Reflect: how would Rust’s ownership model change the way you design a shared cache in a multithreaded system? Who owns the entries? Who may borrow them? What is the session that ends the cache?

---

## Worked Examples & Deep Dive

### Worked Example 1: Decoding `T: 'static` — Owned Values vs. Borrowed References

Many Rust learners misread `T: 'static` as “must be a `const` global.” Check four concrete types against `fn spawn_job<T: Send + 'static>(job: T)`:

| Concrete Type of `job` | Contains Borrowed Ref? | Satisfies `T: 'static`? | Why? |
|---|---|---|---|
| `String::from("hello")` | No (owns heap bytes) | **Yes ✓** | Can be kept alive as long as the task wants! |
| `Arc<Config>` | No (shared heap owner) | **Yes ✓** | Refcount keeps `Config` alive until task drops it |
| `&'static str` (`"ok"`) | Borrowed, but lives forever | **Yes ✓** | Embedded in binary `.rodata` |
| `let s = String::new(); &s` (`&'local str`) | Borrowed from stack frame | **No ✗** | Stack frame might pop before task finishes! |

### Worked Example 2: Tracing a Zero-Copy GAT `LendingIterator` Buffer Reuse

Compare reading 1,000,000 lines from a file:
1. **Standard `BufRead::lines()` (`Iterator<Item = io::Result<String>>`):** Must allocate a fresh `String` on every `.next()` call because the trait allows callers to collect all lines simultaneously (`Vec<String>`). **Allocations: 1,000,000.**
2. **GAT `LendingIterator` (`type Item<'a> = &'a str where Self: 'a;`, `fn next<'a>(&'a mut self) -> Option<Self::Item<'a>>`):**
   - Call 1: clears internal `self.buf`, reads line 1 into `self.buf`, returns `Some(&'a str)` borrowing `&'a mut self`.
   - Call 2: borrow checker **forbids** calling `.next()` again until the previous `&'a str` is no longer used! Once dropped, `self.buf` is overwritten in place with line 2. **Allocations: 1 total!**

### Explanation: Choosing an Ownership Architecture for Real Systems

#### Step 1: Never fight `'static` across `tokio::spawn`—wrap shared state in `Arc`
If a task needs access to a database pool, router, or config, clone an `Arc` handle (`let db = Arc::clone(&state.db);`) and `move` it into the `async` block. One atomic increment costs ~5 nanoseconds and keeps your entire architecture lifetime-free.

#### Step 2: Use Arenas when objects share a phase-scoped lifespan
If you build a giant graph of nodes during one HTTP request or one compiler pass and throw the whole graph away together, an arena gives you pointer-speed `&'a Node` traversal with zero `Rc` overhead and O(1) bulk deallocation.

#### Step 3: Reach for GATs only when standard `Iterator` truly forces unacceptable copies
Standard `Iterator` unlocks `.map()`, `.filter()`, `.collect()`, and `for` loops. Use a GAT `LendingIterator` specifically on hot streaming I/O paths where reusing one scratch buffer eliminates millions of allocations.

#### Step 4: Match lifetime complexity to the layer of the stack
Low-level parsers and arenas love explicit lifetimes (`'a`, `'tcx`); high-level application services and async actors love owned types (`String`, `Bytes`, `Arc<T>`). Separating those two layers keeps both fast and ergonomic.

## Exercises

1. **Conceptual.** Classify each as session-lifetime or task-lifetime: a rustc `Ty`, a Tokio spawned logger, a Wasmtime `Store`, a per-HTTP-request `Bytes`.
2. **Code fix.** Repair `naive` above with `Arc` without cloning `policy` bytes.
3. **API design.** Change `Session::get` to return `Sym` only to the caller, and add a separate `resolve` at the print boundary. What call sites got simpler?
4. **Interior mutability.** Replace `Session`’s `&mut self` intern table with `RefCell<Vec<String>>` so `check` can intern on a miss. Then explain why a compiler would still prefer `&mut` during a phase.
5. **Implementation.** Build a tiny interned-string table with `Arc<Session>` shared across two threads that only *read* after intern completes (`OnceLock` or a barrier). No data races, no cloned payloads.

## References

- [Announcing Rust 1.65.0](https://blog.rust-lang.org/2022/11/03/Rust-1.65.0/) — GATs stable.
- [Tokio shared state](https://tokio.rs/tokio/tutorial/shared-state) — `Arc` + lock patterns.
- [WASI 0.2 Launched](https://bytecodealliance.org/articles/WASI-0.2) — Component Model as a lifetime/ABI choice.
- [The Rustonomicon — ownership of resources](https://doc.rust-lang.org/nomicon/ownership.html).
- rustc-dev-guide, “Type interning and arenas” — [rustc-dev-guide.rust-lang.org](https://rustc-dev-guide.rust-lang.org/).

## Recap

- Production lifetimes are sessions, tasks, or ids — not decorations.
- Tokio pays `Arc` at scheduler boundaries; rustc pays arenas inside a session.
- GATs (2022) are why modern traits can return borrows and futures without a box.
- Interior mutability is an operational choice (poison, fairness, once) as much as a type.

## Video tham khảo

- [Crust of Rust: Send, Sync, and their implementors](https://www.youtube.com/watch?v=yOezcP-XaIw) — Jon Gjengset, English. **xem trước.** Watch first: Send and Sync, the reason a Tokio task stores Arc instead of a borrowed client.
- [Crust of Rust: Lifetime Annotations](https://www.youtube.com/watch?v=rAl-9HwD858) — Jon Gjengset, English. Lifetime annotations as a session scope — the same idea as rustc’s arena lifetime.
- [Crust of Rust: Smart Pointers and Interior Mutability](https://www.youtube.com/watch?v=8O0Nt9qY_vo) — Jon Gjengset, English. Arc, Weak, and interior mutability as the sharing tools those runtimes actually reach for.
