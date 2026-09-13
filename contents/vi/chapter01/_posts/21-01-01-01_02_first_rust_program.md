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

Bài này tập trung vào cú pháp Rust bạn sẽ chạm liên tục: binding, tính khả biến, IO cơ bản, và luồng điều khiển đơn giản. Mục tiêu không phải thuộc lòng cú pháp, mà xây thói quen đáng tin: viết chương trình nhỏ, làm cho đúng, và thể hiện lỗi một cách tường minh.

## Kế hoạch giảng 60 phút

- 0 đến 10 phút: Hình dạng một chương trình Rust và `fn main()`.
- 10 đến 25 phút: Biến: `let`, `mut`, và shadowing.
- 25 đến 40 phút: IO: đọc một dòng, `trim`, parse, xử lý thất bại.
- 40 đến 50 phút: Định dạng output: `println!` và debug formatting.
- 50 đến 60 phút: Bài tập: prompt nhỏ vững chắc, không panic khi input xấu.

## Mục tiêu học

Cuối bài, bạn có thể:

- Giải thích khác biệt giữa bất biến và khả biến trong Rust.
- Dùng shadowing để biến đổi đổi kiểu (ví dụ `String` thành số).
- Đọc stdin, parse an toàn, và báo lỗi mà không panic.
- Viết hàm phụ nhỏ với chữ ký rõ.

## Chương trình Rust tối thiểu

Bắt đầu từ đây và chú thích từng phần:

```rust
fn main() {
    println!("Hello, world!");
}
```

Điểm giảng:

- `fn` định nghĩa hàm.
- `main` là điểm vào của binary.
- `println!` là macro (sẽ giải thích macro sau; lúc này hãy coi nó là "cú pháp giống hàm nhưng đặc biệt").

## Binding, tính khả biến, và shadowing

### `let` tạo một binding

```rust
let x = 10;
```

`x` bất biến theo mặc định. Đây là tính năng, không phải hạn chế: nó ngăn thay đổi ngoài ý muốn.

### `mut` chọn tham gia mutation

```rust
let mut count = 0;
count += 1;
```

Nhấn mạnh: trong Rust, tính khả biến thuộc về **binding**, không thuộc về giá trị.

### Shadowing (cùng tên, binding mới)

Shadowing thường là cách sạch nhất để biến đổi dữ liệu từng bước.

```rust
let input = "  42 ";
let input = input.trim();
let input: i32 = input.parse().unwrap();
```

Điểm giảng:

- Mỗi `let input = ...` tạo binding mới.
- Kiểu có thể đổi qua các lần shadow.
- Ta dùng `unwrap()` chỉ để minh họa; code thật nên ưu tiên `Result`.

## In và định dạng

### Formatting cơ bản

```rust
let name = "Rust";
println!("Hello, {name}!");
println!("{} + {} = {}", 2, 3, 2 + 3);
```

### Debug formatting

```rust
let v = vec![1, 2, 3];
println!("v = {:?}", v);
```

Nếu học viên hỏi "sao không luôn debug print?": debug dành cho người phát triển; display formatting dành cho thông điệp người dùng.

## Đọc stdin (live coding)

Đây là mẫu vững chắc đơn giản nhất:

```rust
use std::io;

fn read_line() -> io::Result<String> {
    let mut s = String::new();
    io::stdin().read_line(&mut s)?;
    Ok(s)
}
```

Điểm giảng:

- `String::new()` cấp phát chuỗi rỗng, có thể lớn dần.
- `read_line` **nối thêm** vào chuỗi.
- `?` trả về sớm nếu có lỗi IO.

## Parse an toàn

### Bản ngây thơ (chỉ ra, rồi cải thiện)

```rust
let n: i32 = input.trim().parse().unwrap();
```

Bản này panic khi input không hợp lệ. Với CLI, điều đó không chấp nhận được.

### Bản tốt hơn với `Result`

```rust
fn parse_i32(s: &str) -> Result<i32, String> {
    s.trim()
        .parse::<i32>()
        .map_err(|e| format!("not a valid integer: {e}"))
}
```

Điểm giảng:

- `parse::<i32>()` trả về `Result<i32, ParseIntError>`.
- `map_err` biến đổi kiểu lỗi.
- Trả về `String` không lý tưởng lâu dài, nhưng đủ cho Bài 1; sau này ta sẽ định nghĩa kiểu lỗi thật.

## Trả `Result` từ `main`

Nhờ đó bạn dùng `?` trong `main`.

```rust
fn main() -> Result<(), String> {
    let line = read_line().map_err(|e| e.to_string())?;
    let n = parse_i32(&line)?;
    println!("n = {n}");
    Ok(())
}
```

Ghi chú giảng viên: nếu bài trước đã dùng `anyhow`, bản này còn gọn hơn. Cả hai cách đều ổn; giữ nhất quán xuyên khóa học.

## Ví dụ làm việc: prompt nhỏ không bao giờ panic

Mục tiêu: hỏi người dùng một số nguyên, và hỏi lại cho đến khi họ nhập hợp lệ.

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

- Chương trình vẫn có thể thất bại ở đâu? (lỗi IO)
- Vì sao lỗi parse không được trả về như một error? (vì ta chọn phục hồi bằng cách hỏi lại)

## Sai lầm thường gặp (và điều compiler dạy)

### 1. Quên `mut`

```rust
let s = String::new();
io::stdin().read_line(&mut s)?;
```

Lỗi vì `read_line` cần mutable reference. Sửa: `let mut s = ...`.

### 2. `trim()` quá muộn

Học viên thường parse dòng thô còn `\n`. Dạy: `trim()` trước khi parse.

### 3. Lạm dụng `unwrap()`

Quy tắc khóa học này: trong chương trình CLI, `unwrap()` là phương án cuối. Ưu tiên `match` hoặc `?`.

## Bài tập

1. Sửa `prompt_i32` để nhận một khoảng đóng (ví dụ 1 đến 10). Hỏi lại khi ngoài khoảng.
2. Viết `prompt_f64` nhận số thực và từ chối `NaN`.
3. Thêm `prompt_yes_no` nhận `y/n` và trả về `bool`.

## Tóm tắt

- Bất biến theo mặc định; chọn mutation bằng `mut`.
- Shadowing là cách sạch để biến đổi giá trị.
- Ưu tiên xử lý lỗi tường minh, không panic.
- Dùng `Result` và `?` sớm; cách này mở rộng được sang chương trình lớn hơn.
