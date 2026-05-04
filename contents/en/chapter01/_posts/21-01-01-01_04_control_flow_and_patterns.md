---
layout: post
title: "01-04 Control Flow and Pattern Basics"
chapter: "01"
order: 4
owner: "OpenCode"
lang: en
categories:
  - chapter01
lesson_type: required
---

Rust encourages you to express decisions with `match` and patterns so that impossible states are unrepresentable. This lesson builds the habit of using expressions (`if`, `match`) and enums to make logic explicit.

## 60-minute teaching plan

- 0 to 10 min: `if` as an expression and basic branching.
- 10 to 30 min: `match`: exhaustiveness, patterns, and guards.
- 30 to 45 min: Loops: `for`, `while`, `loop`, and `break` values.
- 45 to 60 min: Mini-lab: a typed REPL with a command enum and `match` dispatch.

## Learning goals

By the end of this lesson, students can:

- Use `if` and `match` as expressions (returning values).
- Write exhaustive `match` statements and interpret compiler errors when cases are missing.
- Use common pattern forms: literals, ranges, destructuring, `_` catch-all.
- Build a simple command interpreter that parses input into an enum.

## `if` as an expression

Rust `if` returns a value:

```rust
let n = 10;
let kind = if n % 2 == 0 { "even" } else { "odd" };
println!("{n} is {kind}");
```

Teaching point: both branches must return the same type.

## `match`: the workhorse

### Basic `match`

```rust
let x = 3;

let label = match x {
    0 => "zero",
    1 => "one",
    2 => "two",
    _ => "many",
};
```

Exhaustiveness is a feature: Rust forces you to think about every case.

### Patterns you will use constantly

#### Ranges

```rust
let grade = 87;
let letter = match grade {
    90..=100 => 'A',
    80..=89 => 'B',
    70..=79 => 'C',
    60..=69 => 'D',
    _ => 'F',
};
```

#### Guards

```rust
let n = -3;
let desc = match n {
    x if x < 0 => "negative",
    0 => "zero",
    _ => "positive",
};
```

#### Destructuring tuples

```rust
let p = (10, 20);
let s = match p {
    (0, 0) => "origin",
    (0, y) => "on y-axis",
    (x, 0) => "on x-axis",
    (_, _) => "somewhere else",
};
```

### Matching `Option` and `Result`

This is where `match` becomes a safety tool:

```rust
let s = "42";
let n = match s.parse::<i32>() {
    Ok(n) => n,
    Err(e) => {
        eprintln!("parse failed: {e}");
        return;
    }
};
```

Later we will prefer `?`, but `match` is the clearest way to teach control flow.

## Loops

### `for` over a range

```rust
for i in 0..3 {
    println!("i = {i}");
}
```

### `while` when you have a condition

```rust
let mut n = 3;
while n > 0 {
    println!("{n}");
    n -= 1;
}
```

### `loop` when you break explicitly

Rust `break` can return a value:

```rust
let mut n = 0;
let found = loop {
    n += 1;
    if n == 5 {
        break n;
    }
};
assert_eq!(found, 5);
```

Teaching point: `loop` is ideal for REPLs and retry loops.

### `if let` / `while let` (common pattern)

Use these when you only care about one pattern:

```rust
let maybe = Some(3);
if let Some(x) = maybe {
    println!("x = {x}");
}
```

```rust
let mut it = vec![1, 2, 3].into_iter();
while let Some(x) = it.next() {
    println!("{x}");
}
```

## Mini-lab: a typed REPL

Goal: read a line, parse it into a `Command` enum, then execute with `match`.

### Step 1: define the command model

```rust
#[derive(Debug)]
enum Command {
    Add(i32, i32),
    Mul(i32, i32),
    Help,
    Quit,
}
```

### Step 2: parse input into a command

Start with a strict format: `add 1 2` or `mul 3 4`.

```rust
fn parse_command(line: &str) -> Result<Command, String> {
    let parts: Vec<&str> = line.split_whitespace().collect();
    if parts.is_empty() {
        return Err("empty command".to_string());
    }

    match parts[0] {
        "add" if parts.len() == 3 => {
            let a = parts[1].parse::<i32>().map_err(|_| "bad a")?;
            let b = parts[2].parse::<i32>().map_err(|_| "bad b")?;
            Ok(Command::Add(a, b))
        }
        "mul" if parts.len() == 3 => {
            let a = parts[1].parse::<i32>().map_err(|_| "bad a")?;
            let b = parts[2].parse::<i32>().map_err(|_| "bad b")?;
            Ok(Command::Mul(a, b))
        }
        "help" => Ok(Command::Help),
        "quit" | "exit" => Ok(Command::Quit),
        _ => Err("unknown command or wrong arity".to_string()),
    }
}
```

Teaching points:

- Guards (`if parts.len() == 3`) keep parsing logic clear.
- `Result` lets you report parse errors without panicking.

### Step 3: execute with `match`

```rust
fn execute(cmd: Command) -> bool {
    match cmd {
        Command::Add(a, b) => {
            println!("{}", a + b);
            true
        }
        Command::Mul(a, b) => {
            println!("{}", a * b);
            true
        }
        Command::Help => {
            println!("commands: add <a> <b>, mul <a> <b>, help, quit");
            true
        }
        Command::Quit => false,
    }
}
```

Returning `bool` makes the REPL loop simple: continue or exit.

### Step 4: connect to a loop

Use `loop` and `break`:

```rust
use std::io::{self, Read};

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

        if !execute(cmd) {
            break;
        }
    }
}
```

Instructor note: `use std::io::{self, Read};` is not needed for `read_line`; you can simplify imports in the live code.

## Common pitfalls

1. Using `_` too early: you lose exhaustiveness benefits. Prefer explicit cases.
2. Panicking on bad input: keep parse errors recoverable in a REPL.
3. Mixing parsing and execution: keep them separate so they are testable.

## Exercises

1. Add a `sub` command.
2. Add `repeat <n> <word>`.
3. Add unit tests for `parse_command`.
4. Improve errors to mention the exact wrong input.

## Recap

- `if` and `match` are expressions.
- `match` + enums makes states explicit.
- `loop` + `break` values are perfect for interactive flows.
- Separate parsing from execution for maintainability.
