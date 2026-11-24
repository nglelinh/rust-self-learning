---
layout: post
title: "04-20 Interior Mutability"
chapter: "04"
order: 20
owner: "OpenCode"
lang: en
categories:
  - chapter04
lesson_type: required
---

Rust’s golden rule—*many shared references (`&T`) OR one exclusive mutable reference (`&mut T`), never both*—is wonderfully strict. Yet sometimes a method’s external contract really *is* logically read-only (`fn get(&self, key: &str) -> Value`), while under the hood it wants to bump a cache-hit counter or lazily memoize an expensive computation! If you change `&self` to `&mut self`, every caller holding a shared `&Cache` or `Rc<Cache>` suddenly fails to compile.

**Interior mutability** is Rust’s carefully engineered escape hatch: mutating internal data behind a shared `&T` reference while upholding memory safety through either **copy-only swaps (`Cell<T>`)** or **runtime borrow counting (`RefCell<T>`)**. This lesson reveals how `UnsafeCell<T>` anchors the entire mechanism, traces the exact runtime borrow-state integer inside `RefCell`, and teaches you how to keep borrow guards tiny so your code never panics.

---

## Background Knowledge Map: Compile-Time vs. Runtime Aliasing Enforcement

### 1. Shared XOR Mutable & `UnsafeCell<T>`
LLVM assumes that memory behind a plain `&T` never mutates while borrowed—enabling aggressive register caching. **`UnsafeCell<T>`** is the *sole* primitive in Rust that tells the compiler “bytes inside here may change through `&self`.” Every interior-mutability type (`Cell`, `RefCell`, `Mutex`, `RwLock`, `AtomicUsize`) is built directly on top of `UnsafeCell<T>`.
- **Core questions:** Why is casting `&T` to `&mut T` without `UnsafeCell` instant Undefined Behavior?
- **Real-world reach:** Memoization caches, mock objects in tests, reference-counted graphs, and lock primitives.

### 2. Zero-Overhead Copy Mutation (`Cell<T>`)
`Cell<T>` never hands out a reference (`&T` or `&mut T`) to its inner value! Instead, it **copies** values out (`get()`) and **overwrites** values in (`set(val)`). Because no reference can ever dangle inside a single thread (`!Sync`), `Cell<T>` needs **zero runtime borrow flags** and **cannot panic**!
- **Core questions:** Why is `Cell<usize>` strictly better than `RefCell<usize>` for counters and flags?
- **Real-world reach:** Hit counters, dirty flags, `Rc` internal refcounts, and state flags.

### 3. Runtime Borrow Checking (`RefCell<T>`)
When `T` is non-`Copy` (like a `HashMap` or `Vec`), you need temporary references inside `Cell`—so `RefCell<T>` moves Rust’s borrow checker from **compile time to runtime**, tracking active readers/writers in a `Cell<isize>` counter and panicking if you violate aliasing rules dynamically.
- **Core questions:** How do `Ref<'_, T>` and `RefMut<'_, T>` RAII guards automatically decrement the borrow flag on drop?
- **Real-world reach:** Single-threaded caches, graph traversal visited sets, and GUI widget state (`Rc<RefCell<T>>`).

## 60-minute teaching plan

- 0 to 10 min: Why interior mutability exists (API ergonomics + shared ownership).
- 10 to 25 min: `Cell<T>`: copy-in/copy-out mutation.
- 25 to 40 min: `RefCell<T>`: runtime borrow checking, `borrow()` vs `borrow_mut()`.
- 40 to 60 min: Mini-lab: build a small cache with controlled access.

## Learning goals

By the end of this lesson, students can:

- Explain what interior mutability means.
- Choose between `Cell` and `RefCell`.
- Avoid runtime borrow panics by keeping borrows short.
- Build a cache API that prevents misuse.

## Prerequisites

- Lesson 02-07 (Borrowing Rules): shared vs exclusive borrows.
- Lesson 04-19 (Smart Pointers): `Rc<T>` for shared ownership context.

---

## Key concept: the borrow checker's limitations

The borrow checker is conservative. It rejects some programs that are *actually* safe, because it reasons about the structure of code, not runtime values.

A common safe pattern it struggles with:

