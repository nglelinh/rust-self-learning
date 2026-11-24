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

Khi một chương trình Rust vượt quá 200 dòng, đặt toàn bộ mã vào một tệp `src/main.rs` nhanh chóng biến thành cơn ác mộng khó kiểm thử và khó bảo trì. Nhưng nhiều người mới từ Python hoặc JavaScript sang Rust hay bị lúng túng ở bước tách tệp: tại sao tạo thêm tệp `src/math.rs` rồi mà trình biên dịch vẫn bảo *"không tìm thấy"*? Câu trả lời nằm ở thiết kế tường minh của Rust: **cây module không tự động quét thư mục** — mọi tệp chỉ tồn tại trong chương trình khi được khai báo rõ bằng `mod` từ gốc crate!

Bài học này gỡ rối hoàn toàn hệ thống tổ chức mã của Rust: phân biệt chính xác **Package**, **Crate** (`lib.rs` vs `main.rs`) và **Module**, hiểu nguyên tắc **riêng tư mặc định (private by default)**, và áp dụng kiến trúc chuẩn công nghiệp *thin `main.rs` + testable `lib.rs`*.

---

## Bản đồ kiến thức nền: Kiến trúc phân tầng Package – Crate – Module trong Rust

Nắm vững bốn quy tắc sau sẽ giúp bạn tổ chức dự án Rust hàng chục nghìn dòng mà không bao giờ lạc đường:

1. **Đơn vị biên dịch là Crate, không phải Tệp:** Một `package` (có 1 `Cargo.toml`) có thể chứa tối đa 1 library crate (`src/lib.rs`) và nhiều binary crates (`src/main.rs`, `src/bin/*.rs`). Trình biên dịch `rustc` biên dịch nguyên một **crate** trong một lần, nên các module trong cùng crate có thể gọi chéo lẫn nhau mà không cần tệp header `.h` như C/C++.
2. **Khai báo `mod` vs. Nhập đường dẫn `use`:** `mod foo;` ra lệnh cho trình biên dịch *gắn tệp `foo.rs` vào cây module* (chỉ khai báo **1 lần** ở module cha!). Còn `use crate::foo::bar;` chỉ tạo một tên viết tắt để đỡ phải gõ đường dẫn dài.
3. **Riêng tư mặc định (Private by Default) & Đóng gói bất biến:** Mọi hàm, struct và từng trường (field) của struct đều mặc định `private` đối với module cha và bên ngoài, nhưng **module con luôn nhìn thấy phần private của module cha** — rất lý tưởng để đặt `#[cfg(test)] mod tests` ngay trong cùng tệp mà vẫn kiểm thử được các hàm nội bộ!
4. **Kiến trúc `lib.rs` + `main.rs` mỏng:** Đưa toàn bộ logic nghiệp vụ vào `src/lib.rs` để tái sử dụng và viết integration tests trong `tests/`; giữ `src/main.rs` siêu mỏng chỉ lo đọc tham số dòng lệnh và gọi thư viện.

---

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

---

### Ví dụ truy vết từng dòng (Cây Module & Quyền truy cập): Truy vết lỗi kinh điển khi tách tệp

Giả sử package `calc_app` có cấu trúc gồm cả `src/lib.rs`, `src/math.rs` và `src/main.rs`:

```text
calc_app/
├── Cargo.toml       (name = "calc_app")
└── src/
    ├── lib.rs       --> mod math; pub use math::add;
    ├── math.rs      --> pub fn add(a: i32, b: i32) -> i32 { helper(a, b) }
    │                    fn helper(a: i32, b: i32) -> i32 { a + b }
    └── main.rs      --> fn main() { println!("{}", calc_app::add(2, 3)); }
```

Truy vết đường đi và quyền nhìn thấy (visibility) từ từng vị trí:

| Vị trí gọi | Gọi `add(2, 3)` | Gọi `helper(2, 3)` | Giải thích của trình biên dịch |
| :--- | :--- | :--- | :--- |
| Bên trong `src/math.rs` | Hợp lệ | Hợp lệ | Cùng module |
| Bên trong `math::tests` (module con) | Hợp lệ | **Hợp lệ** | Module con được phép nhìn vào `private` của cha! |
| Bên trong `src/lib.rs` | Hợp lệ (`math::add`) | **Lỗi E0603** | `helper` là private bên trong `math` |
| Bên trong `src/main.rs` | Hợp lệ (`calc_app::add`) | **Lỗi E0603** | `main.rs` là **crate riêng** — chỉ thấy API `pub` của `lib.rs`! |

### Khung giải thích: Bốn nguyên tắc tổ chức mã chuẩn công nghiệp trong Rust (nên đọc chậm)

1. **Đừng bao giờ khai báo `mod math;` ở cả `lib.rs` lẫn `main.rs`:** Làm vậy sẽ buộc trình biên dịch biên dịch tệp `math.rs` thành **hai module tách biệt** thuộc hai crate khác nhau (gây lỗi kiểu trùng tên nhưng khác danh tính!). Chỉ khai báo `mod math;` ở `lib.rs`, còn `main.rs` gọi qua `use calc_app::...`.
2. **Dùng `pub use` để thiết kế mặt tiền API gọn gàng (Re-exporting):** Thay vì bắt người dùng gõ `my_crate::internal::parser::v2::parse`, hãy `pub use internal::parser::v2::parse;` ngay ở gốc `lib.rs`.
3. **Dùng `pub(crate)` cho các hàm dùng chung nội bộ:** Khi một hàm cần chia sẻ giữa các module trong cùng dự án nhưng **không** muốn lộ ra ngoài như API công khai, `pub(crate)` là lựa chọn hoàn hảo.
4. **Ưu tiên đường dẫn tuyệt đối `crate::...` trong nội bộ thư viện:** Đường dẫn bắt đầu bằng `crate::` luôn ổn định khi bạn di chuyển tệp sang thư mục khác, dễ đọc hơn chuỗi `super::super::...`.

**Kiểm tra nhanh.** Trong dự án `calc_app` ở trên, nếu bạn xóa dòng `pub use math::add;` khỏi `src/lib.rs` mà chỉ giữ lại `mod math;`, liệu `src/main.rs` có còn gọi được `calc_app::math::add(2, 3)` không — và vì sao?

---

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
