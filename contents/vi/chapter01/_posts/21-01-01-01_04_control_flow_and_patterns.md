---
layout: post
title: "01-04 Luồng điều khiển và pattern cơ bản"
chapter: "01"
order: 4
owner: "OpenCode"
lang: vi
categories:
  - chapter01
lesson_type: required
---

Rust khuyến khích diễn đạt logic rẽ nhánh bằng `match` và pattern matching để các trạng thái bất hợp lệ không thể tồn tại trong chương trình. Bài học này xây dựng thói quen sử dụng `if`, `match` như các biểu thức trả về giá trị, kết hợp với `enum` để luồng xử lý trở nên tường minh.

## Kế hoạch giảng dạy 60 phút

- 0 đến 10 phút: `if` với vai trò biểu thức và rẽ nhánh cơ bản.
- 10 đến 30 phút: `match`: tính đầy đủ (exhaustiveness), các dạng pattern và điều kiện canh (guard).
- 30 đến 45 phút: Vòng lặp `for`, `while`, `loop` và trả về giá trị qua `break`.
- 45 đến 60 phút: Bài thực hành ngắn: xây dựng REPL có kiểm tra kiểu với `enum` lệnh và điều phối bằng `match`.

## Mục tiêu học tập

Sau bài học này, bạn có thể:

- Sử dụng `if` và `match` như các biểu thức trả về giá trị.
- Viết khối `match` đầy đủ mọi trường hợp và đọc hiểu thông báo lỗi của trình biên dịch khi thiếu nhánh.
- Vận dụng các dạng pattern phổ biến: hằng số (literal), khoảng giá trị, phân rã cấu trúc (destructuring) và mẫu đại diện `_`.
- Xây dựng bộ thông dịch lệnh đơn giản bằng cách phân tích đầu vào thành `enum`.

## `if` là một biểu thức

Trong Rust, `if` có thể trả về giá trị trực tiếp:

```rust
let n = 10;
let kind = if n % 2 == 0 { "even" } else { "odd" };
println!("{n} is {kind}");
```

Lưu ý: cả hai nhánh của `if`/`else` phải trả về cùng một kiểu dữ liệu.

## `match`: công cụ rẽ nhánh chủ lực

### `match` cơ bản

```rust
let x = 3;

let label = match x {
    0 => "zero",
    1 => "one",
    2 => "two",
    _ => "many",
};
```

Tính đầy đủ (exhaustiveness) là một cơ chế bảo vệ quan trọng: trình biên dịch buộc bạn phải xử lý mọi trường hợp có thể xảy ra.

### Các dạng pattern thường dùng

#### Khoảng giá trị

```rust
let grade = 87;
let letter = match grade {
    90..=100 => 'A',
    80..=89 => 'B',
    70..=79 => 'C',
    60..=69 => 'D',
    _ => 'F',
};
```

#### Điều kiện canh (guard)

```rust
let n = -3;
let desc = match n {
    x if x < 0 => "negative",
    0 => "zero",
    _ => "positive",
};
```

#### Phân rã tuple (destructuring)

```rust
let p = (10, 20);
let s = match p {
    (0, 0) => "origin",
    (0, y) => "on y-axis",
    (x, 0) => "on x-axis",
    (_, _) => "somewhere else",
};
```

### Đối khớp `Option` và `Result`

Đây là ngữ cảnh `match` phát huy rõ nhất giá trị an toàn kiểu:

```rust
let s = "42";
let n = match s.parse::<i32>() {
    Ok(n) => n,
    Err(e) => {
        eprintln!("parse failed: {e}");
        return;
    }
};
```

Ở các phần sau, ta sẽ dùng nhiều toán tử `?` để mã ngắn gọn hơn, nhưng `match` là cách trực quan nhất để nắm rõ luồng điều khiển.

## Vòng lặp

### `for` duyệt trên một khoảng

```rust
for i in 0..3 {
    println!("i = {i}");
}
```

### `while` lặp theo điều kiện

```rust
let mut n = 3;
while n > 0 {
    println!("{n}");
    n -= 1;
}
```

### `loop` lặp vô hạn và thoát tường minh bằng `break`

Trong Rust, `break` có thể kèm theo một giá trị trả về cho biểu thức `loop`:

```rust
let mut n = 0;
let found = loop {
    n += 1;
    if n == 5 {
        break n;
    }
};
assert_eq!(found, 5);
```

Lưu ý: `loop` rất phù hợp cho vòng lặp tương tác (REPL) hoặc cơ chế thử lại (retry).

### `if let` và `while let`

Sử dụng khi bạn chỉ quan tâm đến một pattern duy nhất và muốn bỏ qua các nhánh còn lại:

```rust
let maybe = Some(3);
if let Some(x) = maybe {
    println!("x = {x}");
}
```

```rust
let mut it = vec![1, 2, 3].into_iter();
while let Some(x) = it.next() {
    println!("{x}");
}
```

## Bài thực hành ngắn: xây dựng REPL có kiểm tra kiểu

