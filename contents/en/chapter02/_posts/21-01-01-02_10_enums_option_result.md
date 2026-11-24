---
layout: post
title: "02-10 Enums, Option, and Result"
chapter: "02"
order: 10
owner: "OpenCode"
lang: en
categories:
  - chapter02
lesson_type: required
---

Tony Hoare famously called his 1965 invention of the **null reference** his *"billion-dollar mistake"*—responsible for countless `NullPointerException` crashes and C segfaults across half a century of software. Rust refuses to add `null` to the language. Instead, absence and failure are ordinary, type-checked **algebraic data types (`enum`)**: `Option<T>` says *"a value might be absent"*, and `Result<T, E>` says *"this operation might fail with error `E`"*.

This lecture unlocks Rust’s richest modeling tool: enums that carry per-variant payloads, how `Option<T>` and `Result<T, E>` compose cleanly with `?` and combinators (`map`, `and_then`), and a delightful compiler trick—the **Null Pointer Optimization (NPO)**—that makes `Option<&T>` cost literally **zero extra bytes** in memory!

---

## Background Knowledge Map: Algebraic Data Types, Tagged Unions, and Zero-Cost Safety

Four computer-science ideas explain why Rust’s `enum` is far more powerful than C/Java enums:

1. **Sum Types (Tagged Unions):** A `struct` is a *product type* (has field A **AND** field B). A Rust `enum` is a *sum type* (is either variant A with payload $$P_A$$ **OR** variant B with payload $$P_B$$). In memory, an enum occupies `discriminant tag + max(variant sizes)` plus alignment padding.
2. **Null Pointer Optimization (NPO):** Because a Rust reference `&T` or `Box<T>` is *guaranteed* never to be address `0x0`, `rustc` represents `Option<&T>::None` as the bit pattern `0x0` and `Some(ptr)` as the raw pointer itself—so `std::mem::size_of::<Option<&T>>() == 8` bytes (zero overhead compared to a nullable C pointer, yet 100% checked at compile time!).
3. **Explicit Absence & Recoverable Errors (`Option` & `Result`):** You cannot accidentally dereference a `None` or ignore an `Err`—the type system forces you to unwrap, match, or propagate explicitly.
4. **Early-Return Propagation (`?`):** The `?` operator is pure syntactic sugar over `match`: on `Ok(v)` it yields `v`; on `Err(e)` it immediately converts `e` via `From` and returns `Err(e.into())` from the enclosing function.

---

## 60-minute teaching plan

- 0 to 10 min: Why enums matter: modeling states, not flags.
- 10 to 25 min: `Option<T>` patterns and common combinators.
- 25 to 40 min: `Result<T, E>` patterns and propagation.
- 40 to 60 min: Mini-lab: expression AST + evaluator with typed errors.

## Learning goals

By the end of this lesson, students can:

- Design enums that represent real domain states.
- Use `match` to destructure and handle cases exhaustively.
- Use `Option` to represent "maybe" values without null.
- Use `Result` to represent fallible operations with explicit errors.

## Enums as domain models

Enums are not just "one of several values"; they are how you encode state machines.

Example: a user input command.

```rust
#[derive(Debug, Clone, PartialEq, Eq)]
enum Command {
    Add,
    Remove,
    Quit,
}
```

The key win: `Command` can only be one of these states. No invalid values.

## `Option<T>`: safe absence

### Basic matching

```rust
fn first(xs: &[i32]) -> Option<i32> {
    xs.get(0).copied()
}

fn main() {
    match first(&[1, 2, 3]) {
        Some(x) => println!("first={x}"),
        None => println!("empty"),
    }
}
```

### Common combinators

Teach 3 that students will use immediately:

- `map`: transform the inside
- `and_then`: chain options
- `unwrap_or`: provide a default

```rust
let maybe = Some(" 42 ");
let parsed = maybe
    .map(|s| s.trim())
    .and_then(|s| s.parse::<i32>().ok());

let n = parsed.unwrap_or(0);
```

Teaching point: combinators reduce nested matches, but `match` remains the clearest tool when logic is complex.

## `Result<T, E>`: explicit failure

### Basic matching

```rust
fn parse_i32(s: &str) -> Result<i32, String> {
    s.trim().parse::<i32>().map_err(|e| e.to_string())
}
```

### Propagation with `?`

```rust
fn add_two_numbers(a: &str, b: &str) -> Result<i32, String> {
    let a = parse_i32(a)?;
    let b = parse_i32(b)?;
    Ok(a + b)
}
```

Teach: `?` is "early return on error" and forces you to be honest about failure.

### Mapping results

```rust
let x = "42".parse::<i32>().map(|n| n * 2);
```

## Mini-lab: expression AST + evaluator

Goal: model arithmetic expressions and evaluate them with typed errors.

### Step 1: define the AST

```rust
#[derive(Debug, Clone, PartialEq)]
enum Expr {
    Num(f64),
    Add(Box<Expr>, Box<Expr>),
    Sub(Box<Expr>, Box<Expr>),
    Mul(Box<Expr>, Box<Expr>),
    Div(Box<Expr>, Box<Expr>),
}
```

Teaching points:

- `Box<Expr>` allows recursive enums.
- The AST is an explicit structure, not a string.

### Step 2: define a typed error

