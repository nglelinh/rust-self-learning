---
layout: post
title: "06-29 FFI and Interop"
chapter: "06"
order: 29
owner: "OpenCode"
lang: en
categories:
  - chapter06
lesson_type: required
---

FFI is where Rust meets the outside world: C libraries, system APIs, and foreign runtimes. This lesson teaches the core FFI surface area — ABI compatibility, string handling, error translation — and how to wrap unsafe calls in safe Rust APIs that callers can use confidently.

## 60-minute teaching plan

- 0 to 10 min: What FFI is and why it is unsafe by default.
- 10 to 25 min: ABI basics: `extern "C"` and `repr(C)`.
- 25 to 40 min: Strings and buffers across boundaries.
- 40 to 55 min: Mini-lab: wrap a C buffer-sum function behind a safe Rust API.
- 55 to 60 min: Safety checklist and recap.

## Learning goals

By the end of this lesson, students can:

- Declare C functions in Rust with `extern "C"`.
- Use `#[repr(C)]` for structs shared with C.
- Pass strings safely using `CString` and `CStr`.
- Build safe wrappers that validate inputs and translate errors.

## Prerequisites

- Lesson 06-28 (Unsafe Rust): raw pointers, safety contracts, `// Safety:` comments.

---

## Key concept: why FFI is inherently unsafe

Rust's safety model is enforced entirely within Rust's type system. At the FFI boundary, the type system ends:

- C doesn't know about lifetimes — it can keep pointers after Rust has freed them.
- C doesn't know about aliasing rules — it can create multiple mutable pointers freely.
- C doesn't know about `Send`/`Sync` — it might call from multiple threads unsafely.
- C error handling is convention-based (return codes, out-params, `errno`) — Rust doesn't track these.

**The job of a Rust FFI wrapper is to reintroduce Rust's guarantees** at the boundary:

1. Validate inputs before crossing.
2. Encapsulate raw pointers behind owned types.
3. Translate C error codes into `Result`.
4. Document ownership semantics (who allocates, who frees).

---

## ABI basics

### Application Binary Interface (ABI)

An ABI defines how functions are called at the machine level: register usage, stack layout, calling conventions, struct layout. C's ABI is the universal "lingua franca" of native interop — nearly every language can call C functions.

`extern "C"` tells Rust to use C calling conventions for a function:

```rust
extern "C" {
    // Declare a C function without defining it — Rust will link to the compiled C
    fn strlen(s: *const std::ffi::c_char) -> usize;
}
```

Calling it:

```rust
use std::ffi::CString;

let s = CString::new("hello").unwrap();
let len = unsafe { strlen(s.as_ptr()) };
assert_eq!(len, 5);
```

### Exposing Rust to C with `#[no_mangle]`

```rust
/// This function can be called from C.
#[no_mangle]
pub extern "C" fn add_i32(a: i32, b: i32) -> i32 {
    a + b
}
```

`#[no_mangle]` prevents Rust from mangling the symbol name (Rust normally adds type info to names). C code can then call `add_i32(3, 4)`.

---

## `repr(C)` for shared structs

By default, Rust may reorder struct fields for efficiency. C code expects a specific layout. `#[repr(C)]` ensures C-compatible layout:

```rust
#[repr(C)]
pub struct Point {
    pub x: i32,
    pub y: i32,
}
```

Without `#[repr(C)]`, passing `Point` to C is undefined behavior — the fields might be in the wrong order.

### Size and alignment

Use `std::mem::size_of::<T>()` and `std::mem::align_of::<T>()` to verify layout:

```rust
assert_eq!(std::mem::size_of::<Point>(), 8);   // 4 + 4 bytes
assert_eq!(std::mem::align_of::<Point>(), 4);  // aligned to i32
```

---

## Strings across the boundary

### The mismatch

