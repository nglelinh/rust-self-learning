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

Lập trình Rust chủ yếu là làm chủ vòng phản hồi: viết code, đọc thông báo của trình biên dịch, rồi chỉnh nhanh. Bài này dựng vòng đó và giải thích vài công cụ bạn sẽ dùng mỗi ngày.

## Kế hoạch giảng 60 phút

- 0 đến 10 phút: Tooling Rust là gì (và không phải là gì). Cài đặt và kiểm tra.
- 10 đến 25 phút: Nền tảng Cargo: tạo, chạy, build, test.
- 25 đến 40 phút: Cấu trúc dự án + đọc `Cargo.toml` + hiểu target.
- 40 đến 50 phút: Vòng chất lượng: `fmt` rồi `clippy`.
- 50 đến 60 phút: Tài liệu, lệnh help, và checklist ngắn khi build lỗi.

## Mục tiêu học

Cuối bài, bạn có thể:

- Kiểm tra toolchain Rust hoạt động và giải thích `rustup`, `rustc`, `cargo` làm gì.
- Tạo một crate, thêm dependency, và chạy local.
- Dùng vòng làm việc hàng ngày: `cargo fmt` rồi `cargo clippy` rồi `cargo test`.
- Tìm tài liệu nhanh (stdlib và dependency).

## Mô hình toolchain (bản đồ tư duy)

Tooling Rust tách thành ba lớp:

- `rustup`: cài và quản lý toolchain (phiên bản Rust và các component).
- `rustc`: trình biên dịch.
- `cargo`: hệ thống build + quản lý dependency + runner test.

Thực tế, bạn hầu như chỉ gõ `cargo ...` và để Cargo gọi compiler.

## Cài đặt và kiểm tra

### Xác nhận bạn biên dịch được

```bash
rustc --version
cargo --version
```

Kỳ vọng: cả hai in ra phiên bản. Nếu có `rustc` mà không có `cargo`, bản cài chưa đủ.

### Toolchain (vì sao bạn cần quan tâm)

Rust ra phiên bản nhanh. Toolchain quan trọng vì:

- Một repo có thể đòi compiler mới hơn.
- CI thường ghim một phiên bản.
- Tính năng nightly cần nightly.

Các lệnh `rustup` thường dùng:

```bash
rustup show
rustup update
rustup toolchain list
```

Nếu đang dạy, quy tắc đơn giản nhất: giữ mọi người trên **stable** trừ khi có lý do rõ để lệch.

## Nền tảng Cargo

### Tạo binary crate mới

```bash
cargo new hello-cli
cd hello-cli
```

Chạy:

```bash
cargo run
```

Build mà không chạy (nhanh hơn khi chỉ cần compile):

```bash
cargo build
```

Chạy test:

```bash
cargo test
```

### Hiểu cấu trúc vừa được tạo

Cargo tạo:

- `Cargo.toml`: metadata gói và dependency.
- `src/main.rs`: điểm vào của binary crate.
- `target/`: artifact build (thường không commit).

Mở `Cargo.toml` và chỉ ra các trường chính:

```toml
[package]
name = "hello-cli"
version = "0.1.0"
edition = "2021"

[dependencies]
```

Ghi chú:

- `edition` đổi mặc định ngôn ngữ và idiom. Hầu hết dự án mới dùng 2021.
- Dependency được resolve từ `crates.io` theo mặc định.

## Thêm dependency (demo trực tiếp)

Chọn một dependency nhỏ để minh họa. Ví dụ: `anyhow` để xử lý lỗi nhanh.

Sửa `Cargo.toml`:

```toml
[dependencies]
anyhow = "1"
```

Rồi trong `src/main.rs`:

```rust
use anyhow::Result;

fn main() -> Result<()> {
    println!("hello-cli ready");
    Ok(())
}
```

Build và chạy:

```bash
cargo run
```

Điểm giảng: Cargo sẽ tải, compile, và cache dependency. Lần build thứ hai nhanh hơn nhiều.

## Vòng chất lượng hàng ngày

### Format: `cargo fmt`

Format Rust đã được chuẩn hóa. Ở hầu hết repo, format không phải chủ đề tranh luận.

```bash
cargo fmt
```

Nếu thiếu `cargo fmt`, cài component:

```bash
rustup component add rustfmt
```

### Lint: `cargo clippy`

Clippy là linter của Rust: bắt lỗi thường gặp và gợi ý code idiomatic hơn.

```bash
cargo clippy
```

Nếu thiếu Clippy:

```bash
rustup component add clippy
```

Ghi chú giảng viên: dạy học viên **đọc** thông báo clippy, không tắt mặc định.

### Thứ tự nên dùng

Để vòng phản hồi gọn:

```bash
cargo fmt
cargo clippy
cargo test
```

Nhiều pipeline CI nghĩ theo đúng thứ tự này: style, kiểm tra tĩnh, rồi tính đúng.

## Tài liệu và help (cách tự gỡ tắc)

### Docs có sẵn

```bash
cargo doc --open
```

Lệnh này sinh tài liệu local cho crate của bạn và các dependency.

### Mẫu help nhanh

```bash
cargo --help
cargo test --help
rustc --explain E0382
```

Lệnh cuối (`rustc --explain ...`) là một trong những cách học Rust tốt nhất. Thấy mã lỗi thì giải thích mã đó.

## Sự cố thường gặp và cách gỡ

### 1. Máy mình build được, CI thì không

Nguyên nhân điển hình:

- Khác phiên bản toolchain.
- Dựa vào hành vi riêng của nền tảng.
- Thiếu feature của dependency.

Việc cần làm: so `rustc --version` local với CI. Chỉ `cargo clean` khi nghi artifact cũ thật sự.

### 2. Rối dependency

Nếu thêm dependency thất bại:

- Kiểm tra bạn sửa đúng section `[dependencies]`.
- Chạy `cargo build -vv` để thấy chi tiết hơn.

### 3. Build chậm

Dạy phần cơ bản:

- Lần build đầu chậm, các lần sau incremental.
- `cargo check` nhanh hơn `cargo build` khi bạn chỉ cần type check.

```bash
cargo check
```

## Bài tập trên lớp (10 đến 15 phút)

1. Tạo crate tên `hello-cli`.
2. Thêm một dependency.
3. Để `main` trả về `Result`.
4. Chạy `cargo fmt`, `cargo clippy`, và `cargo test`.
5. Thêm một unit test (dù tầm thường) rồi chạy lại `cargo test`.

## Bài về nhà

- Tạo crate thứ hai `sandbox` và thử:
  - `cargo doc --open`
  - `cargo check`
  - `rustc --explain` với một lỗi compiler bạn gặp

## Tóm tắt

- `rustup` quản lý toolchain.
- `cargo` là cửa vào hàng ngày.
- Dùng vòng: format, lint, test.
- Học cách tự gỡ tắc bằng `--help` và `rustc --explain`.
