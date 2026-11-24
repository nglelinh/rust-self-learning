---
layout: post
title: "05-26 Modern Applications — Async Rust in Production Systems"
chapter: "05"
order: 26
owner: "OpenCode"
lang: en
categories:
  - chapter05
lesson_type: optional
---

For years, high-performance cloud infrastructure—reverse proxies, load balancers, service meshes, and HTTP/3 gateways—belonged almost exclusively to C and C++ (NGINX, Envoy, HAProxy), with teams paying an endless tax in memory-safety CVEs and tail-latency debugging. Between 2022 and 2026, **async Rust** quietly rewrote that landscape: Cloudflare replaced NGINX across its global edge with **Pingora** (serving over **1 trillion requests per day** with 70% less CPU and 67% less memory!), `hyper` 1.0 stabilized Rust’s zero-copy HTTP core, and `tracing` unified structured async observability across millions of concurrent spans.

This optional capstone follows Chapter 5’s building blocks—Tokio tasks, bounded channels, cancellation, and structured spans—straight into the production cloud systems running the modern internet.

---

## Background Knowledge Map: Production Async Rust Infrastructure (2022–2026)

### 1. Multi-Threaded Work-Stealing Proxies (Cloudflare Pingora)
Classic NGINX spawns isolated OS worker processes that cannot easily share connection pools across cores. **Pingora** runs a single multi-threaded **Tokio** process with work-stealing scheduling and lock-free/shared `Arc` connection pools—improving upstream connection reuse while eliminating C buffer overflows by construction.
- **Core questions:** How does shared-process async Rust cut both memory usage and tail latency compared to multi-process C?
- **Real-world reach:** CDN edge proxies, API gateways, L4/L7 load balancers, and Kubernetes ingress controllers.

### 2. Modular HTTP & Middleware Composition (`hyper` 1.0 + `tower` + `axum`)
`hyper` 1.0 decoupled low-level HTTP/1 & HTTP/2 framing from specific runtimes and high-level routers. On top of it, `tower::Service` models every middleware—timeouts, rate limits, retries, load shedding—as a composable generic wrapper around `async fn call(req) -> Result<resp, err>`.
- **Core questions:** How does `tower` compose timeouts, concurrency limits, and tracing with zero dynamic allocation per middleware layer?
- **Real-world reach:** `axum` web backends, `tonic` gRPC services, and `reqwest` HTTP clients.

### 3. Span-Based Async Observability (`tracing`)
Plain `println!` or thread-local logs turn into unreadable soup when 10,000 async tasks interleave on 8 OS threads! **`tracing`** attaches structured **spans** (`request_id`, `user_id`) directly to each `Future` via `.instrument(span)` so every log line automatically inherits its exact async causal context—even as the task hops across worker threads!
- **Core questions:** Why do thread-local loggers fail in async Rust, and how does `tracing::Instrument` fix them?
- **Real-world reach:** OpenTelemetry distributed tracing, latency flamegraphs, and production incident debugging.

## 60-minute teaching plan

- 0 to 10 min: Why Cloudflare replaced nginx workers with a Rust async runtime.
- 10 to 25 min: Pingora’s model: multithreaded async, shared state, custom HTTP.
- 25 to 40 min: The Tokio/`hyper`/`axum` stack after 2023 and RFC 3185.
- 40 to 50 min: Cancellation, timeouts, and `JoinSet` as the production API.
- 50 to 60 min: `io_uring` experiments (`tokio-uring`, glommio, monoio) and when not to leave epoll.

## Objectives

You will be able to describe a real 2022–2026 service in Chapter 5 vocabulary: which work is a task, which state is `Arc`, which boundary is a channel, and which failure is a timeout rather than a panic. You will treat Pingora and Tokio not as brand names but as answers to “how many connections per core, and what happens when the peer stalls.” The mental-model shift is from “I spawned a future” to “I am scheduling a million futures and I will be paged when one of them never ends.”

## Prerequisites

Required Chapter 5: threads and `Mutex`, channels and work queues, futures and `async`/`await`, Tokio-style timeouts, unit/integration tests. You should know why holding `std::sync::Mutex` across `.await` is forbidden in the required lessons. You do not need to have operated a CDN.

## Introduction

On 14 September 2022 Cloudflare published *How we built Pingora*: an in-house Rust HTTP proxy serving over a trillion requests a day, using about a third of the CPU and memory of the nginx-based system it replaced. The architectural complaint was classic Chapter 5. nginx’s process-per-worker model left CPU on the table; extending it was painful; C made memory safety a standing incident risk. Pingora chose a multithreaded async runtime in Rust and, later, a custom HTTP implementation.

