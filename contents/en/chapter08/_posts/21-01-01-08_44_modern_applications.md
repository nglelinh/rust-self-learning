---
layout: post
title: "08-44 Modern Applications — Embedded, Kernel, and Certified Systems"
chapter: "08"
order: 44
owner: "OpenCode"
lang: en
categories:
  - chapter08
lesson_type: optional
---

This optional lesson does not reteach `#![no_std]`, MMIO, syscalls, or the Linux module sketch from 08-43. It maps those ideas onto 2022–2026 *deployments*: Embassy on battery devices, Ferrocene as a qualified rustc, Rust-for-Linux Binder going from LPC talk to “delete `binder.c`,” and WASI 0.2 as a userspace “OS ABI” written in Rust.

## 60-minute teaching plan

- 0 to 10 min: Three OS-shaped products — kernel driver, bare-metal async, certified toolchain.
- 10 to 25 min: Embassy: async as an RTOS replacement (and its limits).
- 25 to 40 min: Rust for Linux + Binder as an IPC rewrite (ties to 08-41).
- 40 to 50 min: Ferrocene and ISO 26262 — when “it compiles” is not an argument.
- 50 to 60 min: WASI 0.2 / Firecracker-adjacent isolation: OS concepts without a full kernel.

## Objectives

You will be able to name a 2022–2026 system for each Chapter 8 topic: processes/signals → Firecracker/WASI guests; files and fds → WASI 0.2 filesystem/sockets; memory → Embassy’s no-heap executor and kernel allocators; IPC → Binder; syscalls → `nix` still, plus WIT imports; kernel/embedded → RFL + Embassy + Ferrocene. The mental-model shift is from “OS homework on Linux” to “Rust is now how vendors *replace* pieces of the OS, and how they certify the compiler that builds those pieces.”

## Prerequisites

Required Chapter 8 lessons on processes, files, virtual memory, IPC, and syscalls/`nix`. The optional 08-43 `no_std`/kernel module lesson is strongly recommended; this one assumes you have seen a panic handler and an `extern "C"` entry. Automotive safety standards are *named*, not taught.

## Introduction

Linux 6.1 (December 2022) let Rust into the kernel tree. That was a language-policy event. The product event came later: Android’s Binder, the IPC every app uses, was rewritten in Rust (LPC 2023; patches through 2024–2026). In September 2026 Google engineers proposed removing the C `binder.c` after the Rust driver had run on devices with feature parity and comparable throughput. Chapter 8’s IPC lecture is no longer hypothetical.

On the other side of the syscall boundary, Embassy made `async/await` a first-class embedded scheduler: no heap required, interrupt-driven wakeups, multiple priority executors. Tweede Golf’s public “async Rust vs RTOS” measurements (and Embassy’s own README) are the citations students should read before saying “we need FreeRTOS for concurrency.”

A third axis appeared in 2023: Ferrocene, Ferrous Systems’ downstream rustc, qualified by TÜV SÜD for ISO 26262 (ASIL D) and IEC 61508 — later also IEC 62304. You can now *buy a story* about rustc in a car. That is an OS-adjacent application of Chapter 1’s toolchain and Chapter 6’s unsafe discipline.

This lesson is the field report. 08-43 remains the mechanism lecture.

## Key Concepts

### Async as a replacement kernel on the metal

Embassy’s executor turns each `async fn` into a state machine the compiler allocates in static storage. When a future waits on a UART or a timer, the core sleeps. That is Chapter 5’s cooperative scheduling *and* Chapter 8’s “there is no OS.” It is not magic: you still write `#![no_std]`, you still map ARM vectors, you still do not block.

```rust
// Shape — Embassy on a Cortex-M; see embassy.dev for the real HAL.
#[embassy_executor::main]
async fn main(_sp: embassy_executor::Spawner) {
    loop {
        // await a timer or a GPIO edge; the executor idles the core.
        Timer::after_secs(1).await;
    }
}
```

Compare to a process on Linux: the kernel’s scheduler is the Embassy executor; a signal is an interrupt; a pipe is a `channel` in static RAM.

