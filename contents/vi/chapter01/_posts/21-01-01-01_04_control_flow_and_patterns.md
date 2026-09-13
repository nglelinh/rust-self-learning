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

Rust khuyến khích bạn diễn đạt quyết định bằng `match` và pattern để các trạng thái bất khả thi không thể biểu diễn. Bài này xây thói quen dùng biểu thức (`if`, `match`) và enum để logic trở nên tường minh.

## Kế hoạch giảng 60 phút

- 0 đến 10 phút: `if` như một biểu thức và rẽ nhánh cơ bản.
- 10 đến 30 phút: `match`: tính cạn kiệt, pattern, và guard.
- 30 đến 45 phút: Vòng lặp: `for`, `while`, `loop`, và giá trị `break`.
- 45 đến 60 phút: Mini-lab: REPL có kiểu với enum lệnh và điều phối bằng `match`.

## Mục tiêu học

Cuối bài, bạn có thể:

- Dùng `if` và `match` như biểu thức (trả về giá trị).
- Viết `match` cạn kiệt và đọc lỗi compiler khi thiếu nhánh.
- Dùng các dạng pattern thường gặp: literal, khoảng, destructuring, `_` bắt tất cả.
- Xây bộ thông dịch lệnh đơn giản: parse input thành enum.

## `if` như một biểu thức

`if` trong Rust trả về giá trị:

```rust
let n = 10;
let kind = if n % 2 == 0 { "even" } else { "odd" };
println!("{n} is {kind}");
```

Điểm giảng: hai nhánh phải trả cùng kiểu.

## `match`: con ngựa kéo

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

Tính cạn kiệt là tính năng: Rust buộc bạn nghĩ về mọi trường hợp.

### Pattern bạn sẽ dùng liên tục

#### Khoảng

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

#### Guard

```rust
let n = -3;
let desc = match n {
    x if x < 0 => "negative",
    0 => "zero",
    _ => "positive",
};
```

#### Destructuring tuple

```rust
let p = (10, 20);
let s = match p {
    (0, 0) => "origin",
    (0, y) => "on y-axis",
    (x, 0) => "on x-axis",
    (_, _) => "somewhere else",
};
```

### Khớp `Option` và `Result`

Đây là chỗ `match` trở thành công cụ an toàn:

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

Sau này ta sẽ ưu tiên `?`, nhưng `match` là cách rõ nhất để dạy luồng điều khiển.

## Vòng lặp

### `for` trên một khoảng

```rust
for i in 0..3 {
    println!("i = {i}");
}
```

### `while` khi bạn có điều kiện

```rust
let mut n = 3;
while n > 0 {
    println!("{n}");
    n -= 1;
}
```

### `loop` khi bạn `break` tường minh

`break` trong Rust có thể trả về giá trị:

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

Điểm giảng: `loop` lý tưởng cho REPL và vòng thử lại.

### `if let` / `while let` (pattern thường gặp)

Dùng khi bạn chỉ quan tâm một pattern:

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

## Mini-lab: REPL có kiểu

Mục tiêu: đọc một dòng, parse thành enum `Command`, rồi thực thi bằng `match`.

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

### Bước 2: parse input thành lệnh

Bắt đầu với format chặt: `add 1 2` hoặc `mul 3 4`.

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

Điểm giảng:

- Guard (`if parts.len() == 3`) giữ logic parse rõ.
- `Result` cho phép báo lỗi parse mà không panic.

### Bước 3: thực thi bằng `match`

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

Trả `bool` làm vòng REPL đơn giản: tiếp tục hoặc thoát.

### Bước 4: nối với một vòng lặp

Dùng `loop` và `break`:

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

Ghi chú giảng viên: `use std::io::{self, Read};` không cần cho `read_line`; bạn có thể giản import trong live code.

## Cạm bẫy thường gặp

1. Dùng `_` quá sớm: mất lợi ích cạn kiệt. Ưu tiên các nhánh tường minh.
2. Panic khi input xấu: giữ lỗi parse có thể phục hồi trong REPL.
3. Trộn parse với thực thi: hãy tách để dễ test.

## Bài tập

1. Thêm lệnh `sub`.
2. Thêm `repeat <n> <word>`.
3. Thêm unit test cho `parse_command`.
4. Cải thiện lỗi để nêu đúng input sai.

## Tóm tắt

- `if` và `match` là biểu thức.
- `match` + enum làm trạng thái tường minh.
- `loop` + giá trị `break` rất hợp luồng tương tác.
- Tách parse khỏi thực thi để dễ bảo trì.
