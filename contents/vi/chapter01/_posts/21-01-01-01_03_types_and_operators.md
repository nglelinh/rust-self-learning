---
layout: post
title: "01-03 Kiểu dữ liệu và toán tử"
chapter: "01"
order: 3
owner: "OpenCode"
lang: vi
categories:
  - chapter01
lesson_type: required
---

Rust kiểm soát hệ thống kiểu dữ liệu rất chặt chẽ, đổi lại chương trình tránh được các phép ép kiểu ngầm và hạn chế lỗi bất ngờ khi chạy. Bài học này trình bày hệ thống kiểu cơ bản của Rust và các cách chuyển đổi kiểu an toàn.

## Kế hoạch giảng dạy 60 phút

- 0 đến 10 phút: Cơ chế suy luận kiểu và lý do Rust yêu cầu rõ ràng về kiểu.
- 10 đến 25 phút: Các kiểu số, hằng số (literal) và cách chọn kiểu phù hợp.
- 25 đến 40 phút: Các hình thức chuyển đổi kiểu: `as`, `try_into` và `parse`.
- 40 đến 50 phút: Tuple, mảng và vấn đề an toàn khi truy cập theo chỉ số.
- 50 đến 60 phút: Bài thực hành ngắn: tính giá trị trung bình (mean) và phương sai (variance) an toàn, kèm kiểm thử các trường hợp biên.

## Mục tiêu học tập

Sau bài học này, bạn có thể:

- Phân biệt các nhóm kiểu số trong Rust và biết khi nào nên chọn từng loại.
- Nhận biết những tình huống Rust yêu cầu chuyển đổi kiểu tường minh.
- Tránh các phép ép kiểu gây mất dữ liệu và tràn số (overflow) ngoài ý muốn.
- Sử dụng đúng tuple và mảng, đồng thời hiểu rằng truy cập chỉ số trực tiếp có thể gây panic.

## Suy luận kiểu

Trong nhiều trường hợp, Rust có thể tự suy luận kiểu dữ liệu dựa trên ngữ cảnh sử dụng:

```rust
let x = 1;
let y = x + 2;
```

Tuy nhiên, cơ chế suy luận có giới hạn. Khi trình biên dịch không thể xác định duy nhất một kiểu, nó sẽ yêu cầu bạn chú thích rõ. Thói quen tốt là chỉ bổ sung chú thích kiểu tối thiểu vừa đủ để xóa bỏ sự nhập nhằng.

## Các kiểu số trong Rust

### Số nguyên

- Có dấu: `i8`, `i16`, `i32`, `i64`, `i128`, `isize`
- Không dấu: `u8`, `u16`, `u32`, `u64`, `u128`, `usize`

Nguyên tắc chọn kiểu thực dụng:

- Dùng `i32` cho hầu hết bộ đếm thông thường trong bộ nhớ, trừ khi có yêu cầu đặc biệt.
- Dùng `usize` cho chỉ số và kích thước tập hợp (tương thích với `slice` và `Vec`).
- Dùng `u64` cho các đại lượng đếm không bao giờ âm và có thể rất lớn (ví dụ kích thước tính theo byte).

### Số thực

- `f32` và `f64`

Nguyên tắc chọn kiểu: mặc định dùng `f64`, chỉ chuyển sang `f32` khi cần tối ưu bộ nhớ/băng thông hoặc làm việc với API hướng GPU.

### Hằng số (literal)

Các hằng số trong Rust có thể gắn trực tiếp hậu tố kiểu dữ liệu:

```rust
let a = 10u64;
let b = 3.14f64;
let c = 1_000_000usize;
```

Gắn chú thích kiểu trực tiếp vào literal là cách gọn gàng để định hướng cho bộ suy luận kiểu.

## Tràn số (overflow) và phép toán an toàn

Hành vi tràn số nguyên trong Rust phụ thuộc vào chế độ biên dịch:

- Ở chế độ **debug**, phần lớn phép toán bị tràn số sẽ gây panic.
- Ở chế độ **release**, tràn số nguyên thực hiện phép quay vòng theo biểu diễn bù hai (two's complement wrapping), trừ khi bạn chủ động dùng các phương thức kiểm tra hoặc bão hòa.

Các phương thức tính toán an toàn hơn:

```rust
let (sum, overflowed) = a.overflowing_add(b);
let sum = a.checked_add(b);
let sum = a.saturating_add(b);
```

Lưu ý: nếu thuật toán thực sự cần hành vi quay vòng (wraparound), hãy dùng phương thức tường minh (như `wrapping_add`). Trong các tính toán quan trọng khác, hãy dùng nhóm phương thức `checked_*` để phát hiện tràn số.

## Ba nhóm chuyển đổi kiểu

### 1. Phân tích từ chuỗi (`parse`)

`parse` luôn trả về `Result`:

```rust
let n: i64 = "42".parse()?;
```

Nếu cần thông báo lỗi rõ ngữ cảnh hơn, hãy bắt lỗi và chuyển đổi thông điệp trước khi trả về.

### 2. Ép kiểu bằng `as` (nhanh nhưng có thể mất dữ liệu)

Toán tử `as` không kiểm tra miền giá trị khi ép kiểu:

```rust
let x: u8 = 300u16 as u8; // becomes 44
```

Chỉ nên dùng `as` trong một phạm vi hẹp:

- Mở rộng số nguyên sang kiểu rộng hơn (ví dụ `u8` sang `u64`).
- Chuyển đổi giữa hai kiểu số thực (`f32` ↔ `f64`).

Tránh dùng `as` để thu hẹp miền giá trị, trừ khi bạn đã kiểm tra biên trước đó.

### 3. Chuyển đổi số nguyên có kiểm tra (`TryFrom` / `TryInto`)

Sử dụng `TryFrom` hoặc `TryInto` để tránh tình trạng cắt cụt dữ liệu âm thầm:

```rust
use std::convert::TryFrom;

let x = u8::try_from(300u16);
assert!(x.is_err());
```

Trong mã nguồn thực tế:

```rust
let x: u8 = value.try_into().map_err(|_| "out of range")?;
```

## Tuple và mảng

### Tuple

Tuple nhóm nhiều giá trị có thể khác kiểu lại với nhau:

```rust
let p: (i32, i32) = (10, 20);
let (x, y) = p;
```

### Mảng

Mảng có kích thước cố định tại thời điểm biên dịch và mọi phần tử phải cùng kiểu:

```rust
let a: [i32; 3] = [1, 2, 3];
```

Truy cập phần tử bằng cặp ngoặc vuông `[i]` sẽ gây panic nếu chỉ số vượt quá biên:

```rust
let v = vec![1, 2, 3];
// v[10] would panic
let maybe = v.get(10);
```

Khi chỉ số có khả năng nằm ngoài phạm vi hợp lệ, hãy dùng `.get()` để nhận về `Option`.

## Ví dụ thực hành: tính mean và variance an toàn

Ta xây dựng hai hàm tính giá trị trung bình (mean) và phương sai tổng thể (population variance) cho lát cắt đầu vào `&[u64]`, đáp ứng hai yêu cầu:

- Tránh tràn số nguyên khi tính tổng.
- Đảm bảo độ ổn định số học ở mức hợp lý.

### Giá trị trung bình (`mean`)

Dùng `u128` cho biến tổng trung gian để giảm thiểu nguy cơ tràn số:

```rust
pub fn mean(xs: &[u64]) -> Option<f64> {
    if xs.is_empty() {
        return None;
    }

    let sum: u128 = xs.iter().map(|&x| x as u128).sum();
    let n = xs.len() as f64;
    Some((sum as f64) / n)
}
```

Các điểm cần lưu ý:

- Trả về `Option<f64>` thể hiện rõ ràng rằng giá trị trung bình không xác định khi đầu vào rỗng.
- Chỉ chuyển sang `f64` sau khi đã cộng dồn trên kiểu nguyên rộng hơn (`u128`).

### Phương sai (`variance`)

Một cách tính ổn định là thuật toán hai lượt duyệt:

```rust
pub fn variance(xs: &[u64]) -> Option<f64> {
    let m = mean(xs)?;
    let n = xs.len() as f64;
    let mut acc = 0.0f64;

    for &x in xs {
        let dx = (x as f64) - m;
        acc += dx * dx;
    }

    Some(acc / n)
}
```

Các điểm cần lưu ý:

- Cách tính hai lượt dễ hiểu và đủ ổn định cho nhiều bài toán thông thường.
- Ở mức nâng cao hơn, có thể áp dụng thuật toán Welford để tính phương sai theo luồng (streaming) với độ ổn định số học cao.

## Kiểm thử các trường hợp biên

```rust
#[test]
fn mean_empty_is_none() {
    assert_eq!(mean(&[]), None);
}

#[test]
fn mean_simple() {
    assert_eq!(mean(&[1, 2, 3]).unwrap(), 2.0);
}

#[test]
fn variance_zero_for_constant() {
    assert_eq!(variance(&[5, 5, 5]).unwrap(), 0.0);
}
```

Ở ví dụ này, các bộ dữ liệu kiểm thử cho kết quả chính xác nên có thể so sánh trực tiếp bằng `assert_eq!`. Với các phép toán số thực có sai số làm tròn về sau, cần dùng phép so sánh gần đúng với ngưỡng sai số (epsilon).

## Bài tập

1. Chỉnh sửa hàm `variance` để tính phương sai mẫu (sample variance, chia cho `n - 1`) và trả về `None` khi `n < 2`.
2. Viết hàm tính giá trị trung bình theo luồng (cập nhật dần khi nhận từng giá trị mới).
3. Viết thêm hàm trả về `(min, max)` bằng iterator.
4. Thay một phép ép kiểu `as` bằng `try_into` và lan truyền lỗi nếu chuyển đổi thất bại.

## Tóm tắt

- Rust không tự động ép kiểu ngầm; mọi chuyển đổi kiểu đều phải tường minh.
- Tránh dùng `as` để thu hẹp miền giá trị khi chưa kiểm tra biên.
- Sử dụng kiểu trung gian rộng hơn để giảm nguy cơ tràn số.
- Ưu tiên `Option` và `Result` để biểu diễn kết quả không xác định hoặc phép tính có thể thất bại.

## Video tham khảo

- [\[Rust Cơ Bản\] Bài 2: Kiểu dữ liệu cơ bản và biến trong Rust  – Khóa Học Lập Trình Rust Cho Người Mới](https://www.youtube.com/watch?v=CojJ6Sc6LYI) — Vie Lang (tiếng Việt). Nên xem trước: trình bày kiểu dữ liệu cơ bản và biến bằng tiếng Việt, khớp với phần mở đầu bài học.
- [Rust from Zero -- Primitive number types and how to do safe arithmetic (i32, u32, usize, isize, etc)](https://www.youtube.com/watch?v=4w2tRIZbQ7E) — rvne (tiếng Anh). Giải thích `i32`, `u32`, `usize` và các phép toán có kiểm tra tràn số.
- [Visualizing memory layout of Rust's data types](https://www.youtube.com/watch?v=7_o-YRxf_cc) — Sreekanth (tiếng Anh). Minh họa cách kiểu số, tuple và mảng được bố trí trong bộ nhớ.
