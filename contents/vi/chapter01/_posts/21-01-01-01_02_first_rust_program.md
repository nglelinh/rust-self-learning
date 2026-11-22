---
layout: post
title: "01-02 Chương trình Rust đầu tiên"
chapter: "01"
order: 2
owner: "OpenCode"
lang: vi
categories:
  - chapter01
lesson_type: required
---

Bài học này tập trung vào những cú pháp Rust xuất hiện thường xuyên nhất: gắn biến (binding), tính khả biến, nhập/xuất cơ bản và luồng điều khiển đơn giản. Mục tiêu không phải là học thuộc lòng cú pháp, mà là hình thành thói quen lập trình tin cậy: viết chương trình nhỏ gọn, đảm bảo tính đúng đắn và xử lý lỗi một cách tường minh.

## Kế hoạch giảng dạy 60 phút

- 0 đến 10 phút: Cấu trúc cơ bản của một chương trình Rust và hàm `fn main()`.
- 10 đến 25 phút: Biến trong Rust: `let`, `mut` và kỹ thuật che khuất biến (shadowing).
- 25 đến 40 phút: Nhập/xuất (IO): đọc một dòng từ đầu vào chuẩn, dùng `trim`, phân tích kiểu (parse) và xử lý lỗi.
- 40 đến 50 phút: Định dạng đầu ra với `println!` và định dạng gỡ lỗi (debug formatting).
- 50 đến 60 phút: Bài tập thực hành: xây dựng lời nhắc nhập liệu an toàn, không bị panic khi dữ liệu đầu vào không hợp lệ.

## Mục tiêu học tập

Sau bài học này, bạn có thể:

- Giải thích sự khác biệt giữa bất biến (immutable) và khả biến (mutable) trong Rust.
- Sử dụng shadowing để biến đổi giá trị kèm thay đổi kiểu dữ liệu (ví dụ từ `String` sang kiểu số).
- Đọc dữ liệu từ `stdin`, chuyển đổi kiểu an toàn và báo lỗi mà không làm chương trình bị panic.
- Viết các hàm phụ nhỏ gọn với chữ ký hàm rõ ràng.

## Chương trình Rust tối thiểu

Hãy bắt đầu từ chương trình tối giản sau và phân tích từng thành phần:

```rust
fn main() {
    println!("Hello, world!");
}
```

Các điểm cần lưu ý:

- Từ khóa `fn` dùng để định nghĩa hàm.
- `main` là điểm khởi đầu (entry point) của một binary crate.
- `println!` là một macro (khái niệm macro sẽ được trình bày chi tiết ở phần sau; hiện tại có thể hiểu đây là cú pháp gần giống lời gọi hàm nhưng có cơ chế mở rộng riêng).

## Gắn biến (binding), tính khả biến và shadowing

### Tạo binding bằng `let`

```rust
let x = 10;
```

Theo mặc định, `x` là bất biến. Đây là chủ đích thiết kế của Rust nhằm ngăn chặn các thay đổi dữ liệu ngoài ý muốn, không phải là hạn chế của ngôn ngữ.

### Cho phép thay đổi giá trị bằng `mut`

```rust
let mut count = 0;
count += 1;
```

Điểm quan trọng: trong Rust, tính khả biến thuộc về **binding** (tên gắn với giá trị), không phải thuộc tính cố hữu của bản thân giá trị.

### Shadowing (cùng tên, tạo binding mới)

Shadowing thường là cách gọn gàng để biến đổi dữ liệu qua nhiều bước liên tiếp mà không cần đặt thêm nhiều tên biến trung gian.

```rust
let input = "  42 ";
let input = input.trim();
let input: i32 = input.parse().unwrap();
```

Các điểm cần lưu ý:

- Mỗi câu lệnh `let input = ...` tạo ra một binding hoàn toàn mới.
- Kiểu dữ liệu có thể thay đổi qua từng lần shadow.
- Lời gọi `unwrap()` ở trên chỉ dùng để minh họa ngắn gọn; trong mã nguồn thực tế, hãy ưu tiên xử lý lỗi bằng `Result`.

## In và định dạng đầu ra

### Định dạng cơ bản

```rust
let name = "Rust";
println!("Hello, {name}!");
println!("{} + {} = {}", 2, 3, 2 + 3);
```

### Định dạng gỡ lỗi (debug formatting)

```rust
let v = vec![1, 2, 3];
println!("v = {:?}", v);
```

Lưu ý về mục đích sử dụng: định dạng debug (`{:?}`) dành cho lập trình viên khi kiểm tra cấu trúc dữ liệu, còn định dạng hiển thị (`{}`) dành cho thông báo hướng tới người dùng cuối.

## Đọc dữ liệu từ `stdin`

Dưới đây là mẫu đọc một dòng từ đầu vào chuẩn đơn giản và tin cậy:

```rust
use std::io;

fn read_line() -> io::Result<String> {
    let mut s = String::new();
    io::stdin().read_line(&mut s)?;
    Ok(s)
}
```

Các điểm cần lưu ý:

- `String::new()` tạo một chuỗi rỗng trên heap, có thể tự động mở rộng kích thước.
- Phương thức `read_line` **nối thêm** nội dung đọc được vào cuối chuỗi `s`.
- Toán tử `?` tự động trả về sớm nếu phát sinh lỗi IO.

## Phân tích chuỗi thành số một cách an toàn

### Cách làm đơn giản (chưa an toàn)

```rust
let n: i32 = input.trim().parse().unwrap();
```

