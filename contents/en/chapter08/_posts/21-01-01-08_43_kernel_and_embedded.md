---
layout: post
title: "08-43 Rust in the Kernel and on Bare Metal"
chapter: "08"
order: 43
owner: "OpenCode"
lang: en
categories:
  - chapter08
lesson_type: optional
---

What happens when you strip away Linux, macOS, Windows, the allocator, the file system, and `println!`—and run your Rust binary directly on a bare $2 ARM Cortex-M microcontroller or inside the Linux kernel itself? Suddenly there is no OS to catch your stack overflow, no `std::fs`, and no `Vec` unless you bring your own allocator.

Remarkably, **almost everything that makes Rust great survives intact**: ownership, borrowing, zero-cost generics, traits, enums, and fearless concurrency—all packaged in the tiny, dependency-free `core` crate. That is why Rust became the first new language ever accepted alongside C in the Linux kernel and is rapidly reshaping firmware engineering.

## Key ideas: What Changes When You Leave `std` Behind for Bare Metal?

Writing `#![no_std]` Rust for kernels and microcontrollers rests on four core architectural pillars:

1. **The Crate Triad (`core` vs. `alloc` vs. `std`)**:
   Rust's standard library is cleanly layered:
   - **`core`**: platform-independent, zero-OS, zero-allocation primitives (`Option`, `Result`, iterators, slices, atomics, `ptr`, `fmt`). Always available on every bare-metal chip.
   - **`alloc`**: heap collections (`Box`, `Vec`, `String`, `Arc`) that work in `#![no_std]` as soon as you supply a `#[global_allocator]`.
   - **`std`**: OS-dependent APIs (`File`, `TcpStream`, `Thread`, `Instant`) built on top of `core` + `alloc` + OS syscalls.

2. **Memory-Mapped I/O (MMIO) & Volatile Access**:
   Microcontrollers do not have a "LED syscall." Hardware peripherals (GPIO pins, UART serial ports, SPI buses, timers) are wired directly into fixed physical memory addresses. Writing a `1` to bit 5 of address `0x4002_0014` physically drives 3.3 V onto pin PA5. Because an optimizing compiler assumes normal RAM never changes on its own, all MMIO reads/writes **must** use `ptr::read_volatile` and `ptr::write_volatile` so LLVM never deletes or reorders hardware register accesses.

3. **Type-State Hardware Drivers (`embedded-hal`)**:
   In C firmware, accidentally reading from a GPIO pin configured as an output—or using a UART peripheral before enabling its clock—compiles silently and fails mysteriously on the circuit board. Rust's `embedded-hal` uses **zero-sized type states** (e.g., `Pin<PA5, Output<PushPull>>` vs. `Pin<PA5, Input<Floating>>`) so invalid hardware states fail to compile with zero runtime overhead.

4. **Interrupts, Critical Sections & Async Bare Metal (`Embassy`)**:
   Without an OS thread scheduler, microcontrollers respond to hardware events via **Interrupt Service Routines (ISRs)**. Modern Rust embedded frameworks like **Embassy** compile `async fn` state machines directly into interrupt-driven, zero-heap executors that automatically put the CPU into microamp deep-sleep (`WFE`/`WFI`) whenever no task is ready.

## 60-minute teaching plan

- 0 to 10 min: `#![no_std]` — what the standard library provides and what you lose.
- 10 to 25 min: Bare-metal setup: linker scripts, startup code, MMIO.
- 25 to 40 min: Writing a Linux kernel module in Rust.
- 40 to 55 min: The `embedded-hal` ecosystem and running Rust on a microcontroller.
- 55 to 60 min: Exercises and recap.

## Learning goals

By the end of this lesson, students can:

- Write a `#![no_std]` crate and explain what is unavailable.
- Read and write memory-mapped hardware registers with `volatile`.
- Compile and load a minimal Linux kernel module written in Rust.
- Describe the `embedded-hal` abstraction and why it matters for portability.

