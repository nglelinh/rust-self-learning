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

Trong C hay Java, một câu lệnh `switch` quên viết nhánh `default` (hoặc quên từ khóa `break`) có thể âm thầm trôi qua kiểm thử và gây ra sự cố nghiêm trọng khi một trạng thái mới được thêm vào hệ thống nhiều tháng sau đó. Rust biến luồng điều khiển thành một **công cụ kiểm chứng tính đầy đủ tại thời điểm biên dịch**: `if` là một *biểu thức* trả về giá trị, còn `match` bắt buộc phải **bao quát mọi khả năng (exhaustiveness)** — nếu bạn bỏ sót dù chỉ một trường hợp, chương trình sẽ từ chối biên dịch!

Bài học này giúp bạn tư duy theo phong cách biểu thức (expression-oriented) của Rust: viết `if` gọn gàng không cần biến tạm `mut`, làm chủ sức mạnh của `match` (khoảng giá trị, điều kiện canh `if` guard, phân rã tuple), xử lý an toàn `Option`/`Result`, và chọn đúng giữa `match`, `if let` và `while let`.

---

## Bản đồ kiến thức nền: Tư duy biểu thức và Đối khớp mẫu toàn diện trong Rust

Bốn nguyên lý sau phân biệt luồng điều khiển trong Rust với các ngôn ngữ mệnh lệnh truyền thống:

1. **Mọi thứ đều là Biểu thức (Expression-Oriented):** Trong Rust, khối `{ ... }`, `if/else`, `match`, thậm chí cả `loop` đều có thể trả về một giá trị. Nhờ đó bạn có thể khởi tạo trực tiếp biến bất biến `let status = if ok { "OK" } else { "ERR" };` mà không cần khai báo biến rỗng `mut` trước.
2. **Tính bao quát tuyệt đối (Exhaustiveness Checking):** Trình biên dịch phân tích toàn bộ không gian giá trị của biểu thức trong `match`. Nếu bạn thêm một biến thể mới vào `enum`, trình biên dịch sẽ lập tức gạch đỏ *mọi* câu lệnh `match` trong toàn dự án chưa xử lý biến thể đó!
3. **Phân rã cấu trúc (Destructuring) & Ràng buộc mẫu:** Pattern matching không chỉ so sánh bằng — nó vừa kiểm tra hình dạng dữ liệu (tuple, struct, enum, khoảng `1..=5`), vừa **bóc tách** các thành phần bên trong ra thành biến cục bộ trong đúng một bước.
4. **Vòng lặp an toàn & Đường gọn (`if let` / `while let`):** Khi bạn chỉ quan tâm đúng 1 nhánh của `Option`/`Result` và muốn bỏ qua phần còn lại, `if let` và `while let` giữ mã ngắn gọn mà vẫn đảm bảo an toàn kiểu.

---

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

---

### Ví dụ truy vết từng dòng (Đánh giá biểu thức & Bẫy dấu chấm phẩy `;`): Vì sao một dấu `;` đổi cả kiểu dữ liệu?

Hãy truy vết từng dòng đoạn mã sử dụng biểu thức `if`, `match` có guard và phân rã tuple:

```rust
fn classify(pair: (i32, Option<i32>)) -> &'static str {
    match pair {
        (x, Some(y)) if x + y == 10 => "tổng bằng 10", // Nhánh 1: khớp mẫu + guard đúng
        (0, Some(_))                => "x bằng 0",     // Nhánh 2: không chạy nếu Nhánh 1 đã khớp
        (x, None) if x < 0          => "âm và thiếu y",// Nhánh 3: phân rã x và kiểm tra None
        _                           => "trường hợp khác", // Nhánh 4: bao quát toàn bộ phần còn lại
    } // KHÔNG có dấu `;` -> toàn bộ `match` trả về `&'static str`!
}
```

Truy vết 3 đầu vào cụ thể đi qua các nhánh theo đúng thứ tự từ trên xuống dưới:

| Đầu vào `pair` | Nhánh 1 `(x, Some(y)) if x+y==10` | Nhánh 2 `(0, Some(_))` | Nhánh 3 `(x, None) if x<0` | Kết quả trả về |
| :--- | :--- | :--- | :--- | :--- |
| `(0, Some(10))` | Khớp `x=0, y=10`, guard `0+10==10` **Đúng** | (không xét nữa) | (không xét nữa) | `"tổng bằng 10"` |
| `(0, Some(3))` | Khớp mẫu nhưng guard `0+3==10` **Sai** | Khớp `0` và `Some(3)` **Đúng** | (không xét nữa) | `"x bằng 0"` |
| `(-4, None)` | Sai (vì `None` $$
eq$$ `Some`) | Sai | Khớp `x=-4 < 0` **Đúng** | `"âm và thiếu y"` |

*Lưu ý sống còn về dấu chấm phẩy:* Nếu ở dòng cuối của hàm bạn vô tình thêm `;` sau dấu `}` của `match`, biểu thức `match` biến thành **câu lệnh (statement)** có kiểu unit `()` — lập tức gây lỗi biên dịch vì không khớp kiểu trả về `&'static str`!

### Khung giải thích: Bốn thói quen viết luồng điều khiển sạch chuẩn Rust (nên đọc chậm)

1. **Dùng biểu thức để giảm biến `mut`:** Thay vì khai báo `let mut label = ""; if ... { label = ... }`, hãy gán thẳng `let label = match ... { ... };`. Mã ngắn hơn và đảm bảo biến không bao giờ bị quên khởi tạo.
2. **Cẩn thận khi dùng nhánh bắt tất cả `_ =>`:** Với `enum` do chính bạn định nghĩa, hạn chế dùng `_ =>` nếu có thể — liệt kê tường minh từng biến thể giúp trình biên dịch tự động nhắc bạn mỗi khi hệ thống mở rộng thêm trạng thái mới.
3. **Kiểm soát thứ tự nhánh có guard:** Khi một nhánh có `if` guard bị sai, Rust sẽ tự động trượt xuống kiểm tra nhánh tiếp theo — đặt các nhánh cụ thể hơn ở phía trên nhánh tổng quát.
4. **Dùng `loop` trả về giá trị cho vòng lặp thử lại (retry):** `let val = loop { if let Ok(v) = try_op() { break v; } };` là mẫu thiết kế cực kỳ thanh lịch chỉ có ở ngôn ngữ hướng biểu thức.

**Kiểm tra nhanh.** Trong hàm `classify` ở trên, nếu bạn đưa vào cặp `(5, Some(5))` và trước đó vô tình đảo vị trí nhánh `_ => "trường hợp khác"` lên đầu tiên của `match`, chuyện gì sẽ xảy ra lúc biên dịch và lúc chạy?

---

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
