---
layout: post
title: "01-01 Cài đặt và công cụ"
chapter: "01"
order: 1
owner: "OpenCode"
lang: vi
categories:
  - chapter01
lesson_type: required
---

Học lập trình Rust gắn liền với việc làm chủ vòng lặp phản hồi: viết mã, đọc thông báo của trình biên dịch và điều chỉnh nhanh chóng. Bài học này thiết lập quy trình làm việc đó và giới thiệu các công cụ cốt lõi bạn sẽ sử dụng hằng ngày.

## Kế hoạch giảng dạy 60 phút

- 0 đến 10 phút: Giới thiệu phạm vi của bộ công cụ (tooling) Rust; cài đặt và kiểm tra môi trường.
- 10 đến 25 phút: Nền tảng Cargo: tạo dự án, chạy, biên dịch và kiểm thử.
- 25 đến 40 phút: Cấu trúc dự án, cách đọc `Cargo.toml` và vai trò của thư mục `target/`.
- 40 đến 50 phút: Quy trình đảm bảo chất lượng mã nguồn với `cargo fmt` và `cargo clippy`.
- 50 đến 60 phút: Tra cứu tài liệu, dùng lệnh trợ giúp và danh sách kiểm tra khi gặp lỗi biên dịch.

## Mục tiêu học tập

Sau bài học này, bạn có thể:

- Kiểm tra môi trường chuỗi công cụ (toolchain) Rust và giải thích vai trò của `rustup`, `rustc`, `cargo`.
- Tạo mới một crate, khai báo thư viện phụ thuộc (dependency) và chạy chương trình trên máy cục bộ.
- Áp dụng quy trình làm việc hằng ngày: `cargo fmt` → `cargo clippy` → `cargo test`.
- Tra cứu nhanh tài liệu của thư viện chuẩn (stdlib) và các thư viện phụ thuộc.

## Mô hình chuỗi công cụ

Bộ công cụ của Rust được tổ chức thành ba lớp rõ ràng:

- `rustup`: cài đặt và quản lý chuỗi công cụ (các phiên bản Rust và thành phần đi kèm).
- `rustc`: trình biên dịch Rust.
- `cargo`: công cụ quản lý build, quản lý thư viện phụ thuộc và chạy kiểm thử.

Trong thực tế làm việc hằng ngày, bạn chủ yếu tương tác qua lệnh `cargo ...`, còn Cargo sẽ tự động gọi trình biên dịch `rustc` ở bên dưới.

## Cài đặt và kiểm tra

### Xác nhận khả năng biên dịch

```bash
rustc --version
cargo --version
```

Kết quả mong đợi là cả hai lệnh đều in ra số phiên bản. Nếu máy có `rustc` nhưng thiếu `cargo`, bản cài đặt hiện tại chưa đầy đủ.

### Quản lý chuỗi công cụ (toolchain)

Rust phát hành phiên bản mới theo chu kỳ ngắn, nên việc quản lý toolchain rất quan trọng vì các lý do sau:

- Một kho mã nguồn có thể yêu cầu phiên bản trình biên dịch mới hơn.
- Hệ thống CI thường cố định (ghim) một phiên bản cụ thể để đảm bảo tính tái lập.
- Một số tính năng thử nghiệm chỉ hoạt động trên kênh `nightly`.

Các lệnh `rustup` thường dùng:

```bash
rustup show
rustup update
rustup toolchain list
```

Khi học tập hoặc giảng dạy, nguyên tắc đơn giản nhất là giữ toàn bộ môi trường ở kênh **stable**, trừ khi có lý do cụ thể cần chuyển sang kênh khác.

## Nền tảng Cargo

### Tạo mới một binary crate

```bash
cargo new hello-cli
cd hello-cli
```

Biên dịch và chạy chương trình:

```bash
cargo run
```

Chỉ biên dịch mà không chạy (nhanh hơn khi chỉ cần kiểm tra quá trình build):

```bash
cargo build
```

Chạy bộ kiểm thử:

```bash
cargo test
```

### Cấu trúc thư mục vừa tạo

Cargo tạo sẵn các thành phần cơ bản sau:

- `Cargo.toml`: tệp khai báo siêu dữ liệu (metadata) của gói và danh sách thư viện phụ thuộc.
- `src/main.rs`: điểm khởi đầu của một binary crate.
- `target/`: thư mục chứa các tệp sinh ra sau quá trình biên dịch (thường không đưa vào Git).

Mở `Cargo.toml` để quan sát các trường chính:

```toml
[package]
name = "hello-cli"
version = "0.1.0"
edition = "2021"

[dependencies]
```

Lưu ý:

- Trường `edition` quy định phiên bản quy ước ngôn ngữ và cú pháp mặc định; phần lớn dự án hiện nay dùng edition `2021` (hoặc `2024` trên các bản Rust mới hơn).
- Theo mặc định, các thư viện phụ thuộc được phân giải và tải từ `crates.io`.

## Thêm thư viện phụ thuộc (minh họa trực tiếp)

Chọn một thư viện nhỏ để minh họa, chẳng hạn `anyhow` giúp xử lý lỗi gọn nhẹ.

Cập nhật tệp `Cargo.toml`:

```toml
[dependencies]
anyhow = "1"
```

Chỉnh sửa `src/main.rs`:

```rust
use anyhow::Result;

fn main() -> Result<()> {
    println!("hello-cli ready");
    Ok(())
}
```

Biên dịch và chạy lại:

```bash
cargo run
```

Lưu ý: ở lần chạy đầu tiên, Cargo sẽ tải xuống, biên dịch và lưu vào bộ nhớ đệm (cache) thư viện phụ thuộc. Từ lần build thứ hai trở đi, tốc độ sẽ nhanh hơn đáng kể.

