---
layout: post
title: "02-11 Modern Applications — Memory Safety in Browsers, Kernels, and Cloud"
chapter: "02"
order: 11
owner: "OpenCode"
lang: en
categories:
  - chapter02
lesson_type: optional
---

This optional lesson does not reteach moves, borrows, slices, or `Option`/`Result`. It follows those rules out of the classroom and into systems that spent 2022–2026 rewriting C and C++ *because* use-after-free is no longer an acceptable cost of doing business.

## 60-minute teaching plan

- 0 to 10 min: Memory-safety roadmaps as policy (CISA 2023, ONCD 2024).
- 10 to 25 min: Linux 6.1 Rust support and why Binder was the proving ground.
- 25 to 40 min: How ownership maps onto kernel refcounts and browser heaps.
- 40 to 55 min: Cloud and consumer OS: Android, Windows, Cloudflare’s framing.
- 55 to 60 min: What ownership cannot save you from (logic bugs).

## Objectives

You will be able to point at a 2022–2026 production rewrite and say *which ownership rule* made the rewrite worth the cost. You will distinguish “Rust eliminated a class of CVEs” from “Rust made the program correct.” The mental-model shift is from “the borrow checker rejected my homework” to “the borrow checker is why a government agency and a phone OS vendor now publish memory-safe language roadmaps.”

## Prerequisites

Required Chapter 2: ownership, borrowing, slices, structs, and `Option`/`Result`. You should be able to predict a move error and explain why a `&mut` cannot alias. Kernel or browser internals are not required; treat those systems as *clients* of the same three ownership rules.

## Introduction

For thirty years, systems software treated use-after-free as a tax. Android’s Binder IPC driver — the path almost every app-to-system call takes on a phone — accumulated more than a decade of C complexity. Over half of Binder’s historical vulnerabilities were use-after-free. At Linux Plumbers 2023, Google engineers Alice Ryhl and Carlos Llamas described a Rust rewrite whose Drop glue was, in one slide, simply a closing brace.

In December 2023 CISA (with NSA, FBI, and international partners) published *The Case for Memory Safe Roadmaps*, asking vendors to say publicly how they will move new code to memory-safe languages. In February 2024 the White House Office of the National Cyber Director followed with *Back to the Building Blocks*. Ownership left the textbook and entered procurement language.

This lesson is about that translation. We will not re-derive the borrow checker. We will watch it show up in a kernel driver, a browser, and a cloud proxy, and we will be honest about the bugs it does *not* catch.

## Key Concepts

### Ownership as a CVE class, not a style guide

Chapter 2’s first rule — each value has one owner, and it is dropped when that owner goes away — is the same rule a C driver encodes with `kfree` on every error path. Humans miss a path; Rust’s `Drop` does not. The LWN write-up of the Binder talk is explicit: the C version’s cleanup was a goto-chain at the end of a function; the Rust version was the `}` that ends the scope.

That is not “nicer syntax.” It is a reduction in the set of states the program can occupy after a failure. When students ask why Rust forbids using a moved `String`, the production answer is: because the other name would be a dangling owner, and dangling owners are how Binder used to get compromised.

### Borrowing as an API contract with hardware and other processes

Kernel code cannot always *own* the bytes. A userspace buffer arriving through `copy_from_user` is borrowed for the duration of a syscall. A browser’s CSS tree is borrowed by the layout engine for one frame. The exclusive-`&mut` rule is how those systems say “no other writer exists,” without a sanitizer running in production.

Android and Linux still need `unsafe` to talk to C and to MMIO. The claim is not “zero unsafe.” The claim is “unsafe is a reviewed module boundary, and the safe API on top still obeys Chapter 2.”

### `Option` and `Result` as the public face of absence

Binder objects go away when a process dies. In C that was a nullable pointer plus a comment. In Rust it is `Option<Handle>` or a `Result` from a lookup. The type is the comment, and the compiler is the reviewer who never skips a day.

## Code Walkthroughs

