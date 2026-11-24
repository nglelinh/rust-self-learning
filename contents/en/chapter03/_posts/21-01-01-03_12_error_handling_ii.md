---
layout: post
title: "03-12 Error Handling II (Panics and Invariants)"
chapter: "03"
order: 12
owner: "OpenCode"
lang: en
categories:
  - chapter03
lesson_type: required
---

After learning `Result<T, E>`, many Rust learners swing too far in one direction—either sprinkling `.unwrap()` everywhere (turning user typos into crashes) or wrapping even impossible internal bugs in awkward `Result` chains. The secret to idiomatic Rust error handling is knowing the crisp boundary between two worlds: **expected external failures** (bad user input, missing files, dropped sockets $$	o$$ `Result`) versus **broken programmer invariants** (bugs that should never happen if your code is correct $$	o$$ `panic!` / `assert!`).

This lecture sharpens that exact engineering judgment: when `unwrap()` is a liability, how `expect("why this holds")` documents invariants, and how `assert!` differs from `debug_assert!` in `--release` builds.

---

## Background Knowledge Map: Recoverable Errors vs. Unrecoverable Panics

Four systems principles define the boundary between `Result` and `panic!` in Rust:

1. **External Boundary vs. Internal Invariant:** Never trust input crossing a process boundary (CLI args, network bytes, files)—always return `Result`. Inside trusted code, if a mathematical invariant is violated (e.g., an internal queue length disagrees with its backing buffer), failing fast via `panic!` prevents silent data corruption.
2. **What `panic!` Actually Does (Unwind vs. Abort):** By default, a panic **unwinds** the stack cleanly, running every `Drop` destructor on the way up so files close and locks release (unlike a C segfault!). In embedded or size-constrained binaries (`panic = "abort"`), it terminates immediately.
3. **`expect()` as an Executable Proof Comment:** Never write `.expect("failed")`—write `.expect("regex literal is verified at compile time")`, stating *why* the `None`/`Err` case is believed impossible.
4. **Release-Build Semantics of `assert!` vs. `debug_assert!`:** `assert!` stays active in both `debug` and `--release` builds (protecting safety-critical invariants); `debug_assert!` is **completely stripped out** in `--release` builds (ideal for expensive O(n) sanity checks during testing).

---

## 60-minute teaching plan

- 0 to 10 min: Failure taxonomy: user error vs programmer error.
- 10 to 25 min: `panic!`, `unwrap`, `expect`: when are they acceptable?
- 25 to 40 min: Invariants and assertions (`debug_assert!`).
- 40 to 60 min: Mini-lab: write a small parser that uses `Result` externally and internal invariants.

## Learning goals

By the end of this lesson, students can:

- Decide between `Result` and `panic!` based on who can fix the issue.
- Use `expect` with meaningful messages when a panic is justified.
- Write internal invariants with `debug_assert!`.
- Structure parsing code so invalid input is handled as `Result`.

## The rule of thumb

- Use `Result` for anything that can fail due to the environment or user input.
- Use `panic!` for impossible states and violated invariants (bugs).

Ask: if this fails in production, can the user do something different? If yes, return `Result`.

## Panics and unwraps

### `unwrap()`

`unwrap()` is a shortcut that panics on `Err`/`None`.

```rust
let n: i32 = "42".parse().unwrap();
```

Avoid in user-facing paths.

### `expect()`

If you do panic, provide context:

```rust
let n: i32 = "42".parse().expect("hardcoded number must parse");
```

Use `expect` when:

- you have a hardcoded constant,
- you are in a test,
- you are in a quick prototype and explicitly accept panic behavior.

## Assertions and invariants

### `assert!` vs `debug_assert!`

- `assert!` runs in debug and release.
- `debug_assert!` runs only in debug.

Use `debug_assert!` for internal invariants that should never fail if your code is correct.

Example:

```rust
fn midpoint(a: i32, b: i32) -> i32 {
    debug_assert!(a <= b);
    a + (b - a) / 2
}
```

Teaching point: invariants are about your program’s internal logic, not about user input.

## Mini-lab: parser with external `Result` and internal invariants

We will parse a tiny command language:

- `add <i32> <i32>`
- `mul <i32> <i32>`

### Step 1: model the command

```rust
#[derive(Debug, Clone, PartialEq, Eq)]
enum Command {
    Add(i32, i32),
    Mul(i32, i32),
}
```

### Step 2: define errors

```rust
#[derive(Debug, Clone, PartialEq, Eq)]
enum ParseError {
    Empty,
    UnknownCommand,
    WrongArity { expected: usize, got: usize },
    BadInt { which: &'static str },
}
```

### Step 3: implement parsing

