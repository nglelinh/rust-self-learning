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

Once a Rust program grows past a couple of hundred lines, stuffing everything into `src/main.rs` quickly becomes painful to read and impossible to integration-test. Yet newcomers arriving from Python or JavaScript often hit a classic surprise: creating `src/math.rs` does *not* automatically make it part of your program! In Rust, the module tree is **explicitly declared**—a file only exists to the compiler when its parent module mounts it with `mod`.

This lecture demystifies Rust’s code organization: the exact boundary between **Package**, **Crate** (`lib.rs` vs. `main.rs`), and **Module**, the **private-by-default** encapsulation model, and the industry-standard *thin `main.rs` + testable `lib.rs`* architecture.

---

## Key ideas: Package, Crate, Module, and Visibility in Rust

Four architectural rules keep even 100,000-line Rust codebases crystal clear:

1. **The Compilation Unit Is the Crate, Not the File:** A single `package` (one `Cargo.toml`) may contain at most one library crate (`src/lib.rs`) and any number of binary crates (`src/main.rs`, `src/bin/*.rs`). `rustc` compiles an entire crate at once, so sibling modules can reference each other without C/C++ header files.
2. **Declaration (`mod`) vs. Path Aliasing (`use`):** `mod math;` tells the compiler to *include* `src/math.rs` in the module tree (declare it **once** in its parent). `use crate::math::add;` merely brings an already-mounted item into local scope.
3. **Private by Default, Child Sees Parent:** Items and struct fields are private to outer scopes unless marked `pub`, yet **child modules can inspect private items of their ancestors**—which is why `#[cfg(test)] mod tests` can unit-test private helpers right inside the same file!
4. **Thin Binary + Rich Library Layout:** Put core domain logic in `src/lib.rs` (accessible to `tests/` integration tests and benchmarks) and keep `src/main.rs` as a thin CLI wrapper calling into the library.

---

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

---

### Worked Line-by-Line Trace (Module Tree & Visibility): Resolving `lib.rs` vs. `main.rs`

Consider a package `calc_app` containing both `src/lib.rs` (library crate) and `src/main.rs` (binary crate):

```text
calc_app/
├── Cargo.toml       (name = "calc_app")
└── src/
    ├── lib.rs       --> mod math; pub use math::add;
    ├── math.rs      --> pub fn add(a: i32, b: i32) -> i32 { helper(a, b) }
    │                    fn helper(a: i32, b: i32) -> i32 { a + b }
    └── main.rs      --> fn main() { println!("{}", calc_app::add(2, 3)); }
```

Trace which calls compile from each scope:

| Call Site | Calling `add(2, 3)` | Calling `helper(2, 3)` | Compiler Verdict |
| :--- | :--- | :--- | :--- |
| Inside `src/math.rs` | Compiles | Compiles | Same module |
| Inside `math::tests` (child module) | Compiles | **Compiles** | Child modules can access parent’s private items |
| Inside `src/lib.rs` | Compiles (`math::add`) | **Error E0603** | `helper` is private inside `math` |
| Inside `src/main.rs` | Compiles (`calc_app::add`) | **Error E0603** | `main.rs` is a **separate crate**—only sees `lib.rs`’s `pub` API! |

### Explanation: Four Module Habits That Prevent Link & Visibility Headaches

1. **Never declare `mod math;` in *both* `src/lib.rs` and `src/main.rs`:** Doing so compiles `math.rs` twice into two unrelated crates—producing baffling errors where `calc_app::math::Config` and `crate::math::Config` are incompatible types! Mount it once in `lib.rs` and import it via `calc_app::...` in `main.rs`.
2. **Use `pub use` to design a clean public facade:** Re-export key types at the crate root so users write `calc_app::add` instead of memorising deep internal paths.
3. **Use `pub(crate)` for internal shared utilities:** Share helpers freely across your own modules without committing to them as a public API contract.
4. **Prefer `crate::...` absolute paths inside a library:** Absolute crate paths stay valid even when you reorganize nested files.

**Quick check.** In `calc_app` above, if you delete `pub use math::add;` from `src/lib.rs` while leaving `mod math;` private, can `src/main.rs` still call `calc_app::math::add(2, 3)`—and why?

---

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

## Video tham khảo

- [Rust Modules - Explained Like I'm 5](https://www.youtube.com/watch?v=969j0qnJGi8) — Let's Get Rusty, English. **xem trước.** Watch first for a short picture of mod, crates, and why files are not automatically modules.
- [Rust's Module System Explained!](https://www.youtube.com/watch?v=5RPXgDQrjio) — Let's Get Rusty, English. The fuller module, pub, and use story for the lib-plus-bin refactor later in the lesson.