```rust
// You have multiple &self methods that logically mutate internal state
// (e.g., a counter, a cache hit count), but your public API promises
// no observable mutation to the caller.
```

**Interior mutability** is the solution: a way to tell the compiler "I'm doing this safely, but you can't verify it statically — trust me, and I'll prove it at runtime."

The key types:

| Type | Check happens | For |
|------|---------------|-----|
| `Cell<T>` | Compile time (no refs given out) | `Copy` types |
| `RefCell<T>` | Runtime (panics on violation) | All types |
| `Mutex<T>` | Runtime (blocks on contention) | Thread-safe mutation |
| `RwLock<T>` | Runtime (read-write locking) | Thread-safe many-readers |

---

## `Cell<T>`

`Cell` works by swapping entire values — it never gives out a reference to its interior. This makes it safe to implement with no runtime checking, because you can't alias a value that's never borrowed.

```rust
use std::cell::Cell;

#[derive(Debug)]
struct HitCounter {
    hits: Cell<u64>,
}

impl HitCounter {
    fn new() -> Self {
        Self { hits: Cell::new(0) }
    }

    fn record_hit(&self) {     // &self, not &mut self!
        self.hits.set(self.hits.get() + 1);
    }

    fn count(&self) -> u64 {
        self.hits.get()
    }
}
```

Usage:

```rust
let c = HitCounter::new();
c.record_hit();
c.record_hit();
assert_eq!(c.count(), 2);

// Can share via Rc without needing &mut
let c = Rc::new(HitCounter::new());
let c2 = Rc::clone(&c);
c.record_hit();
c2.record_hit();
assert_eq!(c.count(), 2);
```

Teaching points:

- `record_hit` takes `&self` but mutates — this is the whole point.
- `Cell` does **not** give you references to its contents — only copies.
- `Cell<T>` requires `T: Copy` for `get()`, but `set()` works for any `T`.

### Other useful `Cell` methods

```rust
let c = Cell::new(vec![1, 2, 3]);
let v = c.take();           // replaces with Default::default()
c.set(vec![4, 5, 6]);
```

---

## `RefCell<T>`

`RefCell` allows borrowing **references** at runtime. It maintains a dynamic borrow count:

- `borrow()` → `Ref<T>`: increments shared borrow count; panics if mutably borrowed.
- `borrow_mut()` → `RefMut<T>`: takes exclusive borrow; panics if any borrow is active.
- When guards drop, counts are decremented.

```rust
use std::cell::RefCell;

let data = RefCell::new(vec![1, 2, 3]);

// Shared borrow
{
    let r = data.borrow();
    println!("len = {}", r.len());
}  // r dropped, borrow ends

// Exclusive borrow
{
    let mut m = data.borrow_mut();
    m.push(4);
}  // m dropped, borrow ends

assert_eq!(*data.borrow(), vec![1, 2, 3, 4]);
```

### Understanding the Ref/RefMut guards

`Ref<T>` and `RefMut<T>` implement `Deref` (and `DerefMut`), so they behave like `&T` and `&mut T` in most contexts. But they also carry the "I'm borrowed" information that `RefCell` tracks.

When a guard is dropped, the borrow ends — that's the runtime enforcement.

---

## Common runtime panic scenario

Show a simplified mistake:

```rust
use std::cell::RefCell;

let data = RefCell::new(vec![1, 2, 3]);
let r = data.borrow();          // shared borrow active

// data.borrow_mut();           // would panic: "already borrowed"

println!("{:?}", r);            // r still alive here
drop(r);                        // explicit drop

let mut m = data.borrow_mut(); // now ok
m.push(4);
```

**The most common mistake in RefCell usage**: holding a `Ref` or `RefMut` alive longer than expected, often by returning it from a function:

```rust
fn get_items(cache: &RefCell<Vec<String>>) -> Ref<Vec<String>> {
    cache.borrow()  // ok to return — caller must manage it
}

// But if caller then also tries to borrow_mut, it panics!
```

Rule: keep borrows short. Use blocks to scope them:

```rust
let len = {
    let guard = data.borrow();
    guard.len()
};  // guard drops here
// now safe to borrow_mut
```

### Try variants: `try_borrow` and `try_borrow_mut`

For code that cannot panic (embedded, error recovery):

