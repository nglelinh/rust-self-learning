---
layout: post
title: "03-16 Modern Applications — Traits, Errors, and Collections at Scale"
chapter: "03"
order: 16
owner: "OpenCode"
lang: en
categories:
  - chapter03
lesson_type: optional
---

Open any modern Rust repository from 2024–2026—a web backend in `axum`, a CLI tool like `ripgrep`, or a database engine—and you will notice something striking: Chapter 3’s everyday tools (`?`, traits, and `HashMap`) have evolved into the **public ABI of the entire Rust ecosystem**. Applications and libraries split cleanly between `anyhow` and `thiserror`; Rust 1.75 stabilized `async fn` right inside traits (retiring heap-boxing macros on hot paths); and every `std::collections::HashMap` you instantiate is actually powered under the hood by Google’s SIMD-accelerated SwissTable (`hashbrown`).

This optional capstone shows how Chapter 3’s abstractions scale to production systems—connecting error-design ergonomics, zero-allocation async traits, and hardware-friendly hash tables with concrete byte-level traces.

---

## Key ideas: Production Rust’s Three Chapter-3 Pillars

### 1. Structured vs. Erased Error Handling (`thiserror` vs. `anyhow`)
Libraries need **structured error enums** (`thiserror`) so callers can `match` on specific failure variants without losing type information. Applications (CLIs, servers) mostly log, attach human context, and bubble errors up—where **type-erased error containers** (`anyhow::Error`) shine by storing any `std::error::Error + Send + Sync + 'static` behind a slim single-word pointer.
- **Core questions:** Why shouldn’t a public library expose `anyhow::Error` in its API? How does `#[from]` automate `?` conversions?
- **Real-world reach:** Virtually every modern Rust crate on `crates.io`.

### 2. Return-Position `impl Trait` in Traits (RPITIT) & Native `async fn` in Traits
Before Rust 1.75, writing `async fn` inside a trait required the `#[async_trait]` macro, which silently wrapped every call in `Pin<Box<dyn Future>>`—forcing a heap allocation on *every* async method call! Rust 1.75 introduced native **RPITIT**, letting each trait implementation return its own concrete, stack-allocated state-machine `Future` with zero heap overhead.
- **Core questions:** What happens when you need `Send` bounds on an async trait method in multi-threaded runtimes like Tokio?
- **Real-world reach:** Embedded async I/O (`embedded-hal-async`), `tower` middleware, and zero-allocation network drivers.

### 3. SIMD Probing & Cache-Friendly Hash Tables (`hashbrown` / SwissTable)
Traditional hash tables chase linked-list pointers on collisions (cache misses!). Rust’s `HashMap` uses **open addressing with SIMD control bytes** (`hashbrown`): 16 one-byte metadata tags sit contiguously in a single 128-bit SSE2/NEON register, letting the CPU test **16 hash slots in one single instruction**!
- **Core questions:** Why is `HashMap` fast in practice despite using SipHash for DoS resistance?
- **Real-world reach:** Compilers (`rustc`), browsers, key-value stores, and high-throughput telemetry aggregators.

## 60-minute teaching plan

- 0 to 10 min: The application/library error split that crates.io actually follows.
- 10 to 25 min: `thiserror` / `anyhow` in a small service, then in a library crate.
- 25 to 40 min: RFC 3185 — `async fn` in traits (Rust 1.75) and what is still not dyn-safe.
- 40 to 50 min: `hashbrown`, SwissTable, and why `std::collections::HashMap` changed underneath you.
- 50 to 60 min: Iterator pipelines in compilers and data services (`rayon`, rustc query-shaped thinking).

## Objectives

You will stop treating traits as a classroom abstraction and start reading them as *stability promises*: a trait method that becomes `async fn` is an RFC, a release blog post, and a migration for every HTTP client crate. You will choose `anyhow` versus `thiserror` the way production codebases do, and you will know that your `HashMap` is a SwissTable (hashbrown) even when you never typed that name. The mental-model shift is from “I implemented `Display`” to “I am designing a boundary that Pingora, Tokio, and rustc all have to live with.”

## Prerequisites

Required Chapter 3: `Result`/`?`, panic versus recoverable errors, collections and iterators, traits, and generics versus trait objects. You should be able to write a custom error enum and a trait with a generic parameter. Async syntax helps but is not required; we will treat `async fn` as “a function that returns `impl Future`.”

## Introduction

By 2022 the Rust error story had converged socially even though the language did not pick a winner. Library crates published typed enums via `thiserror`. Application binaries collapsed everything into `anyhow::Error` (or `eyre` / `color-eyre` for prettier reports). That split is now as conventional as `cargo fmt`.

