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

Structs help you model data with invariants, and `impl` blocks help you keep behavior close to the data. This lesson teaches how to design structs that cannot represent invalid state, and how to choose method receivers intentionally.

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
