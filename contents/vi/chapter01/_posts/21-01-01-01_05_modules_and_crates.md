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

Dự án Rust dễ mở rộng nhờ ranh giới module rõ ràng và quy tắc phạm vi truy cập (visibility) tường minh. Bài học này trình bày cách tổ chức một dự án Rust, cách phân giải tên (name resolution) và cách thiết kế một crate nhỏ với API gọn gàng.

## Kế hoạch giảng dạy 60 phút

- 0 đến 10 phút: Phân biệt thuật ngữ: crate, module và package.
- 10 đến 25 phút: Bố cục tệp nguồn: `main.rs`, `lib.rs`, `mod.rs` và cách khai báo `mod`.
- 25 đến 40 phút: Phạm vi truy cập (visibility): `pub`, `pub(crate)` và thiết kế API.
- 40 đến 55 phút: Tái cấu trúc trực tiếp một chương trình đơn tệp thành mô hình `lib + bin`.
- 55 đến 60 phút: Danh sách kiểm tra để giữ cấu trúc module gọn gàng và dễ kiểm thử.

## Mục tiêu học tập

Sau bài học này, bạn có thể:

- Phân biệt rõ: package (dự án Cargo), crate (đơn vị biên dịch) và module (không gian tên).
- Tách mã nguồn thành nhiều module mà không gặp lỗi phân giải đường dẫn.
- Sử dụng `pub` có chủ đích để kiểm soát bề mặt API công khai.
- Tổ chức ứng dụng CLI sao cho `main.rs` mỏng gọn còn logic chính nằm trong `lib.rs`.

## Hệ thống thuật ngữ trong Rust

### Package

Package là một thư mục dự án chứa tệp `Cargo.toml`.

### Crate

Crate là đơn vị mà trình biên dịch Rust trực tiếp biên dịch:

- **Binary crate** có điểm khởi đầu (`fn main`) và biên dịch ra tệp thực thi.
- **Library crate** biên dịch ra thư viện để các crate khác tái sử dụng.

Một package có thể chứa nhiều crate (chẳng hạn một library crate đi kèm nhiều binary crate).

### Module

Module là không gian tên (namespace) dùng để tổ chức mã nguồn bên trong một crate.

## Các bố cục dự án mặc định

### Chỉ có binary

```
src/main.rs
```

### Chỉ có library

```
src/lib.rs
```

### Kết hợp library và binary (khuyến nghị cho hầu hết dự án thực tế)

```
src/lib.rs
src/main.rs
```

Bố cục này mang lại hai lợi ích rõ rệt:

- Phần logic nghiệp vụ nằm trong library nên có thể kiểm thử đơn vị độc lập.
- Tệp binary (`main.rs`) giữ vai trò lớp vỏ mỏng: đọc tham số CLI, gọi hàm từ library và in kết quả.

## Module và bố cục tệp nguồn

Một bố cục module phổ biến trong dự án Rust có dạng:

```
src/
  lib.rs
  main.rs
  parser.rs
  math/
    mod.rs
    stats.rs
```

Để gắn một module vào cây module của crate, ta dùng từ khóa `mod`.

Ví dụ trong `src/lib.rs`:

```rust
pub mod parser;
pub mod math;
```

Trong `src/math/mod.rs`:

```rust
pub mod stats;
```

Sau đó, trong `src/math/stats.rs`, bạn định nghĩa các hàm và kiểu dữ liệu thuộc module `stats`.

Lưu ý: Rust cũng hỗ trợ bố cục mới đặt tệp `src/math.rs` ngang cấp với thư mục `src/math/` thay cho `src/math/mod.rs`. Tuy nhiên, cách dùng `mod.rs` vẫn rất phổ biến trong nhiều mã nguồn hiện có và dễ hình dung khi mới học.

## Đường dẫn (path) và từ khóa `use`

Đường dẫn trong Rust luôn tường minh:

```rust
use crate::math::stats::mean;
```

Các điểm cần lưu ý:

- Tiền tố `crate::` trỏ từ gốc của crate hiện tại.
- Tiền tố `super::` tham chiếu đến module cha trực tiếp.
- Tránh lặp lại các đường dẫn `use` quá sâu ở nhiều tệp; có thể re-export (dùng `pub use`) tại một điểm trung tâm để việc import thuận tiện hơn.

## Phạm vi truy cập (`pub`)

Trong Rust, mọi thành phần đều là **private theo mặc định**. Nguyên tắc này giúp kiểm soát chặt chẽ thiết kế API.

### Private (mặc định)

```rust
fn helper() {}
```

### Công khai với các module khác và người dùng thư viện

```rust
pub fn parse_command(...) { ... }
```

