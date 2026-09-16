---
layout: post
title: "01-06 Ứng dụng hiện đại — Toolchain, WASM và Cargo trong sản xuất"
chapter: "01"
order: 6
owner: "OpenCode"
lang: vi
categories:
  - chapter01
lesson_type: optional
---

Bài tùy chọn này không dạy lại `rustup`, Cargo hay module. Nó chỉ ra chỗ những công cụ ấy đang chạy ở quy mô công nghiệp: chỉ mục crates.io kiểu sparse, language server chính thức, toolchain WebAssembly component, và các đợt chuyển edition mà team sản xuất thực sự ship.

## Kế hoạch giảng 60 phút

- 0 đến 10 phút: Vì sao tooling *chính là* sản phẩm với shop Rust (phút CI, lockfile, MSRV).
- 10 đến 25 phút: Sparse index của Cargo (RFC 2789) và đồ thị workspace trong crate lớn.
- 25 đến 40 phút: rust-analyzer và rustc — hai trình biên dịch, một ngôn ngữ.
- 40 đến 55 phút: WASM / WASI 0.2 và `wasm-bindgen` như một target thứ hai, không phải đồ chơi.
- 55 đến 60 phút: Edition 2024 như một cuộc di cư sản xuất, không phải bài blog.

## Mục tiêu

Sau bài này bạn giải thích được vì sao vòng toolchain Chương 1 — tạo crate, ghim toolchain, thêm dependency, format, lint, test — cũng chính là vòng mà Cloudflare, nhóm rustc và các runtime WASM đang dùng, chỉ khác ở kỷ luật lockfile. Bạn nhận ra giao thức sparse registry, vai trò LSP chính thức của rust-analyzer, và WASI 0.2 như một *target* chọn bằng đúng cử chỉ `rustup target add` đã học. Chuyển mô hình tư duy: từ “công cụ cài để làm bài tập” sang “công cụ chặn cửa các dịch vụ nghìn tỷ request.”

## Điều kiện tiên quyết

Bạn đã xong các bài bắt buộc Chương 1: `rustup`/`cargo` chạy được, crate binary đầu tiên, kiểu nguyên thủy, luồng điều khiển, và tách `lib.rs` + `main.rs`. Không cần async, unsafe hay WASM. Đọc được `Cargo.toml` và một file CI YAML là đủ.

## Giới thiệu

Từ 2018 đến khoảng 2021, “Rust trong sản xuất” thường nghĩa là một service hoặc một hot path viết lại. Giữa 2022 và 2026 nút thắt đổi chỗ. Team không còn hỏi rustc có phát ra mã nhanh không; họ hỏi *workspace* có clone, index, lint và ship được trên laptop, trên CI, và lên `wasm32-wasip2` mà không cần toolchain tự chế hay không. Chỉ mục git cũ của Cargo cho crates.io thành bài toán quy mô. rust-analyzer thành compiler mặc định cho công việc tương tác. WebAssembly hết là target demo và có ABI component.

Shop C/C++ vẫn coi compiler, trình quản lý gói và language server là ba nhà cung cấp. Cược bất thường của Rust là chúng là một sản phẩm. Bài này lần theo cược đó vào các hệ thống bạn bấm được hôm nay.

## Khái niệm cốt lõi

### Sparse index là giao thức sản xuất, không phải cờ tiện lợi

Đến 2023, Cargo biết crates.io bằng cách clone một kho git khổng lồ chứa file index. RFC 2789 (“Sparse HTTP protocol for Cargo”) thay bằng HTTP GET *chỉ những crate bạn phụ thuộc*. Rust 1.68 (tháng 3/2023) ổn định giao thức; Rust 1.70 (tháng 6/2023) đặt `sparse` làm mặc định cho crates.io. Bài Inside Rust kêu gọi cộng đồng thử `CARGO_REGISTRIES_CRATES_IO_PROTOCOL=sparse` với `https://index.crates.io/`.

Thay đổi ấy vô hình trong repo bài tập mười crate và quyết định trong workspace 400 crate. Nó cũng nhắc rằng **Cargo là chương trình hệ thống có mạng**: HTTP/2, cache và hash lockfile cũng “Rust” không kém `let`.

```toml
# Ghim lịch sử — hữu ích khi dạy image CI cũ.
# Từ 1.70 đây là mặc định của crates.io.
[registries.crates-io]
protocol = "sparse"
```

Vì sao dự án tốn một RFC cho cái index? Vì chương trình đầu tiên bạn viết (`cargo new`) cũng là chương trình đầu tiên một cache CDN phải phục vụ cả triệu lần mỗi ngày. Tooling không scale không phải “tooling cho người mới”; đó là hạ tầng hỏng.

