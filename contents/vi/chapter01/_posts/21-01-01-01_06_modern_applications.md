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

Ở năm bài trước, bạn đã làm quen với Cargo, `rustc` và cây module ở quy mô bài tập cá nhân. Nhưng khi bước vào một kho mã nguồn sản xuất hàng trăm nghìn dòng với 200 thư viện phụ thuộc, chuỗi công cụ Rust bộc lộ những thiết kế kỹ thuật đáng nể: làm sao `cargo build` tải thông tin gói trong chưa đầy 1 giây thay vì clone hàng trăm megabyte lịch sử Git? Làm sao IDE báo lỗi kiểu trong 50 mili-giây khi bạn đang gõ? Và làm sao **cùng một hàm Rust** vừa biên dịch thành dịch vụ máy chủ Linux, vừa chạy ngay trong trình duyệt qua WebAssembly (WASM)?

Bài đọc mở rộng này kết nối toàn bộ nền tảng Chương 01 với bốn trụ cột sản xuất giai đoạn 2022–2026: **giao thức sparse index**, **kiến trúc `rust-analyzer`**, **đích biên dịch WebAssembly (`wasm32`)**, và **kỷ luật nâng cấp Rust Edition (2021 $$	o$$ 2024)**.

---

## Kiến thức cần nhớ: Bốn công nghệ sản xuất đứng sau hệ sinh thái chuỗi công cụ Rust

Mỗi công cụ bạn gõ hàng ngày ở Chương 01 đều giải quyết một bài toán kỹ thuật hệ thống thực thụ:

1. **Giao thức Sparse Index của Cargo:** Thay vì bắt máy khách clone toàn bộ kho Git `crates.io-index` khổng lồ, giao thức sparse chỉ gửi yêu cầu HTTP GET lấy đúng tệp siêu dữ liệu JSON của những crate thực sự có trong `Cargo.toml` — cắt giảm thời gian cập nhật chỉ mục trên CI/CD từ vài chục giây xuống dưới 1 giây.
2. **Hai trình biên dịch cho hai ngân sách độ trễ (`rustc` vs. `rust-analyzer`):** `rustc` tối ưu cho *tính đúng đắn tuyệt đối và tốc độ mã máy* (chạy theo lô vài giây–vài phút), còn `rust-analyzer` chạy nền theo kiến trúc truy vấn tăng dần (incremental query — Salsa) để phản hồi gõ phím trong dưới 100 ms, chấp nhận mã nguồn đang dở dang đầy lỗi cú pháp.
3. **Biên dịch chéo & Target Triple (WASM):** Với kiến trúc biên dịch tách biệt front-end (Rust) và back-end (LLVM), WebAssembly không phải ngôn ngữ mới — nó đơn giản là một **đích biên dịch** (`wasm32-unknown-unknown` hoặc `wasm32-wasip2`) chạy an toàn trong hộp cát (sandbox) trên trình duyệt lẫn edge server.
4. **Cơ chế Edition không phá vỡ tương thích ngược:** Rust 2015, 2018, 2021 và 2024 cùng tồn tại và liên kết mượt mà trong **cùng một tệp binary** nhờ mỗi crate tự khai báo trường `edition` trong `Cargo.toml` — giải quyết bài toán kinh điển "cú sốc chia đôi hệ sinh thái" (như Python 2 vs. Python 3).

---

## Kế hoạch giảng dạy 60 phút

- 0 đến 10 phút: Vì sao bộ công cụ là yếu tố quyết định với các nhóm kỹ thuật dùng Rust (thời gian CI, tệp khóa `Cargo.lock`, MSRV).
- 10 đến 25 phút: Giao thức sparse index của Cargo (RFC 2789) và đồ thị workspace trong dự án lớn.
- 25 đến 40 phút: `rust-analyzer` và `rustc` — hai trình biên dịch cùng phục vụ một ngôn ngữ.
- 40 đến 55 phút: WASM / WASI 0.2 và `wasm-bindgen` dưới góc nhìn một đích biên dịch (target) thực thụ.
- 55 đến 60 phút: Edition 2024 dưới góc nhìn một đợt chuyển đổi kỹ thuật trong môi trường sản xuất.

## Mục tiêu học tập

