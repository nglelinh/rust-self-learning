---
layout: post
title: "08-42 System Calls, libc, and the nix Crate"
chapter: "08"
order: 42
owner: "OpenCode"
lang: en
categories:
  - chapter08
lesson_type: required
---

Everything the kernel provides to user space — file I/O, process control, networking, memory mapping — is exposed through system calls. This lesson explains how syscalls work at the hardware level, how `libc` wraps them, and how Rust's `nix` crate gives you safe, idiomatic access to the full POSIX API.

## 60-minute teaching plan

- 0 to 10 min: What a system call is — user mode vs kernel mode, the syscall mechanism.
- 10 to 25 min: `libc` in Rust — FFI to the C standard library.
- 25 to 40 min: The `nix` crate — POSIX API in safe Rust.
- 40 to 55 min: Direct syscalls with `syscall!` / `linux-syscall`; writing a strace-style tracer.
- 55 to 60 min: Exercises and recap.

## Learning goals

By the end of this lesson, students can:

- Explain the user-mode / kernel-mode boundary and how syscalls cross it.
- Call POSIX functions from Rust via `libc` with correct types and error handling.
- Use the `nix` crate for safe, typed POSIX access (file, process, socket, time).
- Trace syscalls made by a program using `strace` and interpret the output.

## What is a system call?

User-space programs run in **unprivileged mode** (ring 3 on x86-64). They cannot directly touch hardware, manipulate page tables, or access other processes' memory. To do any of that they must ask the kernel.

A **system call** is the controlled entry point into kernel space:

1. The program places the syscall number and arguments into specific CPU registers.
2. It executes a special instruction (`syscall` on x86-64, `svc` on ARM64).
3. The CPU switches to **kernel mode** (ring 0), saves user registers, and jumps to the kernel's syscall handler.
4. The kernel validates arguments, performs the operation, and returns the result in a register.
5. The CPU switches back to user mode and the program continues.

This switch costs roughly 100–300 ns — fast, but not free. Batching I/O (e.g. with `BufWriter`) and using `io_uring` for async I/O exist precisely to reduce syscall frequency.

### Syscall numbers (Linux x86-64)

```
read   →  0
write  →  1
open   →  2
close  →  3
mmap   →  9
fork   → 57
execve → 59
exit   → 60
```

The full list is in `/usr/include/asm/unistd_64.h` and `man 2 syscall`.

## Using libc from Rust

The `libc` crate provides raw FFI bindings to the C standard library and POSIX APIs:

```toml
[dependencies]
libc = "0.2"
```

### Pattern: call, check errno, return Result

POSIX functions signal errors by returning -1 and setting `errno`. Translate this to Rust's `Result`:

```rust
use std::io;

fn set_nonblocking(fd: libc::c_int) -> io::Result<()> {
    let flags = unsafe { libc::fcntl(fd, libc::F_GETFL, 0) };
    if flags == -1 {
        return Err(io::Error::last_os_error());
    }
    let ret = unsafe { libc::fcntl(fd, libc::F_SETFL, flags | libc::O_NONBLOCK) };
    if ret == -1 {
        return Err(io::Error::last_os_error());
    }
    Ok(())
}
```

`io::Error::last_os_error()` reads `errno` and wraps it as a Rust error with its OS message.

### Getting system information

```rust
unsafe {
    // Number of CPUs
    let cpus = libc::sysconf(libc::_SC_NPROCESSORS_ONLN);
    println!("CPUs: {cpus}");

    // Page size
    let page_size = libc::sysconf(libc::_SC_PAGESIZE);
    println!("page size: {page_size}");

    // Hostname
    let mut buf = [0i8; 256];
    libc::gethostname(buf.as_mut_ptr(), buf.len());
    let name = std::ffi::CStr::from_ptr(buf.as_ptr());
    println!("hostname: {}", name.to_str().unwrap());
}
```

### Calling a raw syscall directly

```rust
unsafe {
    // getpid() via raw syscall number
    let pid = libc::syscall(libc::SYS_getpid);
    println!("pid via syscall: {pid}");
}
```

Direct syscalls bypass `libc` entirely — occasionally needed for syscalls not yet wrapped by `libc`, or in `#![no_std]` environments.

## The nix crate — idiomatic POSIX in Rust

`libc` is raw FFI. `nix` wraps the same POSIX API with proper Rust types, `Result` returns, and no `unsafe` at the call site:

```toml
[dependencies]
nix = { version = "0.29", features = ["process", "fs", "signal", "time", "socket"] }
```

### Process

