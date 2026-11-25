---
layout: post
title: "03-13 Collections and Iteration"
chapter: "03"
order: 13
owner: "OpenCode"
lang: en
categories:
  - chapter03
lesson_type: required
---

In Python or Java, looping over a list with `for x in items` hides a quiet ambiguity: are you borrowing references, mutating in place, or consuming the collection? And if you modify a collection while iterating over it, you risk a `ConcurrentModificationException` at runtime. In Rust, **collections and iterators obey the exact same ownership and borrowing rules you learned in Chapter 2**—making iterator invalidation impossible at compile time while compiling high-level iterator chains down to hand-tuned C-speed loops!

This lecture maps Rust’s three workhorse standard collections (`Vec<T>`, `HashMap<K, V>`, `BTreeMap<K, V>`), traces the exact difference between the three iteration modes (`&v`, `&mut v`, `v`), and shows how zero-cost iterator adapters and the `HashMap::entry` API write cleaner, faster code.

---

## Key ideas: Memory Complexity, Ownership in Loops, and Zero-Cost Iterators

Four systems fundamentals govern collections and iteration in Rust:

1. **Choosing the Right Data Structure (`Vec`, `HashMap`, `BTreeMap`):**
   - `Vec<T>`: contiguous heap array; O(1) indexed access & amortized O(1) push; unbeatable CPU cache locality.
   - `HashMap<K, V>`: hash table (SipHash by default for HashDoS resistance); O(1) average lookup; **unordered** iteration.
   - `BTreeMap<K, V>`: cache-friendly B-tree; O($$\log n$$) lookup; **always sorted by key** and supports range queries (`range(a..b)`).
2. **Three Iteration Modes = Three Ownership Contracts:** `for x in &v` calls `.iter()` (yields `&T`, keeps `v`); `for x in &mut v` calls `.iter_mut()` (yields `&mut T`, mutates elements in place); `for x in v` calls `.into_iter()` (yields owned `T`, **consumes `v`**!).
3. **Lazy, Zero-Cost Iterator Chains:** Adapters like `.filter()`, `.map()`, and `.take()` allocate **zero intermediate `Vec`s**—they fuse into a single tight machine-code loop only when driven by a consumer like `.collect()` or `.sum()`.
4. **The `Entry` API (Single-Lookup Mutation):** `map.entry(key).or_insert(0)` hashes `key` **once** instead of twice (unlike checking `contains_key` followed by `insert`), avoiding both redundant hashing and borrow-checker conflicts.

---

## 60-minute teaching plan

- 0 to 10 min: Picking a collection: `Vec`, `HashMap`, `BTreeMap`.
- 10 to 25 min: Iteration modes: by value vs `&T` vs `&mut T`.
- 25 to 40 min: Sorting and deterministic output.
- 40 to 60 min: Mini-lab: word frequency + top-k with tie-breaking.

## Learning goals

By the end of this lesson, students can:

- Choose between `Vec`, `HashMap`, and `BTreeMap` based on ordering and performance needs.
- Iterate in the correct ownership mode.
- Use `HashMap::entry` to update counts without borrow conflicts.
- Produce deterministic results by sorting.

## The core collections

### `Vec<T>`

Use a `Vec` for ordered, indexable sequences.

```rust
let mut v = Vec::new();
v.push(1);
v.push(2);
```

### `HashMap<K, V>`

Use for fast key lookup (unordered).

```rust
use std::collections::HashMap;

let mut m: HashMap<String, u64> = HashMap::new();
m.insert("a".to_string(), 1);
```

### `BTreeMap<K, V>`

Use when you need keys in sorted order.

```rust
use std::collections::BTreeMap;

let mut m: BTreeMap<String, u64> = BTreeMap::new();
m.insert("b".to_string(), 1);
m.insert("a".to_string(), 2);
```

Teaching point: `BTreeMap` gives deterministic iteration order by key.

## Iteration modes (the key idea)

### By reference: `for x in &v`

```rust
let v = vec![1, 2, 3];
for x in &v {
    println!("{x}");
}
// v is still usable
```

### By mutable reference: `for x in &mut v`

```rust
let mut v = vec![1, 2, 3];
for x in &mut v {
    *x += 1;
}
```

### By value: `for x in v`

```rust
let v = vec![1, 2, 3];
for x in v {
    println!("{x}");
}
// v is moved; cannot use it now
```

Teach: choose based on whether you want to keep the collection.

## Map update patterns

### The naive way (often triggers borrow issues)

```rust
// pseudo: if map contains key then update else insert
```

Teach the right way: use `entry`.

### `entry` API

```rust
use std::collections::HashMap;

fn bump(map: &mut HashMap<String, u64>, key: &str) {
    *map.entry(key.to_string()).or_insert(0) += 1;
}
```

Teaching points:

- `entry` returns a handle that lets you insert or update in one place.
- This avoids holding multiple borrows of the map.

## Deterministic output and sorting

`HashMap` iteration order is not stable. If you print directly, output will vary.

To produce deterministic output, collect into a vector and sort.

```rust
let mut items: Vec<(&String, &u64)> = map.iter().collect();
items.sort_by(|(a_word, a_count), (b_word, b_count)| {
    b_count.cmp(a_count).then_with(|| a_word.cmp(b_word))
});
```

Teaching points:

- sort by count descending
- tie-break by word ascending

## Mini-lab: word frequency + top-k

Goal: given input text, print the top-k most frequent words.

### Step 1: tokenize

Start simple (whitespace split). Later you can add punctuation handling.

```rust
fn words(s: &str) -> impl Iterator<Item = &str> {
    s.split_whitespace()
}
```

### Step 2: count

```rust
use std::collections::HashMap;

fn count_words(text: &str) -> HashMap<String, u64> {
    let mut map = HashMap::new();
    for w in words(text) {
        *map.entry(w.to_lowercase()).or_insert(0) += 1;
    }
    map
}
```

Teaching point: `.to_lowercase()` allocates; acceptable for now.

### Step 3: compute top-k

```rust
fn top_k(map: &HashMap<String, u64>, k: usize) -> Vec<(String, u64)> {
    let mut items: Vec<(String, u64)> = map.iter().map(|(w, c)| (w.clone(), *c)).collect();
    items.sort_by(|(aw, ac), (bw, bc)| bc.cmp(ac).then_with(|| aw.cmp(bw)));
    items.truncate(k);
    items
}
```

Teaching points:

- cloning is ok at the boundary for output.
- deterministic tie-breakers make tests stable.

### Step 4: tests

```rust
#[test]
fn top_k_is_deterministic() {
    let map = count_words("b a b a c");
    assert_eq!(top_k(&map, 2), vec![("a".to_string(), 2), ("b".to_string(), 2)]);
}
```

## Common mistakes

1. Printing a `HashMap` and expecting stable order.
2. Iterating by value and accidentally moving your collection.
3. Updating `HashMap` counts without `entry`.

---

### Worked Line-by-Line Trace (Ownership in Loops & `HashMap::entry`): Comparing All 3 `for` Modes

Trace ownership and heap state across all three loop forms on a `Vec<String>` and a `HashMap`:

```rust
use std::collections::HashMap;

fn main() {
    let mut words = vec![String::from("rust"), String::from("safe")];

    for w in &words {             // 1) BORROWED READ: `w: &String` (words stays owned & intact)
        println!("read: {w}");
    }
    for w in &mut words {         // 2) BORROWED WRITE: `w: &mut String` (mutates in place!)
        w.push('!');
    }
    let mut counts: HashMap<String, usize> = HashMap::new();
    for w in words {              // 3) OWNED MOVE: `w: String` (`words` is CONSUMED here!)
        *counts.entry(w).or_insert(0) += 1; // Moves `w` into map with ZERO extra `.clone()`!
    }
    // println!("{words:?}");     // COMPILE ERROR E0382: `words` was moved into the loop!
    println!("{counts:?}");       // {"rust!": 1, "safe!": 1}
}
```

| Loop Form | Trait Method Called | Type of `w` | Heap Clones? | Is `words` Usable After Loop? |
| :--- | :--- | :--- | :--- | :--- |
| `for w in &words` | `(&words).into_iter()` (`iter`) | `&String` | 0 | **Yes** |
| `for w in &mut words` | `(&mut words).into_iter()` (`iter_mut`) | `&mut String` | 0 | **Yes** (elements mutated) |
| `for w in words` | `words.into_iter()` | `String` (owned!) | 0 (moves into `counts`!) | **No** (consumed!) |

### Explanation: Four Performance & Idiom Habits for Rust Collections

1. **Default to `&v` in `for` loops unless you intentionally want to consume `v`:** Forgetting `&` in `for x in v` is one of the most common causes of unexpected `E0382` move errors for beginners.
2. **Use `entry()` for frequency counters and caches:** Writing `*counts.entry(k).or_insert(0) += 1;` performs a single hash lookup and cleanly avoids holding a simultaneous `&` and `&mut` borrow on the map.
3. **Prefer `BTreeMap` whenever deterministic sorted output matters:** `HashMap` randomizes its hash seed per process run—never rely on `HashMap` iteration order in unit tests or serialized output!
4. **Trust iterator chains over manual index loops:** `v.iter().filter(|x| **x > 0).sum()` eliminates manual bounds checks (`v[i]`) and lets LLVM auto-vectorize cleanly.

**Quick check.** In the trace above, why did moving `words` with `for w in words` let us insert each `String` into `counts` with **zero `.clone()` calls**, whereas looping with `for w in &words` would have forced `w.clone()`?

---

## Exercises

1. Strip punctuation (`. , ! ?`) before counting.
2. Add a minimum word length filter.
3. Return both total word count and unique word count.
4. Replace `HashMap` with `BTreeMap` and compare what changes.

## Recap

- Choose collections based on requirements.
- Iteration mode determines ownership.
- Use `entry` for updates.
- Sort for deterministic results and testability.

## Video tham khảo

- [Iterators in Rust](https://www.youtube.com/watch?v=4GcKrj4By8k) — Let's Get Rusty, English. **xem trước.** Watch first: iterators, adaptors, and consumers — the iteration-mode section.
- [Rust Collections Explained](https://www.youtube.com/watch?v=oiWATcjyUEI) — Code to the Moon, English. Vec, HashMap, and the other collections the word-frequency lab chooses among.
- [Crust of Rust: Iterators](https://www.youtube.com/watch?v=yozQ9C69pNs) — Jon Gjengset, English. A long Crust of Rust session on how iterators are implemented, after the primer.