Sau bài học này, bạn có thể giải thích vì sao quy trình chuỗi công cụ của Chương 1 — tạo crate, cố định toolchain, thêm thư viện phụ thuộc, định dạng, phân tích tĩnh và kiểm thử — cũng chính là quy trình mà các tổ chức như Cloudflare, nhóm phát triển `rustc` và các runtime WASM đang vận hành, chỉ khác ở mức độ kỷ luật với tệp khóa và CI. Bạn cũng sẽ nắm được vai trò của giao thức sparse registry, vị trí LSP chính thức của `rust-analyzer`, và cách xem WASI 0.2 như một *target* được chọn bằng chính lệnh `rustup target add` đã học. Từ đó, bạn chuyển góc nhìn từ "công cụ cài để làm bài tập" sang "công cụ kiểm soát chất lượng cho các dịch vụ quy mô lớn".

## Điều kiện tiên quyết

Bạn đã hoàn thành các bài bắt buộc của Chương 1: cài đặt và chạy được `rustup`/`cargo`, tạo binary crate đầu tiên, nắm kiểu nguyên thủy, luồng điều khiển và cách tách `lib.rs` với `main.rs`. Bài này không yêu cầu kiến thức về `async`, `unsafe` hay lập trình WASM chuyên sâu; chỉ cần đọc hiểu được `Cargo.toml` và một tệp cấu hình CI dạng YAML là đủ.

## Giới thiệu

Giai đoạn 2018–2021, "dùng Rust trong sản xuất" thường gắn với việc viết lại một dịch vụ đơn lẻ hoặc một đoạn mã nhạy cảm về hiệu năng. Sang giai đoạn 2022–2026, bài toán trọng tâm đã dịch chuyển: các nhóm kỹ thuật không còn băn khoăn liệu `rustc` có sinh mã đủ nhanh hay không, mà quan tâm liệu một *workspace* lớn có thể clone, lập chỉ mục, chạy lint và phát hành ổn định trên máy cá nhân, trên CI lẫn lên đích `wasm32-wasip2` mà không cần tự chế công cụ hay không. Chỉ mục Git truyền thống của Cargo cho `crates.io` trở thành điểm nghẽn về quy mô; `rust-analyzer` trở thành trình biên dịch mặc định cho trải nghiệm tương tác; còn WebAssembly vượt qua giai đoạn thử nghiệm để định hình chuẩn ABI hướng thành phần (Component Model).

Trong hệ sinh thái C/C++, trình biên dịch, trình quản lý gói và máy chủ ngôn ngữ thường do các bên khác nhau phát triển rời rạc. Lựa chọn khác biệt của Rust là xây dựng cả ba như một hệ sinh thái thống nhất. Bài học này lần theo lựa chọn đó trong các hệ thống thực tế hiện nay.

## Khái niệm cốt lõi

### Sparse index là giao thức sản xuất, không chỉ là tùy chọn tiện lợi

Cho đến năm 2023, Cargo cập nhật thông tin từ `crates.io` bằng cách clone toàn bộ một kho Git rất lớn chứa chỉ mục gói. RFC 2789 (*Sparse HTTP protocol for Cargo*) thay thế cơ chế này bằng các yêu cầu HTTP GET *chỉ tải thông tin của những crate mà dự án thực sự phụ thuộc*. Rust 1.68 (tháng 3/2023) ổn định giao thức này, và Rust 1.70 (tháng 6/2023) đặt `sparse` làm mặc định cho `crates.io`. Bài thông báo trên Inside Rust trước đó đã kêu gọi cộng đồng thử nghiệm biến môi trường `CARGO_REGISTRIES_CRATES_IO_PROTOCOL=sparse` với `https://index.crates.io/`.

Thay đổi này gần như khó nhận thấy ở một dự án bài tập có mười crate, nhưng lại tạo khác biệt lớn trong một workspace gồm 400 crate. Điều này cũng cho thấy **Cargo là một phần mềm hệ thống có tương tác mạng**: HTTP/2, cơ chế bộ nhớ đệm và mã băm trong tệp khóa là một phần thiết yếu của hệ sinh thái Rust không kém gì từ khóa `let`.

```toml
# Ghim lịch sử — hữu ích khi dạy image CI cũ.
# Từ 1.70 đây là mặc định của crates.io.
[registries.crates-io]
protocol = "sparse"
```

Vì sao cộng đồng phải dành riêng một bản RFC cho chỉ mục gói? Bởi lệnh đầu tiên người mới chạy (`cargo new` và tải dependency) cũng là lưu lượng mà hệ thống CDN phải phục vụ hàng triệu lượt mỗi ngày. Công cụ không mở rộng được về quy mô không đơn thuần là "bất tiện cho người mới", mà là một điểm nghẽn hạ tầng.

