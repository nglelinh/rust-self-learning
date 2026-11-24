---
layout: post
title: "02-09 Structs and Methods"
chapter: "02"
order: 9
owner: "OpenCode"
lang: en
categories:
  - chapter02
lesson_type: required
---

In loosely typed codebases, a configuration object or bank account is often just a bag of public fields where any function anywhere can set `port = 0` or `balance = -500`—forcing every downstream function to repeat defensive `if` checks forever. Rust turns `struct` definitions and module privacy into a **compile-time invariant fortress**: keep the fields private, validate *once* inside a constructor (`new`), and from that moment on, holding a value of that type *proves* the invariant holds!

This lecture explores how structs lay out data contiguously in memory, how `impl` blocks attach methods with exact ownership receivers (`&self`, `&mut self`, `self`), and how encapsulation makes illegal states unrepresentable.

---

## Background Knowledge Map: Data Layout, Method Receivers, and Type-Driven Invariants

Four core ideas connect Rust structs directly to systems engineering and clean API design:

1. **Inline Contiguous Memory Layout (Zero Hidden Indirection):** Unlike Java or Python objects (where every object reference is a heap pointer), a Rust `struct` stores its fields **inline** where it is declared (on the stack or packed directly inside a `Vec<MyStruct>`), delivering superb CPU cache locality.
2. **Three Exact Method Receivers (`&self`, `&mut self`, `self`):** Every method explicitly declares its ownership contract: `&self` reads without moving, `&mut self` mutates in place exclusively, and `self` *consumes* the value (ideal for state-machine transitions or builder `.build()` methods so the old state can never be reused!).
3. **Encapsulation via Module Privacy:** Struct fields are private outside their defining module by default. Pairing private fields with a fallible constructor (`fn new(...) -> Result<Self, Err>`) guarantees no caller can ever forge an invalid instance.
4. **Automatic Referencing & Dereferencing:** When calling `cfg.port()`, `rustc` automatically inserts `&`, `&mut`, or `*` as needed to match the method’s receiver signature.

---

## 60-minute teaching plan

- 0 to 10 min: Struct basics and why modeling matters.
- 10 to 25 min: `impl` blocks: constructors, methods, associated functions.
- 25 to 40 min: Encapsulation: private fields and validating invariants.
- 40 to 55 min: Live coding: a `Config` struct for a CLI.
- 55 to 60 min: Exercises and recap.

## Learning goals

By the end of this lesson, students can:

- Design a struct with a clear responsibility and minimal fields.
- Use `impl` to add constructors and methods.
- Choose between `self`, `&self`, and `&mut self`.
- Keep invariants intact using private fields and validated constructors.

## Struct fundamentals

```rust
#[derive(Debug, Clone)]
struct User {
    name: String,
    age: u8,
}
```

Teaching points:

- `derive` adds common trait implementations.
- Start with simple structs; add behavior with `impl`.

## `impl`: methods and associated functions

```rust
impl User {
    fn new(name: String, age: u8) -> Self {
        Self { name, age }
    }

    fn greeting(&self) -> String {
        format!("hi, {}", self.name)
    }

    fn birthday(&mut self) {
        self.age = self.age.saturating_add(1);
    }
}
```

Receiver choices:

- `&self`: read-only method.
- `&mut self`: mutating method.
- `self`: consumes the value (often used for builders or conversions).

## Encapsulation and invariants

The main design goal: make invalid states unrepresentable.

Example invariant: a port must be in `1..=65535`.

### Bad design (public fields)

```rust
pub struct Config {
    pub host: String,
    pub port: u16,
}
```

Anyone can set invalid combinations, and validation becomes everyone’s problem.

### Better design (private fields + validated constructor)

```rust
#[derive(Debug, Clone)]
pub struct Config {
    host: String,
    port: u16,
    verbose: bool,
}

#[derive(Debug, Clone, PartialEq, Eq)]
pub enum ConfigError {
    EmptyHost,
    InvalidPort,
}

impl Config {
    pub fn new(host: String, port: u16, verbose: bool) -> Result<Self, ConfigError> {
        if host.trim().is_empty() {
            return Err(ConfigError::EmptyHost);
        }
        if port == 0 {
            return Err(ConfigError::InvalidPort);
        }
        Ok(Self { host, port, verbose })
    }

    pub fn host(&self) -> &str {
        &self.host
    }

    pub fn port(&self) -> u16 {
        self.port
    }

    pub fn verbose(&self) -> bool {
        self.verbose
    }
}
```

Teaching points:

- Errors should be typed, not strings.
- Accessors expose data without letting callers violate invariants.

## Live coding: parsing a config from args (minimal)

We avoid heavy CLI libraries for now. Teach basic `std::env::args`.

```rust
use std::env;

pub fn parse_args() -> Result<Config, String> {
    let mut args = env::args().skip(1);
    let host = args.next().ok_or("missing host")?;
    let port = args
        .next()
        .ok_or("missing port")?
        .parse::<u16>()
        .map_err(|_| "invalid port")?;
    let verbose = args.any(|a| a == "--verbose" || a == "-v");

    Config::new(host, port, verbose).map_err(|e| format!("config error: {:?}", e))
}
```