## `#![no_std]` — the standard library boundary

`std` provides: heap allocation (`Box`, `Vec`, `String`), threads, file I/O, networking, process control, and more — all of which require an OS. Without an OS, none of that exists.

`#![no_std]` opts out of `std`. You still have:

- `core` — language primitives: iterators, `Option`, `Result`, math, raw pointers, atomics, `fmt`.
- `alloc` (optional) — heap collections (`Box`, `Vec`, `String`) if you provide an allocator.

```rust
#![no_std]
#![no_main]   // we provide our own entry point

use core::panic::PanicInfo;

#[panic_handler]
fn panic(_info: &PanicInfo) -> ! {
    loop {}   // on bare metal: spin or reset
}

#[no_mangle]
pub extern "C" fn _start() -> ! {
    // entry point — called by the bootloader or OS loader
    loop {}
}
```

Without `std`, you must provide:
- A **panic handler** (`#[panic_handler]`).
- A **global allocator** (`#[global_allocator]`) if you use `alloc`.
- An **entry point** that matches the ABI expected by the platform.

## Memory-mapped I/O (MMIO)

On embedded and kernel-level hardware, peripherals are controlled by reading and writing specific physical addresses. The compiler must not optimize away those accesses, so they must be **volatile**:

```rust
use core::ptr::{read_volatile, write_volatile};

const UART_BASE: usize = 0x4000_C000;
const UART_DR: *mut u32 = UART_BASE as *mut u32;    // data register
const UART_FR: *mut u32 = (UART_BASE + 0x18) as *mut u32;  // flag register
const TXFF: u32 = 1 << 5;   // transmit FIFO full

fn uart_putc(c: u8) {
    unsafe {
        // Wait until transmit FIFO is not full
        while read_volatile(UART_FR) & TXFF != 0 {}
        write_volatile(UART_DR, c as u32);
    }
}
```

The `volatile_register` and `vcell` crates provide safer wrappers that enforce volatile access at the type level.

For production embedded code, use the **Peripheral Access Crate (PAC)** generated by `svd2rust` from the chip's SVD (System View Description) file — it gives you named, typed register access:

```rust
// Generated PAC code (example):
peripheral.uart0.dr.write(|w| w.data().bits(c));
```

## Linux kernel modules in Rust

Since Linux 6.1, Rust is an optional first-class language for kernel modules. The kernel provides Rust bindings in `rust/` and a set of safe abstractions over kernel internals.

### Prerequisites

```bash
# Requires a kernel built with CONFIG_RUST=y
# Check: zcat /proc/config.gz | grep CONFIG_RUST
CONFIG_RUST=y
CONFIG_RUST_IS_AVAILABLE=y
```

### Minimal "Hello, kernel" module

```rust
// src/lib.rs
use kernel::prelude::*;

module! {
    type: HelloModule,
    name: "hello_rust",
    author: "Your Name",
    description: "A minimal Rust kernel module",
    license: "GPL",
}

struct HelloModule;

impl kernel::Module for HelloModule {
    fn init(_module: &'static ThisModule) -> Result<Self> {
        pr_info!("Hello from Rust kernel module!\n");
        Ok(HelloModule)
    }
}

impl Drop for HelloModule {
    fn drop(&mut self) {
        pr_info!("Goodbye from Rust kernel module!\n");
    }
}
```

```makefile
# Kbuild
obj-m := hello_rust.o
```

```bash
make -C /lib/modules/$(uname -r)/build M=$(pwd) modules
sudo insmod hello_rust.ko
dmesg | tail -5       # should show "Hello from Rust kernel module!"
sudo rmmod hello_rust
dmesg | tail -5       # should show "Goodbye..."
```

### What the kernel Rust environment provides

The kernel Rust bindings (`kernel::prelude::*`) include:

- `pr_info!`, `pr_err!`, `pr_warn!` — kernel logging macros (write to `dmesg`).
- `GFP_KERNEL`, `Box::try_new()` — allocation with explicit failure handling (no panics allowed in kernel).
- Safe wrappers for mutexes, spinlocks, linked lists, work queues.
- `File`, `INode`, `Address Space` bindings for filesystem drivers.
- `NetDevice` for network drivers.

Key difference from user-space Rust: **no unwinding panics**. Any allocation can fail; code must handle `Result::Err` at every step. Kernel Rust uses `Result<T, kernel::error::Error>` throughout.

## The embedded-hal ecosystem

`embedded-hal` defines platform-agnostic traits for common hardware peripherals: GPIO, SPI, I2C, UART, ADC, delays. A driver crate written against `embedded-hal` traits works on any microcontroller that has an `embedded-hal` implementation.

```toml
[dependencies]
embedded-hal = "1.0"
# Plus an HAL implementation for your chip, e.g.:
stm32f4xx-hal = { version = "0.21", features = ["stm32f411"] }
```

### Blinky on an STM32 (illustrative)

```rust
#![no_std]
#![no_main]

use cortex_m_rt::entry;
use stm32f4xx_hal::{pac, prelude::*};

#[entry]
fn main() -> ! {
    let dp = pac::Peripherals::take().unwrap();
    let gpioa = dp.GPIOA.split();
    let mut led = gpioa.pa5.into_push_pull_output();

    let rcc = dp.RCC.constrain();
    let clocks = rcc.cfgr.freeze();
    let mut delay = dp.TIM1.delay_ms(&clocks);

    loop {
        led.set_high();
        delay.delay_ms(500u32);
        led.set_low();
        delay.delay_ms(500u32);
    }
}
```

The `cortex_m_rt::entry` macro sets up the interrupt vector table and calls `main`. There is no OS, no scheduler, no heap unless you add one.

### Running and debugging embedded Rust

```bash
# Cross-compile
cargo build --target thumbv7em-none-eabihf --release

# Flash with probe-rs
probe-rs run --chip STM32F411CEUx target/thumbv7em-none-eabihf/release/blinky

# Or with OpenOCD + GDB
openocd -f interface/stlink.cfg -f target/stm32f4x.cfg &
arm-none-eabi-gdb target/thumbv7em-none-eabihf/release/blinky
```

The `defmt` crate provides efficient logging for embedded — messages are encoded as integer IDs in the binary and decoded on the host, reducing the bandwidth over debug probes.

## RTOS integration: Embassy

`Embassy` is an async executor for embedded Rust — it brings `async`/`await` to microcontrollers:

```rust
#[embassy_executor::main]
async fn main(spawner: Spawner) {
    let p = embassy_stm32::init(Default::default());
    let mut led = Output::new(p.PA5, Level::High, Speed::Low);

    loop {
        led.set_high();
        Timer::after_millis(500).await;
        led.set_low();
        Timer::after_millis(500).await;
    }
}
```

Embassy implements cooperative multitasking with zero-cost async state machines — no heap allocation per task.

### Explanation: Hand-Tracing a Volatile MMIO GPIO Register Toggle

Let us trace down to the exact bit level what happens inside an STM32 ARM Cortex-M microcontroller when your `#![no_std]` Rust firmware turns on an onboard LED wired to **GPIO Port A, Pin 5 (`PA5`)**—and why omitting `volatile` breaks the hardware.

Suppose the chip's Reference Manual specifies:
- `GPIOA` peripheral base address: `0x4002_0000`
- `GPIOA_ODR` (Output Data Register) offset: `0x14` → exact address **`0x4002_0014`** (32-bit register where bit `n` controls pin `PAn`).
- Current value of `GPIOA_ODR` before our code runs: `0x0000_0009` (pins `PA0` and `PA3` are currently HIGH, `PA5` is LOW).

#### Step 1: Compute the Exact Bitmask by Hand