On 28 February 2024 they open-sourced the framework (Apache 2.0). By then Pingora had handled on the order of a quadrillion requests and was being positioned, with the ISRG Prossimo project, as infrastructure other people could run. The GitHub repo’s own README (still current in 2026) claims tens of millions of requests per second in Cloudflare’s network.

The rest of the ecosystem moved in parallel. `hyper` 1.0, `axum`’s rise as the Tokio-rs web framework, RFC 3185’s `async fn` in traits (Rust 1.75), and `tokio-console` turned “we wrote an async server” into something you can debug. This lesson is a field guide, not a second Tokio tutorial.

## Key Concepts

### Multithreaded async is a *scheduler* choice

Pingora’s public description is “async multithreaded,” not “one task per process.” That is Tokio’s default flavor (`rt-multi-thread`): a work-stealing pool, many sockets, shared configuration behind `Arc`. nginx’s model was isolation via processes; Pingora’s model is isolation via types plus tasks. Chapter 5’s `Arc<Mutex<T>>` versus channels debate is exactly the design review Cloudflare ran — they needed cross-core sharing that processes made expensive.

### HTTP is an async state machine

A proxy is not `read(); write();`. It is: accept, negotiate TLS, parse headers, acquire an upstream connection from a pool, stream a body with backpressure, honor a timeout, and cancel the loser of a `select`. Every one of those words is a Chapter 5 primitive. Pingora exists because doing that in C next to a process-model server did not compose.

### Cancellation is the production API

The required 05-24 lesson already said it: cancellation is cooperative. In a CDN, a canceled request *must* drop the upstream socket or you leak file descriptors at planetary scale. `timeout`, `JoinSet`, `CancellationToken`, and `Drop` on a connection guard are not extras; they are the product.

### Observability is part of the runtime

`tracing` spans plus `tokio-console` (spawned as a sidecar in many 2023–2026 shops) let you see *which task* is stuck. Without that, async Rust fails the same way Go goroutines fail: silently, on a random core, at 3 a.m.

## Code Walkthroughs

### Naive: sequential proxy

```rust
fn proxy_naive(mut client: TcpStream, mut upstream: TcpStream) -> std::io::Result<()> {
    let mut buf = [0u8; 8192];
    let n = client.read(&mut buf)?;
    upstream.write_all(&buf[..n])?;
    let n = upstream.read(&mut buf)?;
    client.write_all(&buf[..n])?;
    Ok(())
}
```

This is a textbook first server. It cannot overlap the two directions, cannot time out, and occupies a thread per connection. nginx workers scale this with processes; at Cloudflare’s 2022 load that was the problem statement.

### Compiler-shaped async: still missing policy

```rust
async fn proxy_better(mut client: TcpStream, mut upstream: TcpStream) -> std::io::Result<()> {
    let mut buf = vec![0u8; 8192];
    let n = client.read(&mut buf).await?;
    upstream.write_all(&buf[..n]).await?;
    Ok(())
}
```

You now have a future. You still have no bidirectional copy, no timeout, and a `Vec` allocated per request without pooling. rustc is happy; SRE is not.

### Idiomatic: copy both ways, bound the wait, cancel the rest

```rust
use tokio::io::{copy_bidirectional, AsyncWriteExt};
use tokio::time::{timeout, Duration};

async fn proxy(
    mut client: tokio::net::TcpStream,
    mut upstream: tokio::net::TcpStream,
) -> std::io::Result<()> {
    let work = copy_bidirectional(&mut client, &mut upstream);
    match timeout(Duration::from_secs(30), work).await {
        Ok(Ok(_)) => {}
        Ok(Err(e)) => return Err(e),
        Err(_) => {
            let _ = client.shutdown().await;
            let _ = upstream.shutdown().await;
        }
    }
    Ok(())
}
```

```rust
use tokio::task::JoinSet;

async fn accept_loop(listener: tokio::net::TcpListener) {
    let mut set = JoinSet::new();
    loop {
        let (client, _) = listener.accept().await.expect("accept");
        set.spawn(async move {
            // look up upstream, then proxy(client, upstream).await
            let _ = client;
        });
        while set.len() > 10_000 {
            let _ = set.join_next().await; // crude admission control
        }
    }
}
```