### `rust-analyzer` là trình biên dịch thứ hai với ngân sách độ trễ khác

`rust-analyzer` là triển khai chính thức theo giao thức LSP (Language Server Protocol) của Rust, không phải công cụ tô màu cú pháp gắn ngoài lên `rustc`. Nó thực hiện kiểm tra kiểu trên mã nguồn chưa hoàn chỉnh, chạy build script và phân tích đồ thị workspace — chính là đồ thị module bạn học ở bài 01-05 — nhưng phải phản hồi ở mức độ trễ theo từng phím gõ. Trong môi trường sản xuất, `rust-analyzer` gặp sự cố cũng nghiêm trọng như lỗi trình biên dịch: nếu IDE không phân giải được `crate::auth::Token`, người đánh giá mã (reviewer) khó có thể tin tưởng vào thay đổi đó.

Điểm đáng chú ý nằm ở sự khác biệt kiến trúc: `rustc` tối ưu cho việc tạo ra *tệp biên dịch chính xác*, còn `rust-analyzer` tối ưu cho việc phân tích *chương trình đang viết dở*. Cả hai cùng đọc `Cargo.toml`, cùng tuân thủ `edition` và cùng phải hiểu các cờ biên dịch điều kiện `cfg`. Do đó, khi thấy "trình biên dịch chậm", cần xác định rõ là *trình biên dịch nào* và trên *đồ thị crate nào*.

### WASM là một đích biên dịch (target triple), không phải ngôn ngữ mới

Lệnh `rustup target add wasm32-unknown-unknown` và gần đây là WASI 0.2 (`wasm32-wasip2` cùng Component Model) sử dụng đúng cơ chế `cargo build --target` như khi biên dịch cho máy cục bộ. Ngày 25 tháng 1 năm 2024, Bytecode Alliance phát hành WASI 0.2 (Preview 2), bổ sung giao diện WIT cho đồng hồ, sinh số ngẫu nhiên, hệ thống tệp, socket, CLI và HTTP, đưa WASI từ mô hình ABI kiểu C sang Component Model. Wasmtime và `jco` là hai triển khai đầu tiên vượt qua bộ kiểm thử tính di động.

```bash
rustup target add wasm32-unknown-unknown
cargo build --target wasm32-unknown-unknown --release
```

`wasm-bindgen` và `wasm-pack` hoạt động *trên* đích biên dịch đó: chúng sinh mã cầu nối JavaScript, chứ không tạo ra một hệ thống kiểu riêng biệt. Nguyên lý ở Chương 1 — "crate là đơn vị biên dịch gắn với một target" — giải thích vì sao Rust trở thành ngôn ngữ phổ biến cho các runtime WASM như Wasmtime, WasmEdge hay wasmCloud. Bạn không học một ngôn ngữ mới, mà chỉ chọn một đích biên dịch khác.

### Edition là một đợt chuyển đổi có kỷ luật trong sản xuất

Edition 2024 ra mắt cùng Rust 1.85 vào ngày 20 tháng 2 năm 2025 (RFC 3501). Đây là phiên bản edition có phạm vi thay đổi lớn nhất từ trước đến nay: bổ sung `Future` và `IntoFuture` vào prelude, đánh dấu một số API môi trường là `unsafe`, và điều chỉnh thứ tự giải phóng (drop order). Trong thực tế sản xuất, các nhóm kỹ thuật không nâng cấp cảm tính mà cập nhật `edition = "2024"` trong `Cargo.toml`, chạy `cargo fix --edition` và cố định phiên bản Rust tối thiểu được hỗ trợ (MSRV) trên CI.

```toml
[package]
name = "edge-filter"
version = "0.4.2"
edition = "2024"
rust-version = "1.85"
```

Trường `rust-version` báo cho Cargo biết không nên biên dịch crate này trên các bản Rust cũ hơn (chẳng hạn 1.76). Đây chính là nguyên tắc cố định phiên bản bạn đã làm quen với `rustup show`, được mở rộng cho toàn bộ hệ thống máy chủ build.

## Minh họa từng bước qua mã nguồn

Ta bắt đầu từ một crate cơ bản như đã học, sau đó mở rộng dần theo cấu trúc kho mã nguồn dịch vụ hiện đại mà không cần thêm tính năng ngôn ngữ nâng cao.

