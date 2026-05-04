---
layout: post
title: "04-18 Ownership-Friendly APIs"
chapter: "04"
order: 18
owner: "OpenCode"
lang: en
categories:
  - chapter04
lesson_type: required
---

Most ergonomic Rust is about choosing the right ownership model at API boundaries. This lesson teaches a practical decision framework: what to borrow, what to own, and how to avoid clones without making APIs painful.

## 60-minute teaching plan

- 0 to 10 min: The API boundary problem (call-site ergonomics vs performance).
- 10 to 25 min: Borrowed inputs (`&str`, `&[T]`) and why they are the default.
- 25 to 40 min: Return types: borrowed vs owned vs hybrid (`Cow`).
- 40 to 55 min: Refactor lab: improve signatures from earlier lessons.
- 55 to 60 min: Checklist and recap.

## Learning goals

By the end of this lesson, students can:

- Prefer borrowed inputs to maximize usability.
- Decide when returning `&str` is safe and when returning `String` is required.
- Use `AsRef` for flexible inputs when it helps.
- Understand `Cow` as a "borrow-or-own" return type option.

## Prerequisites

- Lesson 04-16 (Lifetimes I): lifetime parameters on functions.
- Lesson 04-17 (Lifetimes II): lifetime parameters on structs.
- Lesson 02-06 (Ownership): move vs borrow.

---

## Key concept: the API design spectrum

Every function parameter sits somewhere on this spectrum:

```
Own everything                                  Borrow everything
   String, Vec<T>   ←——————————————→  &str, &[T], &T
   maximum control                    maximum flexibility
   minimum call-site ergonomics       minimum overhead
```

In practice, Rust code leans heavily toward borrowing inputs and being explicit about owned outputs. The reason: if you take `String`, the caller with a `&str` must `.to_string()` just to call your function — that's an allocation the caller shouldn't pay for.

---

## The guiding principles

1. **Accept borrowed inputs** when possible (`&str` over `String`, `&[T]` over `Vec<T>`).
2. **Return owned outputs** when you create new data (transform, concatenate, etc.).
3. **Return borrowed outputs** only when returning a view into an input.
4. **Clone at boundaries**: clone when you store data long-term or need multiple owners.

---

## Borrowed inputs: the default

### String inputs

```rust
// Bad: forces caller to allocate if they have &str
fn count_words_bad(text: String) -> usize {
    text.split_whitespace().count()
}

// Good: accepts both String and &str
fn count_words(text: &str) -> usize {
    text.split_whitespace().count()
}
```

Call-site comparison:

```rust
let s = String::from("hello world");
let lit = "hello world";

// bad: forced clone
count_words_bad(s.clone());
count_words_bad(lit.to_string());

// good: just borrow
count_words(&s);
count_words(lit);    // &str is already &str
```

### Slice inputs

```rust
// Bad: Vec<T> input
fn sum_bad(xs: Vec<i32>) -> i32 {
    xs.iter().sum()
}

// Good: slice input
fn sum(xs: &[i32]) -> i32 {
    xs.iter().sum()
}
```

`&[i32]` is a "fat pointer" (pointer + length) that is cheaply created from `Vec<i32>` or array references with `&xs[..]` or just `&xs`.

---

## Return types: borrowed vs owned

### Return borrowed when you can

Return a view into an input when no new data is created:

```rust
fn first_word(s: &str) -> &str {
    s.split_whitespace().next().unwrap_or("")
}

fn extension(path: &str) -> Option<&str> {
    let dot = path.rfind('.')?;
    Some(&path[dot+1..])
}
```

Teaching point: output is a view into `s`, so borrowing is correct and zero-allocation.

### Return owned when you must

Return `String` when you create new data:

```rust
fn normalize(s: &str) -> String {
    s.trim().to_lowercase()
}

fn join_parts(parts: &[&str]) -> String {
    parts.join(", ")
}
```

Rule: if you call `.to_string()`, `.to_owned()`, format!, or string concatenation — you own the result.

### The `&str` vs `String` decision tree

