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

Not all failures are equal. Some are recoverable (invalid user input), and others indicate a bug (an invariant was violated). This lesson teaches where panics belong, how to keep invariants local, and how to design APIs that fail predictably.

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

## Exercises

1. Add `sub` and `div`, with `div` rejecting division by zero via `Result`.
2. Add a `Help` error message with a usage string.
3. Add a test that ensures unknown commands produce `UnknownCommand`.

## Recap

- `Result` is for recoverable failures.
- Panics are for bugs and broken invariants.
- `debug_assert!` documents internal assumptions.
- Keep structured errors separate from user-facing formatting.
