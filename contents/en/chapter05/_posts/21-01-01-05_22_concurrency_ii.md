---
layout: post
title: "05-22 Concurrency II (Channels and Work Queues)"
chapter: "05"
order: 22
owner: "OpenCode"
lang: en
categories:
  - chapter05
lesson_type: required
---

Message passing is often a simpler way to build concurrency than shared mutable state. This lesson uses channels to coordinate work, designs explicit shutdown protocols, and builds a worker pool — a pattern you will use in real CLI tools, servers, and pipeline processors.

## 60-minute teaching plan

- 0 to 10 min: Why message passing (reduce shared mutable state).
- 10 to 25 min: `mpsc` channels: `Sender`, `Receiver`, `send`, `recv`.
- 25 to 40 min: Designing message types (jobs, results, stop signals).
- 40 to 60 min: Mini-lab: worker pool that processes jobs and returns results.

## Learning goals

By the end of this lesson, students can:

- Use channels for producer/consumer coordination.
- Design typed message enums for work protocols.
- Implement a worker pool with explicit shutdown.
- Avoid deadlocks by avoiding shared locks in the work path.

## Prerequisites

- Lesson 05-21 (Concurrency I): threads, `Arc<Mutex<T>>`, `Send`/`Sync`.

---

## Key concept: shared state vs message passing

In the previous lesson, we used `Arc<Mutex<T>>` — threads share memory and lock to coordinate. This works but has drawbacks:

- Lock ordering is your responsibility (deadlock risk).
- Logic is scattered across threads (hard to reason about).
- State is implicit — who modified what when?

**Message passing** inverts this: threads don't share mutable memory. They communicate by sending values through typed channels. The channel itself is the synchronization point.

```
Go proverb: "Do not communicate by sharing memory; share memory by communicating."
```

Rust's `std::sync::mpsc` (multi-producer, single-consumer) channels implement this pattern.

---

## Channel basics

```rust
use std::sync::mpsc;
use std::thread;

fn main() {
    let (tx, rx) = mpsc::channel::<String>();

    thread::spawn(move || {
        tx.send("hello".to_string()).unwrap();
        tx.send("world".to_string()).unwrap();
        // tx drops here — signals end-of-stream to rx
    });

    while let Ok(msg) = rx.recv() {
        println!("got: {msg}");
    }
    // loop ends when all Senders are dropped
}
```

Teaching points:

- `channel()` returns a `(Sender<T>, Receiver<T>)` pair.
- `Sender` can be cloned (multiple producers, hence "mpsc").
- `Receiver` is typically owned by one consumer (single-consumer).
- `recv()` blocks until a message arrives or all senders are dropped.
- `try_recv()` is non-blocking — returns `Err(TryRecvError::Empty)` if no message yet.

### Channel backpressure: bounded vs unbounded

`mpsc::channel()` is **unbounded** — the sender never blocks. For backpressure (where producers slow down if consumers lag), use `mpsc::sync_channel(bound)`:

```rust
let (tx, rx) = mpsc::sync_channel::<String>(10);
// tx.send(...) blocks when the buffer has 10 items waiting
```

Backpressure is crucial in real systems: without it, a fast producer can overwhelm a slow consumer, exhausting memory.

---

## Message design: model the protocol

Use an enum so the protocol is explicit and compile-time safe:

```rust
#[derive(Debug)]
enum Job {
    Compute { id: u64, value: u64 },
    Stop,
}

#[derive(Debug, Clone, PartialEq)]
struct JobResult {
    id:     u64,
    input:  u64,
    output: u64,
}
```

Teaching points:

- `Stop` is an explicit shutdown message — workers know when to exit.
- Including `id` in both request and response lets callers match results to requests.
- Separate job and result channels keep flows clear — jobs flow in, results flow out.

---

## Mini-lab: worker pool

Goal: spawn N workers. The main thread sends jobs and receives results. Workers process jobs concurrently.

