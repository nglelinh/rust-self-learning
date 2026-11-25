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

In languages with unchecked exceptions, a single line like `load_config(path)` hides a minefield: could it throw a file-not-found error, a permission error, a syntax error, or a network timeout? You often don’t know until a stack trace wakes you up at 3 a.m. Rust takes the opposite road: **recoverable failures are ordinary return values (`Result<T, E>`)**—visible right in the function signature, composable with `?`, and exhaustively matchable by callers.

This lecture shows you how to design clean, structured local error enums (wrapping `io::Error` and parse errors), wire automatic error conversion via `From`, and build a real file-based config loader where every failure path is explicit and testable.

---

## Key ideas: Error-as-Value Architecture in Rust

Four pillars separate mature Rust error handling from ad-hoc string errors:

1. **Errors as Values (`Result<T, E>`):** Because errors flow through normal return channels rather than unwinding hidden exception stacks, control flow stays transparent and predictable—critical for systems code, async tasks, and embedded targets.
2. **Structured Domain Error Enums:** Instead of flattening every failure into `Result<T, String>`, defining `enum ConfigError { Io(io::Error), MissingKey(String), BadInt(ParseIntError) }` lets callers programmatically branch on *why* something failed (e.g., retry on I/O timeout, fail fast on bad syntax).
3. **Automatic Conversion via `From` + `?`:** Implementing `From<io::Error> for ConfigError` lets the `?` operator seamlessly lift lower-level library errors into your domain error enum without repetitive `.map_err(...)` boilerplate.
4. **The `std::error::Error` Trait Ecosystem:** Implementing `Display` + `Debug` + `Error` (or using crates like `thiserror` for libraries and `anyhow` for applications) integrates your custom errors with Rust’s entire logging and reporting ecosystem.

---

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

---

### Worked Line-by-Line Trace (Error Conversion & `?`): How Two Different Errors Flow Into One `ConfigError`

Trace how `?` converts two completely different error types into a single `ConfigError` enum:

```rust
use std::io;
use std::num::ParseIntError;

#[derive(Debug)]
enum ConfigError {
    Io(io::Error),
    Parse(ParseIntError),
}
impl From<io::Error> for ConfigError {
    fn from(e: io::Error) -> Self { Self::Io(e) }
}
impl From<ParseIntError> for ConfigError {
    fn from(e: ParseIntError) -> Self { Self::Parse(e) }
}

fn load_port(path: &str) -> Result<u16, ConfigError> {
    let text = std::fs::read_to_string(path)?; // Error type: io::Error -> ConfigError::Io
    let port = text.trim().parse::<u16>()?;    // Error type: ParseIntError -> ConfigError::Parse
    Ok(port)
}
```

| Runtime Scenario | Failing Line | Raw Error Type | Conversion Triggered by `?` | Returned Value |
| :--- | :--- | :--- | :--- | :--- |
| File missing | `read_to_string(path)?` | `io::Error` | `ConfigError::from(io_err)` | `Err(ConfigError::Io(...))` |
| File contains `"xyz"` | `parse::<u16>()?` | `ParseIntError` | `ConfigError::from(parse_err)` | `Err(ConfigError::Parse(...))` |
| File contains `"8080
"` | None | — | Both `?` extract `Ok` values | `Ok(8080)` |

### Explanation: Four Rules for Designing Maintainable Rust Errors

1. **Libraries use structured enums; top-level apps can use boxed/context errors:** A library cannot guess how callers want to react to each failure, so it exposes a concrete `enum`. A CLI `main()` simply wants rich human-readable context.
2. **Add context (like file path or line number) where the error happens:** Raw `io::Error` ("No such file or directory") doesn’t tell the user *which* file was missing—attach the path or key name in your error variant!
3. **Keep error enums small and local to each module:** One giant monolithic `AppError` with 40 variants couples unrelated subsystems; prefer focused error types per component that compose upward via `From`.
4. **Test error paths as thoroughly as happy paths:** Write unit tests asserting `matches!(load_port("bad"), Err(ConfigError::Parse(_)))` so error contracts never regress silently.

**Quick check.** In `load_port` above, what compile-time error occurs if you delete the `impl From<ParseIntError> for ConfigError` block while keeping `text.trim().parse::<u16>()?`?

---

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

## Video tham khảo

- [A Simpler Way to See Results](https://www.youtube.com/watch?v=s5S2Ed5T-dc) — Logan Smith, English. **xem trước.** Watch first: Result as data, which is the mental model behind the config-loader lab.
- [Rust Error Handling Explained: thiserror vs anyhow (Best Practices)](https://www.youtube.com/watch?v=-c9JEexiHPE) — Semicolon, English. Where a small local error enum stops and thiserror or anyhow starts — the design section.
