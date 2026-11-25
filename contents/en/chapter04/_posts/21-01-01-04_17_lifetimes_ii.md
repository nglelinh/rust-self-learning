---
layout: post
title: "04-17 Lifetimes II (Borrowed Structs)"
chapter: "04"
order: 17
owner: "OpenCode"
lang: en
categories:
  - chapter04
lesson_type: required
---

Imagine parsing a 2-gigabyte CSV log file with 20 million rows. If your parser copies every single column into its own heap-allocated `String`, your program performs **100 million `malloc` calls** and burns gigabytes of RAM just duplicating bytes you already have in memory! Instead, you want a **zero-copy view struct**—`struct CsvRow<'a>`—whose fields are lightweight `&'a str` slices pointing directly into the original input buffer, allocating **zero extra bytes** on the heap.

The moment you put a reference inside a `struct`, however, lifetimes become part of the type’s identity. This lesson teaches you how borrowed structs work in memory, why `struct Foo<'a>` binds the struct’s lifespan to its backing buffer, how to write clean `impl<'a>` methods, and why **self-referential structs** (storing both a `String` and a `&str` into that same `String`) are rejected by Rust’s move semantics.

---

## Key ideas: Zero-Copy Views & Memory Ownership in Structs

Three systems-programming principles explain every rule of borrowed structs.

### 1. Zero-Copy Parsing & Slice Views (`&'a [u8]` / `&'a str`)
A `&'a str` is just a 16-byte `(ptr, len)` window into an existing buffer. By storing slices inside a `struct View<'a>`, parsers (HTTP headers, JSON tokens, CSV rows) inspect gigabytes of data in place with near-zero allocation overhead.
- **Core questions:** When does zero-copy borrowing pay off massively, and when does an owned struct (`String`) produce a much simpler API?
- **Real-world reach:** `serde` zero-copy deserialization, `nom` parsers, `hyper` HTTP request parsing, and `polars` string views.

### 2. Contagious Lifetimes in Type Design
Once a struct carries `<'a>`, every outer struct containing it must also declare `<'a>`—signaling to the compiler that the entire view tree **cannot outlive** the backing buffer and cannot be moved into a `'static` background thread while borrowing stack data.
- **Core questions:** How do you decouple long-lived application state from short-lived borrowed views?
- **Real-world reach:** Separating wire-protocol parsers (borrowed) from domain entities (owned) in network services.

### 3. Move Semantics & Why Self-Referential Structs Break Safety
In Rust, moving a struct performs a shallow bitwise `memcpy` of its stack bytes to a new address. If a struct held a `String` *and* a `&str` pointing into that same `String` (or into an inline array), moving the struct could invalidate internal pointers! That is why Rust forbids safe self-referential structs—and why **byte-offset indices (`Range<usize>`)** or `Pin` are used instead.
- **Core questions:** Why does storing `(buffer: String, span: Range<usize>)` survive moves effortlessly while `(buffer: String, slice: &'a str)` fails?
- **Real-world reach:** AST token spans in `rustc`, lexer design, and `async` state-machine pinning.

## 60-minute teaching plan

- 0 to 10 min: Why borrowed structs exist (zero-copy views).
- 10 to 25 min: Define a borrowed struct with `<'a>` and explain what `'a` means.
- 25 to 40 min: Implement methods and accessors with lifetimes.
- 40 to 60 min: Mini-lab: build `CsvRow<'a>` and parse fields safely.

## Learning goals

By the end of this lesson, students can:

- Define a struct that borrows from input data (`struct X<'a> { ... }`).
- Explain how the borrowed struct cannot outlive the data it references.
- Build "view" types that avoid allocations.
- Recognize and avoid self-referential struct designs.

## Prerequisites

- Lesson 04-16 (Lifetimes I): lifetime syntax, elision, "does not live long enough".
- Lesson 02-08 (Slices and Strings): `&str` vs `String`, slice mechanics.

---

## Key concept: view types vs owned types

There are two families of types in Rust:

| | **Owned type** | **Borrowed ("view") type** |
|---|---|---|
| Example | `String`, `Vec<T>` | `&str`, `&[T]`, `CsvRow<'a>` |
| Stores | The data itself | A reference to data elsewhere |
| Lifetime | Independent | Tied to the backing store |
| Allocation | On heap | None |
| Flexibility | Can outlive its creation | Cannot outlive the source |

View types are the key to **zero-copy parsing**: you hand the parser a buffer once, and every result is a window into that buffer.

---

## Borrowed struct basics

Example: a view into a config line like `host=127.0.0.1`.

```rust
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
struct Kv<'a> {
    key:   &'a str,
    value: &'a str,
}
```

Teaching points:

- The struct holds references, not owned strings.
- `'a` ties those references to some external buffer.
- Both fields share the same lifetime `'a` — they both borrow from the same source.