Teaching points:

- Keep parsing separate from validation.
- `Config::new` is the single place that enforces invariants.

## Struct update syntax (quick mention)

```rust
let base = Config::new("localhost".to_string(), 8080, false).unwrap();
let other = Config::new(base.host().to_string(), base.port(), true).unwrap();
```

In early lessons, avoid over-emphasizing update syntax; focus on invariants and accessors.

## Tests (show the habit)

```rust
#[test]
fn config_rejects_empty_host() {
    let err = Config::new(" ".to_string(), 8080, false).unwrap_err();
    assert_eq!(err, ConfigError::EmptyHost);
}

#[test]
fn config_rejects_zero_port() {
    let err = Config::new("localhost".to_string(), 0, false).unwrap_err();
    assert_eq!(err, ConfigError::InvalidPort);
}
```

## Common mistakes

1. Making fields public too early.
2. Letting parsing and validation leak everywhere.
3. Using `String` errors when a small enum would be clearer.

---

### Worked Line-by-Line Trace (Ownership Receivers & Invariants): `&self` vs. `&mut self` vs. `self`

Trace how each method receiver interacts with a validated `Port` struct and ownership:

```rust
pub struct Port(u16); // Private inner field: nobody outside can write `Port(0)`!

impl Port {
    pub fn new(n: u16) -> Result<Self, &'static str> {
        if n == 0 { Err("port 0 is reserved") } else { Ok(Self(n)) }
    }
    pub fn number(&self) -> u16 { self.0 }        // Shared borrow: caller keeps ownership
    pub fn bump(&mut self) { self.0 += 1; }       // Exclusive borrow: mutates in place
    pub fn into_u16(self) -> u16 { self.0 }       // Takes ownership: consumes `Port`!
}

fn main() {
    let mut p = Port::new(8080).unwrap(); // Stack: 2 bytes (`8080`), invariant n != 0 PROVED!
    println!("{}", p.number());           // Borrows `&p` -> prints 8080; `p` still alive
    p.bump();                             // Borrows `&mut p` -> mutates to 8081
    let raw = p.into_u16();               // MOVES `p` -> returns 8081
    // println!("{}", p.number());        // COMPILE ERROR if Port is non-Copy!
}
```

| Method Call | Receiver Type | Stack / Ownership Effect | Can Caller Still Use `p` Afterward? |
| :--- | :--- | :--- | :--- |
| `Port::new(8080)` | Associated fn (no `self`) | Validates `8080 != 0`, returns `Ok(Port(8080))` | Creates `p` |
| `p.number()` | `&self` | Temporary shared borrow | **Yes** |
| `p.bump()` | `&mut self` | Temporary exclusive borrow; `8080` $$	o$$ `8081` | **Yes** |
| `p.into_u16()` | `self` (by value) | Moves `p` into method and consumes it | **No** (if `Port` is not `Copy`) |

### Deep-Dive Explanation: Four Rules for Designing Idiomatic Rust Structs (Read Slowly)

1. **Validate at the boundary, trust inside the core:** Once `Port::new` returns `Ok(p)`, no internal function taking `Port` ever needs `if port == 0` checks again.
2. **Default to `&self`; escalate to `&mut self` or `self` deliberately:** Taking `self` by value is a superpower for *typestate* APIs (e.g., `Connection::close(self)` makes calling `conn.send()` after closing a compile-time error!).
3. **Derive standard traits thoughtfully (`Debug`, `Clone`, `PartialEq`):** Keep `Copy` only for small, cheap stack types whose duplication is semantically trivial—never derive `Copy` on types representing unique handles or state machines.
4. **Watch out for struct update syntax `..other` moving fields:** Writing `Config { port: 9000, ..base }` *moves* any non-`Copy` fields (like `String` hostnames) out of `base`, partially invalidating `base`.

**Quick check.** Why is marking `pub struct Port(pub u16);` dangerous if your application relies on the invariant that `port != 0`?

---

## Exercises

1. Add a `timeout_ms: u64` field with validation (`timeout_ms > 0`).
2. Add a `fn url(&self) -> String` method that formats `host:port`.
3. Change the error formatting so user-facing errors are friendly.
4. Write a unit test for `parse_args` by extracting an internal helper that parses a slice of strings.

## Recap

- Structs model state; `impl` attaches behavior.
- Keep invariants in one place (validated constructors).
- Use private fields to prevent invalid states.
- Choose method receivers based on whether you read, mutate, or consume the value.

## Video tham khảo

- [Structs in Rust](https://www.youtube.com/watch?v=n3bPhdiJm9I) — Let's Get Rusty, English. **xem trước.** Watch first: struct layout, fields, and constructing values before methods.
- [Implement Methods on Rust Structs 🦀](https://www.youtube.com/watch?v=7EYSXQFRyKY) — Trevor Sullivan, English. impl blocks, methods, and associated functions for the config-parsing exercise.