```rust
match data.try_borrow_mut() {
    Ok(mut m)  => m.push(42),
    Err(_)     => eprintln!("borrow conflict — skipping"),
}
```

---

## Mini-lab: cache with controlled access

Goal: expose a cache API that prevents callers from holding onto references in dangerous ways.

### Step 1: define the cache

```rust
use std::cell::RefCell;
use std::collections::HashMap;

#[derive(Debug)]
struct Cache {
    map:  RefCell<HashMap<String, String>>,
    hits: Cell<u64>,
    misses: Cell<u64>,
}

impl Cache {
    fn new() -> Self {
        Self {
            map:    RefCell::new(HashMap::new()),
            hits:   Cell::new(0),
            misses: Cell::new(0),
        }
    }
}
```

### Step 2: safe API design

Key choice: do not return `Ref<'_, HashMap<...>>` to callers. Return owned values instead.

```rust
impl Cache {
    fn get(&self, key: &str) -> Option<String> {
        let result = self.map.borrow().get(key).cloned();
        if result.is_some() {
            self.hits.set(self.hits.get() + 1);
        } else {
            self.misses.set(self.misses.get() + 1);
        }
        result
    }

    fn set(&self, key: &str, value: &str) {
        self.map.borrow_mut().insert(key.to_string(), value.to_string());
    }

    fn get_or_insert_with(&self, key: &str, f: impl FnOnce() -> String) -> String {
        // Check first (short borrow)
        if let Some(v) = self.get(key) {
            return v;
        }
        // Compute and insert (no borrow held during computation)
        let v = f();
        self.set(key, &v);
        v
    }

    fn stats(&self) -> (u64, u64) {
        (self.hits.get(), self.misses.get())
    }

    fn len(&self) -> usize {
        self.map.borrow().len()
    }

    fn clear(&self) {
        self.map.borrow_mut().clear();
    }
}
```

Teaching points:

- Returning `String` (owned) avoids leaking borrow guards to callers.
- `Cell<u64>` is used for the counters — no borrow needed, just copy-in/copy-out.
- `get_or_insert_with` avoids double borrowing: it borrows to check, drops the borrow, computes, then borrows again to insert.

### Step 3: tests

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn cache_basic() {
        let c = Cache::new();
        assert_eq!(c.get("a"), None);
        c.set("a", "1");
        assert_eq!(c.get("a"), Some("1".to_string()));
    }

    #[test]
    fn cache_stats() {
        let c = Cache::new();
        c.get("x");           // miss
        c.set("x", "hello");
        c.get("x");           // hit
        c.get("x");           // hit
        assert_eq!(c.stats(), (2, 1));
    }

    #[test]
    fn cache_get_or_insert_with() {
        let c = Cache::new();
        let mut computed = 0;
        let v1 = c.get_or_insert_with("k", || { computed += 1; "val".to_string() });
        let v2 = c.get_or_insert_with("k", || { computed += 1; "other".to_string() });
        assert_eq!(v1, "val");
        assert_eq!(v2, "val");   // cached from first call
        assert_eq!(computed, 1); // f called only once
    }

    #[test]
    fn cache_clear() {
        let c = Cache::new();
        c.set("a", "1");
        c.clear();
        assert_eq!(c.len(), 0);
        assert_eq!(c.get("a"), None);
    }
}
```

---

## Code walkthrough: naive → idiomatic

### Naive: `&mut self` everywhere

```rust
struct Counter {
    value: u64,
}

impl Counter {
    fn inc(&mut self) { self.value += 1; }
    fn get(&self)     -> u64 { self.value }
}
```

This is fine when you have sole ownership. The issue is when you need to share a `Counter` via `Rc` — you can't get `&mut Counter` through `Rc<Counter>`.

### Idiomatic: `Cell<u64>` for shared counter

```rust
use std::cell::Cell;

struct Counter {
    value: Cell<u64>,
}

impl Counter {
    fn inc(&self)     { self.value.set(self.value.get() + 1); }
    fn get(&self)     -> u64 { self.value.get() }
}

let c = Rc::new(Counter { value: Cell::new(0) });
let c2 = Rc::clone(&c);
c.inc();
c2.inc();
assert_eq!(c.get(), 2);  // both see the same value
```

---

## Applications in systems programming

### Lazy initialization

```rust
use std::cell::OnceCell;