### rust-analyzer là rustc thứ hai với ngân sách độ trễ khác

`rust-analyzer` là triển khai LSP chính thức. Nó không phải highlighter gắn lên rustc. Nó type-check file chưa hoàn chỉnh, chạy build script, và hiểu workspace — cùng đồ thị module bạn học ở bài 01-05, truy vấn ở độ trễ từng phím. Team sản xuất coi rust-analyzer hỏng như compiler hỏng: nếu IDE không resolve được `crate::auth::Token`, reviewer sẽ không tin thay đổi đó.

Điểm dạy nằm ở kiến trúc. rustc tối ưu *artifact đúng*. rust-analyzer tối ưu *chương trình dở dang*. Cả hai đọc `Cargo.toml`, cả hai tôn trọng edition, cả hai phải hiểu `cfg`. Khi sinh viên nói “compiler chậm,” hãy hỏi *compiler nào* và *đồ thị crate nào*.

### WASM là target triple, không phải ngôn ngữ mới

`rustup target add wasm32-unknown-unknown` và gần đây WASI 0.2 (`wasm32-wasip2` cùng Component Model) dùng đúng `cargo build --target` bạn đã chạy cho máy mình. Bytecode Alliance phát hành WASI 0.2 (Preview 2) ngày 25 tháng 1 năm 2024: giao diện WIT cho clock, random, filesystem, socket, CLI và HTTP, đưa WASI khỏi ABI kiểu C sang Component Model. Wasmtime và `jco` là hai triển khai đầu tiên vượt bộ kiểm thử tính di động.

```bash
rustup target add wasm32-unknown-unknown
cargo build --target wasm32-unknown-unknown --release
```

`wasm-bindgen` và `wasm-pack` nằm *trên* target đó: chúng sinh glue JS, không phải hệ kiểu thứ hai. Ý Chương 1 “crate là đơn vị biên dịch có target” chính là lý do Rust thành ngôn ngữ mặc định của runtime WASM nghiêm túc (Wasmtime, WasmEdge, wasmCloud). Bạn không học ngôn ngữ mới; bạn chọn backend.

### Edition là cuộc di cư sản xuất

Edition 2024 đi cùng Rust 1.85 ngày 20 tháng 2 năm 2025 (RFC 3501). Đây là edition lớn nhất từ trước: thêm prelude (`Future`, `IntoFuture`), một số API môi trường thành `unsafe`, và chỉnh thứ tự drop. Team sản xuất không “nâng Rust” bằng một tweet; họ đổi `edition = "2024"` trong `Cargo.toml`, chạy `cargo fix --edition`, và ghim MSRV trên CI.

```toml
[package]
name = "edge-filter"
version = "0.4.2"
edition = "2024"
rust-version = "1.85"
```

Trường `rust-version` là cách crate nói với Cargo “đừng cố biên dịch tôi trên 1.76.” Đó là đúng bản năng ghim bạn luyện với `rustup show`, áp lên cả đội máy.

## Walkthrough mã

Bắt đầu từ crate bạn đã biết cách tạo, rồi lớn lên theo kiểu repo service 2024 — không bịa thêm tính năng ngôn ngữ.

### Ngây thơ: một binary “chỉ cần build được”

```rust
// src/main.rs
fn main() {
    println!("edge-filter ready");
}
```

```toml
# Cargo.toml
[package]
name = "edge-filter"
version = "0.1.0"
edition = "2021"
```

Nó biên dịch. Nó chẳng dạy gì về cách Tokio, rustc hay Wasmtime thực sự sống. Lỗi sản xuất đầu tiên không phải lỗi kiểu; đó là “CI dùng toolchain khác laptop.”

### Sửa theo compiler: ghim toolchain trong repo

```toml
# rust-toolchain.toml  (commit cạnh Cargo.toml)
[toolchain]
channel = "1.85.0"
components = ["rustfmt", "clippy", "rust-analyzer"]
targets = ["wasm32-unknown-unknown"]
```

`rustup` đọc file này khi bạn `cd` vào repo. Đó là mô hình Chương 1 — rustup quản lý toolchain — áp cho cả nhóm. Nếu `cargo --version` của bạn khác CI, file này là hợp đồng.

Lỗi phổ biến thứ hai: “máy tôi chạy vì index git còn cũ.” Trên image trước 1.70 bạn sẽ thấy dòng `Updating crates.io index` kéo vài phút. Giao thức sparse biến nó thành vài HTTP GET. Sinh viên cần *gọi tên* được giao thức, không chỉ ngồi chờ.

### Idiomatic: workspace + thư viện + hai target

