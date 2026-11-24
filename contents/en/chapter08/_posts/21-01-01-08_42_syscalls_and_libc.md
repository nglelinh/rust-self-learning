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

Every safe Rust abstraction we have used so far—`File::open`, `TcpStream::connect`, `Command::spawn`, `Instant::now`—eventually hits the exact same hardware gate: a **system call**. Your user-space code runs in CPU Ring 3, strictly forbidden from touching disk controllers, network cards, or other processes' memory. Whenever your program needs the outside world, it loads arguments into specific CPU registers and executes a single privileged trap instruction (`syscall` on x86-64, `svc #0` on ARM64) to ask the kernel (Ring 0) to perform the work on its behalf.

In this lesson, we peel away the layers between `std` and the kernel: raw `libc` bindings, safe POSIX wrappers in the `nix` crate, `errno` translation, and watching every syscall live with `strace`.

## Background Knowledge Map: What Layers Sit Between Rust Code and the OS Kernel?

Understanding the syscall stack lets you debug production issues with `strace` and call OS features that `std` does not yet wrap:

1. **CPU Privilege Rings & the Syscall ABI**:
   Modern CPUs enforce hardware isolation between **user space** (unprivileged) and **kernel space** (privileged). On Linux x86-64, a syscall is *not* a normal C function call: the kernel reads the **syscall number** from `rax` (e.g., `0 = read`, `1 = write`, `2 = open`, `60 = exit`), up to 6 arguments from `rdi, rsi, rdx, r10, r8, r9`, and returns the result in `rax` (values in `-4095..-1` encode negated error codes like `-EACCES = -13`).

2. **The `libc` Layer & `errno` Convention**:
   Most Unix programs do not invoke `syscall` directly—they call thin C wrappers in `libc` (glibc/musl on Linux, `libSystem` on macOS—where macOS does *not* even guarantee stable raw syscall numbers!). When the kernel returns a negative error like `-13`, `libc` stores `13` (`EACCES`) in the thread-local variable `errno` and returns `-1` to the caller.

3. **Zero-Cost Safe Wrappers (`nix` & `rustix`)**:
   Calling `libc::open` directly in Rust requires `unsafe`, raw C pointers, integer bitmasks, and manual `errno` checks. Crates like **`nix`** (safe POSIX wrappers over `libc`) and **`rustix`** (direct safe syscalls/libc) turn raw integer flags into type-safe `bitflags!`, wrap file descriptors in RAII `OwnedFd` (closing automatically on `Drop`), and convert `-1` + `errno` into idiomatic `Result<T, Errno>`.

4. **Dynamic Syscall Observation (`strace` / `dtruss`)**:
   Because every external side effect must cross the syscall boundary, running `strace ./my_binary` gives you X-ray vision into any compiled binary—even without source code—revealing exact files opened, bytes written, failed permissions, and unexpected blocking calls.

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

### Deep-Dive Explanation: Tracing `write(1, "hi\n", 3)` Through CPU Registers, `libc`, and `nix::Result` (Read Slowly)

Let us trace a single 3-byte write to standard output—and one failed permission check—all the way from Rust down to the exact x86-64 CPU registers and back up into a safe `Result`.

#### Trace 1: Successful `write(1, "hi\n", 3)` on Linux x86-64

Suppose our byte buffer `"hi\n"` (`[0x68, 0x69, 0x0A]`) sits at virtual address `0x7fff_1234_0000`.

1. **Before `syscall` instruction (CPU Registers loaded by `libc::write`)**:
   | Register | Role in Linux x86-64 Syscall ABI | Exact Value |
   | :---: | :--- | :--- |
   | `rax` | Syscall number (`SYS_write`) | `1` |
   | `rdi` | Arg 1: `fd` (`STDOUT_FILENO`) | `1` |
   | `rsi` | Arg 2: `buf` pointer | `0x7fff_1234_0000` |
   | `rdx` | Arg 3: `count` (bytes) | `3` |
2. **Hardware Trap (`syscall`)**: CPU switches from Ring 3 → Ring 0, validates `fd = 1` in the current process's FD table, copies 3 bytes to the terminal driver, and places the number of bytes written (`3`) into `rax`.
3. **Return to User Space**:
   - `libc::write` sees `rax = 3 >= 0` → returns `3_isize`.
   - `nix::unistd::write` wraps `3_isize` as **`Ok(3)`**.

#### Trace 2: Failed `open("/etc/shadow", O_RDONLY)` (`EACCES`)

Now trace what happens when a non-root user tries to open `/etc/shadow`:

1. **Kernel check fails**: Permission bits are `0640` (`root:shadow`), so the kernel rejects the request and returns **`rax = -13`** (where `13 == EACCES`).
2. **`libc` translation**: `libc` checks if `rax` is in `[-4095, -1]`. Because `-13` is in that range, `libc` sets thread-local **`errno = 13`** and returns **`-1`** to Rust!
3. **Why raw `libc` is error-prone vs. `nix`**:
   - In raw `libc`, if you forget to check `ret < 0` and pass `fd = -1` into `read(-1, ...)`, you silently cascade into `EBADF`—or worse, if you call another `libc` function before reading `errno`, `errno` gets overwritten!
   - In `nix::fcntl::open`, the wrapper immediately checks `res == -1`, reads `Errno::last()` atomically, and returns **`Err(nix::errno::Errno::EACCES)`**—impossible to ignore thanks to `#[must_use]` on `Result`.

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

## Video tham khảo

- [System Calls](https://www.youtube.com/watch?v=lhToWeuWWfw) — Neso Academy, English. **xem trước.** Watch first: what a system call is, before libc and the nix crate.
- [strace feels like magic — let’s fix that (with Rust)](https://www.youtube.com/watch?v=engduNoI6DE) — fasterthanlime, English. strace, explained by writing the tracer in Rust — the tracing section of the lesson.
- [Rust Tutorial: Using Linux libc in Rust - with the file-locker Crate](https://www.youtube.com/watch?v=UgNrDb6hQQ0) — Ben Goldberg, English. Calling libc from Rust, which is the unsafe FFI layer nix wraps.