Pingora’s real code is a framework with hooks, not a 20-line function. The *shape* is the same: bounded concurrency, explicit shutdown, async IO. Admission control (`JoinSet` length, a semaphore, a queue) is Chapter 5 work queues wearing a 2024 CDN badge.

A common compiler error at this stage is `error: future cannot be sent between threads safely` because a `std::sync::MutexGuard` lived across `.await`. The required lessons covered the fix (`tokio::sync::Mutex` or release the guard). In Pingora-scale code that error is a *design* signal: you tried to hold a core-wide lock over a network wait.

## Examples

### Example 1 — Pingora as a case study

Read the 2022 engineering post for the resource numbers (about 70% less CPU and 67% less memory versus the old service at equal load) and the 2024 open-source post for the framework framing. Notice what they did *not* do: they did not put a GC next to the data plane, and they did not keep a process-per-core C core. They used Rust’s Send/Sync rules as the isolation boundary.

When you read `cloudflare/pingora` today, look at `pingora-core`, `pingora-proxy`, and `pingora-error`. That crate split is Chapter 1 workspaces plus Chapter 3 error types plus Chapter 5 tasks.

### Example 2 — The Tokio HTTP stack after hyper 1.0

`hyper` 1.0 (November 2023) split client/server more cleanly and became the substrate for `axum` and `reqwest` through 2024–2026. Combined with RFC 3185, service traits could use `async fn` without a mandatory box. A typical 2025 API crate is: `axum` router, `tower` middleware, `tracing` spans, Tokio multi-thread runtime. That is the stack you should be able to *name* after this course, even if this lesson does not make you reimplement it.

### Example 3 — Discord, Deno, and “async as a product language”

Discord’s well-known Go-to-Rust rewrite of Read States (2020) is slightly older than this window, but the 2022–2026 follow-through — more Discord backend in Rust, Deno’s runtime (Tokio + V8 + Rust) shipping as a product — is the same bet: one scheduler, memory safety, no GC pauses on the hot path. Deno in particular is a Chapter 5 + Chapter 6 (FFI to V8) system.

`tokio-console` is how those teams inspect the scheduler. If you can only `println!` in async code, you are not yet operating it.

## Applications in Systems Programming

**Edge and mesh proxies.** Pingora, Envoy-adjacent Rust proxies, and Linkerd’s Rust data plane parts all map connections to tasks.

**API gateways.** Timeouts and `JoinSet` cancellation are the difference between a lab demo and a Monday incident.

**Shared caches.** `Arc<AppState>` plus a channel for invalidation beats one giant `Mutex<HashMap>` held across `.await`.

**io_uring.** `tokio-uring`, glommio, and monoio (by ByteDance and others, 2022+) show Linux completion-based IO. They are optional backends, not a reason to abandon Tokio’s epoll default until you have measured.

## Challenges and Extensions

Structured concurrency is still a discipline, not a language guarantee. Forgetting to await a `JoinHandle` detaches work. `JoinSet` and `CancellationToken` exist because teams got burned.

`Send` bounds on trait futures (the `trait-variant` story from Chapter 3) show up the first time you store a service in an `Arc` and move it to Tokio’s pool.

Head-of-line blocking on a single task that does CPU work will stall a worker thread. The production fix is `spawn_blocking` or a dedicated pool — Chapter 5’s “do not do CPU on the async thread” rule at scale.

Reflect: how would you shut down a Pingora-like proxy so in-flight requests finish, new accepts stop, and idle upstreams drop — using only channels, `Notify`, and `Drop`?

---

## Worked Examples & Deep Dive

### Worked Example 1: Why Thread-Local Logging Fails in Async Rust vs. `tracing::Instrument`

Trace two concurrent HTTP requests (`req_1` and `req_2`) multiplexing on **Worker Thread 1**:

| Event | Naive Thread-Local Logger | `tracing` + `.instrument(span)` |
|---|---|---|
| `t=1`: `req_1` starts on Thread 1 | Sets thread-local `id=1` | Enters `span(req_1)` on poll start |
| `t=2`: `req_1` hits `.await` (yields) | Thread-local still `id=1`! | **Exits `span(req_1)` automatically on `Poll::Pending`!** |
| `t=3`: `req_2` resumes on Thread 1 | Logs under **`id=1` (CORRUPTED!)** | **Enters `span(req_2)` → logs accurately under `req_2` ✓** |
| `t=4`: `req_1` resumes on **Thread 4** | Thread 4 has **no `id`!** | **Enters `span(req_1)` on Thread 4 → intact across threads ✓** |

