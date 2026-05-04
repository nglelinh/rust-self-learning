---
layout: post
title: "05-21 Concurrency I (Threads and Shared State)"
chapter: "05"
order: 21
owner: "OpenCode"
lang: en
categories:
  - chapter05
lesson_type: required
---

Rust's type system pushes many concurrency bugs to compile time. This lesson introduces OS threads, shared-state concurrency using `Arc<Mutex<T>>`, and the crucial difference between data races (which Rust prevents) and deadlocks (which you still must avoid).

## 60-minute teaching plan

- 0 to 10 min: What concurrency is, and what Rust prevents (data races).
- 10 to 25 min: Threads: spawn, join, move closures.
- 25 to 45 min: Shared state with `Arc<Mutex<T>>`.
- 45 to 55 min: Deadlocks: what they are and how to reduce risk.
- 55 to 60 min: Exercises and recap.

## Learning goals

By the end of this lesson, students can:

- Spawn threads and wait for them with `join`.
- Explain the role of `Send` and `Sync` at a high level.
- Share mutable state safely with `Arc<Mutex<T>>`.
- Keep lock scopes small and avoid locking patterns that lead to deadlocks.

## Prerequisites

- Lesson 04-19 (Smart Pointers): `Arc<T>` for shared ownership across threads.
- Lesson 04-20 (Interior Mutability): `Mutex` follows similar ideas to `RefCell` but is thread-safe.

---

## Key concept: what Rust prevents vs what you still must think about

| Bug type | Rust's guarantee | Your responsibility |
|----------|-----------------|---------------------|
| **Data race** | Prevented in safe Rust | — |
| **Deadlock** | Not prevented | Must design to avoid |
| **Starvation** | Not prevented | Must design to avoid |
| **Logic races** | Not prevented | Must test carefully |

A **data race** is two threads accessing the same memory concurrently where at least one access is a write, with no synchronization. It causes undefined behavior in C/C++. Rust's ownership system prevents it: you can't share mutable data without `Mutex`/`RwLock`/channels.

A **deadlock** is two threads waiting on each other forever. The compiler can't prove absence of deadlocks — that requires thinking about lock ordering.

---

## Concurrency vocabulary

- **Concurrency**: multiple tasks make progress — possibly by interleaving on one core.
- **Parallelism**: tasks run at the same time on multiple cores.

Rust's OS threads use true parallelism. Async (next lesson) uses concurrency (cooperative scheduling on fewer threads).

---

## Threads basics

### Spawning and joining

```rust
use std::thread;

fn main() {
    let handle = thread::spawn(|| {
        println!("hello from thread {:?}", thread::current().id());
    });

    handle.join().unwrap();   // waits for the thread to finish
}
```

Teaching points:

- `spawn` returns a `JoinHandle<T>` where `T` is the closure's return type.
- `join()` returns `Result<T, Box<dyn Any>>` — the `Err` case is a thread panic.
- If you don't join, the thread is detached and may be killed when main exits.

### What happens if you don't join?

```rust
fn main() {
    thread::spawn(|| {
        // This might not even start before main exits
        println!("I might not print!");
    });
    // main exits, spawned thread is killed
}
```

Always join threads that do meaningful work.

### Moving data into a thread

If a thread closure uses an outer variable, it may need `move`.

```rust
use std::thread;

fn main() {
    let s = String::from("owned data");
    let handle = thread::spawn(move || {
        println!("thread got: {s}");
    });
    // println!("{s}"); // error: s was moved into the closure
    handle.join().unwrap();
}
```

Teaching points:

- `move` transfers ownership into the closure.
- After `move`, the parent thread can't use `s`.
- The compiler enforces this: you can't share `s` after moving it.

### Why `move` is required for cross-thread data

Without `move`, the closure would capture `s` by reference. But the thread might outlive the function where `s` is defined — the borrow checker rejects this as potentially dangling.

```rust
fn spawn_printer(s: String) {
    thread::spawn(|| println!("{s}"));
    // error: `s` doesn't live long enough
    // The thread might outlive `spawn_printer`
}
```

Fix: `move` or pass an `Arc`.

---

## `Send` and `Sync` intuition

Rust tracks thread-safety through two marker traits:

- **`Send`**: values of this type can be **moved** to another thread.
- **`Sync`**: references to this type can be **shared** between threads (i.e., `&T: Send`).

You don't implement these manually in typical Rust. The compiler derives them automatically from the type's contents:

| Type | `Send` | `Sync` | Why |
|------|--------|--------|-----|
| `i32`, `String`, `Vec<T>` | yes | yes | contains no thread-unsafe parts |
| `Rc<T>` | **no** | **no** | non-atomic ref count |
| `Arc<T>` | yes | yes (if `T: Send+Sync`) | atomic ref count |
| `Cell<T>`, `RefCell<T>` | yes | **no** | interior mutability without locks |
| `Mutex<T>` | yes | yes (if `T: Send`) | protects access with a lock |

When you try to send a non-`Send` type to a thread, you get a compile error:

```
error[E0277]: `Rc<i32>` cannot be sent between threads safely
```

---

## Shared state with `Arc<Mutex<T>>`

When multiple threads need to mutate shared state:

- `Arc<T>`: shared ownership across threads (atomic reference count).
- `Mutex<T>`: exclusive access — only one thread at a time can mutate.

Together: `Arc<Mutex<T>>` is the standard pattern.

### Anatomy of `Mutex`

`Mutex<T>` owns the protected data. `lock()` returns a `MutexGuard<T>` — access the data through the guard. When the guard drops, the lock is released.

```rust
use std::sync::{Arc, Mutex};
use std::thread;

fn main() {
    let counter = Arc::new(Mutex::new(0u64));
    let mut handles = Vec::new();

    for _ in 0..4 {
        let c = Arc::clone(&counter);
        handles.push(thread::spawn(move || {
            for _ in 0..1_000 {
                let mut guard = c.lock().unwrap();
                *guard += 1;
                // guard drops here → lock released
            }
        }));
    }

    for h in handles {
        h.join().unwrap();
    }

    println!("final = {}", *counter.lock().unwrap());  // always 4000
}
```

Teaching points:

- `lock()` blocks until the lock is available.
- `lock()` returns `Err` only if another thread panicked while holding the lock (a "poisoned" mutex).
- Keep the lock scope as **small** as possible.

### Why `unwrap()` on `lock()`?

In production code, you'd handle poisoning. For teaching: a poisoned mutex means a thread panicked while holding it. If that's unexpected, `unwrap()` is reasonable. For critical systems, use `.lock().unwrap_or_else(|e| e.into_inner())` to recover.

---

## Data races vs deadlocks

### Data race

A data race is **unsynchronized concurrent access** where at least one access is a write. Rust prevents this in safe code by making you use `Mutex`, channels, etc. You literally cannot compile a program with a data race in safe Rust.

### Deadlock

A deadlock is a **liveness bug**: threads are waiting on each other forever. Nothing crashes — but nothing progresses either.

Typical cause: multiple locks acquired in inconsistent orders.

```rust
// Thread 1: lock A, then lock B
// Thread 2: lock B, then lock A
// If each thread acquires its first lock before the other, they deadlock
```

Mitigations:

- Keep lock scopes **small** — the less time you hold a lock, the less chance of contention.
- Acquire locks in a **consistent global order** (e.g., always A then B, never B then A).
- Avoid holding locks while doing **slow IO** or computations.
- Consider using **channels** instead of shared locks for complex coordination.

### `Mutex` and the lock scope rule

```rust
// Bad: holding lock while doing expensive work
{
    let mut guard = data.lock().unwrap();
    let result = expensive_computation();  // lock held during this!
    guard.push(result);
}

// Good: compute outside the lock
let result = expensive_computation();
{
    let mut guard = data.lock().unwrap();
    guard.push(result);
}
```

---

## Mini-lab: multi-threaded aggregator

Goal: split a list into chunks, compute partial sums in threads, and combine.

```rust
use std::sync::{Arc, Mutex};
use std::thread;

fn parallel_sum(xs: &[u64], num_workers: usize) -> u64 {
    let total = Arc::new(Mutex::new(0u64));
    let chunk_size = (xs.len() + num_workers - 1) / num_workers;
    let mut handles = Vec::new();

    for chunk in xs.chunks(chunk_size) {
        let chunk = chunk.to_vec();           // clone into thread
        let total = Arc::clone(&total);
        handles.push(thread::spawn(move || {
            let local: u64 = chunk.iter().sum();
            *total.lock().unwrap() += local;  // short lock scope
        }));
    }

    for h in handles {
        h.join().unwrap();
    }

    *total.lock().unwrap()
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn parallel_sum_is_correct() {
        let data: Vec<u64> = (1..=100).collect();
        assert_eq!(parallel_sum(&data, 4), 5050);
    }

    #[test]
    fn parallel_sum_empty() {
        assert_eq!(parallel_sum(&[], 4), 0);
    }

    #[test]
    fn parallel_sum_single() {
        assert_eq!(parallel_sum(&[42], 4), 42);
    }
}
```

Teaching points:

- We clone the chunk into a `Vec` to avoid lifetime issues with threads.
- Compute (`sum`) happens outside the lock; only the final accumulation is locked.
- `xs.chunks(chunk_size)` handles uneven splits cleanly.

---

## Code walkthrough: naive → idiomatic

### Naive: lock held too long

```rust
fn count_words_parallel(docs: Vec<String>, workers: usize) -> usize {
    let total = Arc::new(Mutex::new(0usize));
    let docs = Arc::new(docs);
    let mut handles = Vec::new();

    for i in 0..workers {
        let total = Arc::clone(&total);
        let docs = Arc::clone(&docs);
        handles.push(thread::spawn(move || {
            let mut guard = total.lock().unwrap();  // lock held for entire loop!
            for doc in docs.iter().skip(i).step_by(workers) {
                *guard += doc.split_whitespace().count();
            }
        }));
    }
    for h in handles { h.join().unwrap(); }
    *total.lock().unwrap()
}
```

Problem: each worker holds the lock for the entire computation — zero parallelism.

### Idiomatic: compute locally, accumulate once

```rust
fn count_words_parallel(docs: Vec<String>, workers: usize) -> usize {
    let total = Arc::new(Mutex::new(0usize));
    let docs = Arc::new(docs);
    let mut handles = Vec::new();

    for i in 0..workers {
        let total = Arc::clone(&total);
        let docs = Arc::clone(&docs);
        handles.push(thread::spawn(move || {
            // Compute locally — no lock held
            let local: usize = docs.iter()
                .skip(i).step_by(workers)
                .map(|d| d.split_whitespace().count())
                .sum();
            // Accumulate — lock held briefly
            *total.lock().unwrap() += local;
        }));
    }

    for h in handles { h.join().unwrap(); }
    *total.lock().unwrap()
}
```

---

## Applications in systems programming

### Web server request counters

```rust
struct Metrics {
    requests:  Arc<Mutex<u64>>,
    errors:    Arc<Mutex<u64>>,
}

impl Metrics {
    fn record_request(&self) {
        *self.requests.lock().unwrap() += 1;
    }
}
```

In practice, `AtomicU64` is better for simple counters (no lock needed), but `Mutex<T>` works for complex structures.

### Parallel file processing pipeline

```rust
let results: Arc<Mutex<Vec<ProcessResult>>> = Arc::new(Mutex::new(Vec::new()));
for file in files {
    let results = Arc::clone(&results);
    thread::spawn(move || {
        let r = process_file(&file);
        results.lock().unwrap().push(r);  // short lock
    });
}
```

### Build system job runner

A make/cargo-like system spawns threads for independent build tasks and uses a mutex to protect the dependency graph.

---

## Challenges and extensions

1. **`RwLock<T>`**: replace `Mutex` in the cache lab with `RwLock`. Allow multiple concurrent reads, exclusive writes. When is this better? Worse?

2. **`AtomicU64` vs `Mutex<u64>`**: for a simple counter, `AtomicU64` is lock-free. Compare performance and complexity.

3. **Scoped threads (`std::thread::scope`)**: since Rust 1.63, you can borrow data into threads without `Arc` by guaranteeing all threads finish before the scope ends. Rewrite `parallel_sum` using `thread::scope` and avoid the `.to_vec()` clone.

4. **Deadlock demonstration**: write a program that deliberately deadlocks, then fix it by using a consistent lock ordering.

---

## Exercises

1. Change the aggregator to compute `(min, max, sum)` across all elements in parallel.
2. Add error handling: have one worker return a `Result` containing an error, and surface it via `JoinHandle::join().unwrap_err()`.
3. Explain why holding the lock while counting words (naive version) reduces parallelism to zero.
4. (**harder**) Implement a parallel map: `fn par_map<T, U>(xs: Vec<T>, f: impl Fn(T) -> U + Send + Sync + 'static, workers: usize) -> Vec<U>` that preserves order.

---

## References

- [The Rust Book, Ch. 16 — Fearless Concurrency](https://doc.rust-lang.org/book/ch16-00-concurrency.html)
- [`std::thread` docs](https://doc.rust-lang.org/std/thread/index.html)
- [`std::sync::Mutex` docs](https://doc.rust-lang.org/std/sync/struct.Mutex.html)

## Recap

- Threads are OS-level concurrency — true parallelism.
- `Arc<Mutex<T>>` is the standard shared-state tool for mutable data.
- Rust prevents data races; deadlocks are still your responsibility.
- Keep lock scopes small: compute outside the lock, accumulate inside.
- `Send` and `Sync` are compile-time guarantees, not runtime overhead.