| | Rust `&str` / `String` | C `char*` |
|---|---|---|
| Encoding | UTF-8 | Unspecified (often ASCII or Latin-1) |
| Length | Stored separately (fat pointer) | Null terminator |
| Validity | Guaranteed UTF-8 | No guarantee |
| Allocation | Heap (String) or static | Caller or callee |

### From Rust to C: `CString`

```rust
use std::ffi::CString;

let rust_str = "hello, world";
let c_str = CString::new(rust_str).expect("string contains null byte");
// c_str.as_ptr() gives *const c_char — valid as long as c_str is alive
unsafe { some_c_function(c_str.as_ptr()); }
// c_str drops here, freeing the allocation — DON'T hold the pointer after this
```

**Critical**: `CString::new` fails if the string contains interior null bytes (C strings can't represent those). Always handle this with proper error handling in production code.

### From C to Rust: `CStr`

```rust
use std::ffi::CStr;

// If C gives us a *const c_char:
unsafe fn c_str_to_rust(ptr: *const std::ffi::c_char) -> Option<&'static str> {
    if ptr.is_null() { return None; }
    let c_str = CStr::from_ptr(ptr);       // Safety: ptr must be null-terminated
    c_str.to_str().ok()                    // fails if not valid UTF-8
}
```

`CStr` is a borrowed view — it doesn't own the memory. The C code still owns it.

### Owning a C string: `CString::from_raw` / `CStr::to_owned`

If C allocates a string and gives ownership to Rust, use `CString::from_raw` — but this is complex. In most cases, copy the string into a Rust `String` immediately and let C handle its own deallocation:

```rust
unsafe fn copy_c_string(ptr: *const std::ffi::c_char) -> Option<String> {
    if ptr.is_null() { return None; }
    let c_str = CStr::from_ptr(ptr);
    Some(c_str.to_string_lossy().into_owned())  // copies + handles non-UTF8
}
```

---

## Mini-lab: wrap a C buffer-sum API

### The hypothetical C API

```c
// sum_i32.h
// Sums the first `len` elements of `xs`.
// Returns 0 on success.
// Returns -1 if xs is NULL or len is 0.
// On success, writes the result to `*out`.
int sum_i32(const int* xs, size_t len, int* out);
```

### Step 1: Rust declaration

```rust
use std::ffi::c_int;

extern "C" {
    fn sum_i32(xs: *const i32, len: usize, out: *mut i32) -> c_int;
}
```

### Step 2: typed error

```rust
#[derive(Debug, Clone, PartialEq, Eq)]
pub enum SumError {
    EmptySlice,
    CError { code: i32 },
}

impl std::fmt::Display for SumError {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        match self {
            Self::EmptySlice       => write!(f, "empty input"),
            Self::CError { code } => write!(f, "C function returned error code {code}"),
        }
    }
}
```

### Step 3: safe wrapper

```rust
/// Compute the sum of all elements in `xs`.
///
/// Returns `Err(SumError::EmptySlice)` if `xs` is empty.
pub fn sum(xs: &[i32]) -> Result<i32, SumError> {
    if xs.is_empty() {
        return Err(SumError::EmptySlice);
    }

    let mut out: i32 = 0;

    // Safety:
    // - xs.as_ptr() is non-null (xs is non-empty, checked above)
    // - xs.as_ptr() is valid for xs.len() i32 reads (from the slice contract)
    // - xs.as_ptr() is properly aligned (i32 is 4-byte aligned; slices guarantee this)
    // - &mut out is a valid, aligned, writable i32
    // - No aliasing: xs and &out are different allocations
    let rc = unsafe { sum_i32(xs.as_ptr(), xs.len(), &mut out as *mut i32) };

    if rc != 0 {
        return Err(SumError::CError { code: rc });
    }

    Ok(out)
}
```

### Step 4: tests

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn sum_empty_is_error() {
        assert_eq!(sum(&[]), Err(SumError::EmptySlice));
    }

    #[test]
    fn sum_single() {
        assert_eq!(sum(&[42]), Ok(42));
    }

    #[test]
    fn sum_positive() {
        assert_eq!(sum(&[1, 2, 3, 4, 5]), Ok(15));
    }

    #[test]
    fn sum_mixed() {
        assert_eq!(sum(&[-10, 20, -5]), Ok(5));
    }

    #[test]
    fn sum_large() {
        let xs: Vec<i32> = (1..=100).collect();
        // sum 1..=100 = 5050
        assert_eq!(sum(&xs), Ok(5050));
    }
}
```

---

## Linking to C code

In practice, you link to C libraries via build scripts (`build.rs`) or system library linking.

### Linking to a system library

```rust
// In build.rs:
fn main() {
    println!("cargo:rustc-link-lib=z");   // link libz (zlib)
}