### Constructor

```rust
impl<'a> Kv<'a> {
    fn parse(line: &'a str) -> Option<Self> {
        let (key, value) = line.split_once('=')?;
        Some(Self { key: key.trim(), value: value.trim() })
    }
}
```

Note: the `'a` on `impl<'a>` matches the `'a` on `Kv<'a>`. The constructor takes `line: &'a str` and returns a struct where both slices borrow from that same line.

### Usage

```rust
let config = String::from("host = 127.0.0.1");
let kv = Kv::parse(&config).unwrap();
println!("key={} value={}", kv.key, kv.value);
// kv cannot outlive config
```

---

## The key constraint

If a struct contains `&'a str`, then:

- the struct cannot live longer than `'a`
- you cannot drop the backing buffer while the struct is alive

The compiler enforces this for you. This is what makes "zero-copy parsing" safe.

### What the error looks like

```rust
let kv;
{
    let line = String::from("x=1");
    kv = Kv::parse(&line).unwrap();
}   // line is dropped here
println!("{}", kv.key);  // error: `line` does not live long enough
```

```
error[E0597]: `line` does not live long enough
  --> src/main.rs:5:19
   |
5  |     kv = Kv::parse(&line).unwrap();
   |                    ^^^^^ borrowed value does not live long enough
6  | }   // `line` dropped here while still borrowed
```

---

## Methods on borrowed structs

Methods can return borrowed slices with lifetimes tied to the struct — or to `self`. Understanding this distinction is important.

```rust
impl<'a> Kv<'a> {
    // Returns a slice that lives as long as 'a (the original buffer)
    fn key(&self) -> &'a str {
        self.key
    }

    // Also valid — tied to 'a because the field is 'a
    fn value(&self) -> &'a str {
        self.value
    }
}
```

**Why `&'a str` and not `&str` (with elision)?**

With elision, `fn key(&self) -> &str` would tie the output to `'self`, not to `'a`. That would mean the returned `&str` can only be used as long as the `Kv` struct is alive — even though the underlying buffer might outlive the struct. Explicitly using `'a` is more permissive and correct.

---

## Self-referential structs (what not to do)

You cannot safely build a struct that owns a `String` and also stores `&str` slices into itself, because moving the struct would move the `String` and invalidate the references.

```rust
// This does NOT compile:
struct Bad {
    buf:   String,
    first: &str,    // error: missing lifetime specifier
                    // and even with a lifetime, where does it come from?
}
```

Even if you tried to add a lifetime:

```rust
struct Bad<'a> {
    buf:   String,
    first: &'a str,  // 'a must come from somewhere outside Bad
}
```

This doesn't express self-reference — `'a` refers to data *outside* the struct. True self-reference is not expressible in safe Rust without `Pin` and careful unsafe.

**The solution**: keep the buffer outside the view structs, or store **indices** instead of references:

```rust
// Option 1: keep buffer separate
let buf = String::from("hello,world");
let row = CsvRow::new(&buf);   // buf outlives row

// Option 2: store indices, reconstruct slices on demand
struct CsvRowIndexed {
    fields: Vec<(usize, usize)>,  // (start, end) byte offsets
}
```

---

## Mini-lab: `CsvRow<'a>`

We will build a view over one CSV-like line (comma-separated). This is a teaching example of zero-copy parsing.

### Step 1: define the type

```rust
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
struct CsvRow<'a> {
    line: &'a str,
}
```

We store only the original line and compute views on demand.

### Step 2: constructor

```rust
impl<'a> CsvRow<'a> {
    fn new(line: &'a str) -> Self {
        Self { line }
    }
}
```

### Step 3: field accessors returning `'a` slices

```rust
impl<'a> CsvRow<'a> {
    /// Return the i-th field (trimmed), or None if index is out of range.
    fn get(&self, idx: usize) -> Option<&'a str> {
        self.line.split(',').map(|s| s.trim()).nth(idx)
    }

    /// Iterate over all fields (trimmed).
    fn iter(&self) -> impl Iterator<Item = &'a str> {
        self.line.split(',').map(|s| s.trim())
    }

    /// Count the number of fields.
    fn len(&self) -> usize {
        self.line.split(',').count()
    }

    fn is_empty(&self) -> bool {
        self.line.is_empty()
    }
}
```

Teaching points:

- `split(',')` returns borrowed slices of `self.line`, which itself borrows from `'a`.
- Return type uses `'a` explicitly so callers can use the slices as long as the original buffer lives.

### Step 4: parse typed fields with `Result`

