---
layout: post
title: "01-06 Modern Applications — Toolchains, WASM, and Cargo in Production"
chapter: "01"
order: 6
owner: "OpenCode"
lang: en
categories:
  - chapter01
lesson_type: optional
---

This optional lesson does not reteach `rustup`, Cargo, or modules. It shows where those same tools now run at industrial scale: sparse crates.io indexes, official language-server tooling, WebAssembly component toolchains, and edition migrations that production teams actually ship.

## 60-minute teaching plan

- 0 to 10 min: Why tooling *is* the product for Rust shops (CI minutes, lockfiles, MSRV).
- 10 to 25 min: Cargo sparse index (RFC 2789) and workspace graphs in large crates.
- 25 to 40 min: rust-analyzer and rustc as daily compilers — two programs, one language.
- 40 to 55 min: WASM / WASI 0.2 and `wasm-bindgen` as a second target, not a toy.
- 55 to 60 min: Edition 2024 as a production migration, not a blog post.

## Objectives

After this lesson you will be able to explain how the Chapter 1 toolchain loop — create a crate, pin a toolchain, add a dependency, format, lint, test — is the same loop used by Cloudflare, the Rust compiler team, and WASM runtimes, only with more lockfile discipline. You will recognize the sparse registry protocol, rust-analyzer’s role as the official LSP, and WASI 0.2 as a *target* you select with the same `rustup target add` gesture you already know. The mental-model shift is from “tools I installed for homework” to “tools that gate trillion-request services.”

## Prerequisites

You should have completed the required Chapter 1 lessons: a working `rustup`/`cargo` install, a first binary crate, primitive types, control flow, and a `lib.rs` + `main.rs` split. You do not need async, unsafe, or WASM experience. Comfort reading a `Cargo.toml` and a CI YAML file is enough.

## Introduction

From 2018 through about 2021, “Rust in production” often meant a single service or a rewritten hot path. Between 2022 and 2026 the bottleneck moved. Teams no longer ask whether rustc can emit fast code; they ask whether *the workspace* can be cloned, indexed, linted, and shipped on a laptop, in CI, and onto `wasm32-wasip2` without a custom snowflake toolchain. Cargo’s old git index for crates.io became a scaling problem. rust-analyzer became the de-facto compiler for interactive work. WebAssembly stopped being a demo target and gained a component ABI.

C and C++ shops still treat the compiler, the package manager, and the language server as three vendors. Rust’s unusual bet is that they are one product. This lesson follows that bet into systems you can click through today.

## Key Concepts

### The sparse index is a production protocol, not a convenience flag

Until 2023, Cargo learned about crates.io by cloning a giant git repository of index files. RFC 2789 (“Sparse HTTP protocol for Cargo”) replaced that with HTTPS fetches of *only the crates you depend on*. Rust 1.68 (March 2023) stabilized the protocol; Rust 1.70 (June 2023) made `sparse` the default for crates.io. The Inside Rust announcement asked the community to test `CARGO_REGISTRIES_CRATES_IO_PROTOCOL=sparse` against `https://index.crates.io/`.

That change is invisible in a ten-crate homework repo and decisive in a 400-crate workspace. It is also a reminder that **Cargo is a networked systems program**: HTTP/2, caching, and lockfile hashes are as much “Rust” as `let`.

```toml
# Historical pin — useful when teaching older CI images.
# Since 1.70 this is the crates.io default.
[registries.crates-io]
protocol = "sparse"
```

Why did the project spend an RFC on an index? Because the first program you write (`cargo new`) is also the first program a CDN cache has to serve a million times a day. Tooling that does not scale is not “beginner tooling”; it is broken infrastructure.

### rust-analyzer is a second rustc with a different latency budget

`rust-analyzer` is the official LSP implementation. It is not a syntax highlighter bolted onto rustc. It type-checks incomplete files, runs build scripts, and understands workspaces — the same module graph you learned in lesson 01-05, queried at keystroke latency. Production teams treat a broken rust-analyzer as a broken compiler: if the IDE cannot resolve `crate::auth::Token`, reviewers will not trust the change.

The teaching point is architectural. rustc optimizes for *correct artifacts*. rust-analyzer optimizes for *partial programs*. Both consume `Cargo.toml`, both respect editions, both must understand `cfg`. When students say “the compiler is slow,” ask *which compiler* and *which crate graph*.

### WASM is a target triple, not a new language

`rustup target add wasm32-unknown-unknown` and, more recently, WASI 0.2 (`wasm32-wasip2` and the Component Model) use the same `cargo build --target` you already ran for your host. The Bytecode Alliance launched WASI 0.2 (Preview 2) on 25 January 2024: WIT interfaces for clocks, random, filesystem, sockets, CLI, and HTTP, rebasing WASI off a C-like ABI onto the Component Model. Wasmtime and `jco` were the first two implementations to pass the portability suite.

```bash
rustup target add wasm32-unknown-unknown
cargo build --target wasm32-unknown-unknown --release
```

