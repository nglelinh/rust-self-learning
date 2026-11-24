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

In C or Java, forgetting a `default` branch or a `break` statement inside a `switch` compiles without complaint—and quietly lurks in production until a new enum variant is added six months later. Rust turns control flow into a **compile-time completeness proof**: `if` and `match` are *expressions* that evaluate to values, and every `match` is **exhaustively checked** by the compiler. Miss a single possible case, and your code refuses to compile.

This lecture teaches Rust’s expression-oriented style: replacing mutable scratch variables with `if` and `match` expressions, mastering ranges, guards, and destructuring patterns, and choosing cleanly between `match`, `if let`, and `while let`.

---

## Background Knowledge Map: Expression-Oriented Control Flow & Exhaustive Matching

Four design principles elevate Rust’s control flow far beyond traditional `if`/`switch` statements:

1. **Expressions Over Statements:** Blocks `{ ... }`, `if/else`, `match`, and even `loop` evaluate to values. You can initialize immutable bindings directly (`let label = if ok { "ok" } else { "err" };`) without ever declaring an uninitialized `mut` variable.
2. **Exhaustiveness Checking:** `rustc` statically verifies that the arms of a `match` cover 100% of the input type’s domain. When you later add a new variant to an `enum`, the compiler immediately highlights every `match` across the codebase that needs updating.
3. **Destructuring + Pattern Guards:** Patterns simultaneously inspect the *shape* of tuples, structs, enums, and ranges (`1..=5`) and bind inner fields to local variables in one atomic step, optionally refined by boolean `if` guards.
4. **Focused Sugar (`if let` and `while let`):** When you genuinely care about only one variant (e.g., `Some(x)` or `Ok(v)`), `if let` and `while let` remove boilerplate without sacrificing type safety.

---

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

---

### Worked Line-by-Line Trace (Expression Evaluation & the Semicolon Rule): Tracing `match` Arms & Guards

Trace three inputs step by step through this expression-oriented function:

```rust
fn classify(pair: (i32, Option<i32>)) -> &'static str {
    match pair {
        (x, Some(y)) if x + y == 10 => "sums to 10", // Arm 1: destructure + guard
        (0, Some(_))                => "x is zero",  // Arm 2: reached only if Arm 1 fails
        (x, None) if x < 0          => "neg & none", // Arm 3: negative x with None
        _                           => "other",      // Arm 4: exhaustive fallback
    } // NO trailing semicolon -> the entire `match` is the return expression!
}
```

| Input `pair` | Arm 1 `(x, Some(y)) if x+y==10` | Arm 2 `(0, Some(_))` | Arm 3 `(x, None) if x<0` | Returned Value |
| :--- | :--- | :--- | :--- | :--- |
| `(0, Some(10))` | Binds `x=0, y=10`; guard `0+10==10` is **True** | (skipped) | (skipped) | `"sums to 10"` |
| `(0, Some(3))` | Binds `x=0, y=3`; guard `0+3==10` is **False** | Matches `0` & `Some(3)` -> **True** | (skipped) | `"x is zero"` |
| `(-4, None)` | Pattern mismatch (`None` $$
eq$$ `Some`) | Pattern mismatch | Binds `x=-4 < 0` -> **True** | `"neg & none"` |

*Watch the semicolon:* Adding a `;` right after the closing `}` of `match` turns the expression into a **statement** returning the unit type `()`—which immediately triggers a compile-time return-type mismatch against `&'static str`.

### Deep-Dive Explanation: Four Habits for Clean Rust Control Flow (Read Slowly)

1. **Use expression assignment to eliminate `mut` boilerplate:** Prefer `let msg = match code { ... };` over declaring `let mut msg = "";` and mutating it inside branches.
2. **Avoid blind `_ =>` wildcards on your own enums:** Listing every variant explicitly harnesses the compiler as a refactoring safety net whenever new variants appear.
3. **Order guarded arms from most specific to most general:** When a pattern matches structurally but its `if` guard evaluates to `false`, Rust falls through to test subsequent arms in top-to-bottom order.
4. **Use `loop { ... break value; }` for retry loops:** Because `loop` is guaranteed to run until a `break`, `rustc` knows its return value is always initialized when breaking with a payload.

**Quick check.** In `classify` above, what happens at compile time and at runtime if someone accidentally moves the wildcard arm `_ => "other"` to the very top of the `match`?

---

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

## Video tham khảo

- [Intro to Rust Control Flow Statements & Conditional Expressions 🦀](https://www.youtube.com/watch?v=3XpxkJXggHA) — Trevor Sullivan, English. **xem trước.** Watch first: if as an expression, loops, and the control-flow shapes the typed REPL lab uses.
- [Mastering Rust's Match Control Flow - Full Crash Rust Tutorial for Beginners](https://www.youtube.com/watch?v=sd1M_CLXl4I) — Francesco Ciulla, English. match as the workhorse, including pattern arms you will use in the REPL exercise.