```text
edge-filter/
  Cargo.toml
  rust-toolchain.toml
  crates/
    filter-core/src/lib.rs
    filter-cli/src/main.rs
    filter-wasm/src/lib.rs
```

```toml
# Cargo.toml (workspace gốc)
[workspace]
members = ["crates/*"]
resolver = "2"

[workspace.package]
edition = "2024"
rust-version = "1.85"
```

```rust
// crates/filter-core/src/lib.rs
pub fn allow(host: &str) -> bool {
    !host.ends_with(".invalid")
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn blocks_reserved_tld() {
        assert!(!allow("phishing.invalid"));
    }
}
```

Crate CLI phụ thuộc `filter-core` và giữ mỏng — đúng thiết kế 01-05. Crate WASM *cũng* phụ thuộc `filter-core` và biên dịch bằng `--target wasm32-unknown-unknown`. Một đồ thị module, hai artifact. Đó là cấu trúc của dự án wasm-bindgen, Cloudflare Workers (`workers-rs`) và ví dụ Wasmtime.

Nếu bạn thêm dependency mà không nghĩ, Clippy và `cargo deny` sẽ là giáo viên tiếp theo. `cargo-deny` (Embark Studios) và `cargo-audit` (Rust Secure Code WG) là cách CI sản xuất từ chối crate bị yank và advisory RustSec. Chúng không phải tính năng ngôn ngữ; chúng là *phần còn lại của vòng toolchain* bạn bắt đầu ở 01-01.

## Ví dụ

### Ví dụ 1 — crates.io như hệ phân tán

Sparse index crates.io (`https://index.crates.io/`) là dịch vụ HTTP sản xuất. Cargo là client. Khi bạn viết `serde = "1"` bạn đang gửi truy vấn phân giải phiên bản tới registry phải khớp `Cargo.lock`. RFC 2789 tồn tại vì mô hình clone git cũ không chịu nổi đồ thị crate năm 2023.

Đọc lockfile như một artifact hệ thống:

```toml
[[package]]
name = "serde"
version = "1.0.217"
source = "registry+https://github.com/rust-lang/crates.io-index"
checksum = "..."
```

`checksum` là lý do `cargo install` ở hai lục địa ra cùng byte. Bước “thêm dependency” của Chương 1 là fetch theo địa chỉ nội dung. Sửa lockfile bằng tay như sửa binary bằng tay.

### Ví dụ 2 — rustc và rust-analyzer trên cùng workspace

Trình biên dịch Rust tự nó là workspace Cargo hàng trăm crate (`compiler/`, `library/`, `src/tools/`). rust-analyzer phải nạp *một phần* đồ thị đó khi bạn nhảy tới định nghĩa trong rustc. Tokio, Wasmtime và Zed cũng vậy. Nếu bản đồ module 01-05 của bạn bừa — `pub use` đại trà, module vòng, `lib.rs` 4.000 dòng — cả hai compiler đều trả giá mỗi lần gõ phím.

Bài tập hữu ích: mở một crate tầm trung bạn đang phụ thuộc (`grep serde Cargo.lock`) và chạy `cargo metadata --no-deps --format-version 1 | head`. JSON đó là bản đồ module/crate mà công cụ chia sẻ. Đồ thị build sản xuất không bí ẩn; đó là `cargo metadata` ở quy mô lớn.

### Ví dụ 3 — WASI 0.2 như “HĐH” thứ hai

WASI 0.2 là tập world WIT: `wasi:cli/command`, `wasi:http/proxy`, và các world khác. Crate Rust nhắm WASI vẫn là crate. Thứ đổi là *mặt syscall*: thay libc, bạn import giao diện component. Wasmtime (Bytecode Alliance) triển khai các import đó bằng Rust. Host là chương trình Rust; guest có thể là Rust, JS (`jco`) hoặc C.

```rust
// Tư duy phía guest: bạn vẫn sở hữu main, chỉ link std khác.
fn main() {
    // Trên wasip2 lệnh này in qua wasi:cli, không phải write(2) trên laptop.
    println!("filter-wasm guest started");
}
```

Câu hỏi phản tư: nếu module WASM không chia sẻ linear memory như file `.so` của C, ý tưởng nào ở Chương 1 vừa cứu bạn khỏi một lớp lỗi FFI? (Trả lời: biên giới crate cộng ABI có kiểu, thay vì “cầm lấy `char*`.”)

## Ứng dụng trong lập trình hệ thống

**CI như trang trại compiler.** rustc, Clippy, rustfmt và rust-analyzer được cài bằng cùng `rust-toolchain.toml` trên GitHub Actions, Buildkite và máy local. Team bỏ qua việc ghim sẽ mất tuần đuổi “chạy trên 1.82 / gãy trên 1.85.”