### Kernel Rust is IPC and ownership, not “rewrite Linux”

Binder is Chapter 8 IPC: handles, shared mappings, death notifications. The LWN 2023 write-up is the assigned reading: Drop replaces goto-cleanup; use-after-free stops being the default CVE. The 2026 deletion proposal is the punchline — optional, but historically adjacent to this course’s “today” date. You should still treat Binder as *one driver*, not as “Linux is now Rust.”

### Qualification is a toolchain application

Ferrocene is rustc plus a quality-management story: language specification, tests, long-term support, TÜV SÜD certificates. The 2023 open-source announcement and the 2025 Ferrous safety-critical paper are the citations. Students often ask “can I use Embassy in a car?” The honest answer: the *language* can be qualified via Ferrocene; *your* crate graph and HAL are a separate argument. Ferrocene’s pitch is compatibility with upstream rustc so you do not maintain `#ifdef COMPILER`.

### WASI 0.2 is an OS interface without your kernel

WASI 0.2 (25 January 2024) gives clocks, random, filesystem, sockets, CLI, HTTP as WIT worlds. Wasmtime implements them in Rust. A guest process has fds and a clock without `int 0x80` on your laptop. Firecracker (AWS, Rust since 2018, still the production MicroVM for Lambda/Fargate-style isolation through 2026) is the heavier cousin: a real Linux guest, a tiny VMM. Both are Chapter 8 “what is a process” answers for the cloud.

## Code Walkthroughs

### Naive: bare metal as a Linux program

```rust
fn main() {
    std::fs::write("/sys/class/gpio/gpio17/value", "1").unwrap();
}
```

This is a fine Linux userspace blink, and a category error for Embassy. There is no `/sys` on a Cortex-M. 08-39’s file lecture does not transfer until *you* implement the filesystem (or you are on WASI with a host that did).

### First embedded shape: still blocking

```rust
#![no_std]
#![no_main]

#[no_mangle]
pub extern "C" fn _start() -> ! {
    loop {
        // busy-wait; the battery dies; Chapter 8 timers exist for a reason
    }
}
```

08-43 already showed this entry point. The *application* failure is the busy loop. Embassy’s entire reason to exist is to replace that loop with `await` + WFI.

### Idiomatic userspace: treat Binder-like IPC as owned handles

```rust
use std::collections::HashMap;

struct BinderLike {
    nodes: HashMap<u64, Vec<u8>>,
}

impl BinderLike {
    fn transact(&mut self, handle: u64, payload: Vec<u8>) -> Result<Vec<u8>, &'static str> {
        let node = self.nodes.get_mut(&handle).ok_or("dead")?;
        // own the reply bytes — do not return a pointer into `node` to another process
        Ok(node.clone())
    }

    fn death(&mut self, handle: u64) {
        self.nodes.remove(&handle); // Drop is the cleanup path
    }
}
```

This is a toy. The Linux driver is thousands of lines of carefully reviewed `unsafe` next to safe maps. The walkthrough exists so you can *see* 08-41 (IPC) + 02-11 (ownership) in one type. The production citations are LWN and the kernel list, not this struct.

### Idiomatic cloud: WASI as the syscall table

```rust
fn main() {
    // Compiled for wasm32-wasip2, this write goes through wasi:cli / wasi:io.
    println!("guest fd 1 is a WASI stream, not necessarily a Unix pipe");
}
```

Same Chapter 1 crate, different ABI. The host (Wasmtime) is a Rust program implementing Chapter 8 primitives. That is the mind-bender this lesson wants: *you can be the kernel*.

## Examples

### Example 1 — Embassy in the field

Embassy’s book and the `embassy-rs/embassy` README (actively developed through 2026) document HALs for nRF, STM32, RP2040, and others. Ferrous Systems’ 2025 safety-critical overview mentions Embassy in the context of connected locks (Akiles): not ASIL D, but long-lived, rarely updated, network-facing firmware — exactly where memory unsafety is expensive. Read Tweede Golf’s RTOS comparison before you invent another thread-and-mutex firmware.

