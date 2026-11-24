---
layout: post
title: "06-31 Modern Applications — Macros, Unsafe, FFI, and Compilers"
chapter: "06"
order: 31
owner: "OpenCode"
lang: en
categories:
  - chapter06
lesson_type: optional
---

Chapter 6 introduced the sharpest power tools in the Rust toolbox: declarative and procedural macros, raw pointers inside `unsafe`, Foreign Function Interface boundaries, and property-based testing. In toy exercises, those features can feel abstract—almost like escape hatches you hope never to touch.

In production Rust infrastructure, those "escape hatches" *are* the product. From `serde` and `sqlx` generating zero-cost parsers at compile time, to `PyO3` accelerating Python data pipelines by $10\times$–$100\times$, to `Miri` catching subtle aliasing violations before they ever reach a CPU, this lesson shows how modern teams wield Chapter 6 primitives safely at scale.

## Background Knowledge Map: Where Macros, `unsafe`, and Formal Checking Meet in Industry

Modern Rust ecosystems rely on four interlocking technical pillars from Chapter 6:

1. **Compile-Time Metaprogramming & Macro Hygiene**:
   Rust macros operate on token streams (`TokenStream`) rather than raw text substitution like C's `#define`. **Declarative macros** (`macro_rules!`) enforce *hygiene*—local variables introduced inside a macro cannot accidentally shadow or collide with variables in the caller's scope. **Procedural macros** (`#[derive(...)]`, attribute macros, function-like macros) run arbitrary Rust code inside the compiler to parse syntax trees (`syn`) and emit type-checked Rust code (`quote`), powering `serde`, `tokio::main`, and compile-time SQL verification in `sqlx`.

2. **Aliasing Models (`Stacked Borrows` & `Tree Borrows`)**:
   In safe Rust, the borrow checker guarantees that `&mut T` has exclusive access (no aliases). Inside `unsafe`, you can create raw pointers (`*mut T`, `*const T`) that bypass the borrow checker—but the LLVM optimizer *still* assumes Rust's aliasing rules hold (`noalias` LLVM IR attribute). Violating those rules triggers **Undefined Behavior (UB)** even if your program appears to print the right answer today. Formal operational models like **Stacked Borrows** and **Tree Borrows** define exactly which pointer derivations remain valid at every instruction.

3. **Interpreted UB Detection (`Miri`) & Property Testing**:
   Because UB is invisible to ordinary unit tests, modern Rust teams run their test suites under **Miri**—an interpreter for Rust's Mid-level Intermediate Representation (MIR) that tracks per-byte provenance, initialization state, and borrow stacks on every memory access. Paired with `proptest` or `cargo-fuzz`, Miri turns subtle `unsafe` bugs into deterministic, reproducible failures.

4. **Polyglot FFI Bridges (`PyO3`, `wasm-bindgen`, `cxx`)**:
   Rather than writing hand-rolled `extern "C"` boilerplate, modern ecosystems generate safe FFI glue via procedural macros. `PyO3` wraps CPython's C API so Rust structs appear as native Python classes; `cxx` statically verifies shared type signatures between Rust and C++ at compile time.

## 60-minute teaching plan

- 0 to 10 min: Compilers as Rust applications (rustc, Cranelift, rust-analyzer).
- 10 to 25 min: FFI that ships — PyO3, `cxx`, bindgen — and the `unsafe` boundary.
- 25 to 40 min: Miri, Tree Borrows, and why tests are not enough.
- 40 to 50 min: Fuzzing (`cargo-fuzz`) and property tests on real parsers.
- 50 to 60 min: Proc macros as compiler plugins you will maintain.

## Objectives

You will be able to map each required Chapter 6 topic onto a named production system: macros → `serde`/`thiserror`/`tokio::select`; unsafe → Cranelift’s assembler or a `cxx` bridge; FFI → PyO3 extension modules; testing → Miri in CI. The mental-model shift is from “unsafe is a keyword I should never type” to “unsafe is a reviewed crate that the rest of the world calls through a safe API” — the same ethic the capstone already asked for, now with citations.

## Prerequisites

Required Chapter 6: property/fuzz testing, declarative macros, unsafe, FFI, and the capstone shape of a small product. You should be able to write a `macro_rules!` helper, call a C function through `extern "C"`, and explain why a test that never hits Miri can still hide undefined behavior.

## Introduction

Rust’s compiler is written in Rust. That sentence stopped being cute around the time Cranelift (Bytecode Alliance) became a serious alternative codegen backend and Wasmtime became the reference implementation for WASI 0.2 (January 2024). A JIT, a WASM runtime, and rustc are all *applications* of Chapter 6: they parse, they allocate, they call LLVM or Cranelift through controlled unsafe, and they fuzz the parser.