struct Config {
    computed: OnceCell<String>,
    raw:      String,
}

impl Config {
    fn computed_value(&self) -> &str {
        self.computed.get_or_init(|| {
            // expensive one-time computation
            self.raw.to_uppercase()
        })
    }
}
```

`OnceCell` (stabilized in Rust 1.70) is the standard lazy-init pattern.

### Thread-safe lazy init: `OnceLock`

```rust
use std::sync::OnceLock;

static GLOBAL_CONFIG: OnceLock<Config> = OnceLock::new();

fn get_config() -> &'static Config {
    GLOBAL_CONFIG.get_or_init(|| Config::load())
}
```

### Event systems (pub/sub)

An event bus might use `RefCell<Vec<Box<dyn Fn(&Event)>>>` to allow registering handlers at runtime without `&mut self`:

```rust
struct EventBus {
    handlers: RefCell<Vec<Box<dyn Fn(&str)>>>,
}

impl EventBus {
    fn subscribe(&self, f: impl Fn(&str) + 'static) {
        self.handlers.borrow_mut().push(Box::new(f));
    }

    fn emit(&self, event: &str) {
        for h in self.handlers.borrow().iter() {
            h(event);
        }
    }
}
```

---

## Challenges and extensions

1. **`RefCell<T>` borrow graph**: draw a timeline of borrows/drops for the cache lab. Verify no two borrows overlap with a mutable borrow.

2. **`try_borrow_mut` pattern**: rewrite `get_or_insert_with` to use `try_borrow_mut` instead of panicking, returning `Option<String>` if the borrow fails.

3. **`OnceCell` for memoization**: implement a `Memoize<T: Fn(u64) -> u64>` struct that caches results using `HashMap<u64, u64>` inside `RefCell`.

4. **Thread-safe cache**: convert the `Cache` from this lesson to use `Mutex<HashMap<...>>` instead of `RefCell` so it can be shared across threads via `Arc`.

---

## When to use interior mutability

Use it when:

- you have shared ownership (`Rc`) but need mutation
- you need mutation behind an immutable API (e.g., caching, lazy init)
- you must satisfy a trait method that only gives `&self`

Avoid it when:

- a plain `&mut self` API works
- you are using it to "fight" the borrow checker rather than redesigning
- you need thread safety (use `Mutex`/`RwLock` instead)

---

## Common mistakes

1. Holding a `Ref` or `RefMut` guard alive while trying to take another borrow → runtime panic.
2. Using `RefCell` to avoid the borrow checker without actually knowing why the design calls for it.
3. Returning `Ref<T>` from a function — callers then hold borrows unexpectedly.
4. Using `RefCell` in multi-threaded code (`RefCell` is `!Send` — the compiler will reject it).

---

---

## Worked Examples & Deep Dive

### Worked Example 1: Tracing `RefCell`'s Internal `borrow: isize` State Machine

Inside `RefCell<T>`, a single signed integer `borrow: isize` encodes the exact borrowing state:
- `0` = **unborrowed**
- `+k` (`k > 0`) = **`k` active immutable `Ref` borrows**
- `-1` = **1 active exclusive `RefMut` borrow**

Trace this sequence line by line:

| Step | Code Executed | `borrow` state | Result |
|---|---|---|---|
| 1 | `let c = RefCell::new(vec![1, 2]);` | `0` | Created |
| 2 | `let r1 = c.borrow();` | **`+1`** | OK (`Ref` guard created) |
| 3 | `let r2 = c.borrow();` | **`+2`** | OK (multiple readers allowed!) |
| 4 | `let m = c.borrow_mut();` (while `r1, r2` alive!) | `+2 != 0`! | **PANIC: `already borrowed: BorrowMutError`!** |
| 5 | *(If instead we `drop(r1); drop(r2);` first)* | `0` | Guards dropped → back to `0` |
| 6 | `c.borrow_mut().push(3);` | `0 -> -1 -> 0` | **OK!** Guard drops at semicolon `;`! |

### Worked Example 2: Choosing Between `Cell`, `RefCell`, and `Mutex` on 3 Concrete Fields

| Field Inside `struct AppMetrics` | Best Wrapper | Runtime Cost | Can Panic? | Thread-Safe? |
|---|---|---|---|---|
| `hits: usize` (single-threaded) | **`Cell<usize>`** | **0 bytes overhead, 1 CPU move** | **Never** | No |
| `cache: HashMap<String, Vec<u8>>` (single-threaded) | **`RefCell<HashMap<...>>`** | 1 `isize` counter | Yes (if overlapping borrows) | No |
| `cache: HashMap<String, Vec<u8>>` (multi-threaded) | **`Mutex<HashMap<...>>`** (or `RwLock`) | OS/futex lock | Poisoning on thread panic | **Yes (`Sync`)** |

### Deep-Dive Explanation: Keeping `RefCell` Safe and Panic-Free (Read Slowly)

#### Step 1: Never hold a `Ref` or `RefMut` across a callback or recursive call
Almost every real-world `RefCell` panic comes from borrowing `self.state.borrow_mut()` and then calling another method `self.notify()` that *also* calls `self.state.borrow()`! Always clone the tiny piece of data you need, drop the guard in a tight `{ ... }` block, and *then* invoke callbacks.

#### Step 2: Prefer `Cell<T>` whenever `T: Copy` (or `take()`/`replace()`)
If you only need to read/write an integer, boolean, or enum flag—or swap a `Vec` out temporarily with `cell.take()`—`Cell<T>` gives you compile-time impossibility of borrow panics with zero runtime counter checks.

#### Step 3: Encapsulate `RefCell` strictly as a private field
Never expose `pub state: RefCell<State>` in a public API! Hide `RefCell` inside private methods that acquire the guard, perform one crisp operation, and release the guard before returning—so callers *cannot* accidentally hold overlapping guards.

#### Step 4: Treat rampant `Rc<RefCell<T>>` as an architectural smoke alarm
If half your structs are wrapped in `Rc<RefCell<...>>`, pause and check whether you are translating an OOP pointer-graph directly into Rust. Often, flattening ownership into a `Vec` + indices or splitting read/write phases eliminates `RefCell` completely.

## Exercises

1. Add `len()` and `clear()` to the cache (above solution has them — write the tests independently first).
2. Implement a counter cache: cache values plus a hit counter using `Cell<u64>`.
3. Modify `get_or_insert_with` to track whether it computed or used a cached value, using `Cell<bool>` as a "was_computed" flag.
4. (**harder**) Build a `Memo<T>` struct that lazily computes a `T` exactly once using `OnceCell`, exposing `fn get(&self) -> &T`.

---

## References

- [`std::cell::Cell` docs](https://doc.rust-lang.org/std/cell/struct.Cell.html)
- [`std::cell::RefCell` docs](https://doc.rust-lang.org/std/cell/struct.RefCell.html)
- [`std::cell::OnceCell` docs](https://doc.rust-lang.org/std/cell/struct.OnceCell.html)
- [The Rust Book, Ch. 15.5 — RefCell and Interior Mutability](https://doc.rust-lang.org/book/ch15-05-interior-mutability.html)

## Recap

- Interior mutability trades compile-time checks for runtime checks — a deliberate tradeoff.
- `Cell` is for `Copy` data; no references, no runtime check, no panic.
- `RefCell` is for any type; references are given out; violations panic at runtime.
- Design APIs to avoid leaking `Ref`/`RefMut` guards to callers.
- `OnceCell`/`OnceLock` are the standard lazy-initialization tools.

## Video tham khảo

- [Smart Pointers in Rust - Interior Mutability](https://www.youtube.com/watch?v=77aRH6YBKyY) — Let's Get Rusty, English. **xem trước.** Watch first: Cell and RefCell, and why the borrow checker sometimes moves the check to runtime.
- [Rust Interior Mutability - Sneaking By The Borrow Checker](https://www.youtube.com/watch?v=HwupNf9iCJk) — Code to the Moon, English. The runtime panic when two RefCell borrows overlap — the failure mode in the cache lab.
- [Rc and RefCell Smart Pointers - Rust](https://www.youtube.com/watch?v=KYJ95TxEC18) — danlogs, English. Rc<RefCell<T>> as the combination the lesson’s shared-cache pattern needs.