We will not paste the Linux driver. We will grow a *tiny* userspace model of the bug class Binder was escaping, then let the compiler do what a decade of C review did not.

### Naive: a C-shaped cache of raw pointers

```rust
struct Handle {
    id: u32,
}

struct Table {
    // Pretend this is what C stored: "I might still be valid."
    slots: Vec<Option<*const Handle>>,
}

impl Table {
    fn get(&self, i: usize) -> Option<u32> {
        self.slots.get(i).and_then(|p| p.map(|raw| unsafe { (*raw).id }))
    }
}
```

This type-checks if you sprinkle `unsafe`. It does not *mean* anything. The moment a `Handle` is dropped while a slot still holds the pointer, you have Binder’s historical bug in twelve lines. The compiler is silent because you opted out.

### The error that should have been the design

Try to store a reference instead, with no lifetime:

```rust
struct Table {
    slots: Vec<Option<&Handle>>, // error: missing lifetime specifier
}
```

rustc will ask for a lifetime. That error is the lesson. A table that borrows handles *cannot* outlive them, and the compiler will not let you forget. If the table must outlive individual clients — Binder’s actual situation — you do not “add a lifetime and hope.” You change the ownership story: the table *owns* the data, or it holds an id.

### Idiomatic: own, or borrow with a name the type system can check

```rust
use std::collections::HashMap;

#[derive(Clone, Copy, PartialEq, Eq, Hash)]
struct HandleId(u32);

struct Handle {
    id: HandleId,
    payload: Vec<u8>,
}

struct Table {
    slots: HashMap<HandleId, Handle>,
}

impl Table {
    fn insert(&mut self, handle: Handle) -> HandleId {
        let id = handle.id;
        self.slots.insert(id, handle);
        id
    }

    fn borrow(&self, id: HandleId) -> Option<&Handle> {
        self.slots.get(&id)
    }

    fn take(&mut self, id: HandleId) -> Option<Handle> {
        self.slots.remove(&id)
    }
}

fn deliver(table: &Table, id: HandleId) -> Result<&[u8], &'static str> {
    let h = table.borrow(id).ok_or("dead binder")?;
    Ok(&h.payload)
}
```

Three Chapter 2 moves, all visible in production kernels:

1. The table owns `Handle` values. When a process dies, `take` drops the bytes. No `kfree` on a forgotten goto.
2. Callers borrow (`&Handle`, `&[u8]`) for the duration of `deliver`. No aliasing writer.
3. Absence is `Option`/`Result`, not a sentinel pointer.

Google’s Binder talk added an important caveat: Rust prevented the use-after-free *shape* of a bug that still existed as a logic error in both languages. In C that logic error became exploitable memory corruption. In Rust it became a confused mapping — still serious, much harder to turn into arbitrary write. Ownership is a class-killer, not a proof of total correctness.

## Examples

### Example 1 — Rust in Linux since 6.1

Linux 6.1 (December 2022) merged the first Rust support. That was not a driver showcase; it was the language being allowed in the tree. The following years added abstractions (`rust/kernel/`) and real drivers. Binder became the flagship rewrite because it is security-critical, heavily reviewed, and full of lifetime puzzles that C expresses as comments. By 2024–2026 the Rust Binder patches were on the lists with feature parity claims and throughput within a few percent of C; in September 2026 Google engineers proposed deleting the legacy `binder.c` once the Rust driver had run on devices.

Read that timeline as an ownership story. The kernel did not adopt Rust because it liked turbofish. It adopted Rust because Chapter 2’s drop and aliasing rules are cheaper than another decade of Binder CVEs.

### Example 2 — Browsers and the same heap rules

Firefox has shipped Rust in production components (including earlier Stylo work) for years; Servo’s revival in 2023–2025 put a full Rust engine back in public view. Chromium’s incremental Rust adoption follows the same pattern as Android: new code in a memory-safe language, old C++ left behind a FFI wall. A CSS or style struct that is borrowed for a frame is a `&T` with a lifetime the frame owns. A use-after-free in a style tree is exactly the homework error “I stored a reference in a struct that outlived the owner.”