Meanwhile Python and C++ did not go away. `PyO3` made “rewrite the hot loop in Rust, keep the notebook in Python” a default 2023–2026 strategy for data and CLI tools. `cxx` (dtolnay) made bidirectional Rust/C++ bridges that do not look like 1998 JNI. `bindgen` and `cbindgen` remain the generate-and-pray pair, only now with more CI.

The verification story matured too. Miri gained the Tree Borrows aliasing model (stacked borrows’ successor, developed through 2022–2025) as an experimental and then increasingly default way to catch UB that unit tests miss. `cargo-fuzz` (libFuzzer) is how rustc, wasmtime, and image parsers spend CI cycles.

## Key Concepts

### A compiler is an unsafe-using safe API

Cranelift emits machine code. Someone, somewhere, writes bytes into executable memory. That is `unsafe`. The *product* is the safe `Context::compile` API that Wasmtime and `rustc_codegen_cranelift` call. Chapter 6’s rule — unsafe is allowed when it is sealed and documented — is Cranelift’s existence proof.

```rust
// Shape only — not a Cranelift tutorial.
fn compile_add() {
    // Build a CLIF function, ask Cranelift for a CodeBlob,
    // later: unsafe { transmute and call } inside the runtime, not here.
}
```

Students should be able to *point* at the crate where `unsafe` belongs (the JIT) and the crate where it must not (the CLI that passed the source).

### FFI is an ownership translation

PyO3 translates Python’s refcounted objects into Rust borrows with GIL markers. `cxx` translates C++ unique/shared pointers into `UniquePtr`/`SharedPtr`. Both are Chapter 2 + Chapter 4 + Chapter 6. The 2022–2026 versions of these crates added more safe wrappers, async hooks (PyO3 + pyo3-asyncio / `pyo3[experimental-async]`), and better exception mapping — they did not remove the boundary.

```rust
use pyo3::prelude::*;

#[pyfunction]
fn allow(host: &str) -> bool {
    !host.ends_with(".invalid")
}

#[pymodule]
fn filter_py(m: &Bound<'_, PyModule>) -> PyResult<()> {
    m.add_function(wrap_pyfunction!(allow, m)?)?;
    Ok(())
}
```

The `unsafe` is inside PyO3. Your module is the safe facade. That is the only FFI architecture this course endorses.

### Miri is a semantics CPU

`cargo +nightly miri test` interprets your program and checks aliasing, alignment, and initialization. Tree Borrows (Neven Villani, Ralf Jung, and others; see the 2023–2025 UCG/Miri write-ups) models what `&` and `&mut` mean more closely to what rustc actually optimizes. A crate that wraps a C parser and never runs Miri is guessing.

### Macros are compiler surface area

`serde`’s derive, `tokio::select!`, `thiserror`, and rustc’s own `fluent` diagnostics are proc macros or `macro_rules!`. They expand before type checking. A 2024–2026 production bug class is “the macro expanded to something that only fails on a rare cfg.” `trybuild` and `macrotest` are how library authors snapshot that expansion — Chapter 6 testing applied to the compiler frontend.

### Deep-Dive Explanation: Hand-Tracing a Stacked Borrows Violation That Miri Catches (Read Slowly)

One of the most common `unsafe` surprises for systems programmers coming from C is that **creating a reference can invalidate an earlier raw pointer—even if you never write through the reference**. Let us trace the exact borrow stack step by step as Miri executes the following 6-line snippet:

```rust
let mut x: i32 = 10;
let p1: *mut i32 = &mut x;          // Step 1
let r_shared: &i32 = &x;            // Step 2
let val = *r_shared;                // Step 3
unsafe { *p1 = 20; }                // Step 4: UB!
```

Under **Stacked Borrows**, every memory location maintains a stack of *permissions* (tags) from bottom (oldest owner) to top (most recently derived borrow). A pointer can only access memory if its tag is currently on the stack; writing through a tag pops everything above it, and creating a shared reference pushes a `SharedReadOnly` tag while popping any conflicting `Unique` mutable borrows above the parent.

#### Step-by-Step Borrow Stack Trace for Location `x`