```rust
#[derive(Debug, Clone, PartialEq)]
enum EvalError {
    DivisionByZero,
}
```

### Step 3: implement evaluation

```rust
fn eval(e: &Expr) -> Result<f64, EvalError> {
    match e {
        Expr::Num(x) => Ok(*x),
        Expr::Add(a, b) => Ok(eval(a)? + eval(b)?),
        Expr::Sub(a, b) => Ok(eval(a)? - eval(b)?),
        Expr::Mul(a, b) => Ok(eval(a)? * eval(b)?),
        Expr::Div(a, b) => {
            let denom = eval(b)?;
            if denom == 0.0 {
                return Err(EvalError::DivisionByZero);
            }
            Ok(eval(a)? / denom)
        }
    }
}
```

Teaching points:

- `?` composes naturally for recursive evaluation.
- The division case shows how domain checks become typed errors.

### Step 4: tests

```rust
#[test]
fn eval_add() {
    let e = Expr::Add(Box::new(Expr::Num(1.0)), Box::new(Expr::Num(2.0)));
    assert_eq!(eval(&e).unwrap(), 3.0);
}

#[test]
fn eval_div_by_zero() {
    let e = Expr::Div(Box::new(Expr::Num(1.0)), Box::new(Expr::Num(0.0)));
    assert_eq!(eval(&e), Err(EvalError::DivisionByZero));
}
```

## Beyond `match`: `if let` and `let else`

For simple cases:

```rust
let maybe = Some(10);
if let Some(x) = maybe {
    println!("x={x}");
}
```

Mention: when cases grow, prefer `match` for clarity and exhaustiveness.

## Common mistakes

1. Using `Option` when you need to explain failure (should be `Result`).
2. Returning `bool` to indicate success/failure (lose error details).
3. Using strings for errors everywhere (hard to test and pattern match).

---

### Worked Line-by-Line Trace (Memory Size & `?` Desugaring): Tracing `Option`, `Result`, and `?`

First, check the exact memory layout of two enums on a 64-bit CPU:
- `Option<u64>`: needs a tag byte + 7 padding bytes + 8-byte `u64` = **16 bytes**.
- `Option<&u64>`: uses address `0x0` for `None` (NPO!) = **8 bytes** (zero extra memory!).

Now trace how a two-line function using `?` desugars step by step:

```rust
fn parse_and_double(raw: &str) -> Result<i32, std::num::ParseIntError> {
    let n = raw.trim().parse::<i32>()?; // `?` desugars to the exact match below!
    Ok(n * 2)
}
// Desugared equivalent of line 2:
// let n = match raw.trim().parse::<i32>() {
//     Ok(val) => val,
//     Err(e)  => return Err(e.into()), // Early return! Line 3 never runs.
// };
```

| Input `raw` | `raw.trim().parse::<i32>()` | Action of `?` | Final Return Value |
| :--- | :--- | :--- | :--- |
| `" 21 "` | `Ok(21)` | Extracts `n = 21`, continues to `Ok(n * 2)` | `Ok(42)` |
| `"abc"` | `Err(ParseIntError)` | Immediately returns `Err(...)` (skips `n * 2`) | `Err(ParseIntError)` |

### Deep-Dive Explanation: Four Patterns for Clean Enum, `Option`, and `Result` Code (Read Slowly)

1. **Model states as an `enum`, not a soup of booleans:** Replace `struct State { is_loading: bool, error: Option<String>, data: Option<Data> }` (which allows contradictory states like loading *and* error!) with `enum State { Loading, Failed(String), Ready(Data) }`.
2. **Use `?` for linear happy-path code:** Chaining three fallible steps with `?` reads top-to-bottom like synchronous code while preserving exact error propagation at every step.
3. **Reach for combinators (`map`, `and_then`, `unwrap_or`) for short one-liners:** `opt.map(|x| x * 2).unwrap_or(0)` is cleaner than a 4-line `match`, while complex multi-branch logic still reads clearest as an explicit `match`.
4. **Bridge `Option` to `Result` cleanly with `.ok_or(...)` / `.ok_or_else(...)`:** Use `.ok_or_else(|| ...)` whenever constructing the error allocates a `String`, so the allocation runs *only* on the error path.

**Quick check.** On a 64-bit machine, why does `std::mem::size_of::<Option<Box<i32>>>()` equal **8 bytes** instead of 16 bytes?

---

## Exercises

1. Add a `Neg(Box<Expr>)` variant and implement it.
2. Extend `EvalError` with `NonFinite` and reject `NaN`/`inf` outputs.
3. Implement `simplify(expr) -> Expr` that folds constant sub-expressions.
4. Write a `display(expr) -> String` function to print the AST.

## Recap

- Enums model state explicitly.
- `Option` expresses absence; `Result` expresses failure.
- `match` forces completeness.
- Typed errors make code easier to test and maintain.

## Video tham khảo

- [Enums and Pattern Matching in Rust](https://www.youtube.com/watch?v=DSZqIJhkNCM) — Let's Get Rusty, English. **xem trước.** Watch first: enums, Option, Result, and match as one idea.
- [Rust Option Enum Type Overview](https://www.youtube.com/watch?v=z8k_EViGC10) — Trevor Sullivan, English. Option on its own: Some, None, and the methods you will use instead of null.
