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

For decades, systems programmers faced an uncomfortable bargain. Pick C or C++, and you get raw bare-metal speed—at the price of manually pairing every `malloc` with `free`, where a single mistake creates a **use-after-free**, a **double-free**, or a silent memory leak. Pick Java, Go, or Python, and a **garbage collector (GC)** saves you from memory bugs—at the price of extra RAM overhead and unpredictable pause times.

Rust breaks that binary trade-off with a single, radical invention: **Ownership**. Every heap allocation has *exactly one* owner on the stack at any point in time, and the moment that owner leaves scope, `rustc` deterministically inserts the cleanup call (`drop`) at compile time—zero garbage collector, zero manual `free()`, and zero double-frees! This lecture builds your exact stack-and-heap mental model for **Move**, **Copy**, and **Clone**.

---

## Key ideas: Stack, Heap, and Compile-Time Resource Management

Understanding Ownership becomes effortless once you ground it in four computer-architecture fundamentals:

1. **Stack vs. Heap Memory:** The **stack** is a fast, LIFO region holding fixed-size values known at compile time (`i32`, `bool`, pointers, struct headers). The **heap** is a dynamic allocator for growable or runtime-sized buffers (`String`, `Vec<T>`, `Box<T>`), accessed via a small fixed-size handle stored on the stack.
2. **Affine Type System (Move Semantics by Default):** A `String`’s stack header is 24 bytes `(ptr, len, cap)` pointing to a heap buffer. Assigning `let s2 = s1;` copies only those 24 stack bytes and **invalidates `s1`** at compile time—guaranteeing two variables never try to free the same heap buffer twice.
3. **The `Copy` Marker Trait (Trivial Bitwise Duplication):** Types that live completely on the stack and own no external resources (`i32`, `f64`, `bool`, `(u8, u8)`) implement `Copy`, so `let y = x;` duplicates the bits and keeps *both* variables valid. A type that implements `Drop` (needs cleanup) can **never** implement `Copy`.
4. **RAII Beyond Memory:** Ownership isn’t just for heap bytes—files (`std::fs::File`), TCP sockets, and mutex locks (`MutexGuard`) all use the exact same deterministic `Drop` rule so resources never leak even on early returns.

---

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

---

### Worked Line-by-Line Trace (Stack & Heap Memory): Move vs. Copy vs. Clone

Trace every byte on the stack and heap across these 8 lines:

```rust
fn main() {
    let x: i32 = 42;              // Stack: x = 42 (4 bytes, implements Copy)
    let y = x;                    // COPY: y = 42; BOTH x and y remain valid!

    let s1 = String::from("hi");  // Stack: s1 = (ptr=0x100, len=2, cap=2) -> Heap@0x100: ['h','i']
    let s2 = s1;                  // MOVE:  s2 = (ptr=0x100, len=2, cap=2); s1 is INVALIDATED!
    let s3 = s2.clone();          // CLONE: s3 = (ptr=0x200, len=2, cap=2) -> Heap@0x200: ['h','i']
    println!("{x} {y} {s2} {s3}");
} // Scope ends: drop(s3) frees 0x200; drop(s2) frees 0x100; s1 does NOTHING (already moved)!
```

| Line | Stack State | Heap Allocations Alive | Valid Variables |
| :--- | :--- | :--- | :--- |
| `let x = 42; let y = x;` | `x: 42`, `y: 42` | None | `x`, `y` |
| `let s1 = String::from("hi");` | `s1: (0x100, 2, 2)` | `0x100: "hi"` | `x`, `y`, `s1` |
| `let s2 = s1;` (**Move**) | `s2: (0x100, 2, 2)` (`s1` dead) | `0x100: "hi"` (0 heap copy!) | `x`, `y`, `s2` |
| `let s3 = s2.clone();` (**Clone**) | `s2: (0x100, 2, 2)`, `s3: (0x200, 2, 2)` | `0x100: "hi"`, `0x200: "hi"` | `x`, `y`, `s2`, `s3` |
| End of `main()` | Stack popped | Both freed **once each** | None |

### Explanation: Why Ownership Eliminates Whole Classes of Bugs at Zero Cost

1. **Moves are cheap (just stack-header copies):** Moving a 100 MB `Vec<u8>` into a function copies only its 24-byte `(ptr, len, cap)` header—never the 100 MB buffer on the heap!
2. **Double-free becomes unrepresentable:** Because `s1` is statically invalidated the instant `s2 = s1` executes, it is impossible for both `s1` and `s2` to run `drop()` on `0x100`.
3. **Deep copies are always loud (`.clone()`):** In C++, copying a `std::vector` by value silently allocates on the heap. In Rust, expensive heap duplication only happens when you explicitly write `.clone()`, making performance costs visible in code review.
4. **When you don’t want to transfer ownership, borrow instead:** Passing `s2` by value moves it into a function; passing `&s2` (next lecture) lets the function read the data while you keep ownership.

**Quick check.** Suppose a function signature is `fn print_len(s: String)` and you call `print_len(s2);` followed by `println!("{s2}");`. Which exact ownership rule is violated, and what single character change to the parameter type fixes it without cloning?

---

## Exercises

1. Write `fn takes(s: String)` and `fn borrows(s: &str)` and call both from `main`. Explain which one lets you keep using the original variable.
2. Implement `fn shout(s: &str) -> String` that returns uppercase.
3. Given a function that takes `String` but only reads it, refactor it to borrow.

## Recap

- Moves prevent double-free and use-after-free.
- Copy types are the exception, not the rule.
- RAII means resources are released at scope end.
- Borrowing is the default strategy for read-only access.

## Video tham khảo

- [Understanding Ownership in Rust](https://www.youtube.com/watch?v=VFIOSWy93H0) — Let's Get Rusty, English. **xem trước.** Watch first: move, copy, and scope — the three rules the lesson asks you to learn verbatim.
- [Rust and RAII Memory Management - Computerphile](https://www.youtube.com/watch?v=pTMvh6VzDls) — Computerphile, English. RAII and Drop from a systems angle: who frees the resource when the owner goes out of scope.