```rust
use nix::unistd::{getpid, getppid, getuid, setpriority, Priority};

println!("pid: {}", getpid());
println!("parent pid: {}", getppid());
println!("uid: {}", getuid());

// Lower nice value = higher priority (range -20 to 19)
setpriority(Priority::Process, 0, -5).unwrap();
```

### File and directory

```rust
use nix::fcntl::{open, OFlag};
use nix::sys::stat::Mode;
use nix::unistd::{read, write, close};

let fd = open("/tmp/test.txt",
    OFlag::O_WRONLY | OFlag::O_CREAT | OFlag::O_TRUNC,
    Mode::S_IRUSR | Mode::S_IWUSR,
)?;
write(fd, b"hello nix\n")?;
close(fd)?;
```

### Time

```rust
use nix::time::{clock_gettime, ClockId};

let ts = clock_gettime(ClockId::CLOCK_MONOTONIC)?;
println!("monotonic: {}.{:09} s", ts.tv_sec(), ts.tv_nsec());

let ts = clock_gettime(ClockId::CLOCK_REALTIME)?;
println!("realtime:  {}.{:09} s", ts.tv_sec(), ts.tv_nsec());
```

`CLOCK_MONOTONIC` never goes backwards — use it for duration measurements.
`CLOCK_REALTIME` is wall clock — can jump on NTP sync.

### `ioctl` — device control

`ioctl` is the catch-all syscall for device-specific operations: get terminal size, control network interfaces, etc.

```rust
use nix::libc;

fn terminal_size() -> Option<(u16, u16)> {
    let mut ws: libc::winsize = unsafe { std::mem::zeroed() };
    let ret = unsafe { libc::ioctl(libc::STDOUT_FILENO, libc::TIOCGWINSZ, &mut ws) };
    if ret == 0 {
        Some((ws.ws_col, ws.ws_row))
    } else {
        None
    }
}
```

### `epoll` — efficient I/O event notification

`epoll` lets a single thread wait for events on many file descriptors simultaneously — the foundation of async I/O runtimes:

```rust
use nix::sys::epoll::*;
use nix::unistd::pipe;

let epoll = epoll_create1(EpollCreateFlags::empty())?;
let (read_fd, write_fd) = pipe()?;

let mut event = EpollEvent::new(EpollFlags::EPOLLIN, read_fd.as_raw_fd() as u64);
epoll_ctl(epoll, EpollOp::EpollCtlAdd, read_fd, &mut event)?;

// Write something to trigger the event
nix::unistd::write(write_fd, b"data")?;

let mut events = [EpollEvent::empty(); 10];
let n = epoll_wait(epoll, &mut events, -1)?;   // block until event
println!("got {n} event(s)");
```

`epoll` is what Tokio and async-std use under the hood on Linux. `kqueue` is the equivalent on macOS/BSD; `io_uring` is the modern Linux alternative for even lower overhead.

## Tracing syscalls with strace

`strace` intercepts every syscall made by a process and prints it:

```bash
strace -e trace=openat,read,write -o trace.txt ./your-program
```

Example output:

```
openat(AT_FDCWD, "config.toml", O_RDONLY|O_CLOEXEC) = 3
read(3, "[package]\nname = \"app\"\n", 4096)  = 23
close(3)                                    = 0
write(1, "loaded config\n", 14)             = 14
```

Reading `strace` output is one of the most effective ways to understand what your program is actually doing at the OS level. Use it to:
- Confirm file paths being opened.
- Identify unexpected `stat()` calls slowing down startup.
- Debug permission errors (look for `EACCES` or `EPERM` returns).
- Understand why a program hangs (look for blocking `futex` or `read` calls).

## In-class exercises

1. Write a Rust program that prints its own PID, parent PID, number of CPUs, and page size using only `libc` calls.
2. Use `nix::time::clock_gettime` to write a `sleep_precise(duration: Duration)` function that busy-waits using `CLOCK_MONOTONIC` instead of calling `std::thread::sleep`.
3. Run `strace -c ./your-program` on any Rust binary. Identify the top three syscalls by count and explain what each does.
4. Use `nix` to open a directory fd with `O_DIRECTORY | O_PATH` and call `getdents64` (via `libc::syscall`) to list directory entries without using `std::fs::read_dir`.

## Recap

- Syscalls are the controlled boundary between user-space programs and the kernel; each call costs ~100–300 ns.
- `libc` provides raw POSIX bindings; always check the return value and call `io::Error::last_os_error()` on -1.
- `nix` wraps the same API in safe, typed Rust with `Result` returns — prefer it over `libc` for new code.
- `ioctl` is the device-control interface; `epoll`/`kqueue` are how async runtimes wait for I/O.
- `strace` is your ground truth for what syscalls a program actually makes.