// Or via extern block with link attribute:
#[link(name = "z")]
extern "C" {
    fn compress(dest: *mut u8, dest_len: *mut u64, src: *const u8, src_len: u64) -> i32;
}
```

### The `bindgen` tool

For large C headers, `bindgen` automatically generates Rust FFI declarations:

```bash
cargo install bindgen-cli
bindgen /usr/include/zlib.h -o src/bindings.rs
```

Most real FFI crates (like `libz-sys`, `openssl-sys`) use bindgen to generate their bindings.

---

## Code walkthrough: naive → idiomatic

### Naive: leaking raw pointers to callers

```rust
// Bad: caller gets a raw pointer — they must remember to free it correctly
unsafe fn get_name_bad() -> *const c_char {
    let result = some_c_function();
    result  // raw pointer leaks out
}
```

Problems: caller doesn't know if they must free it, how to free it, or when to free it.

### Idiomatic: own the result in Rust

```rust
// Good: convert to owned Rust type immediately
fn get_name() -> Option<String> {
    let ptr = unsafe { some_c_function() };
    if ptr.is_null() {
        return None;
    }
    let name = unsafe { CStr::from_ptr(ptr).to_string_lossy().into_owned() };
    unsafe { free_c_string(ptr); }  // free the C allocation
    Some(name)
}
```

Now callers have a `String` — normal Rust, no unsafe.

### Idiomatic: RAII for C handles

Wrap C handles in Rust structs that implement `Drop`:

```rust
pub struct DbConnection {
    handle: *mut c_void,  // opaque C handle
}

impl DbConnection {
    pub fn open(url: &str) -> Result<Self, DbError> {
        let url = CString::new(url).map_err(|_| DbError::InvalidUrl)?;
        let handle = unsafe { db_open(url.as_ptr()) };
        if handle.is_null() {
            return Err(DbError::ConnectionFailed);
        }
        Ok(Self { handle })
    }

    pub fn query(&self, sql: &str) -> Result<Vec<Row>, DbError> {
        let sql = CString::new(sql).map_err(|_| DbError::InvalidSql)?;
        // ... call db_query(self.handle, sql.as_ptr()) ...
        todo!()
    }
}

impl Drop for DbConnection {
    fn drop(&mut self) {
        if !self.handle.is_null() {
            unsafe { db_close(self.handle); }
        }
    }
}
```

Now `DbConnection` is a normal Rust type — dropped deterministically when it goes out of scope.

---

## Applications in systems programming

### Calling OpenSSL

`openssl-sys` provides raw FFI bindings; the `openssl` crate provides the safe Rust wrapper:

```rust
use openssl::ssl::{SslConnector, SslMethod};

let connector = SslConnector::builder(SslMethod::tls())?.build();
let stream = connector.connect("example.com", tcp_stream)?;
```

Under the hood: `openssl-sys` has thousands of `extern "C"` declarations. The `openssl` crate wraps them in safe types.

### Calling OS APIs (libc)

```rust
use libc;