**Chính sách chuỗi cung ứng.** `cargo deny check advisories bans licenses sources` là cách nhiều công ty 2024–2026 chặn merge. Nó ngồi cạnh `cargo test`, không thay thế. Vòng chất lượng Chương 1 thêm bước thứ tư: format, lint, test, *policy*.

**Sản phẩm đa target.** Thư viện lọc biên dịch cho `x86_64-unknown-linux-gnu` và `wasm32-wasip2` là cách edge worker và CLI admin dùng chung mã. Sở hữu *cấu hình build* trở nên quan trọng không kém sở hữu `String`.

**Language server như hạ tầng sản xuất.** Zed, VS Code và Helix đều nói LSP với rust-analyzer. Khi rust-analyzer lỗi vì `build.rs` dò `web-sys`, cả team mất một buổi chiều. Đó là sự cố hệ thống không có mã HTTP.

## Thách thức và mở rộng

Resolver của Cargo, thống nhất feature, và phân giải theo MSRV (công việc resolver nhận biết MSRV trong 2024–2025) là lớp tiếp theo sau “biên dịch được.” Workspace có feature tùy chọn có thể kéo hai phiên bản cùng crate; `cargo tree -d` là chẩn đoán.

WASM vẫn hai nhân cách: `wasm32-unknown-unknown` + `wasm-bindgen` cho trình duyệt, versus component WASI 0.2 cho server. Chọn sai target là “libc sai” phiên bản mới.

Edition 2024 biến một số API `std::env` thành `unsafe` vì mutation toàn cục tiến trình chưa bao giờ thực sự an toàn. Di cư là bài toán *xã hội*: mọi crate trong đồ thị phải đồng ý.

Hãy nghĩ: nếu bạn thiết kế cache artifact đã biên dịch (`sccache`, `cargo-cache`) cho tổ chức 200 lập trình viên, bạn sẽ khóa theo đối tượng nào của Chương 1 — hash toolchain, target triple, digest `Cargo.lock`, hay cả ba?

## Bài tập

1. **Suy luận.** Giải thích trong bốn câu vì sao RFC 2789 tồn tại. Chỉ mục git đang lãng phí tài nguyên gì, và ai trả giá tài nguyên đó trên CI?
2. **Sửa mã.** Lấy crate `hello-cli` Chương 1, thêm `rust-toolchain.toml` ghim một bản stable cụ thể, và trường `rust-version`. Xác nhận `rustup show` khớp `cargo --version`.
3. **Metadata.** Chạy `cargo metadata --format-version 1` và chỉ ra id crate, edition, và thư mục target. Phác đồ thị crate trên giấy.
4. **Target thứ hai.** Thêm `wasm32-unknown-unknown`, build crate thư viện cho target đó, và ghi lại đường dẫn artifact dưới `target/wasm32-unknown-unknown/`. Artifact đó *thiếu* gì so với binary native?
5. **Cài đặt.** Tách `hello-cli` thành workspace (`core` + `cli`) theo 01-05, rồi thêm crate `wasm` rỗng phụ thuộc `core`. Không viết lại lý thuyết — chỉ đồ thị gói.

## Tài liệu tham khảo

- [Help test Cargo's new index protocol](https://blog.rust-lang.org/inside-rust/2023/01/30/cargo-sparse-protocol/) — Inside Rust, 30 tháng 1 năm 2023; RFC 2789.
- [Announcing Rust 1.68.0](https://blog.rust-lang.org/2023/03/09/Rust-1.68.0/) — ổn định giao thức sparse.
- [Announcing Rust 1.85.0 and Rust 2024](https://blog.rust-lang.org/2025/02/20/Rust-1.85.0/) — Edition 2024 (RFC 3501).
- [WASI 0.2 Launched](https://bytecodealliance.org/articles/WASI-0.2) — Bytecode Alliance, 25 tháng 1 năm 2024.
- [The Cargo Book — Registry Index](https://doc.rust-lang.org/cargo/reference/registry-index.html) — giao thức git và sparse.
- [rust-analyzer manual](https://rust-analyzer.github.io/) — LSP chính thức.

## Tóm tắt

- Vòng Chương 1 chính là vòng sản xuất: ghim toolchain, phân giải đồ thị, biên dịch target, test.
- crates.io sparse (2023) là Cargo đóng vai client hệ thống có mạng.
- rust-analyzer là compiler thứ hai với ngân sách độ trễ; hãy coi nó là hạ tầng.
- WASM/WASI 0.2 là target triple và ABI, không phải ngôn ngữ mới.
- Edition 2024 là di cư phối hợp, ghi trong `Cargo.toml` và CI, không ghi trên slide.
