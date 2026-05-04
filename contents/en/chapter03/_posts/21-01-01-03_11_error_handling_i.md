---
layout: post
title: "03-11 Error Handling I"
chapter: "03"
order: 11
owner: "OpenCode"
lang: en
categories:
  - chapter03
lesson_type: required
---

Idiomatic Rust error handling is about making failure explicit, keeping errors useful, and making it easy to propagate failures without losing context.

## 60-minute teaching plan

- 0 to 10 min: The error-handling philosophy: no exceptions, explicit failures.
- 10 to 25 min: `Result<T, E>` patterns and `?`.
- 25 to 40 min: Designing error types (small enums) and adding context.
- 40 to 60 min: Mini-lab: file-based config loader with structured errors and tests.

## Learning goals

By the end of this lesson, students can:

- Use `Result` to model fallible operations.
- Propagate errors with `?` without nested `match`.
- Design a small error enum for a module.
- Return `Result` from `main` for clean CLI code.

## The mental model

In Rust:

- `Option<T>` means: value might not exist.
- `Result<T, E>` means: operation might fail, and you want to know why.

This lesson is about `Result`.

## `Result` basics

```rust
fn parse_port(s: &str) -> Result<u16, String> {
    let n = s.trim().parse::<u16>().map_err(|_| "not a number")?;
    if n == 0 {
        return Err("port must be > 0".to_string());
    }
    Ok(n)
}
```

Teaching points:

- `?` returns early on `Err`.
- `Ok(...)` is the success path.

## Returning `Result` from `main`

Rust allows:

```rust
fn main() -> Result<(), Box<dyn std::error::Error>> {
    // ...
    Ok(())
}
```

For teaching, you can either:

- keep error types explicit (good for learning), or
- use a boxed trait object (good for quick CLIs).

This lesson focuses on explicit error types.

## Error type design (small and local)

Prefer a small enum over strings:

```rust
#[derive(Debug, Clone, PartialEq, Eq)]
pub enum ConfigError {
    MissingField(&'static str),
    InvalidNumber { field: &'static str },
    InvalidValue { field: &'static str, reason: &'static str },
    Io,
}
```

Teaching points:

- Use `'static str` for fixed field names.
- Keep errors structured so tests can match variants.

## Mini-lab: file-based config loader

We will parse a tiny config format:

```
host=localhost
port=8080
verbose=true
```

### Step 1: define the output model

```rust
#[derive(Debug, Clone, PartialEq, Eq)]
pub struct Config {
    pub host: String,
    pub port: u16,
    pub verbose: bool,
}
```

Instructor note: in later lessons, you can make fields private and provide accessors. Here we keep it simple.

### Step 2: parse key/value lines

```rust
fn parse_bool(s: &str) -> Option<bool> {
    match s.trim() {
        "true" | "1" | "yes" | "y" => Some(true),
        "false" | "0" | "no" | "n" => Some(false),
        _ => None,
    }
}

fn parse_kv(line: &str) -> Option<(&str, &str)> {
    let line = line.trim();
    if line.is_empty() || line.starts_with('#') {
        return None;
    }
    let (k, v) = line.split_once('=')?;
    Some((k.trim(), v.trim()))
}
```

Teaching points:

- `split_once` is a clean way to avoid manual indexing.
- Returning `Option` here is fine: "this line has no kv".

### Step 3: build the loader with structured errors

```rust
use std::fs;

pub fn load_config(path: &str) -> Result<Config, ConfigError> {
    let text = fs::read_to_string(path).map_err(|_| ConfigError::Io)?;

    let mut host: Option<String> = None;
    let mut port: Option<u16> = None;
    let mut verbose: Option<bool> = None;

    for line in text.lines() {
        let Some((k, v)) = parse_kv(line) else { continue };
        match k {
            "host" => host = Some(v.to_string()),
            "port" => {
                let p = v.parse::<u16>().map_err(|_| ConfigError::InvalidNumber { field: "port" })?;
                if p == 0 {
                    return Err(ConfigError::InvalidValue { field: "port", reason: "must be > 0" });
                }
                port = Some(p);
            }
            "verbose" => {
                let b = parse_bool(v).ok_or(ConfigError::InvalidValue { field: "verbose", reason: "expected true/false" })?;
                verbose = Some(b);
            }
            _ => {
                // ignore unknown keys in this version
            }
        }
    }

    Ok(Config {
        host: host.ok_or(ConfigError::MissingField("host"))?,
        port: port.ok_or(ConfigError::MissingField("port"))?,
        verbose: verbose.unwrap_or(false),
    })
}
```

Teaching points:

- `let else` keeps the loop readable.
- Missing fields become explicit errors.
- `verbose` is optional with a default.

## Tests

Teach that error variants are easy to test:

```rust
#[test]
fn parse_bool_variants() {
    assert_eq!(parse_bool("true"), Some(true));
    assert_eq!(parse_bool("no"), Some(false));
    assert_eq!(parse_bool("maybe"), None);
}
```

If file IO tests are awkward, factor parsing into `load_config_str(&str)` and test it directly.

## Common mistakes

1. Returning `String` errors everywhere (hard to match, easy to break).
2. Losing context (every error becomes "failed").
3. `unwrap()` on parsing or IO.

## Exercises

1. Add `timeout_ms` with validation.
2. Change loader to reject unknown keys instead of ignoring them.
3. Add line numbers to errors (store an index during iteration).
4. Refactor to `load_config_str` + `load_config_file`.

## Recap

- `Result` makes failure explicit.
- `?` keeps the success path readable.
- Typed errors are testable and maintainable.
- Put parsing and validation in one place; keep callers simple.