### Worked Example 2: Back-of-the-Envelope Resource Math — Threads vs. Async Tasks at 50,000 Connections

Compare holding **50,000 idle TLS/HTTP connections** on a 16-core server:
- **1 OS thread per connection** (2 MB default stack): `50,000 × 2 MB = 100 GB RAM` just in stack reservations + massive kernel scheduler thrashing.
- **Async Rust (Tokio + `hyper`)** (~4 KB per task + 16 worker threads): `50,000 × 4 KB ≈ 200 MB RAM` (**500× less memory!**) + zero kernel thread-context switches between tasks on the same core!

### Deep-Dive Explanation: How Production Async Rust Stays Fast Under Load (Read Slowly)

#### Step 1: Bound concurrency at the front door (`tower::limit::ConcurrencyLimit` / `Semaphore`)
Accepting unlimited concurrent requests turns a brief downstream slowdown into an OOM crash. Always place a hard concurrency cap and load-shedding timeout at the outermost middleware layer.

#### Step 2: Share pools with `Arc`, never clone heavy clients per request
Database pools (`sqlx::PgPool`), HTTP clients (`reqwest::Client`), and TLS configs are designed around cheap internal `Arc` cloning so connection pools stay warm across all worker threads.

#### Step 3: Instrument every spawned task explicitly
When you call `tokio::spawn(work.instrument(tracing::info_span!("bg_job", id)))`, the child task carries its parent’s structured trace ID cleanly into OpenTelemetry/Jaeger without losing causality.

#### Step 4: Measure tail latency (`p99`), not just averages
In cooperative async runtimes, one stray 50 ms blocking call on a worker thread might barely move average latency while blowing up `p99` latency for every task parked behind it. Use `tokio-console` and span-busy-time histograms to catch executor stalls immediately.

## Exercises

1. **Conceptual.** In four sentences, contrast nginx’s worker-process model with Pingora’s multithreaded async model. Which Chapter 5 primitive replaces process isolation?
2. **Code fix.** Add a 5-second timeout and a bidirectional copy to `proxy_better`. Confirm that dropping the future closes both sockets (write a test with `tokio::io::duplex` if you can).
3. **Backpressure.** Bound `accept_loop` with a `tokio::sync::Semaphore` instead of `JoinSet` length. What happens to `accept` when permits are gone?
4. **Observability.** Add a `tracing::info_span!("proxy")` around `proxy`. List two fields you would attach in production (peer addr, upstream name).
5. **Implementation.** Write a tiny in-process “upstream pool”: an `mpsc` of `TcpStream`s, a worker that times out idle connections, and a client that `timeout`s on acquire. No new theory — compose 05-22 and 05-24.

## References

- [How we built Pingora](https://blog.cloudflare.com/how-we-built-pingora-the-proxy-that-connects-cloudflare-to-the-internet/) — Cloudflare, 14 September 2022.
- [Open sourcing Pingora](https://blog.cloudflare.com/pingora-open-source/) — Cloudflare, 28 February 2024.
- [cloudflare/pingora](https://github.com/cloudflare/pingora) — framework crates and MSRV notes.
- [Announcing `async fn` in traits](https://blog.rust-lang.org/2023/12/21/async-fn-rpit-in-traits/) — 21 December 2023.
- [Tokio tutorial](https://tokio.rs/tokio/tutorial) and [tokio-console](https://github.com/tokio-rs/console).

## Recap

- 2022–2026 cloud Rust is Chapter 5 at CDN scale: tasks, `Arc` config, timeouts, cancellation.
- Pingora is the canonical public case study; Tokio/`hyper`/`axum` is the canonical stack.
- Cancellation and admission control are the product, not extras.
- If you cannot see tasks (`tracing`, `tokio-console`), you cannot operate them.

## Video tham khảo

- [Getting started with Tokio. The ultimate starter guide to writing async Rust.](https://www.youtube.com/watch?v=dOzrO40jgbU) — Dreams of Code, English. **xem trước.** Watch first: tasks on a Tokio runtime, the vocabulary this lesson uses for a real service.
- [Rust select!: Cancellation & Cancel Safety Explained](https://www.youtube.com/watch?v=0k2Uo_IQlNI) — Semicolon, English. Cancellation when a peer stalls, which is the failure mode the lesson says you will be paged for.
- [1 Hour Dive into Asynchronous Rust](https://www.youtube.com/watch?v=0HwrZp9CBD4) — Ardan Labs, English. A one-hour tour of async Rust at the scale of many connections, after the two shorter videos.