In binary, the low 8 bits of `0x0000_0009` are:
```text
Bit index:   7  6  5  4  3  2  1  0
Initial:     0  0  0  0  1  0  0  1   (0x09: PA3=1, PA0=1)
Mask (1<<5): 0  0  1  0  0  0  0  0   (0x20: bit 5 = 32)
```

#### Step 2: Read-Modify-Write Using `read_volatile` / `write_volatile`

```rust
const GPIOA_ODR: *mut u32 = 0x4002_0014 as *mut u32;

unsafe {
    let cur = core::ptr::read_volatile(GPIOA_ODR);       // Reads 0x0000_0009
    let next = cur | (1 << 5);                           // 0x0000_0009 | 0x0000_0020 = 0x0000_0029
    core::ptr::write_volatile(GPIOA_ODR, next);          // Writes 0x0000_0029 -> PA5 goes 3.3V!
}
```
Check the result:
- Bitwise OR (`|`): `0000_1001 | 0010_0000 = 0010_1001` (`0x29`). Pins `PA0` and `PA3` remain untouched (`1`), while `PA5` flips from `0` to `1`!
- To turn `PA5` back OFF without disturbing `PA0` or `PA3`, compute bitwise AND with the inverted mask `!(1 << 5) = 0xFFFF_FFDF`:
  `0010_1001 & 1101_1111 = 0000_1001` (`0x09`).

#### Step 3: Why Non-Volatile Writes Get Silently Deleted by LLVM

Suppose you wrote a blinking-LED pulse without `volatile`:
```rust
unsafe {
    *GPIOA_ODR = 0x29;  // Turn LED on
    *GPIOA_ODR = 0x09;  // Turn LED off
}
```
To LLVM's Dead Store Elimination (DSE) pass, writing `0x29` to a memory address and immediately overwriting it with `0x09` without reading it in between looks completely redundant—so **`rustc --release` deletes the first write entirely!** The LED never pulses. Using `write_volatile` marks every store as an observable side effect that LLVM is strictly forbidden to eliminate or reorder.

## In-class exercises

1. Write a `#![no_std]` library crate that implements a ring buffer using only `core`. Verify it compiles with `cargo build --target thumbv7m-none-eabi`.
2. (Linux only) Build and load the "Hello, kernel" module above. Check `dmesg` output. Then modify it to log the system uptime using `ktime_get()`.
3. Read a chip's datasheet (e.g. STM32F4 Reference Manual) and identify the base address and register layout for one peripheral (GPIO or UART). Write the MMIO read/write code by hand.
4. Research: what does `CONFIG_RUST_ALLOW_FILES_IN_UNSAFE_CODE` control in the Linux kernel build? Why does the kernel require a different safety contract than user-space Rust?

## Recap

- `#![no_std]` removes `std`; you keep `core` and optionally `alloc`.
- MMIO requires volatile reads/writes; PACs generated by `svd2rust` provide named register access.
- Linux kernel modules in Rust use `kernel::prelude::*`; no unwinding panics; all allocations are fallible.
- `embedded-hal` traits decouple drivers from hardware — write once, run on any supported chip.
- Embassy brings `async`/`await` cooperative scheduling to bare-metal without an RTOS.

## Video tham khảo

- [Blinking an LED: Embedded Rust ecosystem explored](https://www.youtube.com/watch?v=A9wvA_S6m7Y) — The Rusty Bits, English. **xem trước.** Watch first: no_std, HAL, and a blink — the embedded-hal half of the lesson.
- [Intro to Embedded Rust Part 12: Async Programming with Embassy | DigiKey](https://www.youtube.com/watch?v=m9FT1qSm-KM) — DigiKey, English. Embassy’s async executor on bare metal, the RTOS section.
- [Rust for Linux by Miguel Ojeda and Wedson Almeida Filho - Rust Linz, March 2022](https://www.youtube.com/watch?v=fVEeqo40IyQ) — Rust Programming Language, English. Miguel Ojeda and Wedson Almeida Filho on Rust for Linux, the kernel-module half.
