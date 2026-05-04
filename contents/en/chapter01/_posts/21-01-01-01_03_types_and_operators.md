---
layout: post
title: "01-03 Types and Operators"
chapter: "01"
order: 3
owner: "OpenCode"
lang: en
categories:
  - chapter01
lesson_type: required
---

Rust is strict about types, but the payoff is fewer hidden conversions and fewer runtime surprises. This lesson teaches the "type shape" of Rust and the safest ways to convert between types.

## 60-minute teaching plan

- 0 to 10 min: Type inference and why Rust cares.
- 10 to 25 min: Numeric types and literals: picking the right type.
- 25 to 40 min: Conversions: `as`, `try_into`, and parsing.
- 40 to 50 min: Tuples, arrays, and indexing safety.
- 50 to 60 min: Mini-lab: implement mean/variance safely and test edge cases.

## Learning goals

By the end of this lesson, students can:

- Explain Rust’s numeric type families and when to choose each.
- Predict when Rust will require an explicit conversion.
- Avoid lossy casts and accidental overflows.
- Use tuples/arrays correctly, and understand indexing panics.

## Type inference (what Rust does for you)

Rust often infers types from usage:

```rust
let x = 1;
let y = x + 2;
```

But inference has limits. If the compiler cannot determine a single type, it will ask you to annotate.

Instructor habit: when the compiler requests a type, add the smallest annotation that resolves ambiguity.

## Numeric types in Rust (the practical map)

### Integers

- Signed: `i8 i16 i32 i64 i128 isize`
- Unsigned: `u8 u16 u32 u64 u128 usize`

Rules of thumb:

- Use `i32` for most in-memory counters unless you have a reason.
- Use `usize` for indexing/sizes (because slices/vecs use it).
- Use `u64` for counts that should never be negative (e.g., bytes).

### Floats

- `f32` and `f64`

Rule of thumb:

- Default to `f64` unless you are optimizing memory/bandwidth or using GPU-oriented APIs.

### Literals

Rust literals can be annotated:

```rust
let a = 10u64;
let b = 3.14f64;
let c = 1_000_000usize;
```

Teach that annotated literals are a clean way to guide inference.

## Overflow and safety

Key behavior:

- In debug builds, many overflows panic.
- In release builds, integer overflow wraps (two’s complement) unless you use checked/saturating methods.

Safer operations exist:

```rust
let (sum, overflowed) = a.overflowing_add(b);
let sum = a.checked_add(b);
let sum = a.saturating_add(b);
```

Teaching point: if your algorithm relies on wraparound, say so explicitly. Otherwise, use checked math where it matters.

## Conversions: the three buckets

### 1. Parsing from strings

Parsing returns a `Result`:

```rust
let n: i64 = "42".parse()?;
```

If you want better error messages, handle the error and add context.

### 2. Casting with `as` (fast, but can be lossy)

`as` is a blunt tool:

```rust
let x: u8 = 300u16 as u8; // becomes 44
```

Teach: `as` is appropriate for a subset of cases:

- widening integer conversions (e.g., `u8` to `u64`)
- float-to-float conversion (`f32` <-> `f64`)

Avoid: narrowing conversions unless you have validated the range.

### 3. Fallible integer conversions (`try_into`)

Use `TryFrom`/`TryInto` to avoid silent truncation:

```rust
use std::convert::TryFrom;

let x = u8::try_from(300u16);
assert!(x.is_err());
```

In real code:

```rust
let x: u8 = value.try_into().map_err(|_| "out of range")?;
```

## Tuples and arrays

### Tuples

Tuples group different types:

```rust
let p: (i32, i32) = (10, 20);
let (x, y) = p;
```

### Arrays

Arrays are fixed-size and same-type:

```rust
let a: [i32; 3] = [1, 2, 3];
```

Indexing panics if out of bounds:

```rust
let v = vec![1, 2, 3];
// v[10] would panic
let maybe = v.get(10);
```

Teach: use `.get()` when index may be invalid.

## Worked example: mean and variance safely

We will compute mean and (population) variance for `u64` inputs. We want:

- no integer overflow when summing
- reasonable numeric stability

### Mean

Use `u128` for intermediate sum to reduce overflow risk:

```rust
pub fn mean(xs: &[u64]) -> Option<f64> {
    if xs.is_empty() {
        return None;
    }

    let sum: u128 = xs.iter().map(|&x| x as u128).sum();
    let n = xs.len() as f64;
    Some((sum as f64) / n)
}
```

Teaching points:

- Returning `Option` expresses "mean is undefined for empty input".
- We cast only after summing in a wider type.

### Variance

One stable approach is a two-pass algorithm:

```rust
pub fn variance(xs: &[u64]) -> Option<f64> {
    let m = mean(xs)?;
    let n = xs.len() as f64;
    let mut acc = 0.0f64;

    for &x in xs {
        let dx = (x as f64) - m;
        acc += dx * dx;
    }

    Some(acc / n)
}
```

Teaching points:

- Two-pass is easy to understand and sufficiently stable for many use cases.
- Later we can teach Welford’s algorithm as a streaming, numerically stable variant.

## Tests (teach students to test edge cases)

```rust
#[test]
fn mean_empty_is_none() {
    assert_eq!(mean(&[]), None);
}

#[test]
fn mean_simple() {
    assert_eq!(mean(&[1, 2, 3]).unwrap(), 2.0);
}

#[test]
fn variance_zero_for_constant() {
    assert_eq!(variance(&[5, 5, 5]).unwrap(), 0.0);
}
```

If floating comparisons become noisy later, introduce approximate comparisons. For now keep cases exact.

## Exercises

1. Change variance to sample variance (divide by `n - 1`), returning `None` for `n < 2`.
2. Implement a streaming mean (update mean one value at a time).
3. Add a function that returns `(min, max)` using iterators.
4. Replace one `as` cast with a `try_into` and propagate failure.

## Recap

- Rust won’t guess conversions for you; be explicit.
- Avoid narrowing `as` casts unless you validated the range.
- Use wider intermediate types to reduce overflow risk.
- Prefer `Option`/`Result` to represent undefined or failing computations.