### Example 3 — Policy catches up to the type system

CISA’s December 2023 roadmap document and the 2024 ONCD paper do not mention `borrow`. They mention *memory-safe languages*. When a vendor writes “new kernel components in Rust,” they are operationalizing Chapter 2 for auditors. Cloudflare’s Pingora posts (2022, open source 2024) make the same argument from the other direction: a proxy serving a trillion requests a day chose Rust so that memory unsafety would not be the incident.

Why forbid two `&mut` aliases? Because the incident report otherwise starts with “two cores freed the same buffer.”

## Applications in Systems Programming

**Driver APIs.** Safe wrappers over `copy_from_user` return owned `Vec<u8>` or borrowed slices tied to a lock guard. The guard’s lifetime is the borrow.

**IPC handles.** Binder, Unix fds, and cloud request contexts are all “ids in a table the runtime owns.” Once you have seen `HandleId`, you will see it in Tokio, in GPUI entity IDs, and in game engines.

**Shutdown.** Drop order is how a proxy releases sockets. A moved-from connection cannot be used; that is the “use of moved value” error wearing an SRE badge.

**Public APIs.** Returning `Option<&T>` instead of a nullable pointer is how crates.io libraries document absence without a paragraph of prose.

## Challenges and Extensions

Unsafe islands remain. Binder still talks to C. Browsers still have C++ heaps. The discipline is to keep those islands small and to put Chapter 2 types on the boundary.

Logic bugs remain. Confused mappings, incorrect authentication, and protocol desyncs are not ownership errors. Ryhl’s LPC example is the reading assignment for anyone who thinks “we rewrote it in Rust” is a complete sentence.

Scale remains. A borrow checker that is local to a function does not automatically give you a correct distributed cache. How would you design a shared cache in a multithreaded system so that ownership is still obvious — one owner per shard, borrows only under a lock, ids across shards?

## Exercises

1. **Conceptual.** Take a CVE write-up for a use-after-free in any C driver (Binder historical bugs are well documented). Rewrite the report as a Rust compiler error you would have hit first.
2. **Code fix.** The naive `Vec<Option<*const Handle>>` table above: replace it with owned storage so that `unsafe` disappears and `deliver` still compiles.
3. **Lifetime puzzle.** Add a `struct Cursor<'a> { table: &'a Table, id: HandleId }` and show why `Cursor` cannot outlive `Table`. Then replace the reference with `HandleId` only and explain what you lost and gained.
4. **Result discipline.** Make `deliver` distinguish “unknown id” from “payload too large to copy.” Do not add new theory; just split the error type.
5. **Design.** Sketch (no code required) an ownership diagram for a browser style tree: who owns the nodes, who borrows them during layout, what happens when a tab closes.

## References

- [A Rust implementation of Android's Binder](https://lwn.net/Articles/953116/) — LWN, Linux Plumbers 2023 write-up.
- [Linux 6.1 released](https://kernelnewbies.org/Linux_6.1) — first Rust support in mainline (December 2022).
- [The Case for Memory Safe Roadmaps](https://www.cisa.gov/resources-tools/resources/case-memory-safe-roadmaps) — CISA, 6 December 2023.
- [How we built Pingora](https://blog.cloudflare.com/how-we-built-pingora-the-proxy-that-connects-cloudflare-to-the-internet/) — Cloudflare, 14 September 2022.
- [The Rustonomicon — ownership and aliasing](https://doc.rust-lang.org/nomicon/) — for the unsafe edges these rewrites still need.

## Recap

- 2022–2026 rewrites (Linux Rust, Binder, browser components, edge proxies) are Chapter 2 applied to CVEs.
- Drop and exclusive borrows delete a vulnerability *class*; they do not delete logic bugs.
- Production APIs encode absence with `Option`/`Result` and identity with ids, not raw pointers.
- Policy documents now talk about memory-safe languages because ownership scaled past the classroom.
