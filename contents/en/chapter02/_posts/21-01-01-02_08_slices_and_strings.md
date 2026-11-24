---
layout: post
title: "02-08 Slices and Strings"
chapter: "02"
order: 8
owner: "OpenCode"
lang: en
categories:
  - chapter02
lesson_type: required
---

Every Rust beginner eventually hits the same rite of passage: writing `let ch = s[0];` on a `String` and watching the compiler flatly refuse! Why would a modern language forbid indexing a string? Because real-world text is **UTF-8**, where `'a'` takes 1 byte, `'é'` takes 2 bytes, `'ệ'` takes 3 bytes, and `'🦀'` takes 4 bytes—so `s[0]` is either an O(1) broken half-character byte or a deceptive O(n) hidden scan.

This lecture reveals the clean memory layout behind Rust’s **slices (`&[T]` and `&str`)**—16-byte *fat pointers* `(ptr, len)` that borrow contiguous views with zero heap allocation—and explains why accepting `&str` and `&[T]` in function parameters makes your APIs dramatically more flexible and faster.

---

## Background Knowledge Map: Fat Pointers, Deref Coercion, and UTF-8 Invariants

Four systems concepts unlock slices and strings permanently:

1. **Owned Buffer (24 bytes) vs. Borrowed Slice Fat Pointer (16 bytes):** On a 64-bit machine, `Vec<T>` and `String` store `(ptr, len, cap)` (24 bytes) on the stack and *own* a heap buffer. A slice (`&[T]` or `&str`) stores only `(ptr, len)` (16 bytes)—a zero-copy borrowed window into *any* contiguous memory (heap `Vec`, stack array `[T; N]`, or static binary `.rodata` literal!).
2. **Deref Coercion:** Because `String` implements `Deref<Target = str>` and `Vec<T>` implements `Deref<Target = [T]>`, passing `&my_string` to a function expecting `&str` works automatically with zero runtime cost.
3. **UTF-8 Validity Invariant:** Both `String` and `&str` guarantee valid UTF-8 bytes at all times. Indexing by byte offset without checking character boundaries could split a multi-byte character in half, so Rust forces you to choose explicitly between `.bytes()` (raw `u8`) and `.chars()` (Unicode scalar values).
4. **Borrow-Checked Zero-Copy Parsing:** Because a slice `&str` borrows from its parent `String`, the borrow checker guarantees the parent string cannot be mutated or freed while any parsed token slice is still alive.

---

## 60-minute teaching plan

- 0 to 10 min: `Vec<T>` vs `&[T]` and why slices exist.
- 10 to 30 min: `String` vs `&str`: ownership and borrowing.
- 30 to 45 min: UTF-8 and why indexing strings is not allowed.
- 45 to 60 min: Mini-lab: implement `split_words` and `take_chars` with tests.

## Learning goals

By the end of this lesson, students can:

- Explain the difference between owned and borrowed string data.
- Prefer `&str`/`&[T]` in API inputs.
- Avoid invalid string slicing by bytes.
- Use iterators (`chars`, `bytes`, `split_whitespace`) intentionally.

## Slices: borrowed views into data

### `Vec<T>` vs `&[T]`

- `Vec<T>` owns a growable buffer.
- `&[T]` is a borrowed view: pointer + length.

Prefer slice inputs:

```rust
fn sum(xs: &[i32]) -> i32 {
    xs.iter().sum()
}

fn main() {
    let v = vec![1, 2, 3];
    println!("{}", sum(&v));
    println!("{}", sum(&v[0..2]));
}
```

Teaching point: slice inputs accept arrays, vectors, and sub-slices.

## Strings: `String` vs `&str`

### `String` owns data

```rust
let s = String::from("hello");
```

### `&str` borrows string data

```rust
fn shout(s: &str) -> String {
    s.to_uppercase()
}

fn main() {
    let s = String::from("hello");
    println!("{}", shout(&s));
    println!("{}", shout("borrowed literal"));
}
```

Teaching point: accepting `&str` makes the function usable with both owned strings and string literals.

### When should you return `String`?

Return `String` when you create new text (concatenation, case change, trimming to new allocation). Return `&str` only when you are returning a view into an existing string.

## UTF-8: why `s[0]` is not allowed

Rust strings are UTF-8. A "character" may be multiple bytes.

Example:

```rust
let s = "é"; // two bytes in UTF-8
println!("bytes={:?}", s.as_bytes());
```

Because of variable-width encoding, indexing by byte is unsafe for characters. Rust forbids `s[0]` so you don’t accidentally cut a character in half.

## Iterating over a string

Choose based on what you mean:

- `s.bytes()` for raw bytes
- `s.chars()` for Unicode scalar values
- `s.split_whitespace()` for word-ish tokens

Example:

```rust
let s = "hi é";
for c in s.chars() {
    println!("{c}");
}
```

Teaching point: `chars()` is not "grapheme clusters" (what users perceive as a character), but it is the right level for many tasks.

## Mini-lab 1: `split_words`

Goal: return borrowed words as `&str` slices.

```rust
pub fn split_words(s: &str) -> Vec<&str> {
    s.split_whitespace().collect()
}
```

Tests:

```rust
#[test]
fn split_words_basic() {
    assert_eq!(split_words("a  b\n c"), vec!["a", "b", "c"]);
}
```

