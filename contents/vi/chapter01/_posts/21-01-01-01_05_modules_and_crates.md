---
layout: post
title: "01-05 Hàm, module và crate"
chapter: "01"
order: 5
owner: "OpenCode"
lang: vi
categories:
  - chapter01
lesson_type: required
---

Codebase Rust lớn lên nhờ ranh giới module rõ và visibility tường minh. Bài này dạy cách dự án Rust được tổ chức, tên được resolve thế nào, và cách thiết kế một crate nhỏ với API sạch.

## Kế hoạch giảng 60 phút

- 0 đến 10 phút: Crate vs module vs package (từ vựng).
- 10 đến 25 phút: Bố cục file: `main.rs`, `lib.rs`, `mod.rs` (layout hiện đại) và khai báo `mod`.
- 25 đến 40 phút: Visibility: `pub`, `pub(crate)`, và thiết kế API.
- 40 đến 55 phút: Refactor trực tiếp: chương trình một file thành `lib + bin`.
- 55 đến 60 phút: Checklist để module sạch và dễ test.

## Mục tiêu học

Cuối bài, bạn có thể:

- Giải thích: package (dự án Cargo) vs crate (đơn vị biên dịch) vs module (namespace).
- Tách code thành module mà không làm hỏng name resolution.
- Dùng `pub` có chủ đích để định hình API công khai.
- Tổ chức CLI sao cho `main.rs` mỏng và logic nằm trong `lib.rs`.

## Từ vựng (Rust nghĩa là gì)

### Package

Package là một thư mục có `Cargo.toml`.

### Crate

Crate là thứ Rust biên dịch.

- Binary crate có điểm vào (`fn main`) và sinh executable.
- Library crate sinh thư viện tái sử dụng.

Một package có thể sinh nhiều crate (nhiều binary cộng một library, chẳng hạn).

### Module

Module là namespace bên trong một crate.

## Các layout mặc định

### Chỉ binary

```
src/main.rs
```

### Chỉ library

```
src/lib.rs
```

### Library + binary (nên dùng cho hầu hết dự án thật)

```
src/lib.rs
src/main.rs
```

Điểm giảng: layout này lý tưởng vì:

- logic thật sự có thể test trong library
- binary là lớp mỏng (parse CLI, nối dây, in)

## Module và bố cục file

Layout module Rust hiện đại thường trông như:

```
src/
  lib.rs
  main.rs
  parser.rs
  math/
    mod.rs
    stats.rs
```

Để khai báo module, bạn dùng `mod`.

Ví dụ trong `lib.rs`:

```rust
pub mod parser;
pub mod math;
```

Rồi trong `src/math/mod.rs`:

```rust
pub mod stats;
```

Và trong `src/math/stats.rs`, bạn định nghĩa hàm/kiểu.

Ghi chú giảng viên: Rust cũng hỗ trợ layout "thư mục module" mới hơn, đôi khi không cần `mod.rs`, nhưng cách trên dễ dạy và phổ biến trong code cũ.

## Path và `use`

Path trong Rust là tường minh.

```rust
use crate::math::stats::mean;
```

Điểm giảng:

- `crate::` nghĩa là từ gốc crate.
- Dùng `super::` để tham chiếu module cha.
- Tránh chuỗi `use` sâu ở nhiều file; re-export từ một chỗ trung tâm khi giúp ergonomics.

## Visibility (`pub`)

Rust mặc định là private. Điều này rất quan trọng cho thiết kế API.

### Private theo mặc định

```rust
fn helper() {}
```

### Public với module khác (và với người dùng library)

```rust
pub fn parse_command(...) { ... }
```

### Public chỉ trong crate

```rust
pub(crate) fn internal_only() { ... }
```

Quy tắc ngón tay cái: public càng ít càng tốt. API công khai khó đổi hơn.

## Refactor trực tiếp: tool một file thành `lib + bin`

Ta sẽ lấy một chương trình nhỏ (ví dụ REPL bài trước) và tách nó.

### Bước 1: tạo `lib.rs`

Chuyển logic tái sử dụng vào library crate:

- `enum Command`
- `parse_command(&str) -> Result<Command, ...>`
- `execute(Command) -> ...` (hoặc tốt hơn, trả output có cấu trúc)

Ví dụ `src/lib.rs`:

```rust
pub mod command;

pub use command::{parse_command, Command};
```

Ví dụ `src/command.rs`:

```rust
#[derive(Debug)]
pub enum Command {
    Add(i32, i32),
    Mul(i32, i32),
    Help,
    Quit,
}

pub fn parse_command(line: &str) -> Result<Command, String> {
    // parsing logic
    # let _ = line;
    # Ok(Command::Help)
}
```

Điểm giảng: giữ `Command` public, nhưng giữ hàm phụ private trừ khi cần.

### Bước 2: làm mỏng `main.rs`

`src/main.rs` trở thành code nối dây:

```rust
use std::io;

use hello_cli::{parse_command, Command};

fn main() {
    println!("type 'help' for commands");
    loop {
        let mut line = String::new();
        if io::stdin().read_line(&mut line).is_err() {
            eprintln!("stdin error");
            break;
        }

        let cmd = match parse_command(&line) {
            Ok(c) => c,
            Err(e) => {
                eprintln!("error: {e}");
                continue;
            }
        };

        match cmd {
            Command::Quit => break,
            _ => {
                // call into library logic
            }
        }
    }
}
```

Lưu ý: tên crate trong `use hello_cli::...` phải khớp tên package trong `Cargo.toml`.

### Bước 3: thêm test trong library

Vì logic nằm trong `lib.rs`, test trở nên đơn giản:

```rust
#[test]
fn parse_help() {
    let c = parse_command("help").unwrap();
    matches!(c, Command::Help);
}
```

Điểm giảng: unit test cho parse không cần stdin.

## Thiết kế bề mặt API sạch

Dạy một quy tắc đơn giản: `main.rs` chủ yếu nên chứa:

- parse đối số CLI
- gọi hàm library
- in output / mã thoát

Mọi thứ khác nên sống trong module có test.

## Sai lầm thường gặp

1. Biến mọi thứ thành `pub`: refactor sau này đau.
2. Phụ thuộc module vòng: giữ phân lớp (ví dụ `parser` phụ thuộc `model`, không ngược lại).
3. Nhét IO khắp nơi: test trở nên khó.

## Bài tập

1. Tách REPL thành `command` (parse/mô hình) và `engine` (thực thi).
2. Re-export `Command` và `parse_command` từ `lib.rs` để import dễ hơn.
3. Thêm ít nhất 5 unit test cho parse.
4. Thay lỗi `String` bằng `enum ParseError` nhỏ.

## Tóm tắt

- Package chứa crate; crate chứa module.
- Dùng `pub` có chủ đích để định hình API.
- Đặt logic thật trong library; giữ `main` mỏng.
- Cấu trúc này là nền cho dự án Rust dễ bảo trì.
