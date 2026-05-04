---
layout: post
title: "01-01 Setup and Tooling"
chapter: "01"
order: 1
owner: "OpenCode"
lang: en
categories:
  - chapter01
lesson_type: required
---

Rust development is mostly about mastering the feedback loop: write code, read compiler messages, iterate quickly. This lesson sets up that loop and explains the handful of tools you will use every day.

## 60-minute teaching plan

- 0 to 10 min: What Rust tooling is (and is not). Install and verify.
- 10 to 25 min: Cargo fundamentals: create, run, build, test.
- 25 to 40 min: Project layout + reading `Cargo.toml` + understanding targets.
- 40 to 50 min: Code quality loop: `fmt` then `clippy`.
- 50 to 60 min: Docs, help commands, and a short checklist for debugging build issues.

## Learning goals

By the end of this lesson, students can:

- Verify a working Rust toolchain and explain what `rustup`, `rustc`, and `cargo` do.
- Create a crate, add a dependency, and run it locally.
- Use the daily workflow loop: `cargo fmt` then `cargo clippy` then `cargo test`.
- Find documentation quickly (stdlib docs and dependency docs).

## The toolchain model (mental map)

Rust tooling is split into three layers:

- `rustup`: installs and manages toolchains (versions of Rust and components).
- `rustc`: the compiler.
- `cargo`: build system + dependency manager + test runner.

In practice, you mostly run `cargo ...` and let Cargo call the compiler for you.

## Setup and verification

### Verify you can compile

```bash
rustc --version
cargo --version
```

Expected: both print versions. If `rustc` exists but `cargo` does not, your install is incomplete.

### Toolchains (why you care)

Rust versions move quickly. Toolchains matter because:

- One repo may require a newer compiler.
- CI might pin a version.
- Nightly features require nightly.

Common `rustup` commands:

```bash
rustup show
rustup update
rustup toolchain list
```

If you are teaching, the simplest rule is: keep everyone on stable unless you have a reason to deviate.

## Cargo fundamentals

### Create a new binary crate

```bash
cargo new hello-cli
cd hello-cli
```

Run it:

```bash
cargo run
```

Build without running (faster when you only want to compile):

```bash
cargo build
```

Run tests:

```bash
cargo test
```

### Understand the generated structure

Cargo creates:

- `Cargo.toml`: package metadata and dependencies.
- `src/main.rs`: entry point for a binary crate.
- `target/`: build artifacts (usually not committed).

Open `Cargo.toml` and point out the key fields:

```toml
[package]
name = "hello-cli"
version = "0.1.0"
edition = "2021"

[dependencies]
```

Notes:

- `edition` changes language defaults and idioms. Most new projects use 2021.
- Dependencies are resolved from `crates.io` by default.

## Adding a dependency (live demo)

Pick a small dependency to demonstrate. Example: `anyhow` for quick error handling.

Edit `Cargo.toml`:

```toml
[dependencies]
anyhow = "1"
```

Then in `src/main.rs`:

```rust
use anyhow::Result;

fn main() -> Result<()> {
    println!("hello-cli ready");
    Ok(())
}
```

Build and run:

```bash
cargo run
```

Teaching point: Cargo will download, compile, and cache dependencies. The second build is much faster.

## The daily quality loop

### Format: `cargo fmt`

Rust formatting is standardized. In most repos, formatting is not a debate.

```bash
cargo fmt
```

If `cargo fmt` is missing, install the component:

```bash
rustup component add rustfmt
```

### Lint: `cargo clippy`

Clippy is the Rust linter that catches common mistakes and suggests more idiomatic code.

```bash
cargo clippy
```

If Clippy is missing:

```bash
rustup component add clippy
```

Instructor note: teach students to read clippy messages, not to disable them by default.

### Recommended order

For a tight feedback loop:

```bash
cargo fmt
cargo clippy
cargo test
```

This matches how many CI pipelines think: style, static checks, then correctness.

## Docs and help (how to unblock yourself)

### Built-in docs

```bash
cargo doc --open
```

This generates local documentation for your crate and dependencies.

### Fast help patterns

```bash
cargo --help
cargo test --help
rustc --explain E0382
```

The last command (`rustc --explain ...`) is one of the best ways to learn Rust. When you see an error code, explain it.

## Common issues and how to debug them

### 1. It builds on my machine, not on CI

Typical causes:

- Using a different toolchain version.
- Relying on platform-specific behavior.
- Missing a dependency feature.

Action: compare `rustc --version` locally vs CI. Then run `cargo clean` only if you strongly suspect stale artifacts.

### 2. Dependency confusion

If adding a dependency fails:

- Check you edited the correct `[dependencies]` section.
- Run `cargo build -vv` to see more details.

### 3. Slow builds

Teach the basics:

- First build is slow, next builds are incremental.
- `cargo check` is faster than `cargo build` when you only want type checking.

```bash
cargo check
```

## In-class exercises (10 to 15 minutes)

1. Create a crate named `hello-cli`.
2. Add one dependency.
3. Make `main` return a `Result`.
4. Run `cargo fmt`, `cargo clippy`, and `cargo test`.
5. Add one unit test (even a trivial one) and re-run `cargo test`.

## Homework

- Create a second crate `sandbox` and try:
  - `cargo doc --open`
  - `cargo check`
  - `rustc --explain` for one compiler error you encounter

## Recap

- `rustup` manages toolchains.
- `cargo` is your daily entry point.
- Use the loop: format, lint, test.
- Learn to unblock yourself via `--help` and `rustc --explain`.