`wasm-bindgen` and `wasm-pack` sit *on top of* that target: they generate JS glue, not a second type system. The Chapter 1 idea that “a crate is a compilation unit with a target” is exactly why Rust became the default language for serious WASM runtimes (Wasmtime, WasmEdge, wasmCloud). You are not learning a new language; you are selecting a backend.

### Editions are production migrations

Edition 2024 shipped with Rust 1.85 on 20 February 2025 (RFC 3501). It is the largest edition to date: prelude additions (`Future`, `IntoFuture`), newly `unsafe` environment APIs, and drop-order tweaks. Production teams do not “upgrade Rust” by changing a tweet; they change `edition = "2024"` in `Cargo.toml`, run `cargo fix --edition`, and pin an MSRV in CI.

```toml
[package]
name = "edge-filter"
version = "0.4.2"
edition = "2024"
rust-version = "1.85"
```

The `rust-version` field is how a crate tells Cargo “do not even try to compile me on 1.76.” That is the same pinning instinct you practiced with `rustup show`, applied to a fleet.

## Code Walkthroughs

Start with the crate you already know how to create, then grow it the way a 2024 service repo grows — without inventing new language features.

### Naive: a single binary that “just builds”

```rust
// src/main.rs
fn main() {
    println!("edge-filter ready");
}
```

```toml
# Cargo.toml
[package]
name = "edge-filter"
version = "0.1.0"
edition = "2021"
```

This compiles. It teaches nothing about how Tokio, rustc, or Wasmtime actually live. The first production failure is not a type error; it is “CI used a different toolchain than my laptop.”

### Compiler-guided fix: pin the toolchain in the repo

```toml
# rust-toolchain.toml  (committed next to Cargo.toml)
[toolchain]
channel = "1.85.0"
components = ["rustfmt", "clippy", "rust-analyzer"]
targets = ["wasm32-unknown-unknown"]
```

`rustup` reads this file when you `cd` into the repo. That is the Chapter 1 mental model — rustup manages toolchains — applied to a team. If a classmate’s `cargo --version` disagrees with CI, the file is the contract.

A second common error is “it works on my machine because I have a stale git index.” On a pre-1.70 image you would see multi-minute `Updating crates.io index` lines. The sparse protocol turns that into a handful of HTTP GETs. Students should be able to *name* the protocol, not just wait for it.

### Idiomatic: workspace + library + two targets

```text
edge-filter/
  Cargo.toml
  rust-toolchain.toml
  crates/
    filter-core/src/lib.rs
    filter-cli/src/main.rs
    filter-wasm/src/lib.rs
```

```toml
# Cargo.toml (workspace root)
[workspace]
members = ["crates/*"]
resolver = "2"

[workspace.package]
edition = "2024"
rust-version = "1.85"
```

```rust
// crates/filter-core/src/lib.rs
pub fn allow(host: &str) -> bool {
    !host.ends_with(".invalid")
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn blocks_reserved_tld() {
        assert!(!allow("phishing.invalid"));
    }
}
```

The CLI crate depends on `filter-core` and stays thin — the 01-05 design. The WASM crate *also* depends on `filter-core` and is compiled with `--target wasm32-unknown-unknown`. One module graph, two artifacts. That is how wasm-bindgen projects, Cloudflare Workers (via `workers-rs`), and Wasmtime examples are structured.

If you add a dependency without thinking, Clippy and `cargo deny` become the next teachers. `cargo-deny` (Embark Studios) and `cargo-audit` (Rust Secure Code WG) are how production CI rejects yanked crates and known RustSec advisories. They are not language features; they are *the rest of the toolchain loop* you started in 01-01.

## Examples

### Example 1 — crates.io as a distributed system

The crates.io sparse index (`https://index.crates.io/`) is a production HTTP service. Cargo is its client. When you write `serde = "1"` you are performing a version-resolution query against a registry that must stay consistent with `Cargo.lock`. RFC 2789 exists because the previous git clone model did not scale to the crate graph of 2023.

Read the lockfile as a systems artifact:

```toml
[[package]]
name = "serde"
version = "1.0.217"
source = "registry+https://github.com/rust-lang/crates.io-index"
checksum = "..."
```

The `checksum` is why `cargo install` on two continents produces the same bytes. Chapter 1’s “add a dependency” step is a content-addressed fetch. Treat a hand-edited lockfile the way you would treat a hand-edited binary.

### Example 2 — rustc and rust-analyzer on the same workspace

The Rust compiler is itself a Cargo workspace of hundreds of crates (`compiler/`, `library/`, `src/tools/`). rust-analyzer must load a *subset* of that graph when you jump-to-definition inside rustc. The same is true of Tokio, Wasmtime, and Zed. If your 01-05 module map is messy — glob `pub use`, cyclic modules, a 4,000-line `lib.rs` — both compilers pay for it on every keystroke.

A useful exercise: open any mid-size crate you depend on (`grep serde Cargo.lock`) and run `cargo metadata --no-deps --format-version 1 | head`. That JSON is the module/crate map your tools share. Production build graphs are not mysterious; they are `cargo metadata` at scale.