| Step | Statement | Borrow Stack for `x` (bottom → top) | What happened? |
| :---: | :--- | :--- | :--- |
| 0 | `let mut x = 10;` | `[Unique(x)]` | Local variable `x` owns the 4-byte stack slot. |
| 1 | `let p1 = &mut x as *mut i32;` | `[Unique(x), SharedRW(p1)]` | Deriving a raw `*mut` from `&mut x` pushes a `SharedReadWrite` tag for `p1`. |
| 2 | `let r_shared = &x;` | `[Unique(x), SharedRO(r_shared)]` | Reborrowing directly from `x` searches down to `Unique(x)` and **pops `SharedRW(p1)` off the stack**, then pushes `SharedReadOnly(r_shared)`. |
| 3 | `let val = *r_shared;` | `[Unique(x), SharedRO(r_shared)]` | Valid read: `SharedRO(r_shared)` is on the stack (`val = 10`). |
| 4 | `unsafe { *p1 = 20; }` | **ERROR: tag `p1` not in stack!** | Miri halts with Undefined Behavior: `p1` was permanently invalidated at Step 2! |

#### Why Does This Matter to the Compiler?

When LLVM compiles Step 3 and Step 4 with optimizations enabled (`--release`), `r_shared` carries the promise that `x` cannot be mutated through any unrelated pointer while `r_shared` is active. If the optimizer hoists or caches `x` in a CPU register across Step 4, your program silently observes stale data—even though on an unoptimized debug build it "seemed to work."

#### The Fix (Hand-Verifying the Valid Stack)

If you derive `r_shared` *from `p1`* (or finish using `r_shared` before creating `p1` from `x`), `p1`'s tag stays alive beneath it:
```rust
let mut x: i32 = 10;
let p1: *mut i32 = &mut x;                // Stack: [Unique(x), SharedRW(p1)]
let val = unsafe { *p1 };                 // Read via p1: SharedRW(p1) is on top → OK (val = 10)
unsafe { *p1 = 20; }                      // Write via p1: SharedRW(p1) is on top → OK (x = 20)
```
Running `cargo +nightly miri test` checks this exact stack discipline on every byte of your `unsafe` code automatically.

## Code Walkthroughs

### Naive: copy-paste `extern "C"` in the app crate

```rust
extern "C" {
    fn parse_policy(ptr: *const u8, len: usize) -> i32;
}

fn load(s: &str) -> bool {
    unsafe { parse_policy(s.as_ptr(), s.len()) == 0 }
}
```

This will compile. It will also pass a `*const u8` that the C side may free, store, or read past `len`. The app crate now contains UB you cannot see.

### Compiler-guided: isolate the island

```rust
// crates/policy-ffi/src/lib.rs
mod c {
    extern "C" {
        pub fn parse_policy(ptr: *const u8, len: usize) -> i32;
    }
}

/// SAFETY: `s` is borrowed only for the call; C must not retain `ptr`.
pub fn parse(s: &str) -> Result<(), ParseError> {
    let code = unsafe { c::parse_policy(s.as_ptr(), s.len()) };
    if code == 0 { Ok(()) } else { Err(ParseError::Code(code)) }
}

#[derive(Debug)]
pub enum ParseError { Code(i32) }
```

The required FFI lesson already demanded a SAFETY comment. The *application* lesson is: the binary never writes `extern "C"`. Cranelift, PyO3, and `cxx` are this pattern at 50,000 lines.

### Idiomatic: prefer a maintained bridge

```rust
// cxx bridge shape — C++ owns the parser, Rust owns the request.
#[cxx::bridge]
mod ffi {
    unsafe extern "C++" {
        include!("policy.h");
        type Parser;
        fn parse(p: &Parser, host: &str) -> bool;
    }
}
```

`cxx` generates the glue and keeps C++ exceptions from unwinding through Rust. That is a 2022–2026 default when both sides are compiled in one Cargo/CMake graph (Chromium-style, Firefox-style, game-engine-style).

### Testing the island

```bash
cargo test
cargo +nightly miri test -p policy-ffi
cargo fuzz run parse_policy
```

Three layers: examples (capstone), interpreter (Miri), coverage-guided (libFuzzer). rustc and Wasmtime all three. A student project that only has `#[test] fn it_works()` has not yet met Chapter 6’s production bar.

## Examples

### Example 1 — Cranelift and Wasmtime

Cranelift is a fast code generator used by Wasmtime and as `rustc_codegen_cranelift` for debug builds. Wasmtime implementing WASI 0.2 (Bytecode Alliance, 2024) is a compiler *and* an OS ABI. The unsafe sits in signal handlers, JIT mapping, and Cranelift’s emission; the Component Model API is safe Rust. Read the Wasmtime book’s embedding chapter as a Chapter 6 FFI lesson that happens to target WASM instead of C.

### Example 2 — PyO3 in data and tooling

