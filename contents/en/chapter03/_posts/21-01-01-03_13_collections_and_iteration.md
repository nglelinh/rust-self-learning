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

Collections are where ownership, borrowing, and iteration meet in day-to-day Rust. This lesson teaches the most-used collections, how to iterate correctly, and how to avoid borrow checker issues when updating maps.

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
