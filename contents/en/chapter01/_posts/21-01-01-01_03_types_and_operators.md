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

In 1996, the Ariane 5 rocket self-destructed 37 seconds after liftoff—a $370 million loss triggered by a 64-bit floating-point horizontal velocity being narrowed into a 16-bit signed integer and silently overflowing. Where C and C++ allow implicit numeric conversions and treat signed overflow as *undefined behavior*, Rust takes an uncompromising stance: **zero implicit numeric coercions**, explicit bit widths, and first-class APIs for handling overflow and fallible conversions.

This lecture maps Rust’s primitive numeric types (`i8`–`i128`, `u8`–`u128`, `usize`, `f32`/`f64`), shows how local type inference keeps code concise without sacrificing static safety, and compares the three conversion buckets (`parse`, `From`/`TryFrom`, and `as`).

---

## Key ideas: Bit-Exact Types and Overflow Semantics in Rust

Four systems-level guarantees underpin every arithmetic expression in Rust:

1. **Fixed Bit Widths & Architecture-Sized Indices:** An `i32` is *always* 32 bits (4 bytes) on every CPU; `usize` and `isize` match pointer width (8 bytes on 64-bit machines) and are required for indexing slices and vectors.
2. **No Implicit Numeric Widening or Narrowing:** Adding an `i32` and an `i64`—or an `i32` and an `f64`—is a compile-time error until you convert explicitly, preventing accidental precision loss or sign bugs.
3. **Explicit Overflow Policies:** Debug builds panic on overflow while release builds wrap in two’s complement. Whenever overflow is possible in domain logic, Rust provides three explicit method families: `checked_*` (returns `Option`), `saturating_*` (clamps to min/max), and `wrapping_*` (modular arithmetic $$\bmod 2^N$$).
4. **Lossless (`From`/`Into`) vs. Fallible (`TryFrom`) vs. Truncating (`as`):** Prefer `From` when widening safely (`u8` $$	o$$ `u32`), `TryFrom` when narrowing (`i64` $$	o$$ `u8`), and reserve `as` only when bit-truncation is explicitly intended.

---

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

---

### Worked Line-by-Line Trace (Bit Representation & Overflow): What Happens to `250u8 + 10u8`?

An unsigned 8-bit integer (`u8`) holds values in $$[0, 2^8 - 1] = [0, 255]$$. Trace what happens when adding `a = 250u8` (`0b1111_1010`) and `b = 10u8` (`0b0000_1010`), whose mathematical sum is $$260 = 256 + 4$$:

```rust
let a: u8 = 250;
let b: u8 = 10;

let c1 = a.checked_add(b);      // None (overflow caught cleanly!)
let c2 = a.saturating_add(b);   // 255  (clamped at ceiling—great for RGB pixels/audio)
let c3 = a.wrapping_add(b);     // 4    (260 mod 256 = 4—great for hashes/checksums)
let c4 = 300_i32 as u8;         // 44   (TRAP: 300 mod 256 = 44; `as` silently truncates!)
let c5 = u8::try_from(300_i32); // Err(...) (SAFE: forces you to handle out-of-range!)
```

| Expression | Value Returned | Return Type | Safety Profile |
| :--- | :--- | :--- | :--- |
| `250u8.checked_add(10)` | `None` | `Option<u8>` | Safest—explicit failure signal |
| `250u8.saturating_add(10)` | `255` | `u8` | Safe clamping at type bounds |
| `250u8.wrapping_add(10)` | `4` | `u8` | Explicit modular arithmetic |
| `300_i32 as u8` | `44` | `u8` | Hazardous—silently drops high bits |
| `u8::try_from(300_i32)` | `Err(...)` | `Result<u8, ...>` | Safe—checked at runtime |

### Explanation: Four Rules for Bulletproof Numeric Code

1. **Replace narrowing `as` casts with `TryFrom`:** Writing `(-1_i32) as u32` silently produces `4_294_967_295` with zero compiler error. `u32::try_from(x)` turns that silent corruption into a recoverable `Result`.
2. **Never test floating-point equality with `==` after arithmetic:** In IEEE 754 binary64, `0.1 + 0.2 == 0.30000000000000004`. Compare `(a - b).abs() < 1e-9` instead.
3. **Match the overflow method to the domain:** Financial counters and memory sizes belong in `checked_*`; pixel brightness or audio gain belongs in `saturating_*`; cyclic ring buffers and hashes belong in `wrapping_*`.
4. **Annotate boundary literals clearly:** Writing `1_000_000_u64` or `255_u8` documents your exact intent and prevents accidental `i32` default inference.

**Quick check.** What integer value does `(-5_i32) as u8` produce in two’s complement (modulo 256), and what does `u8::try_from(-5_i32)` return instead?

---

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

## Video tham khảo

- [Rust from Zero -- Primitive number types and how to do safe arithmetic (i32, u32, usize, isize, etc)](https://www.youtube.com/watch?v=4w2tRIZbQ7E) — rvne, English. **xem trước.** Watch first for i32, u32, usize, and the checked arithmetic this lesson wants instead of silent wrap.
- [Visualizing memory layout of Rust's data types](https://www.youtube.com/watch?v=7_o-YRxf_cc) — Sreekanth, English. How those numeric types, tuples, and arrays actually sit in memory — the layout behind the type map.