### Chỉ công khai trong nội bộ crate

```rust
pub(crate) fn internal_only() { ... }
```

Nguyên tắc thực hành: chỉ đặt `pub` cho những thành phần thực sự cần thiết, vì một khi đã công khai, việc thay đổi API về sau sẽ khó khăn hơn nhiều.

## Tái cấu trúc trực tiếp: tách chương trình đơn tệp thành `lib + bin`

Ta sẽ lấy một chương trình nhỏ (chẳng hạn vòng lặp REPL ở bài trước) và tách thành cấu trúc thư viện kết hợp tệp thực thi.

### Bước 1: tạo `src/lib.rs`

Đưa phần logic có thể tái sử dụng vào library crate:

- `enum Command`
- `parse_command(&str) -> Result<Command, ...>`
- `execute(Command) -> ...` (hoặc tốt hơn, trả về dữ liệu có cấu trúc thay vì in trực tiếp)

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

Lưu ý: đặt `Command` và `parse_command` là `pub`, còn các hàm phụ trợ bên trong giữ ở mức private trừ khi thực sự cần gọi từ bên ngoài.

### Bước 2: làm mỏng `src/main.rs`

`src/main.rs` lúc này chỉ đảm nhận việc kết nối nhập/xuất với thư viện:

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

Lưu ý: tên crate trong `use hello_cli::...` phải khớp với tên package trong `Cargo.toml` (với dấu gạch ngang `-` được chuyển thành gạch dưới `_`).

### Bước 3: bổ sung kiểm thử trong library

Khi logic đã nằm trong `lib.rs`, việc viết kiểm thử đơn vị trở nên rất gọn nhẹ:

```rust
#[test]
fn parse_help() {
    let c = parse_command("help").unwrap();
    assert!(matches!(c, Command::Help));
}
```

Lưu ý: kiểm thử đơn vị cho hàm phân tích lệnh không cần phụ thuộc vào `stdin`.

## Thiết kế bề mặt API gọn gàng

Một nguyên tắc thực hành đơn giản là chỉ để `main.rs` xử lý ba việc:

- Phân tích đối số dòng lệnh (CLI).
- Gọi các hàm xử lý từ library crate.
- In kết quả đầu ra và trả về mã thoát (exit code).

Toàn bộ logic nghiệp vụ còn lại nên được đặt trong các module có kiểm thử đi kèm.

## Những sai lầm thường gặp

1. Gắn `pub` cho mọi thành phần: khiến việc tái cấu trúc về sau gặp nhiều ràng buộc.
2. Tạo phụ thuộc vòng giữa các module: cần duy trì kiến trúc phân lớp rõ ràng (ví dụ `parser` phụ thuộc `model`, không làm ngược lại).
3. Trộn lẫn thao tác IO vào khắp các hàm xử lý: khiến việc viết kiểm thử đơn vị trở nên khó khăn.

## Bài tập

1. Tách chương trình REPL thành hai module: `command` (mô hình và phân tích lệnh) và `engine` (thực thi lệnh).
2. Dùng `pub use` để re-export `Command` và `parse_command` tại `lib.rs` giúp việc import ngắn gọn hơn.
3. Bổ sung ít nhất 5 kiểm thử đơn vị cho hàm `parse_command`.
4. Thay kiểu lỗi `String` bằng một `enum ParseError` nhỏ gọn.

## Tóm tắt

- Package chứa crate; crate chứa các module.
- Sử dụng `pub` có chủ đích để định hình bề mặt API.
- Đặt logic chính trong library crate và giữ `main.rs` mỏng gọn.
- Bố cục này là nền tảng cho các dự án Rust dễ kiểm thử và dễ bảo trì.

## Video tham khảo

- [LẬP TRÌNH #RUST Part 5: Quản lý dự án và cách sử dụng Packages, Crates, Modules](https://www.youtube.com/watch?v=lbo9RU0pxDE) — Jayden Dang (tiếng Việt). Nên xem trước: giải thích package, crate và module bằng tiếng Việt, khớp với hệ thống thuật ngữ và bố cục tệp của bài học.
- [Rust Modules - Explained Like I'm 5](https://www.youtube.com/watch?v=969j0qnJGi8) — Let's Get Rusty (tiếng Anh). Giải thích ngắn gọn vì sao một tệp nguồn chưa tự động là một module, tập trung vào `mod` và `pub`.
- [Rust's Module System Explained!](https://www.youtube.com/watch?v=5RPXgDQrjio) — Let's Get Rusty (tiếng Anh). Trình bày đầy đủ hơn về `use`, `pub` và hệ thống module phục vụ bước tách `lib` và `bin`.