Cách viết này sẽ làm chương trình panic ngay khi đầu vào không phải số nguyên hợp lệ — điều không thể chấp nhận đối với một ứng dụng CLI.

### Cách làm an toàn hơn với `Result`

```rust
fn parse_i32(s: &str) -> Result<i32, String> {
    s.trim()
        .parse::<i32>()
        .map_err(|e| format!("not a valid integer: {e}"))
}
```

Các điểm cần lưu ý:

- `parse::<i32>()` trả về `Result<i32, ParseIntError>`.
- `map_err` chuyển đổi kiểu lỗi sang định dạng mong muốn.
- Trả về kiểu lỗi `String` phù hợp ở giai đoạn nhập môn này; ở các chương sau, chúng ta sẽ định nghĩa các kiểu lỗi có cấu trúc rõ ràng hơn.

## Trả về `Result` từ hàm `main`

Khi `main` trả về `Result`, bạn có thể sử dụng trực tiếp toán tử `?` bên trong `main`:

```rust
fn main() -> Result<(), String> {
    let line = read_line().map_err(|e| e.to_string())?;
    let n = parse_i32(&line)?;
    println!("n = {n}");
    Ok(())
}
```

Lưu ý: nếu đã cấu hình `anyhow` từ bài trước, chữ ký trả về của `main` sẽ còn ngắn gọn hơn nữa. Cả hai cách đều hợp lệ; điều quan trọng là giữ phong cách nhất quán trong suốt dự án.

## Ví dụ hoàn chỉnh: lời nhắc nhập liệu không bị panic

Mục tiêu: yêu cầu người dùng nhập một số nguyên và lặp lại lời nhắc cho đến khi nhận được giá trị hợp lệ.

```rust
use std::io;

fn read_line() -> io::Result<String> {
    let mut s = String::new();
    io::stdin().read_line(&mut s)?;
    Ok(s)
}

fn prompt_i32(prompt: &str) -> io::Result<i32> {
    loop {
        println!("{prompt}");
        let line = read_line()?;
        match line.trim().parse::<i32>() {
            Ok(n) => return Ok(n),
            Err(_) => {
                println!("Please enter a valid 32-bit integer.");
            }
        }
    }
}

fn main() -> io::Result<()> {
    let n = prompt_i32("Enter an integer:");
    let n = n?;
    println!("You entered: {n}");
    Ok(())
}
```

Câu hỏi thảo luận:

- Chương trình trên vẫn có thể gặp lỗi ở bước nào? (Khi xảy ra lỗi IO trong quá trình đọc `stdin`).
- Vì sao lỗi phân tích số (`parse`) không được trả về ngay như một lỗi của hàm? (Vì chương trình chủ động phục hồi bằng cách yêu cầu người dùng nhập lại).

## Những sai lầm thường gặp

### 1. Quên từ khóa `mut`

```rust
let s = String::new();
io::stdin().read_line(&mut s)?;
```

Trình biên dịch báo lỗi vì `read_line` yêu cầu một tham chiếu khả biến (`&mut String`). Cách sửa: khai báo `let mut s = String::new();`.

### 2. Quên gọi `trim()` trước khi `parse()`

Chuỗi đọc từ `read_line` luôn chứa ký tự xuống dòng (`\n` hoặc `\r\n`) ở cuối. Nếu không gọi `trim()` trước khi gọi `parse()`, phép chuyển đổi sẽ luôn thất bại.

### 3. Lạm dụng `unwrap()`

Nguyên tắc trong khóa học này: đối với ứng dụng dòng lệnh (CLI), chỉ dùng `unwrap()` khi thực sự bất khả kháng; hãy ưu tiên xử lý lỗi tường minh bằng `match` hoặc toán tử `?`.

## Bài tập

1. Chỉnh sửa `prompt_i32` để nhận thêm một khoảng đóng `[min, max]` (ví dụ từ 1 đến 10) và yêu cầu nhập lại nếu giá trị nằm ngoài khoảng.
2. Viết hàm `prompt_f64` nhận số thực dấu phẩy động và từ chối giá trị `NaN`.
3. Viết thêm hàm `prompt_yes_no` nhận `y`/`n` và trả về `bool`.

## Tóm tắt

- Biến trong Rust bất biến theo mặc định; chỉ bật khả biến bằng `mut` khi cần thiết.
- Shadowing giúp biến đổi giá trị qua nhiều bước một cách rõ ràng.
- Luôn ưu tiên xử lý lỗi tường minh thay vì để chương trình panic.
- Tập sử dụng `Result` và toán tử `?` ngay từ đầu để dễ dàng mở rộng sang các chương trình lớn hơn.

## Video tham khảo

- [\[Rust Cơ Bản\] Bài 2: Kiểu dữ liệu cơ bản và biến trong Rust  – Khóa Học Lập Trình Rust Cho Người Mới](https://www.youtube.com/watch?v=CojJ6Sc6LYI) — Vie Lang (tiếng Việt). Nên xem trước: trình bày về biến, kiểu cơ bản và binding bằng tiếng Việt, khớp với nửa đầu bài học.
- [Common Programming Concepts in Rust](https://www.youtube.com/watch?v=2V0JaMVjzws) — Let's Get Rusty (tiếng Anh). Giới thiệu `let`, `mut` và các chương trình nhỏ trước khi làm việc với `stdin`.
- [Rust 101 - Variable shadowing VS mutable variables](https://www.youtube.com/watch?v=WFSapubNpi4) — Rust Ing (tiếng Anh). Phân biệt rõ shadowing và biến khả biến (`mut`).
