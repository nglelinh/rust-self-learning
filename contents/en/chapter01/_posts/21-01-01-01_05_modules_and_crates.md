---
layout: post
title: "01-05 Functions, Modules, and Crates"
chapter: "01"
order: 5
owner: "OpenCode"
lang: en
categories:
  - chapter01
lesson_type: required
---

Rust codebases scale through clear module boundaries and explicit visibility. This lesson teaches how Rust projects are structured, how names resolve, and how to design a small crate with a clean API.

## 60-minute teaching plan

- 0 to 10 min: Crates vs modules vs packages (the vocabulary).
- 10 to 25 min: File layout: `main.rs`, `lib.rs`, `mod.rs` (modern layout) and `mod` declarations.
- 25 to 40 min: Visibility: `pub`, `pub(crate)`, and API design.
- 40 to 55 min: Live refactor: single-file program into `lib + bin`.
- 55 to 60 min: Checklist for keeping modules clean and testable.

## Learning goals

By the end of this lesson, students can:

- Explain: package (Cargo project) vs crate (compilation unit) vs module (namespace).
- Split code into modules without breaking name resolution.
- Use `pub` intentionally to shape a public API.
- Structure a CLI so `main.rs` is thin and logic lives in `lib.rs`.

## Vocabulary (what Rust means)

### Package

A package is a directory with a `Cargo.toml`.

### Crate

A crate is what Rust compiles.

- A binary crate has an entry point (`fn main`) and produces an executable.
- A library crate produces a reusable library.

One package can produce multiple crates (multiple binaries plus one library, for example).

### Module

A module is a namespace inside a crate.

## The default layouts

### Binary-only

```
src/main.rs
```

### Library-only

```
src/lib.rs
```

### Library + binary (recommended for most real projects)

```
src/lib.rs
src/main.rs
```

Teaching point: this layout is ideal because:

- your real logic is testable in the library
- the binary is a thin wrapper (CLI parsing, wiring, printing)

## Modules and file layout

Modern Rust module layout typically looks like:

```
src/
  lib.rs
  main.rs
  parser.rs
  math/
    mod.rs
    stats.rs
```

To declare modules, you use `mod`.

Example in `lib.rs`:

```rust
pub mod parser;
pub mod math;
```

Then in `src/math/mod.rs`:

```rust
pub mod stats;
```

And in `src/math/stats.rs`, you define functions/types.

Instructor note: Rust also supports the newer "module folder" layout without `mod.rs` in some cases, but the above is easy to teach and common in older code.

## Paths and `use`

Paths in Rust are explicit.

```rust
use crate::math::stats::mean;
```

Teaching points:

- `crate::` means from the crate root.
- Use `super::` to reference parent modules.
- Avoid deep `use` chains in many files; re-export from a central place when it helps ergonomics.

## Visibility (`pub`)

Rust defaults to private. This is a big deal for API design.

### Private by default

```rust
fn helper() {}
```

### Public to other modules (and to library users)

```rust
pub fn parse_command(...) { ... }
```

### Public only inside the crate

```rust
pub(crate) fn internal_only() { ... }
```

Rule of thumb: make as little public as possible. Public APIs are harder to change.

## Live refactor: single-file tool to `lib + bin`

We will take a small program (for example, the REPL from the previous lesson) and split it.

### Step 1: create `lib.rs`

Move reusable logic into the library crate:

- `enum Command`
- `parse_command(&str) -> Result<Command, ...>`
- `execute(Command) -> ...` (or better, return structured output)

Example `src/lib.rs`:

```rust
pub mod command;

pub use command::{parse_command, Command};
```

Example `src/command.rs`:

```rust
#[derive(Debug)]
pub enum Command {
    Add(i32, i32),
    Mul(i32, i32),
    Help,
    Quit,
}

pub fn parse_command(line: &str) -> Result<Command, String> {
    // parsing logic
    # let _ = line;
    # Ok(Command::Help)
}
```

Teaching point: keep `Command` public, but keep helper functions private unless needed.

### Step 2: simplify `main.rs`

`src/main.rs` becomes wiring code:

```rust
use std::io;

use hello_cli::{parse_command, Command};

fn main() {
    println!("type 'help' for commands");
    loop {
        let mut line = String::new();
        if io::stdin().read_line(&mut line).is_err() {
            eprintln!("stdin error");
            break;
        }

        let cmd = match parse_command(&line) {
            Ok(c) => c,
            Err(e) => {
                eprintln!("error: {e}");
                continue;
            }
        };

        match cmd {
            Command::Quit => break,
            _ => {
                // call into library logic
            }
        }
    }
}
```

Note: the crate name in `use hello_cli::...` must match the package name in `Cargo.toml`.

### Step 3: add tests in the library

Because logic is in `lib.rs`, tests are simple:

```rust
#[test]
fn parse_help() {
    let c = parse_command("help").unwrap();
    matches!(c, Command::Help);
}
```

Teaching point: unit tests for parsing do not need stdin.

## Designing a clean API surface

Teach a simple rule: `main.rs` should mostly contain:

- CLI argument parsing
- calling library functions
- printing output / exit codes

Everything else should live in modules with tests.

## Common mistakes

1. Making everything `pub`: makes later refactors painful.
2. Circular module dependencies: keep layers (e.g., `parser` depends on `model`, not the other way around).
3. Putting IO everywhere: it makes tests hard.

## Exercises

1. Split the REPL into `command` (parsing/model) and `engine` (execution).
2. Re-export `Command` and `parse_command` from `lib.rs` for nicer imports.
3. Add at least 5 parsing unit tests.
4. Replace `String` errors with a tiny custom `enum ParseError`.

## Recap

- Package contains crates; crates contain modules.
- Use `pub` deliberately to shape your API.
- Put real logic in the library; keep `main` thin.
- This structure is the foundation for maintainable Rust projects.