### Step 1: define the work function

Keep it simple — the real payoff is the concurrency structure around it:

```rust
fn work(value: u64) -> u64 {
    // Simulate CPU work
    (0..value).fold(0u64, |acc, x| acc.wrapping_add(x))
}
```

### Step 2: the receiver-sharing problem

`std::sync::mpsc::Receiver<T>` is single-consumer — it **cannot be cloned**. To share jobs among multiple worker threads, wrap it in `Arc<Mutex<Receiver>>`:

```rust
use std::sync::{mpsc, Arc, Mutex};
use std::thread;

fn run_pool(inputs: &[u64], num_workers: usize) -> Vec<JobResult> {
    let (job_tx, job_rx) = mpsc::channel::<Job>();
    let (res_tx, res_rx) = mpsc::channel::<JobResult>();
    let job_rx = Arc::new(Mutex::new(job_rx));  // shared among workers

    // Spawn workers
    let mut handles = Vec::new();
    for _ in 0..num_workers {
        let rx  = Arc::clone(&job_rx);
        let tx  = res_tx.clone();
        handles.push(thread::spawn(move || loop {
            // Lock scope: receive one job, then release lock immediately
            let job = rx.lock().unwrap().recv();
            match job {
                Ok(Job::Compute { id, value }) => {
                    let output = work(value);
                    tx.send(JobResult { id, input: value, output }).unwrap();
                }
                Ok(Job::Stop) | Err(_) => break,
            }
        }));
    }

    // Send work
    for (i, &value) in inputs.iter().enumerate() {
        job_tx.send(Job::Compute { id: i as u64, value }).unwrap();
    }

    // Send one Stop per worker
    for _ in 0..num_workers {
        job_tx.send(Job::Stop).unwrap();
    }

    // Collect results (one per input)
    let mut out = Vec::with_capacity(inputs.len());
    for _ in inputs {
        out.push(res_rx.recv().unwrap());
    }

    // Wait for all workers to finish
    for h in handles {
        let _ = h.join();
    }

    out
}
```

Teaching points:

- The mutex on `job_rx` is held only during `recv()` — a very short lock.
- Work happens **outside** the lock, enabling true parallelism.
- We send `num_workers` Stop messages so each worker sees exactly one.
- Results may arrive out of order — sort by `id` if order matters.

### Step 3: tests

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn pool_computes_all() {
        let inputs = vec![1, 2, 3, 4];
        let mut results = run_pool(&inputs, 2);
        results.sort_by_key(|r| r.id);
        let outputs: Vec<u64> = results.iter().map(|r| r.output).collect();
        // sum of 0..n = n*(n-1)/2
        assert_eq!(outputs, vec![0, 1, 3, 6]);
    }

    #[test]
    fn pool_handles_empty_input() {
        let results = run_pool(&[], 2);
        assert!(results.is_empty());
    }

    #[test]
    fn pool_more_workers_than_jobs() {
        let results = run_pool(&[10], 8);
        assert_eq!(results.len(), 1);
        assert_eq!(results[0].output, 45);  // 0+1+...+9
    }
}
```

---

## Code walkthrough: shutdown patterns

### Pattern 1: explicit Stop message

```rust
// Producer:
for _ in 0..num_workers {
    job_tx.send(Job::Stop).unwrap();
}
```

Clear and explicit. Every worker sees exactly one Stop and exits cleanly.

### Pattern 2: drop the sender

```rust
drop(job_tx);  // all senders dropped → Receiver::recv() returns Err
```

Workers exit when `recv()` returns `Err`. Simpler but requires no explicit message type for shutdown.

```rust
handles.push(thread::spawn(move || {
    loop {
        match rx.lock().unwrap().recv() {
            Ok(job) => process(job),
            Err(_)  => break,  // channel closed
        }
    }
}));
```

### Pattern 3: cancellation via a flag

For graceful shutdown with cleanup:

```rust
use std::sync::atomic::{AtomicBool, Ordering};

