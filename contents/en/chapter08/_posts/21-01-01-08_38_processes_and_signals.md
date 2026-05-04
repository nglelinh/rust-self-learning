---
layout: post
title: "08-38 Processes and Signals"
chapter: "08"
order: 38
owner: "OpenCode"
lang: en
categories:
  - chapter08
lesson_type: required
---

A process is the OS's fundamental unit of isolation: its own address space, file descriptor table, and scheduling context. This lesson covers how to spawn and supervise child processes from Rust, how signals interrupt execution, and how to handle both correctly.

## 60-minute teaching plan

- 0 to 10 min: What a process is — address space, PID, file descriptors, exit code.
- 10 to 25 min: Spawning processes with `std::process::Command`.
- 25 to 40 min: Signals — what they are, the Unix signal model, safe handling in Rust.
- 40 to 55 min: Process supervision: reaping children, exit status, `SIGCHLD`.
- 55 to 60 min: Exercises and recap.

## Learning goals

By the end of this lesson, students can:

- Spawn a child process, pipe its stdout/stderr, and collect its exit code.
- Explain what a signal is and why signal handlers are restricted to async-signal-safe operations.
- Use `signal-hook` to receive signals safely without undefined behavior.
- Detect and handle child process termination.

## What is a process?

When the kernel runs a program it creates a **process**: an isolated execution environment with:

- **Virtual address space** — the process sees its own address range; the kernel maps physical memory behind the scenes.
- **PID** — a unique integer identifier.
- **File descriptor table** — references to open files, sockets, pipes, and special files like `stdin`/`stdout`/`stderr`.
- **Exit code** — a small integer returned to the parent when the process terminates (0 conventionally means success).

Processes are isolated by default. One process cannot read another's memory unless they explicitly share it (see lesson 08-41 on IPC).

## Spawning processes

### Basic spawn

```rust
use std::process::Command;

let status = Command::new("ls")
    .arg("-la")
    .status()
    .expect("failed to start ls");

println!("exited with: {status}");
assert!(status.success());
```

`status()` waits for the child and returns its exit status. Use `output()` to also capture stdout and stderr:

```rust
let output = Command::new("git")
    .args(["log", "--oneline", "-5"])
    .output()
    .expect("failed to run git");

let stdout = String::from_utf8_lossy(&output.stdout);
let stderr = String::from_utf8_lossy(&output.stderr);
println!("{stdout}");
if !output.status.success() {
    eprintln!("error: {stderr}");
}
```

### Piping I/O

```rust
use std::process::{Command, Stdio};

let mut child = Command::new("cat")
    .stdin(Stdio::piped())
    .stdout(Stdio::piped())
    .spawn()
    .expect("failed to spawn cat");

// Write to child's stdin
use std::io::Write;
child.stdin.take().unwrap().write_all(b"hello\n").unwrap();

let output = child.wait_with_output().unwrap();
println!("cat said: {}", String::from_utf8_lossy(&output.stdout));
```

`stdin.take()` gives ownership of the pipe handle; dropping it closes the pipe, which sends EOF to the child.

### Environment and working directory

```rust
Command::new("cargo")
    .arg("build")
    .current_dir("/path/to/project")
    .env("RUST_LOG", "debug")
    .env_remove("CARGO_INCREMENTAL")
    .status()
    .unwrap();
```

### Non-blocking: spawn then poll

```rust
let mut child = Command::new("long-running-task").spawn().unwrap();

// Do other work...

match child.try_wait().unwrap() {
    Some(status) => println!("finished: {status}"),
    None => {
        println!("still running, killing...");
        child.kill().unwrap();
        child.wait().unwrap();
    }
}
```

## Signals

### What is a signal?

A signal is an asynchronous notification delivered by the kernel (or another process) to a process. Examples:

| Signal | Number | Default action | When sent |
|---|---|---|---|
| `SIGTERM` | 15 | Terminate | `kill <pid>`, graceful shutdown request |
| `SIGINT` | 2 | Terminate | Ctrl-C in the terminal |
| `SIGKILL` | 9 | Terminate (uncatchable) | `kill -9 <pid>` |
| `SIGCHLD` | 17 | Ignore | Child process changed state |
| `SIGHUP` | 1 | Terminate | Terminal closed; often used to reload config |
| `SIGUSR1/2` | 10/12 | Terminate | User-defined |

### Why raw signal handlers are dangerous

Signal handlers execute asynchronously — they can interrupt any instruction, including allocator locks or mutex acquisitions. Only **async-signal-safe** functions (a small OS-defined list) may be called from a raw handler. Almost nothing in a Rust program qualifies.

The safe pattern is: in the handler, write a byte to a self-pipe or set an atomic flag; in the main loop, read that flag.

### signal-hook: the safe approach

```toml
[dependencies]
signal-hook = "0.3"
signal-hook-iterator = "0.3"
```

```rust
use signal_hook::consts::{SIGINT, SIGTERM};
use signal_hook::iterator::Signals;

fn main() {
    let mut signals = Signals::new([SIGINT, SIGTERM]).unwrap();

    // Spawn a thread to handle signals
    std::thread::spawn(move || {
        for sig in signals.forever() {
            match sig {
                SIGINT  => { println!("Ctrl-C received, shutting down..."); std::process::exit(0); }
                SIGTERM => { println!("SIGTERM received, shutting down..."); std::process::exit(0); }
                _       => unreachable!(),
            }
        }
    });

    println!("Running. Press Ctrl-C to stop.");
    loop {
        std::thread::sleep(std::time::Duration::from_secs(1));
        println!("tick");
    }
}
```

`signal_hook` uses the self-pipe trick internally — the `Signals` iterator blocks on a pipe fd, so signal delivery is safe across thread boundaries.

### Atomic flag pattern (no extra crate)

```rust
use std::sync::atomic::{AtomicBool, Ordering};
use std::sync::Arc;

let running = Arc::new(AtomicBool::new(true));
let r = running.clone();

// SAFETY: we only set an atomic flag — async-signal-safe
unsafe {
    signal_hook::low_level::register(signal_hook::consts::SIGINT, move || {
        r.store(false, Ordering::SeqCst);
    }).unwrap();
}

while running.load(Ordering::SeqCst) {
    std::thread::sleep(std::time::Duration::from_millis(100));
}
println!("shutting down");
```

## Process supervision and exit status

When a child terminates, its resources are not fully freed until the parent calls `wait()` — until then it is a **zombie**. Always wait for children you spawn.

```rust
use std::process::{Command, ExitStatus};

fn run_with_retry(cmd: &str, args: &[&str], retries: u32) -> ExitStatus {
    for attempt in 0..=retries {
        let status = Command::new(cmd).args(args).status().unwrap();
        if status.success() || attempt == retries {
            return status;
        }
        eprintln!("attempt {attempt} failed, retrying...");
    }
    unreachable!()
}
```

On Unix, use `ExitStatus::code()` (returns `Option<i32>` — `None` if killed by signal) and `std::os::unix::process::ExitStatusExt::signal()` to distinguish normal exit from signal termination.

## In-class exercises

1. Write a program that runs `ping -c 3 localhost`, captures its stdout, and prints the round-trip times extracted with a simple string search.
2. Write a supervisor that starts a child process and restarts it if it exits with a non-zero code, up to 3 times.
3. Add graceful shutdown: on `SIGTERM`, send `SIGTERM` to the child and wait for it before exiting.
4. Research: what happens on Linux if a parent exits before `wait()`-ing for its child? What is an orphan process vs a zombie?

## Recap

- `Command::new()` spawns children; `output()` captures I/O; `wait_with_output()` is the composable form.
- Signals are asynchronous interrupts; raw handlers are restricted to async-signal-safe ops.
- Use `signal-hook` or the atomic-flag pattern for safe signal handling.
- Always `wait()` for children to avoid zombies; check `ExitStatus::code()` and `::signal()`.
