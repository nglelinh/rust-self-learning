---
layout: post
title: "02-06 Ownership Fundamentals"
chapter: "02"
order: 6
owner: "OpenCode"
lang: en
categories:
  - chapter02
lesson_type: required
---

Ownership is the core idea that unlocks safe memory management without a garbage collector. This lesson builds an intuition for moves, copies, and drops, and teaches you how to read the most common ownership compiler errors.

## 60-minute teaching plan

- 0 to 10 min: The problem Rust solves (use-after-free, double-free) and the ownership rule.
- 10 to 25 min: Move vs copy: what happens when you assign or pass values.
- 25 to 40 min: Drops and scopes (RAII). Why destructors are reliable.
- 40 to 55 min: Live refactor: remove clones in a string pipeline by passing references.
- 55 to 60 min: Quick exercises and recap.

## Learning goals

By the end of this lesson, students can:

- State Rust’s ownership rules in plain language.
- Explain why `String` moves by default while integers copy.
- Predict when values are dropped (and why that matters).
- Fix “use of moved value” errors without random cloning.

## The ownership rules (learn these verbatim)

Rust enforces three core rules:

1. Each value has exactly one owner at a time.
2. When the owner goes out of scope, the value is dropped.
3. You can borrow references to a value, but borrowing has additional rules (next lesson).

Everything else in this chapter is a consequence of these.

## Move vs copy

### A move transfers ownership

```rust
let s1 = String::from("hello");
let s2 = s1;

// println!("{s1}"); // error: use of moved value
println!("{s2}");
```

Teaching points:

- `String` owns a heap allocation.
- When you assign `s1` to `s2`, Rust moves ownership.
- Rust prevents using `s1` afterward because that would risk double-free.

### A copy duplicates bits

```rust
let a = 10;
let b = a;
println!("a={a}, b={b}");
```

Teaching points:

- Simple scalar types implement `Copy`.
- Copy is implicit and cheap.

### The `Copy` trait is a marker

You don’t typically implement `Copy` early in the course. Instead, teach: "if a type owns resources (heap, file handles), it probably is not `Copy`."

## Passing values to functions

### Taking ownership

```rust
fn consume(s: String) {
    println!("consuming: {s}");
}

fn main() {
    let s = String::from("hello");
    consume(s);
    // println!("{s}"); // moved
}
```

### Borrowing instead (preview)

```rust
fn print_len(s: &String) {
    println!("len={}", s.len());
}

fn main() {
    let s = String::from("hello");
    print_len(&s);
    println!("still have s: {s}");
}
```

Teaching point: most "read-only" functions should borrow.

## Drops and RAII

When a value goes out of scope, Rust automatically calls `drop`.

```rust
{
    let s = String::from("temp");
    println!("inside: {s}");
}
// s is dropped here
```

Why this matters:

- Memory is reclaimed deterministically.
- File handles and locks are released reliably.
- You can tie resource lifetime to scope boundaries.

Instructor demo idea (conceptual): mention a `File` being closed when it goes out of scope.

## Live refactor: remove unnecessary clones

Start from a naive pipeline:

```rust
fn normalize(s: String) -> String {
    s.trim().to_lowercase()
}

fn words(s: String) -> Vec<String> {
    s.split_whitespace().map(|w| w.to_string()).collect()
}
```

This forces you to allocate and clone a lot. Improve it by borrowing inputs:

```rust
fn normalize(s: &str) -> String {
    s.trim().to_lowercase()
}

fn words(s: &str) -> Vec<&str> {
    s.split_whitespace().collect()
}
```

Teaching points:

- Borrowed APIs are more flexible.
- Returning `Vec<&str>` ties lifetimes to input; we will discuss this more in later lifetime lessons.
- Sometimes you need owned results; then you allocate once at the boundary.

## How to read the key compiler error

The most common error is E0382 (use of moved value). Teach a stable process:

1. Identify the move site.
2. Ask: should the function take ownership or borrow?
3. If you need to keep using the value, borrow.
4. Clone only when you truly need two owned copies.

## Exercises

1. Write `fn takes(s: String)` and `fn borrows(s: &str)` and call both from `main`. Explain which one lets you keep using the original variable.
2. Implement `fn shout(s: &str) -> String` that returns uppercase.
3. Given a function that takes `String` but only reads it, refactor it to borrow.

## Recap

- Moves prevent double-free and use-after-free.
- Copy types are the exception, not the rule.
- RAII means resources are released at scope end.
- Borrowing is the default strategy for read-only access.
