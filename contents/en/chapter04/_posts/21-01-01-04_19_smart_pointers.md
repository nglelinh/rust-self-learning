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

Smart pointers are explicit ownership strategies: heap allocation (`Box`), shared ownership (`Rc`/`Arc`), and non-owning references (`Weak`). This lesson teaches when to use each, how they interact with the type system, and how to avoid reference cycles that silently leak memory.

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