Teaching point: the returned `&str` slices borrow from the input string; they cannot outlive it.

## Mini-lab 2: take first N characters safely

Goal: implement `take_chars(s, n)` that returns a borrowed slice `&str` if possible.

We can do this by finding the byte index at a valid char boundary:

```rust
pub fn take_chars(s: &str, n: usize) -> &str {
    if n == 0 {
        return "";
    }

    let mut end = s.len();
    let mut count = 0usize;

    for (i, _) in s.char_indices() {
        if count == n {
            end = i;
            break;
        }
        count += 1;
    }

    if count < n {
        // fewer than n chars; return whole string
        s
    } else {
        &s[..end]
    }
}
```

Tests:

```rust
#[test]
fn take_chars_ascii() {
    assert_eq!(take_chars("hello", 2), "he");
}

#[test]
fn take_chars_utf8() {
    assert_eq!(take_chars("héllo", 2), "hé");
}

#[test]
fn take_chars_longer_than_string() {
    assert_eq!(take_chars("hi", 10), "hi");
}
```

Instructor note: there are multiple correct implementations; the key is to avoid slicing at arbitrary byte indices.

## Common mistakes

1. Returning `&str` that points to a temporary `String`:
   - If you allocate a new `String` inside a function, you cannot return `&str` referencing it.
2. Using `as_bytes()[0]` and assuming it’s a character.
3. Calling `.to_string()` everywhere instead of borrowing.

---

### Worked Line-by-Line Trace (Memory Layout & UTF-8 Bytes): Why `s[0]` Is Forbidden

Inspect the exact byte layout of `let s = String::from("aé🦀");` in UTF-8:

```rust
fn main() {
    let s = String::from("aé🦀"); // Stack: (ptr=0x400, len=7, cap=7)
    // Heap@0x400 (7 bytes total!):
    //   byte 0:       0x61             -> 'a' (1 byte)
    //   bytes 1..3:   0xC3, 0xA9       -> 'é' (2 bytes)
    //   bytes 3..7:   0xF0,0x9F,0xA6,0x80 -> '🦀' (4 bytes)

    let view: &str = &s[1..3];    // Stack: (ptr=0x401, len=2) -> "é" (ZERO heap allocation!)
    // let bad = &s[1..2];        // PANIC at runtime: byte index 2 is inside 'é'!
    println!("len={} bytes, chars={}, view={view}", s.len(), s.chars().count());
}
```

| Expression | Stack Size | Heap Allocation? | Result |
| :--- | :--- | :--- | :--- |
| `s: String` | 24 bytes `(0x400, 7, 7)` | 7 bytes on heap | Owns `"aé🦀"` |
| `view: &str = &s[1..3]` | 16 bytes `(0x401, 2)` | **None (0 bytes)** | Borrows `"é"` directly inside `s` |
| `s.len()` | — | — | `7` (always **bytes**, O(1)) |
| `s.chars().count()` | — | — | `3` (scans UTF-8 codepoints, O(n)) |

### Deep-Dive Explanation: Four Rules for Slices and Strings in Real Rust (Read Slowly)

1. **Accept `&str` and `&[T]` in function parameters, not `&String` or `&Vec<T>`:** A function taking `&str` accepts string literals (`"hi"`), full `&String`s, and sub-slices (`&s[1..3]`) seamlessly—zero extra allocations.
2. **Return owned `String` / `Vec<T>` when creating new data:** You can only return `&str` if it borrows from an input parameter or is `'static`; if your function constructs brand-new text, return `String`.
3. **Never slice arbitrary user text by raw integer indices:** Use `.chars()`, `.split(...)`, `.strip_prefix(...)`, or `.get(a..b)` (which returns `Option<&str>` instead of panicking on a mid-character boundary).
4. **Slices lock the backing buffer:** While `view = &s[1..3]` is still live, `rustc` forbids `s.clear()` or `s.push_str(...)` because mutating `s` could invalidate `view`’s pointer `0x401`.

**Quick check.** On a 64-bit machine, how many bytes does `&s[0..1]` occupy on the stack, and why does `s.len()` return `7` for `"aé🦀"` even though there are only 3 visible characters?

---

## Exercises

1. Implement `last_char(s: &str) -> Option<char>`.
2. Implement `trim_prefix<'a>(s: &'a str, prefix: &str) -> &'a str`.
3. Write `fn join_with_comma(parts: &[&str]) -> String`.

## Recap

- Prefer slice inputs for flexibility and performance.
- Use `&str` for borrowed text, `String` for owned text.
- UTF-8 means you must not index strings by byte.
- Use `char_indices` to build safe slicing helpers.

## Video tham khảo

- [All Rust string types explained](https://www.youtube.com/watch?v=CpvzeyzgQdw) — Let's Get Rusty, English. **xem trước.** Watch first for String versus &str and the other string types the lesson separates.
- [Using slices in Rust is pretty powerful!](https://www.youtube.com/watch?v=a4o-kR2ixGc) — Flo Woelki, English. Slices as borrowed views, which is the first half of the lesson before UTF-8 indexing.
- [Mastering Strings and UTF-8 in Rust - Full Crash Rust Tutorial for Beginners](https://www.youtube.com/watch?v=GK9Iz_ihmV8) — Francesco Ciulla, English. Why s[0] is rejected and how to walk scalars and characters the way the labs do.
