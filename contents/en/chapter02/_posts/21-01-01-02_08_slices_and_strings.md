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

Slices (`&[T]`, `&str`) are key to writing fast, ergonomic APIs that avoid copying. This lesson teaches the common string/slice types in Rust, why UTF-8 matters, and how to design slice-based function signatures.

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

## Exercises

1. Implement `last_char(s: &str) -> Option<char>`.
2. Implement `trim_prefix<'a>(s: &'a str, prefix: &str) -> &'a str`.
3. Write `fn join_with_comma(parts: &[&str]) -> String`.

## Recap

- Prefer slice inputs for flexibility and performance.
- Use `&str` for borrowed text, `String` for owned text.
- UTF-8 means you must not index strings by byte.
- Use `char_indices` to build safe slicing helpers.