### Mức cơ bản: một binary "chỉ cần biên dịch được"

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

Dự án này biên dịch thành công, nhưng chưa phản ánh cách các dự án lớn như Tokio, `rustc` hay Wasmtime vận hành. Sự cố thực tế đầu tiên thường không phải lỗi kiểu dữ liệu, mà là sự lệch phiên bản: "CI dùng toolchain khác máy cá nhân".

### Khắc phục lệch môi trường: cố định toolchain ngay trong kho mã nguồn

```toml
# rust-toolchain.toml  (commit cạnh Cargo.toml)
[toolchain]
channel = "1.85.0"
components = ["rustfmt", "clippy", "rust-analyzer"]
targets = ["wasm32-unknown-unknown"]
```

`rustup` tự động đọc tệp này khi bạn di chuyển vào thư mục dự án. Đây là cách mở rộng mô hình quản lý toolchain của Chương 1 cho cả nhóm phát triển: nếu `cargo --version` trên máy cá nhân khác với CI, tệp này đóng vai trò cam kết chung.

Sự cố phổ biến thứ hai thường gặp trên các image CI cũ (trước Rust 1.70) là bước `Updating crates.io index` kéo dài hàng phút do dùng chỉ mục Git. Giao thức sparse rút ngắn thao tác này xuống chỉ còn vài yêu cầu HTTP GET. Hiểu rõ cơ chế này giúp bạn chẩn đoán đúng nguyên nhân thay vì chỉ chờ đợi bị động.

### Cấu trúc chuẩn mực: workspace kết hợp thư viện dùng chung và hai đích biên dịch

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

Crate CLI phụ thuộc vào `filter-core` và giữ lớp vỏ mỏng — đúng thiết kế ở bài 01-05. Crate WASM *cũng* phụ thuộc vào `filter-core` và được biên dịch với cờ `--target wasm32-unknown-unknown`. Một đồ thị module duy nhất tạo ra hai sản phẩm biên dịch khác nhau. Đây là cấu trúc quen thuộc trong các dự án `wasm-bindgen`, Cloudflare Workers (`workers-rs`) và các ví dụ của Wasmtime.

Khi số lượng thư viện phụ thuộc tăng lên, bên cạnh Clippy, các công cụ như `cargo-deny` (do Embark Studios phát triển) và `cargo-audit` (của Rust Secure Code Working Group) thường được tích hợp vào CI để tự động chặn các crate đã bị thu hồi (yanked) hoặc có cảnh báo bảo mật từ RustSec. Chúng không phải tính năng ngôn ngữ mà là phần mở rộng tự nhiên của quy trình công cụ từ bài 01-01.

## Ví dụ thực tế

### Ví dụ 1 — `crates.io` dưới góc nhìn hệ phân tán

Chỉ mục sparse của `crates.io` (`https://index.crates.io/`) là một dịch vụ HTTP vận hành thực tế, còn Cargo đóng vai trò client. Khi bạn khai báo `serde = "1"`, Cargo gửi truy vấn phân giải phiên bản tới registry và đối chiếu với `Cargo.lock`. RFC 2789 ra đời vì mô hình clone Git truyền thống không còn đáp ứng được quy mô đồ thị crate vào năm 2023.

Quan sát một mục trong `Cargo.lock` như một cấu phần hệ thống:

```toml
[[package]]
name = "serde"
version = "1.0.217"
source = "registry+https://github.com/rust-lang/crates.io-index"
checksum = "..."
```

Trường `checksum` đảm bảo quá trình tải và cài đặt ở các vị trí địa lý khác nhau đều thu về cùng một nội dung byte. Bước "thêm dependency" ở Chương 1 thực chất là cơ chế tải dữ liệu có xác thực theo mã băm nội dung; vì vậy không nên chỉnh sửa thủ công tệp `Cargo.lock`.

### Ví dụ 2 — `rustc` và `rust-analyzer` trên cùng một workspace

Bản thân trình biên dịch Rust là một Cargo workspace gồm hàng trăm crate (`compiler/`, `library/`, `src/tools/`). Khi bạn nhảy đến một định nghĩa bên trong `rustc`, `rust-analyzer` phải nạp *một phần* đồ thị phụ thuộc đó. Điều tương tự cũng diễn ra với Tokio, Wasmtime và trình soạn thảo Zed. Nếu cấu trúc module học ở bài 01-05 bị tổ chức thiếu kỷ luật — chẳng hạn `pub use` tràn lan, phụ thuộc vòng hoặc dồn 4.000 dòng mã vào một tệp `lib.rs` — cả hai trình biên dịch đều phải gánh chi phí phân tích lớn sau mỗi lần chỉnh sửa.