### Example 2 — Binder and Rust for Linux

LWN, *A Rust implementation of Android’s Binder* (2023), plus the later lore of `CONFIG_ANDROID_BINDER_IPC_RUST`, is the assigned path. Pair it with 08-41. Ask: which IPC mechanism (sockets, shm, Binder) is the driver *simulating* for userspace? (Binder: object-ish RPC + fd passing + death notification.)

### Example 3 — Ferrocene and WASI as two “certified-enough” stories

Ferrocene (ferrocene.dev): qualified compiler for automotive/industrial/medical targets (Linux, QNX, bare metal on listed Arm). WASI 0.2: portable *guest* ABI with two independent implementations at launch (Wasmtime, `jco`). One story certifies the compiler; the other certifies the syscall interface. Cloud vendors (Firecracker, Wasmtime on the edge) pick the isolation granularity they can afford.

## Applications in Systems Programming

**Firmware products.** Embassy + `embedded-hal` + maybe Ferrocene when a regulator enters the room.

**Phone OS internals.** Rust Binder, other RFL drivers (see kernel newbies / RFL updates through 2025–2026).

**Confidential / multi-tenant compute.** Firecracker MicroVMs; WASI guests for finer-than-VM isolation.

**Teaching OS design.** Theseus and Tock remain research/teaching kernels in Rust; they are extensions, not this lesson’s core citations.

## Challenges and Extensions

Embassy async is cooperative. A blocking I2C driver in an `async fn` starves the chip the same way a blocking `std::sync::Mutex` starves Tokio.

Kernel Rust still needs C. The safe `kernel` crate is incomplete; `unsafe` and bindings are the job.

Qualification does not cover your `unsafe` or your HAL. Ferrocene qualifies the *compiler*.

WASI 0.2 is not POSIX. `fork` is deliberately missing. If your mental model of a process requires fork/exec, you are on Firecracker or real Linux, not WASI.

Reflect: how would you design a shared page cache (08-40) for a Binder-like driver so that userspace mappings die when `Drop` runs — without a use-after-free window?

## Exercises

1. **Conceptual.** Match each system to a Chapter 8 lesson: Embassy, Binder, WASI 0.2, Ferrocene, Firecracker. One sentence each.
2. **Contrast.** List three things `std::fs` gives you that Embassy does not, and three things Embassy gives you that a Linux process does not (hint: power, latency, no page faults).
3. **IPC.** Using only 08-41 vocabulary, explain why Binder death notification is a `Drop`/ownership problem (02-11) as much as a syscall problem.
4. **Toolchain.** Read Ferrocene’s public qualification claims and write four sentences on what is *not* qualified when you `cargo add embassy-nrf`.
5. **Implementation.** Port the `BinderLike` toy so `transact` returns `bytes::Bytes` (owned, cheap to share) and write a test that `death` makes the next `transact` fail. No kernel build.

## References

- [Embassy book](https://embassy.dev/book/) and [embassy-rs/embassy](https://github.com/embassy-rs/embassy).
- [A Rust implementation of Android's Binder](https://lwn.net/Articles/953116/) — LWN / LPC 2023.
- [Ferrocene](https://ferrocene.dev/) and [Open Sourcing Ferrocene](https://ferrous-systems.com/blog/ferrocene-open-source/) — 2023 qualification path.
- [WASI 0.2 Launched](https://bytecodealliance.org/articles/WASI-0.2) — 25 January 2024.
- [Firecracker](https://firecracker-microvm.github.io/) — AWS MicroVM, Rust VMM.
- Lesson 08-43 in this chapter — mechanisms this field report assumes.

## Recap

- 2022–2026 OS-shaped Rust is Embassy (metal), Binder/RFL (kernel IPC), Ferrocene (certified rustc), WASI/Firecracker (guest isolation).
- This lesson does not replace 08-43; it cites the products that use those mechanisms.
- Async on bare metal and qualification of the compiler are now industry paths, not blog dreams.
- You can be the kernel (Wasmtime) or live without one (Embassy); Chapter 8 vocabulary still applies.
