---
layout: post
title: "08-40 Memory Management — Virtual Memory, the Heap, and Custom Allocators"
chapter: "08"
order: 40
owner: "OpenCode"
lang: en
categories:
  - chapter08
lesson_type: required
---

Rust gives you direct control over memory without a garbage collector. This lesson goes beneath the `Box<T>` and explains how the OS provides virtual memory, how the allocator manages the heap, and how to write or swap in a custom allocator when the default is not the right fit.

## 60-minute teaching plan

- 0 to 10 min: Virtual memory — pages, page tables, the kernel's view vs the program's view.
- 10 to 25 min: Stack vs heap: what Rust puts where, and why.
- 25 to 40 min: How the global allocator works: `malloc` internals, fragmentation.
- 40 to 55 min: Custom allocators — `#[global_allocator]`, `jemalloc`, `mimalloc`, arena allocators.
- 55 to 60 min: Exercises and recap.

## Learning goals

By the end of this lesson, students can:

- Explain virtual memory, pages, and why a process's address space is not physical RAM.
- Predict whether a value lives on the stack or heap given its type.
- Measure heap allocations and understand fragmentation.
- Swap in a custom global allocator and explain the trade-offs.

## Virtual memory

### Pages and the MMU

Physical RAM is divided into fixed-size **pages** (typically 4 KiB on x86-64). The CPU's **Memory Management Unit (MMU)** translates every memory access from a **virtual address** (what your program sees) to a **physical address** (actual RAM location) using **page tables** maintained by the kernel.

Consequences:
- Two processes with the same virtual address are accessing completely different physical memory.
- Pages can be **swapped** to disk — the OS transparently moves cold pages out and brings them back on access (a **page fault**).
- The kernel can mark pages read-only, execute-only, or no-access — this is how stack overflow protection and write protection work.

### A process's virtual address space (x86-64 Linux)

```
High addresses
┌──────────────────────────┐  ~0xFFFFFFFFFFFFFFFF
│  Kernel space            │  (not accessible from user mode)
├──────────────────────────┤
│  Stack (grows down)      │  thread stacks
├──────────────────────────┤
│  mmap region             │  shared libraries, mmap'd files
├──────────────────────────┤
│  Heap (grows up)         │  malloc / Box / Vec allocations
├──────────────────────────┤
│  BSS (zero-init globals) │
│  Data (initialized)      │
│  Text (code)             │
└──────────────────────────┘  ~0x0000000000400000
Low addresses
```

`/proc/self/maps` on Linux shows the live mapping:

```bash
cat /proc/self/maps
```

### Page faults

When a program accesses a virtual address whose page is not currently in physical RAM, the CPU raises a **page fault** — a hardware interrupt handled by the kernel. The kernel:
1. Locates the page on disk (or the backing file).
2. Allocates a physical frame.
3. Updates the page table.
4. Resumes the instruction.

Major page faults are expensive (~100 µs). Memory-mapped files and large allocations incur them on first access.

## Stack vs heap in Rust

### Stack

- Fixed-size frame allocated at function call, freed on return.
- Extremely fast — just a register decrement.
- Limited (typically 8 MiB per thread on Linux).
- Holds: local variables with sizes known at compile time.

```rust
let x: i32 = 42;           // stack
let arr: [u8; 1024] = [0; 1024];  // stack — 1 KiB, fine
```

### Heap

- Dynamic allocation managed by the allocator.
- Slower (allocator bookkeeping, possible kernel calls).
- Essentially unbounded (limited by virtual address space and RAM).
- Holds: data whose size is unknown at compile time, or that must outlive its creating scope.

```rust
let v: Vec<i32> = Vec::with_capacity(1_000_000);  // heap: ~4 MB
let s: String = "hello".to_string();               // heap
let b: Box<[u8]> = vec![0u8; 65536].into();       // heap
```

### The `Box<T>` contract

`Box<T>` is exactly one heap allocation containing one `T`. When the `Box` is dropped, the memory is freed. No more, no less. This is why `Box<T>` has zero overhead beyond the allocation itself.

## How the global allocator works

