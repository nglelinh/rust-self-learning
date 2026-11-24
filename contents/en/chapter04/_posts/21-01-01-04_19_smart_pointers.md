---
layout: post
title: "04-19 Smart Pointers"
chapter: "04"
order: 19
owner: "OpenCode"
lang: en
categories:
  - chapter04
lesson_type: required
---

Default Rust ownership is a strict monarchy: every value has **one** owner, and when that owner leaves scope, the value is dropped. That simple rule handles 80% of systems code—until you try to define a recursive expression tree (`enum Expr { Add(Expr, Expr) }` has *infinite* stack size!) or share a read-only configuration table across 16 worker threads without copying it 16 times.

**Smart pointers** are Rust’s explicit, zero-mystery ownership strategies: `Box<T>` moves a single-owner value onto the heap with a fixed 8-byte stack handle; `Rc<T>` and `Arc<T>` share ownership via reference counting (non-atomic vs. atomic); and `Weak<T>` breaks reference cycles so memory never leaks silently. This lesson traces the exact heap control blocks and refcounts behind every pointer so you always pick the lightest tool for the job.

---

## Background Knowledge Map: Memory Ownership Strategies in Systems Programming

### 1. Stack Indirection & Recursive Types (`Box<T>`)
Stack frames must have compile-time known sizes (`Sized`). Wrapping a recursive child in `Box<T>` replaces an unbounded inline struct with an **8-byte heap pointer**—making recursive ASTs, linked structures, and trait objects (`Box<dyn Trait>`) compile with zero reference-counting overhead.
- **Core questions:** How does `Box<T>` give heap allocation with exact single-owner RAII (`Drop`) semantics?
- **Real-world reach:** Compiler ASTs, binary trees, huge structs moved off small stacks, and `Box<dyn Error>`.

### 2. Reference Counting: Single-Threaded (`Rc<T>`) vs. Atomic (`Arc<T>`)
When multiple owners share the same immutable heap allocation, `Rc<T>` stores a `(strong_count, weak_count, T)` control block on the heap and increments/decrements plain integers on `clone()`/`drop()`. Across threads, plain integer increments would be a **data race**—so `Rc<T>` is explicitly `!Send + !Sync`, while `Arc<T>` uses hardware atomic instructions (`fetch_add`/`fetch_sub`) to guarantee thread safety!
- **Core questions:** Why is `Rc::clone(&a)` O(1) (an integer bump, not a deep copy)? Why does `Arc` cost slightly more per clone than `Rc`?
- **Real-world reach:** Shared routing tables, immutable snapshots, graph/DAG caches, and multi-threaded server state.

### 3. Reference Cycles & Weak Handles (`Weak<T>`)
Reference counting frees memory only when `strong_count` hits `0`. If Node A holds an `Rc` to Node B and Node B holds an `Rc` back to Node A, both strong counts stay `>= 1` forever—a classic **memory leak** (which is *memory-safe* in Rust, yet leaks RAM!). Making the parent pointer a `Weak<T>` bumps only `weak_count`, letting strong counts drop cleanly to `0`.
- **Core questions:** How does `Weak::upgrade()` safely return `Option<Rc<T>>` instead of a dangling pointer?
- **Real-world reach:** DOM parent/child trees, observer registries, and LRU caches.

## 60-minute teaching plan

- 0 to 10 min: What a "smart pointer" is in Rust.
- 10 to 25 min: `Box<T>` for heap allocation and recursion.
- 25 to 40 min: `Rc<T>` for shared ownership in single-threaded code.
- 40 to 50 min: `Arc<T>` for shared ownership across threads.
- 50 to 60 min: `Weak<T>` to break cycles (tree parent pointers).

## Learning goals

By the end of this lesson, students can:

- Use `Box` to build recursive data structures.
- Explain reference counting and when `Rc`/`Arc` are appropriate.
- Avoid reference cycles using `Weak`.
- Recognize when shared ownership is a design smell.

## Prerequisites