Một bài tập trực quan là mở một crate quen thuộc trong `Cargo.lock` (`grep serde Cargo.lock`) rồi chạy `cargo metadata --no-deps --format-version 1 | head`. Dữ liệu JSON trả về chính là bản đồ module và crate mà các công cụ dùng chung. Đồ thị build trong sản xuất thực chất chỉ là `cargo metadata` ở quy mô lớn hơn.

### Ví dụ 3 — WASI 0.2 như một "hệ điều hành" thứ hai

WASI 0.2 định nghĩa tập hợp các giao diện (world) bằng WIT như `wasi:cli/command` hay `wasi:http/proxy`. Một crate Rust biên dịch cho WASI vẫn giữ nguyên cấu trúc crate; điểm thay đổi nằm ở *bề mặt lời gọi hệ thống (syscall)*: thay vì gọi qua `libc`, chương trình nhập các giao diện của Component Model. Wasmtime (Bytecode Alliance) triển khai các giao diện phía máy chủ (host) bằng Rust, trong khi mã khách (guest) có thể viết bằng Rust, JavaScript (`jco`) hoặc C.

```rust
// Tư duy phía guest: bạn vẫn sở hữu main, chỉ link std khác.
fn main() {
    // Trên wasip2 lệnh này in qua wasi:cli, không phải write(2) trên laptop.
    println!("filter-wasm guest started");
}
```

Câu hỏi suy ngẫm: khi module WASM không chia sẻ trực tiếp vùng nhớ tuyến tính (linear memory) như tệp thư viện động `.so` của C, ý tưởng nào từ Chương 1 giúp hạn chế một nhóm lớn lỗi FFI? (Gợi ý: ranh giới crate rõ ràng kết hợp với ABI có kiểm tra kiểu, thay vì truyền con trỏ thô `char*`).

## Ứng dụng trong lập trình hệ thống

- **CI như hệ thống biên dịch tập trung:** `rustc`, Clippy, `rustfmt` và `rust-analyzer` được đồng bộ qua cùng một tệp `rust-toolchain.toml` trên GitHub Actions, Buildkite và máy cá nhân. Nhờ đó, nhóm phát triển tránh được tình trạng mã biên dịch được trên phiên bản cũ (như 1.82) nhưng lỗi trên phiên bản mới (như 1.85).
- **Kiểm soát chuỗi cung ứng:** Lệnh `cargo deny check advisories bans licenses sources` được nhiều tổ chức sử dụng trong giai đoạn 2024–2026 để kiểm tra tự động trước khi hợp nhất mã (merge). Nó bổ sung bước thứ tư vào quy trình chất lượng của Chương 1: định dạng, phân tích tĩnh, kiểm thử và *kiểm tra chính sách*.
- **Sản phẩm đa đích biên dịch:** Biên dịch cùng một thư viện lọc cho cả `x86_64-unknown-linux-gnu` và `wasm32-wasip2` giúp edge worker và công cụ CLI quản trị dùng chung một nguồn mã. Việc làm chủ *cấu hình build* trở nên quan trọng không kém việc nắm vững các kiểu dữ liệu cơ bản.
- **Máy chủ ngôn ngữ là hạ tầng phát triển:** Các trình soạn thảo như Zed, VS Code và Helix đều giao tiếp LSP với `rust-analyzer`. Khi `rust-analyzer` gặp sự cố do `build.rs` cấu hình sai (ví dụ dò nhầm `web-sys`), năng suất của cả nhóm bị gián đoạn ngay cả khi dịch vụ không gặp lỗi runtime.

## Thách thức và hướng mở rộng

Bộ phân giải phiên bản (resolver) của Cargo, cơ chế hợp nhất feature (feature unification) và phân giải phụ thuộc có xét đến MSRV (phát triển mạnh trong giai đoạn 2024–2025) là những vấn đề thường gặp tiếp theo sau khi chương trình đã biên dịch được. Một workspace có các feature tùy chọn có thể vô tình kéo theo hai phiên bản khác nhau của cùng một crate; lệnh `cargo tree -d` giúp phát hiện nhanh tình trạng này.

