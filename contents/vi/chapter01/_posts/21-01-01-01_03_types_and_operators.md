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

Rust nghiêm với kiểu, nhưng phần thưởng là ít phép chuyển ẩn và ít bất ngờ lúc chạy. Bài này dạy "hình dạng kiểu" của Rust và những cách chuyển đổi an toàn nhất.

## Kế hoạch giảng 60 phút

- 0 đến 10 phút: Suy luận kiểu và vì sao Rust quan tâm.
- 10 đến 25 phút: Kiểu số và literal: chọn đúng kiểu.
- 25 đến 40 phút: Chuyển đổi: `as`, `try_into`, và parse.
- 40 đến 50 phút: Tuple, mảng, và an toàn khi đánh chỉ số.
- 50 đến 60 phút: Mini-lab: tính mean/variance an toàn và test edge case.

## Mục tiêu học

Cuối bài, bạn có thể:

- Giải thích các họ kiểu số của Rust và khi nào chọn từng họ.
- Dự đoán khi nào Rust đòi chuyển đổi tường minh.
- Tránh cast mất mát và overflow ngoài ý muốn.
- Dùng tuple/mảng đúng, và hiểu indexing có thể panic.

## Suy luận kiểu (Rust làm gì giúp bạn)

Rust thường suy ra kiểu từ cách dùng:

```rust
let x = 1;
let y = x + 2;
```

Nhưng suy luận có giới hạn. Nếu compiler không xác định được một kiểu duy nhất, nó sẽ yêu cầu bạn chú thích.

Thói quen giảng viên: khi compiler đòi kiểu, thêm chú thích nhỏ nhất đủ để hết nhập nhằng.

## Kiểu số trong Rust (bản đồ thực dụng)

### Số nguyên

- Có dấu: `i8 i16 i32 i64 i128 isize`
- Không dấu: `u8 u16 u32 u64 u128 usize`

Quy tắc ngón tay cái:

- Dùng `i32` cho hầu hết bộ đếm trong bộ nhớ, trừ khi có lý do.
- Dùng `usize` cho chỉ số/kích thước (vì slice/vec dùng nó).
- Dùng `u64` cho các đếm không bao giờ âm (ví dụ số byte).

### Số thực

- `f32` và `f64`

Quy tắc ngón tay cái:

- Mặc định `f64` trừ khi bạn tối ưu bộ nhớ/băng thông hoặc dùng API hướng GPU.

### Literal

Literal Rust có thể được chú thích:

```rust
let a = 10u64;
let b = 3.14f64;
let c = 1_000_000usize;
```

Dạy rằng literal có chú thích là cách sạch để dẫn dắt suy luận.

## Overflow và an toàn

Hành vi then chốt:

- Ở bản debug, nhiều overflow sẽ panic.
- Ở bản release, overflow số nguyên wrap (bù hai) trừ khi bạn dùng phương thức checked/saturating.

Các phép an toàn hơn:

```rust
let (sum, overflowed) = a.overflowing_add(b);
let sum = a.checked_add(b);
let sum = a.saturating_add(b);
```

Điểm giảng: nếu thuật toán dựa vào wraparound, hãy nói rõ. Nếu không, dùng checked math ở chỗ quan trọng.

## Chuyển đổi: ba nhóm

### 1. Parse từ chuỗi

Parse trả về `Result`:

```rust
let n: i64 = "42".parse()?;
```

Nếu muốn thông điệp lỗi tốt hơn, bắt lỗi và thêm ngữ cảnh.

### 2. Cast bằng `as` (nhanh, nhưng có thể mất mát)

`as` là công cụ thô:

```rust
let x: u8 = 300u16 as u8; // becomes 44
```

Dạy: `as` phù hợp một tập trường hợp hẹp:

- mở rộng số nguyên (ví dụ `u8` sang `u64`)
- chuyển float-sang-float (`f32` <-> `f64`)

Tránh: thu hẹp trừ khi đã kiểm tra khoảng giá trị.

### 3. Chuyển số nguyên có thể thất bại (`try_into`)

Dùng `TryFrom`/`TryInto` để tránh cắt cụt thầm lặng:

```rust
use std::convert::TryFrom;

let x = u8::try_from(300u16);
assert!(x.is_err());
```

Trong code thật:

```rust
let x: u8 = value.try_into().map_err(|_| "out of range")?;
```

## Tuple và mảng

### Tuple

Tuple nhóm các kiểu khác nhau:

```rust
let p: (i32, i32) = (10, 20);
let (x, y) = p;
```

### Mảng

Mảng có kích thước cố định và cùng kiểu:

```rust
let a: [i32; 3] = [1, 2, 3];
```

Indexing panic nếu ngoài biên:

```rust
let v = vec![1, 2, 3];
// v[10] would panic
let maybe = v.get(10);
```

Dạy: dùng `.get()` khi chỉ số có thể không hợp lệ.

## Ví dụ làm việc: mean và variance an toàn

Ta sẽ tính mean và phương sai (population) cho input `u64`. Ta muốn:

- không overflow số nguyên khi cộng
- ổn định số học ở mức hợp lý

### Mean

Dùng `u128` cho tổng trung gian để giảm rủi ro overflow:

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

Điểm giảng:

- Trả `Option` nói rằng "mean không xác định với input rỗng".
- Chỉ cast sau khi đã cộng ở kiểu rộng hơn.

### Variance

Một cách ổn định là thuật toán hai lượt:

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

Điểm giảng:

- Hai lượt dễ hiểu và đủ ổn định cho nhiều trường hợp.
- Sau này có thể dạy thuật toán Welford như biến thể streaming, ổn định số học.

## Test (dạy học viên test edge case)

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

Nếu so sánh số thực về sau bị nhiễu, hãy giới thiệu so sánh gần đúng. Lúc này giữ các case khớp chính xác.

## Bài tập

1. Đổi variance thành sample variance (chia cho `n - 1`), trả `None` khi `n < 2`.
2. Viết mean dạng streaming (cập nhật mean từng giá trị một).
3. Thêm hàm trả `(min, max)` bằng iterator.
4. Thay một cast `as` bằng `try_into` và lan truyền thất bại.

## Tóm tắt

- Rust không đoán chuyển đổi hộ bạn; hãy tường minh.
- Tránh `as` thu hẹp trừ khi đã kiểm tra khoảng.
- Dùng kiểu trung gian rộng hơn để giảm overflow.
- Ưu tiên `Option`/`Result` để biểu diễn tính toán không xác định hoặc thất bại.