- Lesson 02-06 (Ownership): drops, RAII, move semantics.
- Lesson 04-20 (Interior Mutability): `RefCell` (needed for `Rc`+mutation examples).

---

## Key concept: what is a "smart pointer"?

In C, a pointer is just a memory address. The programmer decides when to free it — or forgets to.

In Rust, a **smart pointer** is a value that:

1. Implements `Deref` (so it can be used like a reference).
2. Implements `Drop` (so cleanup happens automatically).
3. Carries additional ownership semantics — single (`Box`), shared-counted (`Rc`/`Arc`), or weak.

Most `Box<T>` usage is invisible — Rust automatically dereferences it where needed.

---

## `Box<T>`: owning heap allocation

`Box<T>` stores a value **on the heap** and gives you a stable, owned pointer. Unlike a raw pointer, it's always valid (non-null, properly aligned) and drops the value when the `Box` drops.

### Why you need it: recursive types

Recursive enums need indirection because the compiler must know sizes at compile time.

**Without Box (won't compile):**

```rust
enum List {
    Nil,
    Cons(i32, List),  // error: recursive type has infinite size
}
```

The compiler can't compute the size of `List` because it contains itself.

**With Box (works):**

```rust
#[derive(Debug, Clone, PartialEq)]
enum List {
    Nil,
    Cons(i32, Box<List>),
}
```

`Box<List>` is a pointer-sized value (8 bytes on 64-bit). The compiler knows its size.

Usage:

```rust
let xs = List::Cons(1, Box::new(List::Cons(2, Box::new(List::Nil))));
```

### `Box` as a dispatch mechanism: trait objects

`Box<dyn Trait>` is the idiomatic way to store values of different types behind a shared interface:

```rust
trait Shape: std::fmt::Debug {
    fn area(&self) -> f64;
}

#[derive(Debug)] struct Circle  { radius: f64 }
#[derive(Debug)] struct Rect    { w: f64, h: f64 }

impl Shape for Circle { fn area(&self) -> f64 { std::f64::consts::PI * self.radius.powi(2) } }
impl Shape for Rect   { fn area(&self) -> f64 { self.w * self.h } }

fn total_area(shapes: &[Box<dyn Shape>]) -> f64 {
    shapes.iter().map(|s| s.area()).sum()
}

let shapes: Vec<Box<dyn Shape>> = vec![
    Box::new(Circle { radius: 1.0 }),
    Box::new(Rect { w: 2.0, h: 3.0 }),
];
println!("{}", total_area(&shapes));
```

Teaching point: `Box` is still single-owner. Cloning a `Box<T>` clones the `T` (if `T: Clone`).

---

## `Rc<T>`: shared ownership (single-threaded)

`Rc` is reference-counted. Cloning an `Rc` clones **the pointer**, not the underlying data. Both clones point to the same allocation. The data is dropped when the last `Rc` is dropped.

```rust
use std::rc::Rc;

let a = Rc::new(String::from("shared data"));
let b = Rc::clone(&a);   // cheap — only increments a counter
assert_eq!(Rc::strong_count(&a), 2);

drop(b);
assert_eq!(Rc::strong_count(&a), 1);
// data is dropped when a is dropped
```

### Why `Rc` instead of `&T`?

`&T` borrows: the original owner must outlive all borrows. `Rc<T>` shares ownership: nobody is "the original" — any `Rc` clone can outlive others.

Use `Rc` when:

- Multiple parts of a data structure need to hold the same data.
- The "original" doesn't outlive the "users" — ownership is ambiguous.
- You are building graphs, trees with shared nodes, or caches.

Teaching point: **`Rc` is not thread-safe.** It uses a non-atomic counter.

### Combining `Rc` with `RefCell` for mutability

`Rc<T>` gives shared ownership but only immutable access. To mutate, combine with `RefCell`:

```rust
use std::rc::Rc;
use std::cell::RefCell;

let shared = Rc::new(RefCell::new(vec![1, 2, 3]));
let a = Rc::clone(&shared);
let b = Rc::clone(&shared);

a.borrow_mut().push(4);
println!("{:?}", b.borrow());  // [1, 2, 3, 4]
```

---

## `Arc<T>`: shared ownership (thread-safe)

`Arc` is Atomically Reference Counted. Its API is identical to `Rc`, but the counter uses atomic operations — safe to share across threads.

```rust
use std::sync::Arc;
use std::thread;

let data = Arc::new(vec![1, 2, 3]);
let data2 = Arc::clone(&data);

let handle = thread::spawn(move || {
    println!("thread sees: {:?}", data2);
});

handle.join().unwrap();
println!("main sees: {:?}", data);
```

### `Arc` vs `Rc` performance

Atomic operations are more expensive than non-atomic. Use `Rc` for single-threaded code, `Arc` for multi-threaded.

```rust
// Single-threaded graph
type NodeRef = Rc<RefCell<Node>>;

// Multi-threaded task graph
type TaskRef = Arc<Mutex<Task>>;
```

---

## `Weak<T>`: non-owning references to break cycles

`Rc` tracks strong references. When the count hits zero, the data is dropped. But if two `Rc`s point to each other, neither count reaches zero — a **memory leak**.

```rust
// A reference cycle with Rc:
use std::rc::Rc;
use std::cell::RefCell;

#[derive(Debug)]
struct Node {
    next: Option<Rc<RefCell<Node>>>,  // points to another node
}

let a = Rc::new(RefCell::new(Node { next: None }));
let b = Rc::new(RefCell::new(Node { next: Some(Rc::clone(&a)) }));
a.borrow_mut().next = Some(Rc::clone(&b));  // cycle!
// Both a and b have count 2 — neither ever drops to 0.
```

### The solution: `Weak`

`Weak<T>` is a non-owning reference — it doesn't contribute to the strong count. When you want to "see" a value but not "keep it alive", use `Weak`.

### The tree parent-pointer pattern

```rust
use std::cell::RefCell;
use std::rc::{Rc, Weak};

#[derive(Debug)]
struct Node {
    value:    i32,
    parent:   RefCell<Weak<Node>>,           // weak → no cycle
    children: RefCell<Vec<Rc<Node>>>,        // strong → parent owns children
}

fn make_node(value: i32) -> Rc<Node> {
    Rc::new(Node {
        value,
        parent:   RefCell::new(Weak::new()),
        children: RefCell::new(Vec::new()),
    })
}

fn add_child(parent: &Rc<Node>, child: Rc<Node>) {
    *child.parent.borrow_mut() = Rc::downgrade(parent);  // weak reference
    parent.children.borrow_mut().push(child);              // strong reference
}
```

When you want to access the parent:

```rust
fn print_parent_value(node: &Rc<Node>) {
    match node.parent.borrow().upgrade() {
        Some(p) => println!("parent value = {}", p.value),
        None    => println!("no parent (or parent was dropped)"),
    }
}
```

`upgrade()` returns `Option<Rc<T>>` — `None` if the parent was dropped, `Some` otherwise.

Teaching point: the tree can be correctly dropped. Dropping the root removes its children (strong count goes to 0), which removes their children, etc. Parent weak pointers don't prevent this.

---

## Code walkthrough: building an AST evaluator

A practical use case for `Box<dyn Trait>` and recursive enums:

```rust
#[derive(Debug)]
enum Expr {
    Num(f64),
    Add(Box<Expr>, Box<Expr>),
    Mul(Box<Expr>, Box<Expr>),
    Neg(Box<Expr>),
}

fn eval(e: &Expr) -> f64 {
    match e {
        Expr::Num(n)        => *n,
        Expr::Add(a, b)     => eval(a) + eval(b),
        Expr::Mul(a, b)     => eval(a) * eval(b),
        Expr::Neg(e)        => -eval(e),
    }
}

// Build: (1 + 2) * -(3)
let expr = Expr::Mul(
    Box::new(Expr::Add(
        Box::new(Expr::Num(1.0)),
        Box::new(Expr::Num(2.0)),
    )),
    Box::new(Expr::Neg(Box::new(Expr::Num(3.0)))),
);

assert_eq!(eval(&expr), -9.0);
```

Teaching points:

- `Box` enables the recursive enum.
- `eval` takes `&Expr` — borrows the tree, doesn't consume it.
- Evaluation is a simple recursive tree walk.

---

## Applications in systems programming

### Plugin systems

```rust
trait Middleware: Send + Sync {
    fn handle(&self, req: &mut Request) -> Option<Response>;
}

struct Server {
    middlewares: Vec<Arc<dyn Middleware>>,
}
```

`Arc<dyn Middleware>` lets multiple workers share the same middleware instance.

### Interpreter ASTs

A Lisp interpreter might use `Rc<RefCell<Env>>` for environments:

```rust
type Env = Rc<RefCell<HashMap<String, Value>>>;
```

Child environments inherit from parent environments via `Rc` clones — shared ownership.

### Read-only shared configuration

```rust
let config = Arc::new(Config::load());
for _ in 0..num_workers {
    let cfg = Arc::clone(&config);
    thread::spawn(move || run_worker(cfg));
}
```

`Arc<Config>` lets all workers share the same config without copying it.

---

## When shared ownership is a code smell

Before reaching for `Rc`/`Arc`, ask:

- Can I restructure so there is a clear owner?
- Can I store **IDs/indices** instead of pointers?
- Can I return owned values instead of sharing?

Shared ownership makes reasoning harder because:

- You can't tell when data will be dropped.
- Mutation requires `RefCell` or `Mutex`, adding runtime overhead.
- Cycles cause leaks.

Shared ownership is powerful — but it's an escape hatch, not a default.

---

## Challenges and extensions

1. **`Box<dyn Error>`**: look at how `Box<dyn std::error::Error>` is used as a catch-all error type. When is it appropriate? What do you lose compared to typed errors?

2. **Arena allocation**: read about "arena allocators" as an alternative to `Rc` for graphs. How does `bumpalo` work?

3. **Rc cycle detection with `Weak`**: write a function `is_cycle(node: &Rc<Node>) -> bool` that traverses a linked list and detects if it cycles back to the start.

4. **`Arc<Mutex<T>>` vs message passing**: for a shared counter between threads, compare `Arc<Mutex<u64>>` vs a channel-based approach. Which is easier to reason about?

---

---

## Worked Examples & Deep Dive

### Worked Example 1: Tracing `Rc<T>` Strong & Weak Counts Step by Step

Trace the heap control block `(strong, weak)` of `let a = Rc::new(String::from("cfg"));`:

| Step | Code Executed | `strong` | `weak` | Heap `String` Alive? | Control Block Alive? |
|---|---|---|---|---|---|
| 1 | `let a = Rc::new(...)` | **1** | **0** | Yes | Yes |
| 2 | `let b = Rc::clone(&a);` | **2** | **0** | Yes | Yes |
| 3 | `let w = Rc::downgrade(&a);` | **2** | **1** | Yes | Yes |
| 4 | `drop(a); drop(b);` | **0** | **1** | **NO (dropped!)** | Yes (waits for `w`) |
| 5 | `w.upgrade()` | **0** | **1** | Returns **`None`**! | Yes |
| 6 | `drop(w);` | **0** | **0** | No | **Freed!** |

Notice the two-stage teardown: the payload `T` is dropped the instant `strong == 0` (even if `Weak` handles remain), while the tiny control block waits until `weak == 0` so `w.upgrade()` can safely read `strong == 0` and return `None`!

### Worked Example 2: Exact Stack & Heap Footprint of `Box<u64>`, `Rc<u64>`, and `Arc<u64>` (64-bit)

| Smart Pointer | Stack Size | Heap Control Overhead | Thread-Safe (`Send + Sync`)? |
|---|---|---|---|
| `Box<u64>` | **8 bytes** | **0 bytes** (just 8 bytes for `u64`) | Yes (when `T: Send/Sync`) |
| `Rc<u64>` | **8 bytes** | **16 bytes** (`strong: usize`, `weak: usize`) + 8 bytes `u64` | **No (`!Send + !Sync`)** |
| `Arc<u64>` | **8 bytes** | **16 bytes** (`AtomicUsize` strong & weak) + 8 bytes `u64` | **Yes** (when `T: Send + Sync`) |

### Deep-Dive Explanation: Picking the Right Smart Pointer Without Over-Engineering (Read Slowly)

#### Step 1: Start with plain values and references (`T`, `&T`, `&mut T`)
Never reach for `Box` or `Rc` by habit from Java/C++. Plain stack ownership and borrowed slices have zero heap overhead and the cleanest compiler checks.

#### Step 2: Use `Box<T>` for recursive types, trait objects, or moving large structs off the stack
`Box<T>` preserves single ownership completely—you still get `&mut T` directly without any runtime lock or `RefCell`.

#### Step 3: Reach for `Rc<T>` / `Arc<T>` only when ownership is genuinely shared with no clear single owner
Use `Rc` strictly inside single-threaded code (parsers, GUI trees) and `Arc` across threads/async tasks. Remember: both give **shared immutable** access (`&T`) by default—if you also need mutation, you must pair them with interior mutability (`RefCell` or `Mutex`).

#### Step 4: Break every back-edge with `Weak<T>`—or replace pointer graphs with arena indices
In parent–child trees, parents own children (`Rc<Node>`) and children point up via `Weak<Node>`. For complex cyclic graphs, storing nodes in a `Vec<Node>` and connecting them with `usize` indices eliminates cycles, refcount overhead, and lifetime friction all at once!

## Exercises

1. Implement a recursive AST for a simple expression language using `Box` and evaluate it. Include `Add`, `Sub`, `Mul`, `Div`, `Num`, and handle division by zero with `Option<f64>`.
2. Write a function that prints the `Rc::strong_count` at different points in a shared-ownership scenario to illustrate when values are dropped.
3. Build a small tree with parent pointers using `Rc` + `Weak` and prove that dropping the root drops the entire tree (verify with a `Drop` impl that prints).
4. (**harder**) Implement a simple LRU cache using `Rc<RefCell<Node>>` as the underlying doubly-linked list. Use `Weak` for back-pointers.

---

## References

- [The Rust Book, Ch. 15 — Smart Pointers](https://doc.rust-lang.org/book/ch15-00-smart-pointers.html)
- [`std::rc::Rc` docs](https://doc.rust-lang.org/std/rc/struct.Rc.html)
- [`std::sync::Arc` docs](https://doc.rust-lang.org/std/sync/struct.Arc.html)

## Recap

- `Box`: single ownership + heap + recursion + trait objects.
- `Rc`: shared ownership in single-threaded code; clone the pointer, not the data.
- `Arc`: shared ownership across threads; atomic counter, thread-safe.
- `Weak`: break cycles and represent non-owning links.
- Shared ownership is a design choice — prefer clear ownership hierarchies first.

## Video tham khảo

- [Smart Pointers in Rust - Reference Counting](https://www.youtube.com/watch?v=M9Owp3iLigg) — Let's Get Rusty, English. **xem trước.** Watch first: Rc and shared ownership on one thread, before Arc and Weak.
- [Smart Pointers in Rust - The Deref Trait](https://www.youtube.com/watch?v=dYEC6NElVOg) — Let's Get Rusty, English. Deref, so Box and the other pointers feel like the value they own.
- [Crust of Rust: Smart Pointers and Interior Mutability](https://www.youtube.com/watch?v=8O0Nt9qY_vo) — Jon Gjengset, English. The long Crust of Rust session on Box, Rc, RefCell, and when sharing is a smell.
