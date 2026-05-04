---
layout: post
title: "01-02 First Rust Program"
chapter: "01"
order: 2
owner: "OpenCode"
lang: en
categories:
  - chapter01
lesson_type: required
---

This lesson focuses on Rust syntax you will touch constantly: bindings, mutability, basic IO, and simple control flow. The goal is not to memorize syntax, but to build a reliable habit: write a small program, make it correct, make errors explicit.

## 60-minute teaching plan

- 0 to 10 min: The shape of a Rust program and `fn main()`.
- 10 to 25 min: Variables: `let`, `mut`, and shadowing.
- 25 to 40 min: IO: read a line, trim, parse, handle failure.
- 40 to 50 min: Formatting output: `println!` and debug formatting.
- 50 to 60 min: Exercise: build a robust mini prompt that never panics on bad input.

## Learning goals

By the end of this lesson, students can:

- Explain the difference between immutability and mutability in Rust.
- Use shadowing for type-changing transforms (e.g., `String` to number).
- Read from stdin, parse safely, and report errors without panicking.
- Write small helper functions with clear signatures.

## The minimal Rust program

Start from this and annotate it:

```rust
fn main() {
    println!("Hello, world!");
}
```

Teaching points:

- `fn` defines a function.
- `main` is the entry point for binaries.
- `println!` is a macro (we will explain macros later; for now treat it as "special function-like syntax").

## Bindings, mutability, and shadowing

### `let` creates a binding

```rust
let x = 10;
```

`x` is immutable by default. This is a feature, not a restriction: it prevents accidental changes.

### `mut` opts into mutation

```rust
let mut count = 0;
count += 1;
```

Emphasize: in Rust, mutability is a property of the binding, not of the value.

### Shadowing (same name, new binding)

Shadowing is often the cleanest way to transform data step-by-step.

```rust
let input = "  42 ";
let input = input.trim();
let input: i32 = input.parse().unwrap();
```

Teaching points:

- Each `let input = ...` creates a new binding.
- The type can change across shadows.
- We used `unwrap()` only to illustrate; in real code prefer `Result`.

## Printing and formatting

### Basic formatting

```rust
let name = "Rust";
println!("Hello, {name}!");
println!("{} + {} = {}", 2, 3, 2 + 3);
```

### Debug formatting

```rust
let v = vec![1, 2, 3];
println!("v = {:?}", v);
```

If students ask "why not always debug print?": debug is for developers; display formatting is for user-facing messages.

## Reading from stdin (live coding)

This is the simplest robust pattern:

```rust
use std::io;

fn read_line() -> io::Result<String> {
    let mut s = String::new();
    io::stdin().read_line(&mut s)?;
    Ok(s)
}
```

Teaching points:

- `String::new()` allocates an empty growable string.
- `read_line` appends into the string.
- `?` returns early if there is an IO error.

## Parsing safely

### The naive version (show, then improve)

```rust
let n: i32 = input.trim().parse().unwrap();
```

This panics on invalid input. That is unacceptable for a CLI tool.

### A better version with `Result`

```rust
fn parse_i32(s: &str) -> Result<i32, String> {
    s.trim()
        .parse::<i32>()
        .map_err(|e| format!("not a valid integer: {e}"))
}
```

Teaching points:

- `parse::<i32>()` returns `Result<i32, ParseIntError>`.
- `map_err` transforms error types.
- Returning `String` is not ideal long-term, but fine for Lesson 1; later we will define real error types.

## Returning `Result` from `main`

This lets you use `?` in `main`.

```rust
fn main() -> Result<(), String> {
    let line = read_line().map_err(|e| e.to_string())?;
    let n = parse_i32(&line)?;
    println!("n = {n}");
    Ok(())
}
```

Instructor note: if you used `anyhow` in the previous lesson, this becomes even cleaner. Either approach is fine; keep it consistent across the course.

## Worked example: a mini prompt that never panics

Goal: ask the user for an integer, and keep asking until they enter a valid one.

```rust
use std::io;

fn read_line() -> io::Result<String> {
    let mut s = String::new();
    io::stdin().read_line(&mut s)?;
    Ok(s)
}

fn prompt_i32(prompt: &str) -> io::Result<i32> {
    loop {
        println!("{prompt}");
        let line = read_line()?;
        match line.trim().parse::<i32>() {
            Ok(n) => return Ok(n),
            Err(_) => {
                println!("Please enter a valid 32-bit integer.");
            }
        }
    }
}

fn main() -> io::Result<()> {
    let n = prompt_i32("Enter an integer:");
    let n = n?;
    println!("You entered: {n}");
    Ok(())
}
```

Discussion questions:

- Where can the program still fail? (IO errors)
- Why is parse failure not treated as an error return? (because we choose to recover by reprompting)

## Common mistakes (and what the compiler teaches)

### 1. Forgetting `mut`

```rust
let s = String::new();
io::stdin().read_line(&mut s)?;
```

This fails because `read_line` needs a mutable reference. Fix: `let mut s = ...`.

### 2. Using `trim()` too late

Students often parse the raw line including `\n`. Teach: `trim()` before parsing.

### 3. Overusing `unwrap()`

Rule for this course: in CLI programs, `unwrap()` is a last resort. Prefer `match` or `?`.

## Exercises

1. Modify `prompt_i32` to accept an inclusive range (e.g., 1 to 10). Reprompt on out-of-range input.
2. Implement `prompt_f64` that accepts floats and rejects `NaN`.
3. Add a `prompt_yes_no` that accepts `y/n` and returns `bool`.

## Recap

- Immutable by default; opt into mutation with `mut`.
- Shadowing is a clean way to transform values.
- Prefer explicit, non-panicking error handling.
- Use `Result` and `?` early; it scales to larger programs.