```rust
#[derive(Debug, Clone, PartialEq, Eq)]
enum CsvError {
    MissingField { idx: usize },
    BadInt       { idx: usize },
    BadFloat     { idx: usize },
}

impl<'a> CsvRow<'a> {
    fn get_required(&self, idx: usize) -> Result<&'a str, CsvError> {
        self.get(idx).ok_or(CsvError::MissingField { idx })
    }

    fn get_i32(&self, idx: usize) -> Result<i32, CsvError> {
        let s = self.get_required(idx)?;
        s.parse::<i32>().map_err(|_| CsvError::BadInt { idx })
    }

    fn get_f64(&self, idx: usize) -> Result<f64, CsvError> {
        let s = self.get_required(idx)?;
        s.parse::<f64>().map_err(|_| CsvError::BadFloat { idx })
    }
}
```

Teaching point: this is where `Option` (missing field) becomes a `Result` (typed error with context). The `?` operator threads the error through.

### Step 5: tests

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn csv_row_get() {
        let row = CsvRow::new(" 1, 2, 3 ");
        assert_eq!(row.get(0), Some("1"));
        assert_eq!(row.get(2), Some("3"));
        assert_eq!(row.get(3), None);
    }

    #[test]
    fn csv_row_len() {
        assert_eq!(CsvRow::new("a,b,c").len(), 3);
        assert_eq!(CsvRow::new("").len(), 1);  // split on "" gives [""]
    }

    #[test]
    fn csv_row_get_i32() {
        let row = CsvRow::new("1, x");
        assert_eq!(row.get_i32(0), Ok(1));
        assert_eq!(row.get_i32(1), Err(CsvError::BadInt { idx: 1 }));
        assert_eq!(row.get_i32(2), Err(CsvError::MissingField { idx: 2 }));
    }

    #[test]
    fn csv_row_slices_outlive_struct() {
        let line = String::from("a,b,c");
        let first;
        {
            let row = CsvRow::new(&line);
            first = row.get(0).unwrap();
        }  // row is dropped, but line is still alive
        assert_eq!(first, "a");  // ok: first borrows from line, not from row
    }
}
```

The last test demonstrates the power of `&'a str` in methods: the returned slice outlives the `CsvRow` struct itself, because it's tied to the original buffer.

---

## Code walkthrough: naive → idiomatic

### Naive: allocate everything

```rust
fn parse_record(line: &str) -> Vec<String> {
    line.split(',').map(|s| s.trim().to_string()).collect()
}
```

Every field becomes a heap allocation. For a 10-column CSV with millions of rows, that's 10 million `String` allocations just for the fields.

### Idiomatic: zero-copy view

```rust
fn parse_record(line: &str) -> CsvRow<'_> {
    CsvRow::new(line)
}
```

No allocations. Fields are slices of the input buffer. Parsed values (numbers etc.) are constructed on demand from those slices.

---

## Applications in systems programming

### Log parsing

System logs are huge. Parsing them as views avoids allocating every field:

```rust
struct LogLine<'a> {
    timestamp: &'a str,
    level:     &'a str,
    message:   &'a str,
}
```

The log file is `mmap`'d or read in large chunks. Each `LogLine` is a zero-copy view.

### Network protocol buffers

HTTP/2 frames, DNS packets, TLS records — all can be parsed as view types:

```rust
struct HttpHeader<'buf> {
    name:  &'buf [u8],
    value: &'buf [u8],
}
```

### Compiler intermediate representations

Most Rust compilers (including rustc itself) use "arenas" — large allocations that live for the whole compilation — and represent AST nodes as borrowed views into those arenas for zero-cost cloning.

---

## Design discussion: borrowed vs owned

| Situation | Prefer |
|-----------|--------|
| Parse input, process immediately | Borrowed views (`&'a str`) |
| Store parsed data beyond input buffer lifetime | Owned (`String`) |
| Return data from a function that allocated it | Owned |
| High-throughput IO parsing | Borrowed views |
| Simple business logic, readability > performance | Owned |

Rule of thumb: **borrow for processing, own for storing**.

---

## Challenges and extensions

1. **Parameterize the delimiter**: add `delim: char` to `CsvRow` and make all methods use it. What happens to the `Copy` derive?

2. **Iterator implementation**: implement `IntoIterator` for `CsvRow<'a>` so `for field in row { ... }` works.

3. **Header row**: build `CsvTable<'a>` that owns a header row (`CsvRow<'a>`) and a `Vec<CsvRow<'a>>` body. Write `get_column_by_name`.

4. **Lifetime subtyping**: read about `'a: 'b` (lifetime `'a` outlives `'b`) and explain when you'd need it in a view type hierarchy.

---

---

## Worked Examples & Deep Dive

### Worked Example 1: Memory Cost of Owned `CsvRow` vs. Zero-Copy `CsvRow<'a>`

Suppose a single CSV line `"alice,2026-11-24,checkout,49.99"` (31 bytes) is parsed into 4 fields:

| Representation | Stack Size (64-bit) | Extra Heap Allocations | Extra Heap Bytes |
|---|---|---|---|
| Owned `struct OwnedRow { user: String, date: String, action: String, price: String }` | `4 × 24 = 96 bytes` | **4 `malloc` calls** | **31+ bytes** (plus allocator metadata!) |
| Borrowed `struct CsvRow<'a> { user: &'a str, date: &'a str, action: &'a str, price: &'a str }` | `4 × 16 = 64 bytes` | **0 `malloc` calls!** | **0 bytes!** |

Over 10 million rows, zero-copy `CsvRow<'a>` eliminates **40 million heap allocations** while guaranteeing at compile time that no `CsvRow<'a>` can ever outlive the backing buffer!

### Worked Example 2: Tracing Why a Self-Referential Struct Fails—and the Index Fix

Suppose you try to bundle a buffer and a slice into it:

```rust
// BROKEN: self-referential!
struct BadDoc<'a> {
    text: String,        // owns heap buffer
    title: &'a str,      // wants to point into self.text!
}
```
1. If you create `doc` inside `fn load() -> BadDoc<'a>`, returning `doc` **moves** `doc` out of `load`’s stack frame. Even though `String`’s heap bytes stay put, Rust’s borrow checker forbids moving a value *while it is borrowed* (`title` borrows `text`!).
2. **The clean systems fix — store byte offsets (`Range<usize>`):**
```rust
struct GoodDoc {
    text: String,
    title_span: std::ops::Range<usize>, // just 2 usizes (16 bytes), no lifetime!
}
impl GoodDoc {
    fn title(&self) -> &str { &self.text[self.title_span.clone()] }
}
```
`GoodDoc` has **zero lifetime parameters**, moves freely across functions and threads, and still gives zero-copy `&str` access on demand!

### Explanation: Designing Clean Borrowed Structs Without Lifetime Hell

#### Step 1: Read `struct View<'a>` as “`View` is tied to an external owner alive for `'a`”
A lifetime parameter on a struct is a neon warning sign: *this struct does not own its data*. Keep borrowed structs short-lived—typically inside a parsing loop, request handler, or single function call.

#### Step 2: Watch the `impl<'a>` method elision trap!
Inside `impl<'a> CsvRow<'a>`, writing `fn first(&self) -> &str` ties the returned `&str` to `&self` (the borrow of the `CsvRow`), **not** to `'a` (the underlying buffer)! If you want the returned slice to outlive a temporary `CsvRow<'a>`, explicitly write `fn first(&self) -> &'a str`.

#### Step 3: Never fight the compiler to build a self-referential struct
Whenever you want a struct to own a buffer *and* hold parsed views into it, reach immediately for **byte-offset spans** (`Range<usize>`), **arena/index handles**, or **two-stage ownership** (caller owns the `String` buffer; parser produces `View<'a>`).

#### Step 4: Convert to owned at architectural boundaries
Parse hot wire formats with `View<'a>`, filter/validate in zero-copy mode, and convert only the surviving records into owned domain structs (`to_owned()`) before storing them in caches, channels, or long-lived application state.

## Exercises

1. Add `fn len(&self) -> usize` returning number of fields.
2. Add `fn get_required(&self, idx: usize) -> Result<&'a str, CsvError>`.
3. Implement `CsvRow` for tab-separated values by parameterizing the delimiter.
4. (**harder**) Build a `LineIter<'a>` that iterates over `CsvRow<'a>` values for each line in a multi-line string — without any `String` allocations.

---

## References

- [The Rust Book, Ch. 10.3 — Lifetime Syntax](https://doc.rust-lang.org/book/ch10-03-lifetime-syntax.html)
- [Rustonomicon: Lifetime Elision](https://doc.rust-lang.org/nomicon/lifetime-elision.html)
- [Rust Reference: Lifetime Elision](https://doc.rust-lang.org/reference/lifetime-elision.html)

## Recap

- Borrowed structs enable zero-copy views over external buffers.
- Lifetimes on structs express: "this view cannot outlive the data."
- Explicitly returning `&'a str` (not `&str` via elision) can be more permissive.
- Avoid self-referential structs; keep buffers separate from views, or store indices.
- Convert `Option` to `Result` when you need actionable errors.

## Video tham khảo

- [Intro to Embedded Rust Part 8: Lifetimes and Lifetime Annotations | DigiKey](https://www.youtube.com/watch?v=3hzrRgXeNuk) — DigiKey, English. **xem trước.** Watch first: lifetime annotations on structs and functions, close to the CsvRow lab.
- [but what is 'a lifetime?](https://www.youtube.com/watch?v=gRAVZv7V91Q) — leddoo, English. Why a borrowed field ties the struct to someone else’s data — view type versus owned type.
- [A new tool for visualizing Rust lifetimes](https://www.youtube.com/watch?v=NV6Xo_el_2o) — Let's Get Rusty, English. A short look at a lifetime visualizer you can use when a borrowed struct stops compiling.