Pydantic’s Rust core (`jiter`, `pydantic-core`, 2023+), Ruff (Astral, 2022–2026), and Polars are the celebrity citations: Python UX, Rust engine, PyO3 or a thin C ABI. Ruff in particular is a compiler-shaped linter (parser + visitor + macros) that Python teams installed because it was 10–100× faster than the previous stack. That is the capstone ethic — a focused tool — at ecosystem scale.

### Example 3 — Miri and the language itself

The Rust project runs Miri against the standard library. When Tree Borrows flagged a pattern, the discussion happened on the UCG (unsafe code guidelines) repo and in Ralf Jung’s blog. If you maintain a crate that uses `unsafe` to implement `Vec`-like types, Miri is not optional CI. The required testing-II lesson introduced properties; this lesson names the interpreter that checks the *memory* properties.

## Applications in Systems Programming

**Language runtimes.** Wasmtime, Deno (V8 FFI), and PyO3 modules are all embedders.

**OS / browser glue.** `cxx` and bindgen keep Chromium, Firefox, and Android JNI-adjacent stacks moving without a big-bang rewrite.

**Compiler engineering.** rustc, Cranelift, rust-analyzer, and Ruff are the same pipeline: lex, parse (often via macros), lower, emit.

**Supply-chain tests.** `cargo fuzz` + OSS-Fuzz (Google) is how image, font, and WASM crates get CVEs *before* customers do.

## Challenges and Extensions

Proc macros destroy compile times if they parse more than they must. `syn`/`quote` are powerful and expensive; rustc’s own macros are written with that cost in mind.

`unsafe` reviews do not scale linearly. The Ferrocene qualification story (Chapter 8) exists because “we audited the crate once” is not an argument in a car.

FFI ABI drift is a release process. `cbindgen` output must be a CI artifact, not a laptop souvenir.

Reflect: if your capstone grew a Python API and a WASM API, where does `unsafe` live, and which two tests (Miri, fuzz, C-ABI golden) would you refuse to merge without?

## Exercises

1. **Conceptual.** Draw a box around the `unsafe` in PyO3 versus the `unsafe` in a JIT. What does each box promise the code *outside* it?
2. **Code fix.** Move a raw `extern "C"` out of `main.rs` into a helper crate with a safe function and a SAFETY comment. Add one unit test.
3. **Miri.** Run `cargo +nightly miri test` on a crate that uses `Vec` only. Then introduce an intentional dangling pointer in a test marked `#[cfg(miri)]` and watch Miri fail. Do not leave the dangling pointer in `main`.
4. **Macros.** Write a `macro_rules!` constructor for a `thiserror`-like enum with two variants. Snapshot the expansion with `cargo expand` (or by reading `cargo rustc -- -Z unpretty=expanded` on nightly).
5. **Implementation.** Using only the capstone shape, expose one pure function through a PyO3 module *or* a `cdylib` + header. Document the owner of every pointer in the README.

## References

- [WASI 0.2 Launched](https://bytecodealliance.org/articles/WASI-0.2) — Wasmtime as a compiler/runtime product.
- [Cranelift](https://cranelift.dev/) and [Wasmtime book](https://docs.wasmtime.dev/).
- [PyO3 user guide](https://pyo3.rs/) and [cxx](https://cxx.rs/).
- [Miri](https://github.com/rust-lang/miri) and Ralf Jung’s writing on Tree Borrows / stacked borrows.
- [cargo-fuzz](https://rust-fuzz.github.io/book/) — Rust Fuzz Book.

## Recap

- 2022–2026 compiler and FFI products are Chapter 6: sealed unsafe, generated bridges, Miri, fuzz.
- Cranelift/Wasmtime and PyO3/`cxx` are the two canonical shapes (JIT vs language bridge).
- Macros are part of the compiler surface; test their expansions.
- The capstone’s “safe facade over a sharp core” is how these systems ship.

## Video tham khảo

- [Rust NYC: Jon Gjengset - Demystifying unsafe code](https://www.youtube.com/watch?v=QAz-maaH0KM) — Rust Programming Language, English. **xem trước.** Watch first: unsafe as a reviewed core behind a safe API — this lesson’s ethic.
- [PyO3: From Python to Rust and Back Again (with David Hewitt)](https://www.youtube.com/watch?v=UmL_CA-v3O8) — Developer Voices, English. David Hewitt on PyO3, the FFI boundary the lesson names for Python extensions.
- [Unsafe Rust and Miri by Ralf Jung - Rust Zürisee June 2023](https://www.youtube.com/watch?v=svR0p6fSUYY) — Rust Programming Language, English. Ralf Jung on Miri in the compiler toolchain, the testing half of the same map.