The trait story took longer. For years, `async fn` in a trait meant the `async-trait` crate, a box, and a heap allocation per call. RFC 3185 (static async fn in traits) plus RFC 3425 (RPITIT) landed in rustc and stabilized in Rust 1.75 on 28 December 2023. The Rust blog’s 21 December 2023 announcement is the primary source: you can write `async fn` and `-> impl Trait` in traits, with remaining limits on dyn safety and `Send` bounds.

Collections quietly changed earlier and then kept paying rent. Since Rust 1.36, `std::collections::HashMap` has been hashbrown — a Rust port of Google’s SwissTable — but the crate itself remains the `no_std` and SIMD-tuning escape hatch through 2025–2026 releases. If you write `HashMap::new()` in a kernel or a WASM module, you are choosing whether that SwissTable is allowed to talk to an allocator.

## Key Concepts

### Errors are a product decision: typed versus erased

`thiserror` derives `std::error::Error` for enums you intend callers to match on. `anyhow` erases the type so a binary can `?` through files, HTTP, and JSON without naming every failure. The crates.io README for anyhow states the rule in one paragraph: use anyhow when you do not care what the error *is*; use thiserror when you are a library and you do.

```rust
use thiserror::Error;

#[derive(Debug, Error)]
pub enum FilterError {
    #[error("unknown host class: {0}")]
    Class(String),
    #[error("io")]
    Io(#[from] std::io::Error),
}
```

A gateway crate should look like that. The binary that *uses* the gateway should not.

### Traits became the async ABI

`async fn` in a trait desugars to a method returning an anonymous associated type that implements `Future`. That is why GATs (stabilized in 1.65, November 2022) were a prerequisite. The remaining sharp edge is object safety: a trait with `async fn` is not `dyn`-safe in 1.75. The official workaround is `trait-variant` (a rust-lang crate) to emit `Send` and non-`Send` variants, or `async-trait` when you still need `dyn`.

This is Chapter 3’s “generics versus trait objects” lecture wearing a 2024 date stamp. Static dispatch got first-class async. Dynamic dispatch is still a product choice with a cost.

### Hash maps are an algorithm you already shipped

hashbrown documents itself as a SwissTable port: SIMD group lookups, one-byte control tags, empty maps that allocate nothing. `std` wraps it with a DoS-resistant default hasher. The crate keeps a faster default hasher (`foldhash` in recent versions) for `no_std` and specialized hosts. Production services that parse untrusted keys keep the std hasher; compilers and game engines often switch.

Iterators sit on top. rustc, rust-analyzer, and data crates (`rayon`) are iterator machines. `rayon`’s `par_iter()` is the same `Iterator` mental model with a work-stealing backend — Chapter 3 collections, Chapter 5 concurrency, one adaptor name.

## Code Walkthroughs

### Naive: one `Box<dyn Error>` everywhere

```rust
fn load_policy(path: &str) -> Result<String, Box<dyn std::error::Error>> {
    Ok(std::fs::read_to_string(path)?)
}
```

This compiles and is fine for a five-line tool. In a library it is a contract that says “I will never let you match on failure.” Callers start string-matching error text. That is how 2019 prototypes leaked into 2024 incidents.

### Compiler-guided: a typed library error

```rust
use thiserror::Error;

#[derive(Debug, Error)]
pub enum PolicyError {
    #[error("read {path}")]
    Read {
        path: String,
        #[source]
        source: std::io::Error,
    },
}

pub fn load_policy(path: &str) -> Result<String, PolicyError> {
    std::fs::read_to_string(path).map_err(|source| PolicyError::Read {
        path: path.to_string(),
        source,
    })
}
```

`#[source]` preserves the chain `anyhow` and loggers already know how to walk. The required Chapter 3 lesson on `?` still applies; only the *boundary type* changed.

### Idiomatic: library types in, application erasure at `main`

```rust
// library
pub trait Resolver {
    fn lookup(&self, host: &str) -> Result<Class, PolicyError>;
}

// application
fn main() -> anyhow::Result<()> {
    let policy = filter_core::load_policy("policy.txt")?;
    let class = StaticResolver::from_text(&policy)?.lookup("example.com")?;
    println!("{class:?}");
    Ok(())
}
```

```rust
// 1.75+ : async as a trait method, still generic — not dyn
trait Resolver {
    async fn lookup(&self, host: &str) -> Result<Class, PolicyError>;
}

impl Resolver for StaticResolver {
    async fn lookup(&self, host: &str) -> Result<Class, PolicyError> {
        self.lookup_blocking(host)
    }
}
```

