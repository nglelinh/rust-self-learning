---
layout: post
title: "02-07 Borrowing Rules"
chapter: "02"
order: 7
owner: "OpenCode"
lang: en
categories:
  - chapter02
lesson_type: required
---

Borrowing is how Rust lets you share access safely, while preventing data races and invalid references. This lesson teaches the borrowing rules, how to interpret borrow checker errors, and the standard refactor patterns to fix them.

## 60-minute teaching plan

- 0 to 10 min: Why borrowing exists (use-after-free prevention) and the two reference types.
- 10 to 25 min: Shared borrows (`&T`) and multiple readers.
- 25 to 40 min: Mutable borrows (`&mut T`) and exclusive access.
- 40 to 55 min: Refactor patterns: shrink scopes, split work, and avoid holding borrows.
- 55 to 60 min: Exercise: streaming stats update.

## Learning goals

By the end of this lesson, students can:

- Explain the rule: many readers OR one writer.
- Choose between `&T` and `&mut T` based on intent.
- Recognize what a borrow scope is and how it affects compilation.
- Apply common refactor patterns to satisfy the borrow checker without cloning.

## The borrowing rule (learn this verbatim)

At any point in time, for a given value, you can have:

- Any number of immutable references (`&T`), or
- Exactly one mutable reference (`&mut T`).

And references must always be valid (they cannot outlive the value they point to).

## Shared references: `&T`

Shared references allow read-only access.

```rust
fn len(s: &String) -> usize {
    s.len()
}

fn main() {
    let s = String::from("hello");
    let a = &s;
    let b = &s;
    println!("{} {}", a.len(), b.len());
}
```

Teaching points:

- Multiple `&s` is fine.
- The compiler knows these cannot mutate `s`.

## Mutable references: `&mut T`

Mutable references require exclusive access.

```rust
fn push_exclaim(s: &mut String) {
    s.push('!');
}

fn main() {
    let mut s = String::from("hello");
    push_exclaim(&mut s);
    println!("{s}");
}
```

Teaching points:

- You need `let mut s = ...` to take a mutable reference.
- Exclusive access is what prevents data races.

## The classic error: mixing shared + mutable

Show the failing code and have students predict why:

```rust
fn main() {
    let mut s = String::from("hello");
    let r = &s;
    // s is borrowed immutably by r
    let m = &mut s; // error
    println!("{}", r.len());
    m.push('!');
}
```

The fix is to end the immutable borrow before creating the mutable one.

### Fix pattern 1: shrink the borrow scope

```rust
fn main() {
    let mut s = String::from("hello");

    let n = s.len();
    println!("len={n}");

    s.push('!');
    println!("{s}");
}
```

Teaching point: store the data you need (like `len`) instead of holding a reference longer than necessary.

### Fix pattern 2: use a block

```rust
fn main() {
    let mut s = String::from("hello");

    {
        let r = &s;
        println!("{}", r.len());
    } // r ends here

    let m = &mut s;
    m.push('!');
}
```

## Borrow scopes in loops

Borrow scopes can accidentally span longer than expected.

Example: borrowing an element, then trying to modify the vector.

```rust
let mut v = vec![1, 2, 3];
let first = &v[0];
// v.push(4); // error: cannot borrow `v` as mutable because it is also borrowed as immutable
println!("{first}");
```

Teaching point: an element reference ties the whole collection.

Fix: copy the value instead of holding a reference:

```rust
let mut v = vec![1, 2, 3];
let first = v[0];
v.push(4);
println!("{first}");
```

## Reborrowing (useful but optional)

When you have `&mut T`, you can temporarily take `&T` or another `&mut T` from it, as long as scopes don’t overlap.

```rust
fn bump(x: &mut i32) {
    *x += 1;
}

fn main() {
    let mut n = 0;
    let r = &mut n;
    bump(r);
    bump(r);
}
```

Keep this idea simple: "a mutable reference is exclusive, but you can pass it around." The full model comes later.

## Mini-lab: streaming stats updater

We will update running stats without cloning data.

### Model

```rust
#[derive(Debug, Default)]
struct Stats {
    count: u64,
    sum: f64,
    min: f64,
    max: f64,
}
```

### Update function (takes `&mut Stats`)

```rust
fn update(stats: &mut Stats, x: f64) {
    if stats.count == 0 {
        stats.min = x;
        stats.max = x;
    } else {
        if x < stats.min {
            stats.min = x;
        }
        if x > stats.max {
            stats.max = x;
        }
    }

    stats.count += 1;
    stats.sum += x;
}

fn mean(stats: &Stats) -> Option<f64> {
    if stats.count == 0 {
        None
    } else {
        Some(stats.sum / stats.count as f64)
    }
}
```

Teaching points:

- `update` needs `&mut Stats` because it mutates.
- `mean` takes `&Stats` because it only reads.

### Example usage

```rust
fn main() {
    let mut s = Stats::default();
    for x in [1.0, 2.0, 3.0] {
        update(&mut s, x);
    }
    println!("mean={:?}", mean(&s));
}
```

## Common borrow-checker messages (teach students to map them)

- “cannot borrow `x` as mutable because it is also borrowed as immutable”
  - You have a `&x` alive when you try `&mut x`.
- “borrowed value does not live long enough”
  - You returned a reference to a local or temporary.

## Exercises

1. Extend `Stats` to track `sum_sq` and compute variance (two-pass or streaming approximation).
2. Write `fn reset(stats: &mut Stats)`.
3. Create a function that prints a formatted report from `&Stats`.

## Recap

- Many readers or one writer.
- Borrow scopes matter; end borrows early.
- Fix borrow checker errors by restructuring, not by cloning.
- Use `&mut` for mutation, `&` for reading.