Hệ sinh thái WASM hiện vẫn tồn tại hai hướng ứng dụng chính: `wasm32-unknown-unknown` kết hợp `wasm-bindgen` cho trình duyệt, và mô hình thành phần WASI 0.2 cho phía máy chủ. Chọn nhầm đích biên dịch cũng tương tự như liên kết nhầm `libc` trong lập trình hệ thống truyền thống.

Với Edition 2024, một số API trong `std::env` được chuyển sang `unsafe` vì việc thay đổi biến môi trường ở cấp toàn tiến trình vốn không an toàn trong môi trường đa luồng. Quá trình nâng cấp edition vì thế không chỉ là bài toán kỹ thuật mà còn là bài toán *phối hợp cộng đồng*, đòi hỏi các crate trong cùng đồ thị phụ thuộc tương thích với nhau.

Câu hỏi mở: nếu bạn thiết kế hệ thống lưu đệm tệp biên dịch (`sccache`, `cargo-cache`) cho một tổ chức 200 lập trình viên, bạn sẽ chọn khóa bộ nhớ đệm dựa trên yếu tố nào đã học ở Chương 1 — mã băm của toolchain, `target triple`, mã băm của `Cargo.lock`, hay kết hợp cả ba?

---

### Ví dụ truy vết từng dòng (Đồ thị phụ thuộc & Edition): Một dự án kết hợp 2 Edition và 2 đích biên dịch

Hãy truy vết cách Cargo và `rustc` phối hợp khi một dự án `analytics_core` dùng Rust Edition 2024 phụ thuộc vào một thư viện cũ viết ở Edition 2021, và được biên dịch cho cả máy chủ Linux lẫn WebAssembly:

```toml
# Cargo.toml
[package]
name = "analytics_core"
edition = "2024"            # Mã nguồn của bạn dùng cú pháp & quy tắc mới nhất

[dependencies]
legacy_math = "1.2"         # Crate này khai báo edition = "2021" -> VẪN LIÊN KẾT MƯỢT MÀ!
```

| Tầng xử lý | Khi chạy `cargo test` (Native Linux) | Khi chạy `cargo build --target wasm32-unknown-unknown --release` |
| :--- | :--- | :--- |
| **Tải siêu dữ liệu** | Sparse index tải đúng 1 tệp nhỏ cho `legacy_math` | Dùng lại cache cục bộ, không cần tải lại |
| **Biên dịch front-end** | `legacy_math` biên dịch theo luật 2021; `analytics_core` theo luật 2024 | Y hệt — cả hai cùng hạ xuống một tầng MIR/LLVM IR thống nhất! |
| **Sinh mã máy back-end** | Sinh lệnh x86_64/ARM64 gọi syscall hệ điều hành | Sinh bytecode `.wasm` chạy trong sandbox (không có `std::fs` trực tiếp) |

### Giải thích: Bốn bài học kỹ thuật hệ thống rút ra từ chuỗi công cụ Rust

1. **Thiết kế giao thức dữ liệu quyết định tốc độ hạ tầng:** Chuyển từ Git index sang Sparse HTTP index là ví dụ kinh điển cho thấy thay đổi cấu trúc truy cập ($$O(	ext{toàn bộ hệ sinh thái}) 	o O(	ext{số gói phụ thuộc})$$) quan trọng hơn mọi tối ưu vi mô.
2. **Đừng kỳ vọng IDE (`rust-analyzer`) thay thế hoàn toàn `cargo check`:** Do phải ưu tiên độ trễ thấp lúc gõ, đôi khi `rust-analyzer` bỏ qua một số kiểm tra borrow checker sâu — luôn chốt bằng `cargo check` / `cargo test` trên CI.
3. **Tách lõi logic thuần (`no_std` / không phụ thuộc OS) khỏi lớp I/O:** Nếu lõi thuật toán trong `src/lib.rs` chỉ dùng kiểu chuẩn không gọi tệp/mạng trực tiếp, bạn có thể biên dịch nó sang WASM chạy trên trình duyệt chỉ bằng một dòng lệnh!
4. **Nâng cấp Edition theo từng crate với `cargo fix --edition`:** Vì các Edition tương thích nhị phân với nhau, một tổ chức lớn có thể nâng cấp từng thư viện một theo lộ trình an toàn mà không bao giờ phải "đóng băng cả công ty" để viết lại mã.