If you need `Box<dyn Resolver>`, you will feel RFC 3185’s documented limitation. That feeling is the lesson: traits are not “interfaces” in the Java sense until you pay for object safety.

### Collections: prefer the iterator you already have

```rust
fn blocked_hosts(rows: &[Row]) -> Vec<&str> {
    rows.iter()
        .filter(|r| r.action == Action::Deny)
        .map(|r| r.host.as_str())
        .collect()
}
```

A naive `for` with `clone()` everywhere compiles and fails the performance budget of a compiler pass or an edge filter. rustc’s own passes are written in this style for a reason: iterators make the allocation points visible (`collect`) and the borrow points obvious (`&str`).

## Examples

### Example 1 — HTTP client crates after Rust 1.75

`hyper` 1.0 (late 2023) and the Tokio stack (`axum`, `reqwest`) spent 2023–2025 deleting `#[async_trait]` from public traits where static dispatch was enough. Reading a post-1.75 `Service`-shaped trait is Chapter 3 plus RFC 3185. When a crate still depends on `async-trait`, ask whether they need `dyn` or whether they have not migrated yet.

### Example 2 — hashbrown in std and outside it

The hashbrown README (0.14–0.17 line, 2023–2026) is explicit: std adopted the implementation; the crate remains for `no_std`, custom allocators, and rayon parallel iterators. A kernel hashmap and a CLI hashmap share an algorithm and split on hasher and allocator. That is a collections lecture with a safety appendix: pick a hasher the way you pick a lock.

### Example 3 — rustc and rust-analyzer as iterator machines

Both compilers walk token streams, HIR, and type tables with iterator adaptors. Errors are typed (`rustc_errors`) and then rendered. When students write `collect::<Vec<_>>()` inside a hot loop, they are making the same mistake a compiler intern makes in week one. The fix is the Chapter 3 instinct: fuse, filter, take ownership only at the sink.

## Applications in Systems Programming

**Public crate design.** `thiserror` enums plus a small trait are how 2024 networking crates version their failures. Changing a variant is a SemVer event.

**Edge proxies.** Pingora’s crate graph includes dedicated error types (`pingora-error`). A trillion-request proxy that returned `Box<dyn Error>` would be unpageable.

**Data-parallel CLI and build tools.** `rayon` turns `iter()` into `par_iter()` when the adapter is `Send`. That is traits as a concurrency gate.

**`no_std` maps.** Embedded and WASM kernels pull `hashbrown` directly so SwissTable works without std’s hasher or runtime.

## Challenges and Extensions

`async fn` in public traits still wants a `Send` story. The rust-lang `trait-variant` crate exists because “works on Tokio, deadlocks on a local executor” is a support nightmare.

Error types grow. A 40-variant enum is not more professional than four; it is a sign the crate should split.

HashDoS is real. Using hashbrown’s fast hasher on untrusted HTTP headers is a vulnerability. std’s default exists for that reason.

Reflect: if you were designing a plugin trait for a proxy (`async fn on_request`), would you stabilize it as generic, as `dyn`, or as both? What does each choice cost the next five years of crates.io?

---

## Worked Examples & Deep Dive

### Worked Example 1: Tracing One 16-Slot SwissTable (`hashbrown`) Lookup by Hand

Suppose you look up key `"user_42"` inside Rust’s `HashMap` (`hashbrown`). Its 64-bit hash splits into two parts:
- **H1 (top 57 bits):** picks the starting group index `g = H1 % num_groups`.
- **H2 (bottom 7 bits):** a tiny 7-bit fingerprint, say `0x5A` (`0b0101_1010`).

Now inspect the 16-byte control block for group `g`:
```text
Control bytes [0..16]: [FF, 12, 5A, FF, 03, 5A, 80, FF, ...]
  0xFF = EMPTY   |   0x80 = DELETED   |   0x00..0x7F = FULL (stores H2 tag)
```
1. **1 SIMD instruction** (`_mm_cmpeq_epi8`) compares `0x5A` against all 16 control bytes simultaneously, producing a 16-bit bitmask `0b0000_0000_0010_0100` (hits at slots **2** and **5**!).
2. Rust checks key equality **only** at slot 2 and slot 5—completely skipping the other 14 slots without touching their keys in RAM!
3. If neither matches and the group contains at least one `0xFF` (EMPTY) byte, lookup stops immediately (`None`)—zero wasted probing!

### Worked Example 2: Heap Allocations of `#[async_trait]` vs. Native `async fn` (Rust 1.75+)

Compare calling an async trait method `store.get(id).await` 1,000,000 times/second:

| Approach | Return Type of `get` | Heap Allocs / 1M calls | Works with `dyn Store`? |
|---|---|---|---|
| `#[async_trait]` macro | `Pin<Box<dyn Future<Output = ...> + Send>>` | **1,000,000 heap allocs** | Yes ✓ |
| Native `async fn` in trait (1.75+) | Concrete opaque `impl Future<Output = ...>` | **0 heap allocs!** | Not directly (needs wrapper/erasure) |

### Explanation: Designing Chapter-3 Boundaries for Real Production Code

#### Step 1: Split libraries (`thiserror`) from binaries (`anyhow`) on day one
If external callers might branch on *why* something failed (retry on timeout, abort on invalid auth), define a `#[derive(thiserror::Error)]` enum. In your `main.rs` binary or CLI, wrap errors with `anyhow::Context` (`.with_context(|| format!("reading {}", path.display()))?`) so operators get crisp error chains instead of bare `"No such file or directory"`.

#### Step 2: Keep native `async fn` in traits when static dispatch suffices
If your service takes `<S: Storage>` generically, native `async fn` gives zero-cost inlined futures. Only pay for `Box<dyn Future>` (or `trait_variant::make(SendStore: Send)`) when you genuinely need a `dyn` plugin boundary or multi-threaded `tokio::spawn` `Send` bounds.

#### Step 3: Pick your hasher deliberately when CPU profiling proves it matters
`std::collections::HashMap` defaults to **SipHash 1-3**, which defends against HashDoS attacks on untrusted web inputs. For internal trusted integer keys (like compiler AST node IDs), swapping in `FxHashMap` or `AHashMap` keeps the exact same SwissTable SIMD layout while slashing hashing overhead.

#### Step 4: Treat `?`, traits, and collections as one cohesive design language
Notice how `?` relies on the `From` trait, `HashMap` relies on `Eq + Hash`, and `async fn` relies on `Future`. Mastering Chapter 3’s traits unlocks the entire standard library and ecosystem without special-case compiler magic.

## Exercises

1. **Conceptual.** Write the anyhow/thiserror rule in two sentences a new teammate can memorize. Give one crate in this course’s dependency set that should use each.
2. **Code fix.** Convert a function that returns `Box<dyn Error>` into a `thiserror` enum with a `#[from]` I/O variant. Keep `?`.
3. **Traits.** Write a `trait Store { async fn get(&self, k: &str) -> Option<String>; }` on Rust 1.75+. Then try `Box<dyn Store>` and record the compiler error in your notes.
4. **Collections.** Reimplement `blocked_hosts` with an explicit `for` and clones. Measure or at least *count* allocations. Restore the iterator version.
5. **Implementation.** Add a `HashMap<String, Class>` policy table using std, then the same table via the `hashbrown` crate behind `cfg(feature = "fast-map")`. Document hasher choice in a comment.

## References

- [Announcing `async fn` and RPITIT](https://blog.rust-lang.org/2023/12/21/async-fn-rpit-in-traits/) — Rust blog, 21 December 2023.
- [RFC 3185 — static async fn in traits](https://rust-lang.github.io/rfcs/3185-static-async-fn-in-trait.html).
- [Announcing Rust 1.75.0](https://blog.rust-lang.org/2023/12/28/Rust-1.75.0/).
- [anyhow](https://crates.io/crates/anyhow) and [thiserror](https://crates.io/crates/thiserror) — dtolnay; industry split.
- [hashbrown](https://crates.io/crates/hashbrown) — SwissTable implementation behind `std::collections::HashMap`.

## Recap

- Application binaries erase errors; libraries type them. That is a 2022–2026 social standard, not a style preference.
- RFC 3185 made `async fn` a trait-level ABI; dyn dispatch is still a separate bill.
- Your HashMap is hashbrown; hasher and allocator remain your responsibility.
- Iterators are how compilers and edge filters keep allocation honest.

## Video tham khảo

- [Rust Error Handling Explained: thiserror vs anyhow (Best Practices)](https://www.youtube.com/watch?v=-c9JEexiHPE) — Semicolon, English. **xem trước.** Watch first: the anyhow-versus-thiserror product split this lesson treats as convention.
- [Two Ways To Do Dynamic Dispatch](https://www.youtube.com/watch?v=wU8hQvU8aKM) — Logan Smith, English. Dynamic versus static dispatch at an API boundary, the trait half of the lesson.
- [CppCon 2017: Matt Kulukundis “Designing a Fast, Efficient, Cache-friendly Hash Table, Step by Step”](https://www.youtube.com/watch?v=ncHmEUmJZf4) — CppCon, English. Matt Kulukundis’s CppCon talk on SwissTable, the hash table design behind std HashMap. The talk is C++.