### Example 3 — WASI 0.2 as a second “OS”

WASI 0.2 is a set of WIT worlds: `wasi:cli/command`, `wasi:http/proxy`, and friends. A Rust crate targeting WASI is still a crate. What changes is the *syscall surface*: instead of libc, you import component interfaces. Wasmtime (Bytecode Alliance) implements those imports in Rust. The host is a Rust program; the guest can be Rust, JS (`jco`), or C.

```rust
// Guest-shaped thinking: you still own a main, you just link a different std.
fn main() {
    // On wasip2 this prints through wasi:cli, not write(2) on your laptop.
    println!("filter-wasm guest started");
}
```

The reflective question: if WASM modules cannot share linear memory the way C `.so` files do, which Chapter 1 idea just saved you from a class of FFI bugs? (Answer: a crate boundary plus a typed ABI, instead of “here is a `char*`.”)

## Applications in Systems Programming

**CI as a compiler farm.** rustc, Clippy, rustfmt, and rust-analyzer components are installed by the same `rust-toolchain.toml` on GitHub Actions, Buildkite, and local machines. Teams that skip the pin spend weeks chasing “works on 1.82 / fails on 1.85” edition fallout.

**Supply-chain policy.** `cargo deny check advisories bans licenses sources` is how many 2024–2026 companies gate merges. It sits next to `cargo test`, not instead of it. The Chapter 1 quality loop grows a fourth step: format, lint, test, *policy*.

**Multi-target products.** A filtering library compiled for `x86_64-unknown-linux-gnu` and `wasm32-wasip2` is how edge workers and CLI admin tools share code. The ownership of *build configuration* becomes as important as ownership of `String`.

**Language-server as production infra.** Zed, VS Code, and Helix all speak LSP to rust-analyzer. When rust-analyzer misfires on a `build.rs` that probes `web-sys`, whole teams lose an afternoon. That is a systems outage with no HTTP status code.

## Challenges and Extensions

Cargo’s resolver, feature unification, and MSRV-aware resolution (`resolver = "3"` / MSRV-aware resolver work landing across 2024–2025) are the next layer after “it compiled.” Workspaces with optional features can pull two versions of the same crate; `cargo tree -d` is the diagnostic.

WASM still has a split personality: `wasm32-unknown-unknown` + `wasm-bindgen` for browsers versus WASI 0.2 components for servers. Picking the wrong target is the new “wrong libc.”

Edition 2024 makes some `std::env` APIs `unsafe` because process-global mutation was never actually safe. Migration is a *social* problem: every crate in the graph must agree.

Reflect: if you were designing a shared cache of compiled artifacts (`sccache`, `cargo-cache`) for a 200-developer org, which Chapter 1 objects would you key on — toolchain hash, target triple, `Cargo.lock` digest, or all three?

## Exercises

1. **Conceptual.** Explain in four sentences why RFC 2789 exists. What resource was the git index wasting, and who pays for that resource in CI?
2. **Code fix.** Take your Chapter 1 `hello-cli` crate, add a `rust-toolchain.toml` pinning a concrete stable version, and a `rust-version` field. Confirm `rustup show` agrees with `cargo --version`.
3. **Metadata.** Run `cargo metadata --format-version 1` and identify your crate’s id, edition, and target directory. Sketch the crate graph on paper.
4. **Second target.** Add `wasm32-unknown-unknown`, build the library crate for that target, and record the artifact path under `target/wasm32-unknown-unknown/`. What is *not* in that artifact compared to a native binary?
5. **Implementation.** Split `hello-cli` into a workspace (`core` + `cli`) matching 01-05, then add an empty `wasm` crate that depends on `core`. Do not rewrite any theory — only the package graph.

## References

- [Help test Cargo's new index protocol](https://blog.rust-lang.org/inside-rust/2023/01/30/cargo-sparse-protocol/) — Inside Rust, 30 January 2023; RFC 2789.
- [Announcing Rust 1.68.0](https://blog.rust-lang.org/2023/03/09/Rust-1.68.0/) — sparse protocol stabilized.
- [Announcing Rust 1.85.0 and Rust 2024](https://blog.rust-lang.org/2025/02/20/Rust-1.85.0/) — Edition 2024 (RFC 3501).
- [WASI 0.2 Launched](https://bytecodealliance.org/articles/WASI-0.2) — Bytecode Alliance, 25 January 2024.
- [The Cargo Book — Registry Index](https://doc.rust-lang.org/cargo/reference/registry-index.html) — git vs sparse protocols.
- [rust-analyzer manual](https://rust-analyzer.github.io/) — official LSP.

## Recap

- The Chapter 1 loop is the production loop: pin a toolchain, resolve a graph, compile a target, test.
- Sparse crates.io (2023) is Cargo acting as a networked systems client.
- rust-analyzer is a second compiler with a latency budget; treat it as infrastructure.
- WASM/WASI 0.2 is a target triple and an ABI, not a new language.
- Edition 2024 is a coordinated migration, recorded in `Cargo.toml` and CI, not in slides.