pub fn get_hostname() -> std::io::Result<String> {
    let mut buf = vec![0u8; 256];
    let rc = unsafe { libc::gethostname(buf.as_mut_ptr() as *mut libc::c_char, buf.len()) };
    if rc != 0 {
        return Err(std::io::Error::last_os_error());
    }
    let len = buf.iter().position(|&b| b == 0).unwrap_or(buf.len());
    String::from_utf8(buf[..len].to_vec()).map_err(|e| std::io::Error::new(std::io::ErrorKind::InvalidData, e))
}
```

### Embedding Python (PyO3)

PyO3 provides a safe FFI layer over CPython:

```rust
use pyo3::prelude::*;

#[pyfunction]
fn add(a: i64, b: i64) -> i64 {
    a + b
}

#[pymodule]
fn my_module(_py: Python<'_>, m: &PyModule) -> PyResult<()> {
    m.add_function(wrap_pyfunction!(add, m)?)?;
    Ok(())
}
```

PyO3 handles all the FFI details behind a safe Rust API.

---

## Safety documentation standard

Always document:

```rust
// Safety:
// - `ptr` is non-null (checked above).
// - `ptr` points to a null-terminated sequence of UTF-8 bytes (guaranteed by the API contract).
// - The memory is valid for the duration of this call (guaranteed by the caller's ownership of `s`).
```

And for the overall wrapper:

```rust
/// # Safety (for the C API being wrapped)
/// The wrapped C function `sum_i32` must:
/// - Not modify `xs` through the `const int*` parameter.
/// - Write exactly one `int` to `*out` on success.
/// - Be thread-safe (documented in the C API specification).
```

---

## Challenges and extensions

1. **Wrap a string API**: declare and wrap a C function `to_uppercase(const char* s, char* out, size_t out_len) -> int` using `CString` and `CStr`. Handle all failure modes.

2. **RAII handle type**: implement a `LibHandle` RAII type for a hypothetical C library that has `lib_open() -> *mut LibCtx` and `lib_close(*mut LibCtx)`.

3. **Callback FFI**: C APIs often accept function pointers as callbacks. Declare and call a C function that accepts `void callback(int result)`. How do you pass a Rust closure to C?

4. **Bindgen integration**: set up `bindgen` in a `build.rs` to generate bindings for a simple C header. Observe what it generates.

---

## Common mistakes

1. **Returning raw pointers to Rust callers** — they won't know how to free them or when they're valid.
2. **Not validating lengths** — passing unchecked `len` to a C function can cause buffer overreads.
3. **Assuming `&str` is a C string** — it's not null-terminated; always convert with `CString`.
4. **Storing the result of `c_str.as_ptr()` after `c_str` drops** — the pointer is invalidated.
5. **Not handling C's error return codes** — return -1 means different things in different APIs.

---

## Exercises

1. Wrap a C function that writes into a caller-provided buffer:
   ```c
   int read_bytes(const char* path, uint8_t* buf, size_t buf_len, size_t* bytes_read);
   ```
2. Build a safe string wrapper around a `char*` API using `CString` and `CStr`.
3. Add tests that validate wrapper behavior (empty input, null pointer handling, error codes).
4. (**harder**) Implement a `Callback<F>` type that boxes a Rust closure and passes a raw function pointer + user data to a C API using the standard `(fn_ptr, void* data)` callback pattern.

---

## References

- [The Rustonomicon: FFI](https://doc.rust-lang.org/nomicon/ffi.html)
- [`std::ffi` docs](https://doc.rust-lang.org/std/ffi/index.html)
- [`bindgen` documentation](https://rust-lang.github.io/rust-bindgen/)
- [`libc` crate](https://docs.rs/libc/latest/libc/) — bindings to C standard library
- [PyO3 guide](https://pyo3.rs/)

## Recap

- FFI is unsafe because Rust's guarantees stop at the C boundary.
- Use `extern "C"` and `#[repr(C)]` for ABI compatibility.
- Convert strings with `CString` (Rust → C) and `CStr` (C → Rust).
- Encapsulate C handles in RAII structs that implement `Drop`.
- Translate C error codes into Rust `Result` types.
- Never return raw pointers to callers — convert to safe Rust types immediately.
