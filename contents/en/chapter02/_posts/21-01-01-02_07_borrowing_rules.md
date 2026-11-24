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

Imagine handing your friend a sticky note pointing to element #0 of a `Vec<i32>` sitting on the heap, and then pushing a new element into that same `Vec`. If the vector is full, it allocates a brand-new, larger buffer elsewhere on the heap and frees the old one—instantly turning your friend’s sticky note into a **dangling pointer**! In C++, that bug (*iterator invalidation*) compiles cleanly and crashes unpredictably in production.

Rust’s **borrow checker** prevents that entire bug class—along with multi-threaded **data races**—with a single, astonishingly simple compile-time law: **at any instant, you may have either any number of shared readers (`&T`) OR exactly one exclusive writer (`&mut T`), never both.** This lecture traces how the borrow checker tracks borrow lifetimes (including Non-Lexical Lifetimes — NLL) so you stop fighting error `E0502` and start reading it like an X-ray of your memory.

---

## Background Knowledge Map: Aliasing XOR Mutability (The Heart of Rust)

Every borrow-checker error you will ever encounter traces back to one foundational invariant:

1. **Aliasing XOR Mutability (`&T` vs. `&mut T`):** Bugs happen only when **two things occur simultaneously**: *aliasing* (two active paths reaching the same memory) AND *mutation* (one of them writing or reallocating that memory). Rust forbids the intersection: `&T` allows unlimited aliasing with zero mutation; `&mut T` allows mutation with zero aliasing.
2. **Non-Lexical Lifetimes (NLL):** Modern Rust does *not* hold a borrow open until the closing `}` of a block. A borrow begins where the reference is created and **ends immediately after its last use**—letting you read via `&s` and then cleanly mutate via `&mut s` in the very same scope as long as their live spans do not overlap.
3. **Preventing Dangling References:** The compiler guarantees every reference’s lifetime is strictly contained within the owner’s scope, making use-after-free structurally impossible.
4. **Same Law, Single-Threaded and Multi-Threaded:** The exact rule that stops `Vec` iterator invalidation on one thread also stops unsynchronized read/write **data races** across CPU cores.

---

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

---

### Worked Line-by-Line Trace (Borrow Spans & NLL): Why One Line Moves `E0502` From Error to Valid

Trace the active borrow spans on a heap-allocated `Vec<i32>` across two nearly identical snippets:

```rust
fn main() {
    let mut v = vec![10, 20, 30]; // Heap@0x300: [10, 20, 30] (len=3, cap=3)
    let first: &i32 = &v[0];      // Shared borrow `&v` STARTS (points into Heap@0x300)
    println!("first = {first}");  // LAST USE of `first` -> Shared borrow ENDS here (NLL)!

    v.push(40);                   // Exclusive `&mut v` STARTS: reallocates Heap@0x300 -> 0x500!
    println!("v = {v:?}");        // OK! `first` was never used after reallocation.
}
```

Now see what happens if you move `println!("first = {first}");` **below** `v.push(40);`:

| Line | Active Borrows | Heap Buffer of `v` | Borrow-Checker Verdict |
| :--- | :--- | :--- | :--- |
| `let first = &v[0];` | Shared `&v` active (`first` $$\to$$ `0x300`) | `0x300` (`len=3, cap=3`) | OK |
| `v.push(40);` | Needs `&mut v` while `first` is STILL needed later! | Would free `0x300` and move to `0x500`! | **REJECTED (`E0502`)**—prevents dangling pointer! |
| `println!("{first}");` | Would dereference freed `0x300`! | Freed memory | Never reaches runtime! |

### Deep-Dive Explanation: Reading Borrow-Checker Errors Without Guessing (Read Slowly)

1. **Draw two vertical brackets in your margin:** One bracket from where `&v` starts to its *last use*; a second bracket for `&mut v`. If those two brackets overlap on any line, `rustc` will reject the code with `E0502`.
2. **Remember *why* the compiler rejected it:** In our `Vec` trace, `v.push(40)` really *could* free `0x300` and leave `first` pointing at garbage! The borrow checker isn’t being pedantic—it just saved you from a real memory-corruption bug.
3. **Three clean fixes when brackets overlap:** (a) Finish using the shared borrow before taking `&mut`; (b) copy a small `Copy` value out (`let first_val = v[0];` instead of `&v[0]`) so no borrow exists at all; or (c) restructure the function so read and write phases are separated.
4. **Disjoint struct fields can be borrowed independently:** `rustc` understands field-level borrows—you can hold `&mut player.hp` and `&player.name` at the same time as long as you borrow the fields directly rather than passing `&mut player` into a method.

**Quick check.** In the `Vec` snippet above, if you change `let first: &i32 = &v[0];` to `let first: i32 = v[0];`, can you now safely call `v.push(40);` and *then* `println!("{first}");`—and why?

---

## Exercises

1. Extend `Stats` to track `sum_sq` and compute variance (two-pass or streaming approximation).
2. Write `fn reset(stats: &mut Stats)`.
3. Create a function that prints a formatted report from `&Stats`.

## Recap

- Many readers or one writer.
- Borrow scopes matter; end borrows early.
- Fix borrow checker errors by restructuring, not by cloning.
- Use `&mut` for mutation, `&` for reading.

## Video tham khảo

- [Understand Rust's Borrow Checker in 5 Minutes](https://www.youtube.com/watch?v=Nuba5LNy5cY) — notgull, English. **xem trước.** Watch first: one shared-or-mutable rule, in a few minutes, before the longer walkthroughs.
- [Learn these 3 RULES to master Borrowing! | Rust Fundamentals Part 6](https://www.youtube.com/watch?v=8se_9NtF4wQ) — Green Tea Coding, English. The shared reference, mutable reference, and aliasing cases the mini-lab will hit.
- [How to fight Rust's borrow checker... and win.](https://www.youtube.com/watch?v=Pg07HQJ0tvI) — Let's Get Rusty, English. How to read a borrow-checker error and change the code instead of cloning past it.