Mục tiêu: đọc từng dòng đầu vào, phân tích thành kiểu `enum Command`, sau đó thực thi bằng `match`.

### Bước 1: định nghĩa mô hình lệnh

```rust
#[derive(Debug)]
enum Command {
    Add(i32, i32),
    Mul(i32, i32),
    Help,
    Quit,
}
```

### Bước 2: phân tích chuỗi đầu vào thành lệnh

Bắt đầu với cú pháp cố định: `add 1 2` hoặc `mul 3 4`.

```rust
fn parse_command(line: &str) -> Result<Command, String> {
    let parts: Vec<&str> = line.split_whitespace().collect();
    if parts.is_empty() {
        return Err("empty command".to_string());
    }

    match parts[0] {
        "add" if parts.len() == 3 => {
            let a = parts[1].parse::<i32>().map_err(|_| "bad a")?;
            let b = parts[2].parse::<i32>().map_err(|_| "bad b")?;
            Ok(Command::Add(a, b))
        }
        "mul" if parts.len() == 3 => {
            let a = parts[1].parse::<i32>().map_err(|_| "bad a")?;
            let b = parts[2].parse::<i32>().map_err(|_| "bad b")?;
            Ok(Command::Mul(a, b))
        }
        "help" => Ok(Command::Help),
        "quit" | "exit" => Ok(Command::Quit),
        _ => Err("unknown command or wrong arity".to_string()),
    }
}
```

Các điểm cần lưu ý:

- Điều kiện canh (`if parts.len() == 3`) giúp logic kiểm tra số lượng tham số rõ ràng.
- Trả về `Result` cho phép báo lỗi cú pháp mà không làm chương trình panic.

### Bước 3: thực thi lệnh bằng `match`

```rust
fn execute(cmd: Command) -> bool {
    match cmd {
        Command::Add(a, b) => {
            println!("{}", a + b);
            true
        }
        Command::Mul(a, b) => {
            println!("{}", a * b);
            true
        }
        Command::Help => {
            println!("commands: add <a> <b>, mul <a> <b>, help, quit");
            true
        }
        Command::Quit => false,
    }
}
```

Giá trị `bool` trả về giúp điều khiển vòng lặp REPL đơn giản: `true` để tiếp tục, `false` để thoát.

### Bước 4: ghép nối trong vòng lặp chính

Sử dụng `loop` và `break`:

```rust
use std::io::{self, Read};

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

        if !execute(cmd) {
            break;
        }
    }
}
```

Lưu ý: phần `Read` trong `use std::io::{self, Read};` không cần thiết cho `read_line`; khi viết mã thực tế, bạn chỉ cần `use std::io;`.

## Cạm bẫy thường gặp

1. Dùng nhánh đại diện `_` quá sớm: làm mất lợi ích kiểm tra tính đầy đủ của trình biên dịch khi thêm biến thể mới vào `enum`. Nên liệt kê tường minh các nhánh khi có thể.
2. Để chương trình panic khi đầu vào sai: trong vòng lặp REPL, lỗi phân tích cú pháp cần được xử lý để chương trình tiếp tục chạy.
3. Gộp chung phần phân tích cú pháp và phần thực thi: cần tách riêng hai bước này để dễ viết kiểm thử đơn vị.

## Bài tập

1. Bổ sung lệnh `sub` để thực hiện phép trừ.
2. Bổ sung lệnh `repeat <n> <word>` để in chuỗi lặp lại `n` lần.
3. Viết các bài kiểm thử đơn vị cho hàm `parse_command`.
4. Cải thiện thông báo lỗi để chỉ rõ phần dữ liệu đầu vào bị sai.

## Tóm tắt

- `if` và `match` đều là biểu thức trả về giá trị.
- Kết hợp `match` và `enum` giúp biểu diễn các trạng thái hợp lệ một cách tường minh.
- `loop` kết hợp `break` trả về giá trị rất phù hợp cho luồng xử lý tương tác.
- Tách biệt bước phân tích đầu vào khỏi bước thực thi giúp mã nguồn dễ kiểm thử và bảo trì.

## Video tham khảo

- [LẬP TRÌNH #RUST Part 4: Cách sử dụng #Enum, Option Enum & Match](https://www.youtube.com/watch?v=daOoyk9d43I) — Jayden Dang (tiếng Việt). Nên xem trước: giải thích `match`, `Option` và `enum` bằng tiếng Việt — trọng tâm của bài học.
- [Intro to Rust Control Flow Statements & Conditional Expressions 🦀](https://www.youtube.com/watch?v=3XpxkJXggHA) — Trevor Sullivan (tiếng Anh). Trình bày `if` như một biểu thức và các cấu trúc vòng lặp.
- [Mastering Rust's Match Control Flow - Full Crash Rust Tutorial for Beginners](https://www.youtube.com/watch?v=sd1M_CLXl4I) — Francesco Ciulla (tiếng Anh). Phân tích sâu hơn về `match` và pattern, hữu ích cho bài thực hành REPL.