```rust
fn parse_command(line: &str) -> Result<Command, ParseError> {
    let parts: Vec<&str> = line.split_whitespace().collect();
    if parts.is_empty() {
        return Err(ParseError::Empty);
    }

    // Invariant: if we reach argument parsing, we validated the arity.
    match parts[0] {
        "add" => {
            if parts.len() != 3 {
                return Err(ParseError::WrongArity { expected: 3, got: parts.len() });
            }
            debug_assert!(parts.len() == 3);
            let a = parts[1].parse::<i32>().map_err(|_| ParseError::BadInt { which: "a" })?;
            let b = parts[2].parse::<i32>().map_err(|_| ParseError::BadInt { which: "b" })?;
            Ok(Command::Add(a, b))
        }
        "mul" => {
            if parts.len() != 3 {
                return Err(ParseError::WrongArity { expected: 3, got: parts.len() });
            }
            debug_assert!(parts.len() == 3);
            let a = parts[1].parse::<i32>().map_err(|_| ParseError::BadInt { which: "a" })?;
            let b = parts[2].parse::<i32>().map_err(|_| ParseError::BadInt { which: "b" })?;
            Ok(Command::Mul(a, b))
        }
        _ => Err(ParseError::UnknownCommand),
    }
}
```

Teaching points:

- All invalid input returns `Err(ParseError::...)`.
- `debug_assert!` documents an internal assumption after the check.
- The happy path stays readable.

### Step 4: user-facing error messages

Separate structured errors from display:

```rust
fn render_error(e: ParseError) -> String {
    match e {
        ParseError::Empty => "please type a command".to_string(),
        ParseError::UnknownCommand => "unknown command".to_string(),
        ParseError::WrongArity { expected, got } => format!("expected {expected} parts, got {got}"),
        ParseError::BadInt { which } => format!("invalid integer for {which}"),
    }
}
```

Teaching point: keep parsing code free from UI formatting when possible.

## Tests

```rust
#[test]
fn parse_add_ok() {
    assert_eq!(parse_command("add 1 2").unwrap(), Command::Add(1, 2));
}

#[test]
fn parse_wrong_arity() {
    assert_eq!(
        parse_command("add 1"),
        Err(ParseError::WrongArity { expected: 3, got: 2 })
    );
}
```

## Common mistakes

1. Panicking on user input.
2. Using `assert!` for input validation (turns user mistakes into crashes).
3. Returning `bool` instead of a structured error.

---

### Worked Line-by-Line Trace (Debug vs. Release & Boundary Discipline): External `Result` + Internal `assert!`

Trace how a parser separates untrusted input (`Result`) from internal invariants (`assert!` / `debug_assert!`):

```rust
fn split_kv(line: &str) -> Result<(&str, &str), &'static str> {
    let (k, v) = line.split_once('=').ok_or("missing '='")?; // External input -> Result!
    let k = k.trim();
    if k.is_empty() {
        return Err("empty key");                             // External input -> Result!
    }
    debug_assert!(line.contains('=')); // Checked in `cargo test`, ZERO cost in `--release`!
    assert!(!k.is_empty());            // Always active: guards downstream invariant!
    Ok((k, v.trim()))
}
```

| Input `line` | Build Profile | External Check (`Result`) | `debug_assert!` | `assert!` | Outcome |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `"host=localhost"` | `debug` | Passes | Runs & passes | Runs & passes | `Ok(("host", "localhost"))` |
| `"host=localhost"` | `--release` | Passes | **Compiled away (0 ns)** | Runs & passes | `Ok(("host", "localhost"))` |
| `"bad_line"` | Both | Returns `Err("missing '='")` | Never reached | Never reached | Clean `Err` (no crash!) |

### Deep-Dive Explanation: Four Rules for Choosing Between `Result`, `expect`, and `assert!` (Read Slowly)

1. **Never `.unwrap()` untrusted input in library or server code:** A single malformed HTTP header calling `.unwrap()` can crash an entire worker thread (Denial-of-Service vulnerability).
2. **Use `.expect("reason")` only when *you* can prove the invariant:** For example, parsing a hard-coded constant IP `"127.0.0.1".parse::<IpAddr>().expect("valid static IP")` is completely idiomatic because no runtime input can break it.
3. **Never put side effects inside `debug_assert!`:** Writing `debug_assert!(list.pop().is_some())` creates a nightmare Heisenbug—the `.pop()` runs in `debug` tests and vanishes in `--release`!
4. **Test panic invariants explicitly with `#[should_panic]`:** When an internal function documents a panic contract (like out-of-bounds indexing), verify it in unit tests with `#[should_panic(expected = "...")]`.

**Quick check.** Suppose a function runs an expensive O(n) check verifying that a slice is sorted before binary search. Should you write that check with `assert!` or `debug_assert!`—and what happens to it when you build with `cargo build --release`?

---

## Exercises

1. Add `sub` and `div`, with `div` rejecting division by zero via `Result`.
2. Add a `Help` error message with a usage string.
3. Add a test that ensures unknown commands produce `UnknownCommand`.

## Recap

- `Result` is for recoverable failures.
- Panics are for bugs and broken invariants.
- `debug_assert!` documents internal assumptions.
- Keep structured errors separate from user-facing formatting.

## Video tham khảo

- [Rust: Don't Panic](https://www.youtube.com/watch?v=usRlRCNIOG0) — No Boilerplate, English. **xem trước.** Watch first: panic is for bugs, not for ordinary failure — the rule of thumb in this lesson.
- [48: When to panic! vs. when to use Result in Rust](https://www.youtube.com/watch?v=KR6Vb9b4HGw) — Rustfully, English. A direct panic-versus-Result split you can apply to the parser lab’s invariants.
