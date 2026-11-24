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

Every time you hit Enter in a terminal, launch a compiler, or run a web server behind systemd, the operating system performs a carefully choreographed dance: cloning or spawning an isolated execution environment, wiring up its standard input/output streams, monitoring its health, and eventually reaping its exit status so it does not linger as a zombie.

In Chapter 8, we step below application frameworks directly into the **Unix systems programming** layer. This lesson unpacks how processes are created, how asynchronous **signals** (`SIGTERM`, `SIGKILL`, `SIGINT`) interrupt running programs, and how Rust's `std::process::Command` turns fragile C `fork`/`exec` boilerplate into clean, leak-free supervision code.

## Background Knowledge Map: How Does the Kernel Manage Process Lifecycles?

To build reliable CLIs, build tools, shells, and daemons in Rust, you need four foundational Unix process concepts:

1. **Process Identity & Address-Space Isolation (`PID`, `PPID`, Process Groups)**:
   Every running program is tracked by the kernel via a **Process Control Block** (PCB) identified by a unique Process ID (`PID`) and Parent PID (`PPID`). Each process has its own virtual address space, file descriptor table, working directory, and environment variables.

2. **The `fork` + `exec` + `waitpid` Lifecycle (and Zombie Prevention)**:
   On POSIX systems, spawning a new program traditionally duplicates the parent (`fork` using Copy-on-Write page tables) and immediately replaces the child's memory image with the new binary (`execve`). When the child terminates, the kernel frees its memory **except** its small PCB entry holding the exit status—making it a **zombie (`Z`)** until the parent calls `waitpid()` (in Rust: `child.wait()`) to reap it.

3. **Signals: Asynchronous Software Interrupts**:
   Signals are 1-integer notifications delivered by the kernel or another process (`kill(pid, sig)`). Some can be caught for graceful shutdown (`SIGINT` = Ctrl+C, `SIGTERM` = polite stop), while two **cannot be caught, blocked, or ignored** (`SIGKILL` = immediate termination, `SIGSTOP` = freeze). Because signal handlers can interrupt a thread mid-instruction (even while holding a `malloc` lock!), POSIX restricts raw handlers to **async-signal-safe** operations—which is why Rust programs use crates like `signal-hook` or `tokio::signal` to convert signals into safe flags or channel messages.

4. **Exit Status Encoding (`waitpid` 16-bit Bitfield)**:
   A child process can end in two completely different ways: exiting voluntarily (`exit(code)`) or being killed by a signal (`SIGSEGV`, `SIGKILL`). Unix packs both outcomes into a single integer status word decoded by `ExitStatus::code()` and `ExitStatusExt::signal()`.

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

### Deep-Dive Explanation: Hand-Decoding Unix `waitpid` Exit Statuses and Tracing a Zombie Process (Read Slowly)

When a child process finishes on Linux/macOS, `waitpid()` returns a single raw 32-bit integer `wstatus` (whose low 16 bits carry the full termination report). Let us decode two concrete values bit by bit by hand so you understand exactly what Rust's `ExitStatus` is inspecting under the hood.

#### The POSIX/Linux 16-Bit `wstatus` Layout

```text
Bit:   15 14 13 12 11 10  9  8 |  7   |  6  5  4  3  2  1  0
      [     Exit Code (8 bits) | Core |  Terminating Signal  ]
      └────── bits 8..15 ──────┘ bit7 └───── bits 0..6 ──────┘
```

- **Rule A (Normal exit)**: If bits `0..6` (`wstatus & 0x7F`) equal `0`, the process exited normally (`WIFEXITED == true`), and its exit code is `(wstatus >> 8) & 0xFF`.
- **Rule B (Killed by signal)**: If bits `0..6` are non-zero (and not `0x7F` for stopped), the process was killed by signal number `wstatus & 0x7F` (`WIFSIGNALED == true`), and bit `7` (`wstatus & 0x80`) is `1` if a core dump was produced.

#### Worked Example 1: Child Calls `std::process::exit(2)`

Suppose the kernel returns `wstatus = 0x0200` (decimal `512`):
1. Check low 7 bits: `0x0200 & 0x007F = 0x00` → **Normal exit** (`WIFEXITED = true`).
2. Shift right by 8 bits: `(0x0200 >> 8) & 0xFF = 0x02` → **Exit code = `2`**.
3. In Rust: `status.code()` returns `Some(2)`, `status.signal()` returns `None`, and `status.success()` returns `false`.

#### Worked Example 2: Child Is Killed by `SIGKILL` (Signal 9)

Suppose the kernel returns `wstatus = 0x0009` (decimal `9`):
1. Check low 7 bits: `0x0009 & 0x007F = 9` ($\neq 0$) → **Terminated by signal** (`WIFSIGNALED = true`).
2. Check bit 7 (core dump): `0x0009 & 0x0080 = 0` → No core dump.
3. In Rust: `status.code()` returns **`None`** (not `Some(9)`!), `status.signal()` returns **`Some(9)`**, and a shell reports `$? = 128 + 9 = 137` (the famous "exit code 137" you see in Docker/Kubernetes when the OOM killer sends `SIGKILL`!).

#### Tracing a Process Table From `spawn()` to Reaping

| Time | Parent (`PID 4100`) Action | Kernel Process Table Entry for Child (`PID 4101`) |
| :---: | :--- | :--- |
| $t_0$ | `let mut c = Command::new("sleep").arg("1").spawn()?;` | `PID 4101`, `PPID 4100`, state = **`R`/`S` (Running/Sleeping)** |
| $t_1$ | Child finishes after 1 s; parent has **not** called `c.wait()` yet | `PID 4101`, state = **`Z` (Zombie)** — RAM freed, PCB holds `wstatus = 0x0000` |
| $t_2$ | Parent calls `let st = c.wait()?;` (`waitpid(4101)`) | Entry `PID 4101` is **reaped & deleted**; parent receives `Ok(ExitStatus(0))` |

If a long-running Rust daemon spawns thousands of short-lived children and forgets to call `.wait()` (or `.status()` / `.output()`), those zombie PCBs pile up until the OS exhausts its PID limit!

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

## Video tham khảo

- [fork() and exec() System Calls](https://www.youtube.com/watch?v=IFEFVXvjiHY) — Neso Academy, English. **xem trước.** Watch first: fork and exec, the process model behind Rust’s Command. The examples are C.
- [Spawn External Processes with Rust Standard Library 🦀 Rust Programming Tutorial for Developers](https://www.youtube.com/watch?v=RtVzlk4om6U) — Trevor Sullivan, English. std::process in Rust: spawn, wait, and exit status, which is the coding half of the lesson.
- [Sending and Handling Signals in C (kill, signal, sigaction)](https://www.youtube.com/watch?v=83M5-NPDeWs) — Jacob Sorber, English. SIGINT, SIGTERM, and sigaction — the signals section, shown in C.
