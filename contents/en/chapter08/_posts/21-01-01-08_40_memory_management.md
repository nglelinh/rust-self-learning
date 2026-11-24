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

When you write `Box::new(42)` or `Vec::with_capacity(1024)` in Rust, it feels effortless—you get a pointer to memory with no garbage collector pausing your threads. Yet beneath that single line sits one of the greatest illusions in computer engineering: **virtual memory**. The pointer `0x5555_5560_1000` in your Rust process is *not* a physical RAM chip address; it is a private illusion translated on every single CPU load/store by hardware page tables, while a user-space allocator carves OS-provided 4 KB pages into tiny, aligned chunks for your structs.

In this lesson, we pull back the curtain beneath `Box<T>` and `Vec<T>`—exploring virtual-to-physical paging, stack guard pages, how global allocators (`jemalloc`, `mimalloc`, `glibc malloc`) work, and how to plug in your own `#[global_allocator]`.

## Background Knowledge Map: How Does Memory Travel From Silicon DRAM to a Rust `Box<T>`?

Memory management in a systems language spans four stacked layers—from CPU hardware all the way up to Rust's `GlobalAlloc` trait:

1. **Virtual Memory, Page Tables & the TLB**:
   Every process believes it owns a contiguous 48-bit virtual address space ($256\text{ TB}$ on x86-64). The CPU's **Memory Management Unit (MMU)** splits each virtual address into fixed $4\text{ KB}$ ($4096$-byte) **pages** and walks a 4-level radix tree (page tables) to translate a virtual page number into a physical DRAM frame. Recent translations are cached in the **TLB (Translation Lookaside Buffer)** so hits take $< 1\text{ ns}$ instead of four DRAM reads.

2. **OS Page Allocation (`mmap` / `brk`) & Demand Paging**:
   The kernel does not hand out 16-byte chunks. It hands out whole $4\text{ KB}$ pages (via `mmap` or `brk`)—and thanks to **demand paging**, a newly mapped 100 MB buffer consumes **0 bytes of physical RAM** until your code actually touches a page, triggering a minor page fault that zeroes and maps a physical frame on the fly.

3. **User-Space Allocators (`GlobalAlloc`) & Alignment**:
   Asking the kernel for pages on every `String::push` would be far too slow. A user-space **allocator** requests large arenas from the OS, then subdivides them into size classes (e.g., 16 B, 32 B, 64 B, ..., 4 KB) with per-thread caches to avoid lock contention. Rust lets you swap the default system allocator for `jemalloc`, `mimalloc`, or a custom bump/arena allocator via `#[global_allocator]`.

4. **Stack Frames & Guard Pages**:
   Each thread's stack grows downward from a high virtual address. Immediately below the stack sits an unmapped **guard page** (no read/write permission): if infinite recursion or a huge `[0u8; 10_000_000]` local array overflows the stack, the CPU immediately raises a deterministic `SIGSEGV` instead of silently corrupting the heap.

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

### Deep-Dive Explanation: Hand-Walking a 48-Bit x86-64 Virtual Address and a Bump Allocator's Alignment Math (Read Slowly)

Let us trace two concrete calculations by hand: how the CPU hardware splits a Rust pointer into page-table indices, and how a custom bump allocator aligns a pointer without wasting or misaligning bytes.

#### Part 1: Splitting a 48-Bit Virtual Address (`9 + 9 + 9 + 9 + 12` Bits)

On x86-64 (4-level paging with $4\text{ KB} = 2^{12}$ byte pages), the lower 48 bits of any virtual pointer `va` are sliced into five fields:

```text
Bits:  [47..39]   [38..30]   [29..21]   [20..12]   [11..0]
        PML4 (9b)  PDPT (9b)   PD (9b)    PT (9b)   Offset (12b)
       (512 ents) (512 ents) (512 ents) (512 ents) (0..4095 bytes)
```

Suppose your Rust `Box<u64>` lives at virtual address `va = 0x0000_0080_4020_1018`. Convert to exact indices by bit-shifting:

| Level | Bit Slice | Formula | Hex Value | Decimal Index |
| :--- | :---: | :--- | :---: | :---: |
| **Offset inside 4 KB page** | `[11..0]` | `va & 0xFFF` | `0x018` | `24` (byte 24 in page) |
| **PT (Page Table)** | `[20..12]` | `(va >> 12) & 0x1FF` | `0x001` | `1` |
| **PD (Page Directory)** | `[29..21]` | `(va >> 21) & 0x1FF` | `0x001` | `1` |
| **PDPT (Page Dir Ptr)** | `[38..30]` | `(va >> 30) & 0x1FF` | `0x001` | `1` |
| **PML4 (Top Level)** | `[47..39]` | `(va >> 39) & 0x1FF` | `0x001` | `1` |

Verify by reconstructing:
$$(1 \cdot 2^{39}) + (1 \cdot 2^{30}) + (1 \cdot 2^{21}) + (1 \cdot 2^{12}) + 24 = \texttt{0x0080\_0000\_0000} + \texttt{0x4000\_0000} + \texttt{0x20\_0000} + \texttt{0x1000} + \texttt{0x18} = \texttt{0x0080\_4020\_1018}.$$
If `PT[1]` maps to physical frame `0x0000_1A00_0000`, the MMU replaces the top 36 bits and keeps the exact 12-bit offset `0x018`, accessing physical DRAM address **`0x0000_1A00_0018`**!

#### Part 2: Hand-Tracing Alignment Arithmetic Inside a Custom Bump Allocator

Suppose your custom bump allocator manages a 1024-byte arena starting at `addr = 0x1000`, with a cursor `next = 0x1000`:
1. **Alloc 1**: Request `Layout { size: 3, align: 1 }` (e.g., `[u8; 3]`):
   - Align `0x1000` up to multiple of `1`: `start = 0x1000`.
   - Advance cursor: `next = 0x1000 + 3 = 0x1003`. Returns `0x1000`.
2. **Alloc 2**: Request `Layout { size: 8, align: 8 }` (e.g., `u64`):
   - `0x1003` is **not** a multiple of `8`! Using the standard power-of-two bitmask formula:
     $$\text{align\_up}(x, a) = (x + a - 1) \mathbin{\&} \neg(a - 1)$$
     $$\text{align\_up}(\texttt{0x1003}, 8) = (\texttt{0x1003} + 7) \mathbin{\&} \neg 7 = \texttt{0x100A} \mathbin{\&} \texttt{0xFFF8} = \texttt{0x1008}.$$
   - Skip 5 padding bytes (`0x1003..0x1007`), advance cursor `next = 0x1008 + 8 = 0x1010`, and return **`0x1008`**.

Every real Rust `GlobalAlloc::alloc(&self, layout: Layout)` obeys that exact alignment contract—violating it is instant UB on strict-alignment architectures and in Rust's reference validity rules.

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

## Video tham khảo

- [How Computer Memory Works - Computerphile](https://www.youtube.com/watch?v=XETZoRYdtkw) — Computerphile, English. **xem trước.** Watch first: virtual memory, the page model this lesson starts from.
- [GlobalAllocator | Write Your Own Allocator in Rust and Learn How Heap Memory Works](https://www.youtube.com/watch?v=iqQq4sGBDOk) — Semicolon, English. A GlobalAlloc in Rust, matching the custom-allocator section.
- [Rust's Global Allocator](https://www.youtube.com/watch?v=TlkDwWGVZKg) — BehindBlueScreen, English. A longer look at Rust’s global allocator and what it is allowed to assume.