## Quy trình đảm bảo chất lượng hằng ngày

### Định dạng mã nguồn: `cargo fmt`

Quy chuẩn định dạng mã nguồn trong Rust đã được thống nhất. Ở hầu hết các dự án, định dạng mã được tự động hóa thay vì tranh luận thủ công.

```bash
cargo fmt
```

Nếu hệ thống chưa có `cargo fmt`, hãy cài đặt thành phần `rustfmt`:

```bash
rustup component add rustfmt
```

### Phân tích tĩnh (lint): `cargo clippy`

Clippy là công cụ phân tích tĩnh của Rust, giúp phát hiện các lỗi phổ biến và gợi ý cách viết mã đúng chuẩn ngôn ngữ hơn.

```bash
cargo clippy
```

Nếu chưa cài đặt Clippy:

```bash
rustup component add clippy
```

Lưu ý: nên đọc kỹ từng cảnh báo của Clippy để hiểu nguyên nhân thay vì tắt cảnh báo theo thói quen.

### Thứ tự thực hiện khuyến nghị

Để giữ vòng phản hồi gọn gàng và nhất quán, hãy chạy theo thứ tự sau:

```bash
cargo fmt
cargo clippy
cargo test
```

Nhiều hệ thống CI cũng vận hành theo đúng trình tự này: kiểm tra định dạng, phân tích tĩnh, rồi mới xác minh tính đúng đắn qua kiểm thử.

## Tra cứu tài liệu và trợ giúp

### Tạo tài liệu cục bộ

```bash
cargo doc --open
```

Lệnh này tạo và mở tài liệu HTML cục bộ cho chính crate của bạn cùng toàn bộ các thư viện phụ thuộc đang sử dụng.

### Các lệnh trợ giúp nhanh

```bash
cargo --help
cargo test --help
rustc --explain E0382
```

Lệnh `rustc --explain <mã_lỗi>` là một trong những cách tự học Rust hiệu quả nhất: mỗi khi gặp mã lỗi biên dịch, bạn có thể tra trực tiếp lời giải thích chi tiết và ví dụ minh họa ngay trên terminal.

## Sự cố thường gặp và cách xử lý

### 1. Máy cục bộ biên dịch thành công nhưng CI báo lỗi

Các nguyên nhân phổ biến gồm:

- Khác biệt phiên bản toolchain giữa máy cục bộ và CI.
- Mã nguồn phụ thuộc vào hành vi đặc thù của hệ điều hành.
- Thiếu feature cần thiết của một thư viện phụ thuộc.

Cách xử lý: đối chiếu kết quả `rustc --version` trên máy cục bộ với môi trường CI. Chỉ chạy `cargo clean` khi có cơ sở nghi ngờ tệp build cũ bị hỏng.

### 2. Lỗi khi khai báo thư viện phụ thuộc

Nếu bước thêm dependency thất bại:

- Kiểm tra xem khai báo đã đặt đúng trong mục `[dependencies]` hay chưa.
- Chạy `cargo build -vv` để xem thông tin nhật ký chi tiết hơn.

### 3. Thời gian biên dịch chậm

Một số nguyên tắc cơ bản cần nắm:

- Lần biên dịch đầu tiên thường mất thời gian hơn; các lần sau được tăng tốc nhờ biên dịch gia tăng (incremental compilation).
- Khi chỉ cần kiểm tra kiểu dữ liệu và lỗi biên dịch mà chưa cần tạo tệp thực thi, hãy dùng `cargo check` thay vì `cargo build`:

```bash
cargo check
```

## Bài tập trên lớp (10 đến 15 phút)

1. Tạo một crate mới mang tên `hello-cli`.
2. Thêm một thư viện phụ thuộc vào `Cargo.toml`.
3. Chỉnh sửa hàm `main` để trả về kiểu `Result`.
4. Chạy lần lượt `cargo fmt`, `cargo clippy` và `cargo test`.
5. Viết thêm một bài kiểm thử đơn vị (unit test) đơn giản và chạy lại `cargo test`.

## Bài tập về nhà

- Tạo crate thứ hai mang tên `sandbox` và thực hành:
  - `cargo doc --open`
  - `cargo check`
  - `rustc --explain` với một mã lỗi biên dịch mà bạn gặp phải.

## Tóm tắt

- `rustup` quản lý các phiên bản toolchain và thành phần đi kèm.
- `cargo` là công cụ trung tâm cho công việc hằng ngày.
- Duy trì quy trình: định dạng (`fmt`), phân tích tĩnh (`clippy`), kiểm thử (`test`).
- Chủ động tự gỡ lỗi bằng cờ `--help` và lệnh `rustc --explain`.

## Video tham khảo

- [\[Rust Cơ Bản\] Bài 1: Cài Đặt Ngôn Ngữ Lập Trình Rust  – Khóa Học Lập Trình Rust Cho Người Mới](https://www.youtube.com/watch?v=zEcr5XlOgxE) — Vie Lang (tiếng Việt). Nên xem trước: hướng dẫn cài đặt Rust bằng tiếng Việt, bám sát phần `rustup` và chương trình đầu tiên của bài học.
- [Hướng Dẫn Cài Đặt Môi trường làm việc Cho #RUST](https://www.youtube.com/watch?v=XiS7MB23NoE) — Jayden Dang (tiếng Việt). Hướng dẫn thiết lập môi trường phát triển Rust, bổ trợ cho bước kiểm tra toolchain.
- [Rust Installation, Hello World, Hello Cargo -  Full Crash Rust Tutorial for Beginners](https://www.youtube.com/watch?v=R33h77nrMqc) — Francesco Ciulla (tiếng Anh). Trình bày `rustup`, Cargo và vòng lặp chỉnh sửa–chạy thử hằng ngày.
