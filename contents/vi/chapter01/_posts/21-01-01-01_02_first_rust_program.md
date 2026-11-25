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

Ở hầu hết các ngôn ngữ như C, Python hay JavaScript, mọi biến sinh ra đều mặc định *thay đổi được* (mutable) — và hàng nghìn lỗi phần mềm khó lần vết nhất xuất phát từ việc một biến bị âm thầm ghi đè ở đâu đó cách xa hàng trăm dòng mã. Rust lật ngược hoàn toàn mặc định đó: **mọi gắn biến (`let`) đều bất biến (immutable)** trừ khi bạn viết tường minh từ khóa `mut` như một bản cam kết với người đọc mã rằng *"giá trị này sẽ biến đổi"*.

Bài học này dẫn bạn qua chương trình Rust đầu tiên, phân biệt rạch ròi ba khái niệm nền tảng thường bị nhầm lẫn — **bất biến (`let`)**, **khả biến tại chỗ (`let mut`)** và **che bóng (`shadowing`)** — đồng thời làm chủ hệ thống định dạng đầu ra qua `Display` (`{}`) và `Debug` (`{:?}`).

---

## Kiến thức cần nhớ: Bất biến mặc định, Shadowing và Hệ thống định dạng trong Rust

Dưới lớp cú pháp đơn giản của bài học đầu tiên là ba nguyên lý thiết kế ngôn ngữ rất sâu sắc:

1. **Bất biến mặc định (Immutability by Default):** Biến bất biến không chỉ giúp con người đọc mã dễ đoán hơn, mà còn giúp trình biên dịch tối ưu hóa thanh ghi (register) và chuẩn bị nền tảng cho mô hình chia sẻ an toàn đa luồng sau này.
2. **Đột biến tại chỗ (`mut`) vs. Tạo binding mới (`shadowing`):** Khi dùng `let mut x = 1; x = 2;`, ta ghi đè vào *cùng một ô nhớ* trên Stack và kiểu dữ liệu bắt buộc phải giữ nguyên. Ngược lại, **shadowing** (`let x = "42"; let x: i32 = x.parse()?;`) tạo ra một **biến hoàn toàn mới** che khuất tên cũ, cho phép đổi sang kiểu dữ liệu mới mà vẫn giữ nguyên tính bất biến!
3. **Trait định dạng (`Display` vs. `Debug`) và Macro (`println!`):** Dấu chấm than `!` ở `println!` báo hiệu đây là một **macro** được mở rộng và kiểm tra khớp kiểu chuỗi định dạng ngay tại thời điểm biên dịch (compile-time), không bị lỗi tràn bộ đệm hay sai định dạng lúc chạy như `printf` trong C.

---

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

---

### Ví dụ truy vết từng dòng (Stack / Kiểu dữ liệu): So sánh `mut` và `shadowing` trên Stack

Hãy truy vết từng dòng đoạn mã nhỏ sau để thấy sự khác biệt vật lý trên Stack giữa ghi đè (`mut`) và che bóng (`shadowing`):

```rust
fn main() {
    let mut count: i32 = 10;   // Ô nhớ Stack #1 (4 byte): count = 10
    count += 5;                // Ghi đè TRỰC TIẾP vào Ô #1: count = 15 (kiểu vẫn là i32)

    let raw = "  42  ";        // Ô nhớ Stack #2 (&str, 16 byte): ptr + len=6
    let raw = raw.trim();      // Ô nhớ Stack #3 (&str, 16 byte): ptr' + len=2 (SHADOWING!)
    let raw: i32 = raw.parse().unwrap(); // Ô nhớ Stack #4 (i32, 4 byte): 42 (ĐỔI KIỂU!)
    println!("count={count}, raw={raw}");
}
```

| Bước | Tên biến đang nhìn thấy | Kiểu dữ liệu | Hành động trên bộ nhớ Stack |
| :--- | :--- | :--- | :--- |
| Dòng 2–3 | `count` (`mut`) | `i32` (cố định) | Ghi đè tại chỗ vào cùng ô nhớ 4 byte (`10` $$	o$$ `15`) |
| Dòng 5 | `raw` (lần 1) | `&str` | Cấp phát binding mới trỏ chuỗi `"  42  "` |
| Dòng 6 | `raw` (lần 2) | `&str` | Tạo binding mới bất biến trỏ lát cắt `"42"` |
| Dòng 7 | `raw` (lần 3) | `i32` | Tạo binding mới 4 byte chứa số `42`; tên cũ bị che khuất |

### Giải thích: Bốn quy tắc vàng khi làm việc với biến và in ấn trong Rust

1. **Mặc định dùng `let`, chỉ thêm `mut` khi thực sự cần biến đổi:** Nếu bạn lỡ viết `let mut x = 5;` mà không bao giờ sửa `x`, trình biên dịch `rustc` sẽ nhắc nhẹ bằng cảnh báo *unused `mut`* để giữ mã luôn sạch.
2. **Dùng shadowing cho chuỗi biến đổi kiểu dữ liệu:** Thay vì đặt tên rườm rà kiểu `input_str`, `input_trimmed`, `input_num`, shadowing cho phép giữ một tên duy nhất và khóa tính bất biến sau khi chuyển đổi xong.
3. **Phân biệt `{}` (`Display`) và `{:?}` (`Debug`):** `{}` dành cho người dùng cuối (chuỗi, số), còn `{:?}` (và `{:#?}` in đẹp nhiều dòng) dành cho lập trình viên soi cấu trúc bên trong của `Vec`, `tuple` hay `struct` (với `#[derive(Debug)]`).
4. **Nhớ macro `dbg!` khi gỡ lỗi nhanh:** `dbg!(&x)` vừa in tên tệp, số dòng, biểu thức lẫn giá trị ra `stderr`, vừa trả về quyền sở hữu biểu thức để bạn chèn trực tiếp vào giữa công thức tính toán.

**Kiểm tra nhanh.** Nếu bạn viết `let mut s = "42"; s = s.parse::<i32>().unwrap();` (không có từ khóa `let` ở lệnh thứ hai), trình biên dịch sẽ chấp nhận hay báo lỗi — và vì sao?

---

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