Rust uses `ptmalloc` (glibc's `malloc`) by default on Linux, `jemalloc` historically (now optional), and the system allocator on other platforms.

### What malloc does

The allocator maintains **free lists** — linked lists of freed blocks grouped by size class. On `malloc(n)`:
1. Find a free block of size ≥ n in the appropriate size class.
2. If none, request more memory from the OS via `sbrk()` or `mmap()`.
3. Return a pointer to the block.

On `free(ptr)`:
1. Return the block to the appropriate free list.
2. Optionally coalesce adjacent free blocks.

### Fragmentation

**External fragmentation**: many small free blocks exist but no contiguous block is large enough for a new allocation.
**Internal fragmentation**: the allocator rounds up to the next size class — a 17-byte allocation might consume a 32-byte slot.

Tools for measuring heap usage:

```bash
# Linux: use valgrind massif
valgrind --tool=massif ./your-program
ms_print massif.out.* | head -40

# Or heaptrack
heaptrack ./your-program
heaptrack_gui heaptrack.your-program.*.zst
```

In Rust with a tracking allocator:

```toml
[dependencies]
tracking-allocator = "0.4"
```

## Custom global allocators

### Swapping in jemalloc

`jemalloc` reduces fragmentation in multi-threaded workloads through per-thread arenas:

```toml
[dependencies]
tikv-jemallocator = "0.5"
```

```rust
#[global_allocator]
static ALLOC: tikv_jemallocator::Jemalloc = tikv_jemallocator::Jemalloc;

fn main() { /* rest unchanged */ }
```

That's it. Every `Box`, `Vec`, `String` allocation now goes through jemalloc.

### mimalloc — Microsoft's high-performance allocator

```toml
[dependencies]
mimalloc = "0.1"
```

```rust
#[global_allocator]
static GLOBAL: mimalloc::MiMalloc = mimalloc::MiMalloc;
```

`mimalloc` outperforms the system allocator on many workloads, especially those with many short-lived allocations.

### Writing a simple arena allocator

An arena allocates from a pre-reserved block in bulk and frees everything at once. Zero per-object overhead; ideal for request-scoped allocations in a server.

```rust
use std::alloc::{GlobalAlloc, Layout};
use std::cell::Cell;

pub struct BumpAllocator {
    buffer: [u8; 1 << 20],   // 1 MiB static buffer
    offset: Cell<usize>,
}

impl BumpAllocator {
    pub const fn new() -> Self {
        Self {
            buffer: [0; 1 << 20],
            offset: Cell::new(0),
        }
    }

    pub fn reset(&self) {
        self.offset.set(0);  // free everything at once
    }
}

unsafe impl GlobalAlloc for BumpAllocator {
    unsafe fn alloc(&self, layout: Layout) -> *mut u8 {
        let start = self.offset.get();
        let aligned = (start + layout.align() - 1) & !(layout.align() - 1);
        let end = aligned + layout.size();
        if end > self.buffer.len() {
            return std::ptr::null_mut();   // OOM
        }
        self.offset.set(end);
        self.buffer.as_ptr().add(aligned) as *mut u8
    }

    unsafe fn dealloc(&self, _ptr: *mut u8, _layout: Layout) {
        // Individual frees are no-ops; call reset() to free all
    }
}
```

This is a toy example — a real arena needs thread safety (`Mutex` or per-thread arenas) and out-of-buffer fallback.

### Measuring allocator impact

```rust
fn benchmark_vec(n: usize) -> Vec<u64> {
    let mut v = Vec::with_capacity(n);
    for i in 0..n {
        v.push(i as u64);
    }
    v
}
```

Run with `cargo bench` and swap `#[global_allocator]` between system, jemalloc, and mimalloc to measure the difference.

## Stack size and large allocations

Default thread stack size is 8 MiB on Linux. For recursive algorithms or large stack arrays:

```rust
// Spawn a thread with a larger stack
std::thread::Builder::new()
    .stack_size(32 * 1024 * 1024)  // 32 MiB
    .spawn(|| {
        deep_recursion(10_000);
    })
    .unwrap()
    .join()
    .unwrap();
```

Prefer heap allocation (`Vec`, `Box`) over large stack arrays for anything beyond a few KiB.

## In-class exercises

1. Write a program that allocates a `Vec<u8>` of 1 GiB and measures how long the first write to each 4 KiB page takes. Explain the pattern using page fault theory.
2. Swap in `mimalloc` or `jemalloc` as the global allocator for a program that does many small allocations. Benchmark with `criterion` and compare throughput.
3. Read `/proc/self/status` (on Linux) from Rust and parse `VmRSS` (resident set size) and `VmPeak` (peak virtual memory). Display them before and after a large allocation.
4. Implement a thread-safe version of the bump allocator above using `AtomicUsize` instead of `Cell<usize>`.

## Recap

- Virtual memory isolates processes and allows pages to be swapped; the MMU translates addresses.
- Stack is fast but limited; heap is flexible but has allocator overhead.
- `Box<T>` is one heap allocation; `Vec<T>` is a growable heap buffer.
- The global allocator is swappable via `#[global_allocator]`; jemalloc and mimalloc reduce fragmentation in multi-threaded workloads.
- Arenas are the fastest allocator for same-lifetime objects — amortize the allocation cost and free everything at once.