let stop_flag = Arc::new(AtomicBool::new(false));

// In each worker:
while !stop_flag.load(Ordering::Relaxed) {
    match rx.try_recv() {
        Ok(job) => process(job),
        Err(_)  => std::thread::sleep(Duration::from_millis(1)),
    }
}
```

This is less efficient (polling) but allows other threads to signal stop without closing the channel.

---

## Applications in systems programming

### Build system / task runner

A build system like `cargo` uses a worker pool to compile crates in parallel:

- Input channel: compile tasks (crate name, source files, flags).
- Output channel: compile results (success/error, artifacts).
- Workers: compiler invocations.

### Image processing pipeline

```rust
// Three-stage pipeline: decode → filter → encode
let (raw_tx, raw_rx)         = mpsc::channel::<RawImage>();
let (filtered_tx, filtered_rx) = mpsc::channel::<FilteredImage>();

thread::spawn(move || decoder_loop(raw_tx));
thread::spawn(move || filter_loop(raw_rx, filtered_tx));
thread::spawn(move || encoder_loop(filtered_rx));
```

Each stage runs independently; channels buffer between them. This is a classic **pipeline parallelism** pattern.

### Log aggregation

Multiple web server threads produce log entries; one dedicated thread drains the channel and writes to disk:

```rust
let (log_tx, log_rx) = mpsc::sync_channel::<LogEntry>(1000);

// In each request handler:
log_tx.send(entry).ok();  // best-effort; don't block on logging

// Log writer thread:
for entry in log_rx {
    write_to_disk(&entry);
}
```

Backpressure (bounded channel) prevents memory exhaustion if disk is slow.

---

## Challenges and extensions

1. **Ordered results**: modify `run_pool` to return results in the same order as inputs (sort by `id` or use an indexed mechanism).

2. **Error handling in workers**: change `work` to return `Result<u64, String>` and propagate errors through the result channel.

3. **Dynamic work injection**: modify the pool so the main thread can send more jobs while workers are running, not just a pre-defined list.

4. **Progress reporting**: add a third channel where workers send progress events (e.g., "worker N completed job M"). Print a progress bar in the main thread.

5. **`crossbeam` channels**: the `crossbeam-channel` crate provides multi-consumer channels (no mutex needed). Rewrite the pool using it and compare code clarity.

---

## Common mistakes

1. Assuming `Receiver` is clonable in stdlib (it's not — use `Arc<Mutex<Receiver>>` or `crossbeam`).
2. Holding the receiver lock while doing slow work (serializes everything through the lock).
3. Forgetting to send Stop signals (workers block on `recv()` forever).
4. Sending exactly the right number of Stops when workers might consume multiple (race condition — use channel close instead).
5. Not joining threads (they might be killed before finishing).

---

## Exercises

1. Change `work` to compute a fallible operation and send `JobResult` containing `Result<u64, String>`.
2. Add progress reporting: send a progress message after every N jobs.
3. Change shutdown to "drop the sender" and have workers exit on `recv` error.
4. (**harder**) Implement a retry mechanism: if a job fails, re-enqueue it up to 3 times before sending a failure result.

---

## References

- [The Rust Book, Ch. 16.2 — Message Passing](https://doc.rust-lang.org/book/ch16-02-message-passing.html)
- [`std::sync::mpsc` docs](https://doc.rust-lang.org/std/sync/mpsc/index.html)
- [`crossbeam-channel` crate](https://docs.rs/crossbeam-channel/latest/crossbeam_channel/)

## Recap

- Message passing reduces shared mutable state — communication is the synchronization.
- Design message types with enums; include IDs for request/response correlation.
- Stdlib `mpsc` is multi-producer, single-consumer; use `Arc<Mutex<Receiver>>` for multiple consumers.
- Worker pools need explicit shutdown semantics (Stop messages or channel close).
- Never hold the receiver lock while doing work — receive fast, process outside the lock.