```
Does the returned data exist in the input?
    ├─ YES → return &str (borrow from input)
    └─ NO  → return String (you created it)
```

---

## Code walkthrough: naive → idiomatic

### Naive: everything owned

```rust
fn shout(s: String) -> String {
    s.to_uppercase()
}

fn get_value(line: String) -> String {
    line.split_once('=').unwrap().1.to_string()
}
```

Problems:
- `shout`: takes `String` but only reads — forces callers to give up ownership.
- `get_value`: returns `String` but could return `&str` (a view into `line`).

### Idiomatic: borrow inputs, own or borrow outputs appropriately

```rust
fn shout(s: &str) -> String {
    s.to_uppercase()     // creates new data → owned return
}

fn get_value(line: &str) -> &str {
    line.split_once('=').map(|(_, v)| v.trim()).unwrap_or("")
    // returns a view into line → borrowed return
}
```

Caller now has maximum flexibility:

```rust
let s = String::from("hello");
let lit = "hello";
shout(&s);   // works
shout(lit);  // works

let line = String::from("host=127.0.0.1");
let value = get_value(&line);
println!("{value}");  // value borrows from line
```

---

## Flexible inputs: `AsRef<str>` and `AsRef<Path>` (use sparingly)

`AsRef<str>` lets you accept both `String` and `&str` without overloading:

```rust
fn normalize<S: AsRef<str>>(s: S) -> String {
    s.as_ref().trim().to_lowercase()
}
```

Call sites:

```rust
normalize("HELLO WORLD");                    // &str
normalize(String::from("HELLO WORLD"));     // String
normalize(&String::from("HELLO WORLD"));    // &String
```

**When to use `AsRef`**: primarily for public library APIs where callers have many owned-vs-borrowed scenarios. For internal code, plain `&str` is almost always simpler.

**When NOT to use `AsRef`**: don't add generic parameters to avoid one `.as_str()` call at a call site. Compile times and error messages suffer.

---

## Hybrid returns: `Cow<'a, str>`

`Cow` (Clone On Write) is a sum type: it's either a borrowed `&str` or an owned `String`.

```rust
use std::borrow::Cow;

fn trim_cow(s: &str) -> Cow<'_, str> {
    let t = s.trim();
    if t.len() == s.len() {
        Cow::Borrowed(s)     // no change — return the original slice
    } else {
        Cow::Owned(t.to_string())   // changed — allocate
    }
}
```

Teaching points:

- `Cow` is a tool for **performance-sensitive boundaries** where you want to avoid allocating when no modification is needed.
- Classic use case: normalizer/escaper functions that sometimes need to change the string, sometimes don't.
- `Cow<'_, str>` implements `Deref<Target = str>`, so you can use it like `&str`.

### When `Cow` is worth it

```rust
// If 90% of inputs need no trimming, Cow avoids 90% of allocations
let result = trim_cow("  already clean  ");  // allocates
let result = trim_cow("no whitespace");       // borrows, no allocation
```

Don't introduce `Cow` unless you have measured allocation overhead and it matters.

---

## Refactor lab: improve earlier lesson signatures

### Case 1: method that only reads but takes ownership

Before:

```rust
fn words(s: String) -> Vec<String> {
    s.split_whitespace().map(|w| w.to_string()).collect()
}
```

After:

```rust
fn words(s: &str) -> Vec<&str> {
    s.split_whitespace().collect()
}
```

Now there are zero allocations (the returned `Vec` contains slices into `s`). If callers need owned words, they call `.to_string()` on specific items.

### Case 2: returning owned when borrowed works

Before:

```rust
fn get_host(line: &str) -> String {
    line.split_once('=').unwrap().1.to_string()
}
```

After:

```rust
fn get_host(line: &str) -> &str {
    line.split_once('=').map(|(_, v)| v.trim()).unwrap_or("")
}
```

Only do this when you can keep the buffer alive at the call site.

### Case 3: API that needs to store data

If a struct needs to store text long-term, it must own it:

```rust
struct Config {
    host: String,
    port: u16,
}

impl Config {
    fn from_line(line: &str) -> Option<Self> {
        let (host, port_str) = line.split_once(':')?;
        let port = port_str.parse().ok()?;
        Some(Self {
            host: host.to_string(),  // clone at the storage boundary
            port,
        })
    }
}
```

Teaching point: clone **at the boundary** where you decide to store, not at every function call.

---

## Applications in systems programming

### CLI tools

A CLI tool typically reads `&str` arguments from the OS and converts them to structured types:

```rust
fn parse_addr(s: &str) -> Option<(std::net::IpAddr, u16)> {
    let (ip, port) = s.rsplit_once(':')?;
    Some((ip.parse().ok()?, port.parse().ok()?))
}
```

The `&str` arg is valid for the whole process, so no ownership needed.

### Configuration files

Config readers that operate on a buffer:

```rust
fn parse_config_str(text: &str) -> HashMap<&str, &str> {
    text.lines()
        .filter_map(|line| line.split_once('='))
        .map(|(k, v)| (k.trim(), v.trim()))
        .collect()
}
```

Zero allocations — the `HashMap` stores slices of `text`. If you need to store the config beyond `text`'s lifetime, convert to `HashMap<String, String>` once at the boundary.

### Parsers and protocol decoders

Protocol parsers should always take `&[u8]` or `&str` and return typed structs that may borrow from the input:

```rust
fn parse_http_request(buf: &[u8]) -> Option<HttpRequest<'_>> {
    // returns view types into buf — no allocation
    todo!()
}
```

---

## Document ownership expectations

Even simple docs help API users:

```rust
/// Borrows input and returns slices tied to it.
/// The returned vec's elements cannot outlive `s`.
fn words(s: &str) -> Vec<&str> { ... }

/// Allocates and returns a new normalized `String`.
fn normalize(s: &str) -> String { ... }
```

---

## Challenges and extensions

1. **The builder pattern**: how do builder-pattern APIs (e.g., `Command::new("ls").arg("-la")`) handle ownership? Look at `std::process::Command` and trace whether each method borrows or owns.

2. **`Into<String>` parameter**: some APIs accept `impl Into<String>` instead of `AsRef<str>`. When would you prefer this? What does the call site look like?

3. **`Cow` in the standard library**: find one example of `Cow` usage in std. Why is it used there?

4. **Benchmark**: write both owned and borrowed versions of a string pipeline and use `std::hint::black_box` in a benchmark to compare allocation counts.

---

## Common mistakes

1. Taking ownership unnecessarily (`String`, `Vec<T>` parameters everywhere).
2. Returning `&str` to data that won't live long enough.
3. Introducing `Cow` without a performance reason.
4. Adding `AsRef<str>` generics "just in case" (clutters signatures and error messages).

---

## Exercises

1. Refactor these three functions to use borrowed inputs and appropriate return types:
   - `fn join(a: String, b: String) -> String`
   - `fn find_header(headers: Vec<String>, name: String) -> Option<String>`
   - `fn is_prefix(a: String, b: String) -> bool`

2. Change a function that returns `String` to return `&str` if it only returns a substring of its input.

3. Implement `trim_cow` and write tests verifying that identical input returns `Cow::Borrowed` and changed input returns `Cow::Owned`.

4. (**harder**) Design an API for a "config updater" that takes an existing config `&str`, a key, and a new value, and returns the updated config as a `Cow<str>` — borrowing the original if no key was found, or returning a new string if the key was updated.

---

## References

- [Rust API Guidelines: Input parameters](https://rust-lang.github.io/api-guidelines/flexibility.html)
- [`std::borrow::Cow` docs](https://doc.rust-lang.org/std/borrow/enum.Cow.html)
- [The Rust Book: Traits as Parameters](https://doc.rust-lang.org/book/ch10-02-traits.html)

## Recap

- Borrow inputs by default (`&str`, `&[T]`).
- Own outputs when you create data; borrow when you return a view.
- Clone at storage boundaries, not at every call site.
- `AsRef` and `Cow` are tools for specific ergonomics/performance needs, not defaults.
