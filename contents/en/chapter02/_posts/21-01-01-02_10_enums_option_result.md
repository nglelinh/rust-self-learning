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

Enums make state explicit. `Option` and `Result` are the everyday tools for safe absence and failure. This lesson teaches how to model a domain with enums, how to compose `Option`/`Result`, and how `match` ties everything together.

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