**Kiểm tra nhanh.** Nếu dự án của bạn nâng cấp `edition = "2024"` trong `Cargo.toml`, liệu bạn có phải chờ toàn bộ 50 thư viện phụ thuộc trên crates.io nâng cấp lên Edition 2024 trước mới biên dịch được không — và vì sao?

---

## Bài tập

1. **Phân tích:** Giải thích trong khoảng bốn câu vì sao RFC 2789 ra đời. Mô hình chỉ mục Git cũ đã gây lãng phí tài nguyên gì, và chi phí đó ảnh hưởng trực tiếp đến khâu nào trên CI?
2. **Cấu hình dự án:** Lấy crate `hello-cli` từ Chương 1, bổ sung tệp `rust-toolchain.toml` cố định một phiên bản `stable` cụ thể và khai báo trường `rust-version` trong `Cargo.toml`. Xác nhận kết quả `rustup show` khớp với `cargo --version`.
3. **Đọc siêu dữ liệu:** Chạy `cargo metadata --format-version 1`, xác định mã định danh (id) của crate, `edition` và thư mục `target`, sau đó phác thảo đồ thị crate trên giấy.
4. **Đích biên dịch thứ hai:** Cài thêm target `wasm32-unknown-unknown`, biên dịch một library crate cho target đó và ghi nhận đường dẫn tệp kết quả dưới `target/wasm32-unknown-unknown/`. Tệp này *thiếu* những thành phần gì so với tệp nhị phân native?
5. **Tổ chức workspace:** Tách `hello-cli` thành một workspace gồm `core` và `cli` theo bài 01-05, rồi thêm một crate `wasm` rỗng phụ thuộc vào `core` để quan sát cấu trúc đồ thị gói.

## Tài liệu tham khảo

- [Help test Cargo's new index protocol](https://blog.rust-lang.org/inside-rust/2023/01/30/cargo-sparse-protocol/) — Inside Rust, 30 tháng 1 năm 2023; RFC 2789.
- [Announcing Rust 1.68.0](https://blog.rust-lang.org/2023/03/09/Rust-1.68.0/) — ổn định giao thức sparse.
- [Announcing Rust 1.85.0 and Rust 2024](https://blog.rust-lang.org/2025/02/20/Rust-1.85.0/) — Edition 2024 (RFC 3501).
- [WASI 0.2 Launched](https://bytecodealliance.org/articles/WASI-0.2) — Bytecode Alliance, 25 tháng 1 năm 2024.
- [The Cargo Book — Registry Index](https://doc.rust-lang.org/cargo/reference/registry-index.html) — giao thức git và sparse.
- [rust-analyzer manual](https://rust-analyzer.github.io/) — tài liệu LSP chính thức.

## Tóm tắt

- Quy trình học ở Chương 1 cũng là nền tảng của quy trình sản xuất: cố định toolchain, phân giải đồ thị phụ thuộc, biên dịch theo target và kiểm thử.
- Giao thức sparse trên `crates.io` (2023) cho thấy vai trò của Cargo như một phần mềm hệ thống tương tác mạng.
- `rust-analyzer` hoạt động như trình biên dịch thứ hai với yêu cầu độ trễ thấp và cần được xem là một phần hạ tầng phát triển.
- WASM và WASI 0.2 là các đích biên dịch (target triple) và chuẩn ABI, không phải ngôn ngữ lập trình mới.
- Edition 2024 là một đợt chuyển đổi có phối hợp, được quản lý tường minh qua `Cargo.toml` và hệ thống CI.

## Video tham khảo

- [The state of WASI support in Rust and WebAssembly toolchain - Michael Yuan](https://www.youtube.com/watch?v=hGcqJ2zrwIY) — RustLab Conference (tiếng Anh). Nên xem trước: trình bày WASI dưới góc độ một đích biên dịch của Cargo, bám sát phần toolchain trong bài.
- [Lin Clark — WebAssembly: Building a new kind of ecosystem](https://www.youtube.com/watch?v=IBZFJzGnBoU) — WebAssembly Summit (tiếng Anh). Phân tích vai trò của WebAssembly trong việc kết nối hệ sinh thái phần mềm vượt ra ngoài trình duyệt.
- [The Truth about Rust/WebAssembly Performance](https://www.youtube.com/watch?v=4KtotxNAwME) — Greg Johnston (tiếng Anh). Đánh giá khách quan những trường hợp Rust biên dịch sang WASM đạt hiệu năng cao và những giới hạn cần lưu ý để tránh phóng đại.
