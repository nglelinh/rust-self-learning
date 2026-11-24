---
layout: post
title: "00-01 Trạng thái bản tiếng Việt"
chapter: "00"
order: 1
owner: "Nguyen Le Linh"
lang: vi
categories:
  - chapter00
lesson_type: required
---

Khi một kỹ sư C++ hoặc Python lần đầu bước vào Rust, cú sốc lớn nhất không nằm ở cú pháp dấu chấm phẩy mà ở **mô hình tư duy hoàn toàn mới về bộ nhớ và quyền sở hữu**: lần đầu tiên, trình biên dịch không chỉ kiểm tra kiểu dữ liệu mà còn đóng vai trò một *nhà chứng minh toán học*, từ chối biên dịch mọi chương trình có nguy cơ truy cập bộ nhớ đã giải phóng hay tranh chấp dữ liệu (data race).

Trang lộ trình này giúp bạn định vị chính xác nhánh Tiếng Việt của khóa `rust-self-learning` đang đi tới đâu, cách kết hợp mượt mà giữa các bài Chương 01 đã Việt hóa đầy đủ với các chương chuyên sâu về Ownership, Lifetimes và Concurrency ở bản Tiếng Anh.

---

## Bản đồ kiến thức nền: Bức tranh toàn cảnh bốn trụ cột của ngôn ngữ Rust

Trước khi đi vào từng bài học, người tốt nghiệp đại học nên nhìn Rust qua bốn trụ cột kỹ thuật hệ thống liên kết chặt chẽ với nhau:

1. **Quản lý bộ nhớ không cần Garbage Collector (Ownership & RAII):** Mọi giá trị trên Heap đều có đúng một chủ sở hữu (owner) trên Stack; khi chủ sở hữu ra khỏi phạm vi (`scope`), bộ nhớ tự động giải phóng ngay lập tức với chi phí bằng 0 lúc chạy.
2. **Kiểm soát tham chiếu tại thời điểm biên dịch (Borrowing & Lifetimes):** Quy tắc *nhiều tham chiếu đọc `&T` HOẶC duy nhất một tham chiếu ghi `&mut T`* triệt tiêu tận gốc lỗi aliasing nguy hiểm và ngăn ngừa hoàn toàn data race ở lập trình đa luồng.
3. **Hệ thống kiểu đại số (Algebraic Data Types — `enum`, `Option`, `Result`):** Rust loại bỏ con trỏ `null` và ngoại lệ ẩn (`exception`), buộc mọi trường hợp vắng mặt hoặc lỗi phải được xử lý tường minh qua đối khớp mẫu (`match`).
4. **Chuỗi công cụ thống nhất (Cargo, rustc, Clippy, rust-analyzer):** Từ quản lý gói, kiểm thử đơn vị, kiểm tra tĩnh đến biên dịch chéo (WASM/nhúng), toàn bộ hệ sinh thái vận hành nhất quán qua một lệnh `cargo` duy nhất.

---

## Nội dung đã hoàn thiện

- **Chương 1 — Nền tảng và công cụ** đã được biên soạn bằng tiếng Việt, bao gồm: cài đặt chuỗi công cụ (toolchain), chương trình Rust đầu tiên, kiểu dữ liệu và toán tử, luồng điều khiển và pattern cơ bản, hàm, module và crate, cùng bài đọc mở rộng về ứng dụng hiện đại.
- Trang chủ và cấu hình trang web ghi rõ lộ trình tiếng Anh đã đầy đủ, trong khi bản tiếng Việt đang tiếp tục được mở rộng.

## Nội dung chưa có bản tiếng Việt

Các chương từ Chương 2 đến Chương 8 (ownership, trait, lifetime, lập trình đồng thời/bất đồng bộ, `unsafe`/FFI, giao diện đồ họa GUI và lập trình hệ thống) hiện mới có bản tiếng Anh.

## Cách học phù hợp ở giai đoạn hiện tại

1. Học **Chương 1 bằng tiếng Việt** nếu bạn muốn nắm vững các khái niệm mở đầu bằng tiếng mẹ đẻ.
2. Từ Chương 2 trở đi, chuyển sang **bản tiếng Anh** bằng nút chuyển ngôn ngữ trên thanh tiêu đề.
3. Nếu muốn đóng góp bản dịch, hãy giữ nguyên hai trường `chapter` và `order` khớp với bài tiếng Anh tương ứng, vì plugin đa ngôn ngữ dựa vào hai trường này để liên kết các phiên bản bài học.

## Hướng dẫn đóng góp

Nên ưu tiên dịch tuần tự từng chương, bắt đầu từ Chương 2, thay vì dịch rời rạc từng bài. Mỗi bài học đóng góp cần đáp ứng các yêu cầu sau:

- Khai báo đầy đủ YAML frontmatter gồm: `layout`, `title`, `chapter`, `order`, `lang: vi`, `categories`, `lesson_type`, `owner`.
- Giữ nguyên giá trị `chapter` và `order` giống bản tiếng Anh.
- Giữ nguyên các khối mã nguồn Rust; chỉ dịch phần giải thích, mục tiêu học tập và bài tập.
- Trình bày công thức toán học (nếu có) bằng cú pháp `$$ ... $$`.

Cảm ơn bạn đã đồng hành trong giai đoạn hoàn thiện phiên bản song ngữ.

---

### Ví dụ truy vết từng dòng (Stack / Heap / Trình biên dịch): Thấy ngay triết lý của Rust trong 6 dòng mã

Dù bạn mới bắt đầu ở Chương 01, hãy thử đọc trước một đoạn mã 6 dòng minh họa cách Rust nghĩ về Stack và Heap:

```rust
fn main() {
    let a = String::from("rust"); // Stack: (ptr, len=4, cap=4) -> Heap: ['r','u','s','t']
    let b = a;                    // MOVE: quyền sở hữu chuyển từ a sang b (không copy Heap!)
    // println!("{a}");           // LỖI BIÊN DỊCH E0382: a không còn hợp lệ nữa!
    println!("{b}");              // Hợp lệ: b in "rust"
}                                 // Cuối scope: drop(b) giải phóng Heap đúng 1 lần duy nhất!
```

| Dòng lệnh | Biến `a` trên Stack | Biến `b` trên Stack | Vùng nhớ Heap `"rust"` |
| :--- | :--- | :--- | :--- |
| `let a = String::from("rust");` | Chủ sở hữu `(ptr, 4, 4)` | Chưa tồn tại | Được cấp phát 4 byte |
| `let b = a;` | Bị vô hiệu hóa (*moved*) | Chủ sở hữu mới `(ptr, 4, 4)` | Giữ nguyên, **không sao chép** |
| Kết thúc `main()` | Không làm gì | Gọi `drop(b)` | Giải phóng **đúng 1 lần** (0 double-free!) |

### Khung giải thích: Cách học Rust hiệu quả nhất từ ngày đầu tiên (nên đọc chậm)

1. **Đừng chiến đấu với trình biên dịch — hãy đọc thông báo lỗi của nó:** Trình biên dịch `rustc` giải thích nguyên nhân lỗi kèm gợi ý sửa chi tiết hơn hầu hết mọi ngôn ngữ khác.
2. **Tách rõ dữ liệu nằm trên Stack và dữ liệu nằm trên Heap:** Luôn tự hỏi kích thước kiểu dữ liệu có biết trước lúc biên dịch không (Stack) hay tăng giảm động lúc chạy (Heap).
3. **Học vững Chương 01 bằng tiếng Việt rồi nối thẳng sang Chương 02:** Nắm chắc Cargo, biến bất biến mặc định, biểu thức `match` và module ở Chương 01 sẽ giúp bạn tiếp cận Ownership & Borrowing nhẹ nhàng hơn gấp bội.
4. **Chạy thử `cargo test` và `cargo clippy` như thói quen hàng ngày:** Mỗi ví dụ trong khóa đều nên được gõ lại, cố tình làm hỏng 1 dòng và đọc phản hồi của `rustc`.

**Kiểm tra nhanh.** Trong bảng truy vết 6 dòng mã ở trên, tại sao việc vô hiệu hóa `a` ngay sau lệnh `let b = a;` lại giúp ngăn chặn lỗi *double-free* (giải phóng cùng một vùng Heap hai lần) khi hàm `main` kết thúc?

---

## Video tham khảo

- [\[Rust Cơ Bản\] Bài 0: Giới Thiệu Ngôn Ngữ Rust – Khóa Học Lập Trình Rust Cho Người Mới](https://www.youtube.com/watch?v=E5pq7X1IFQA) — Vie Lang (tiếng Việt). Nên xem trước: giới thiệu tổng quan về Rust bằng tiếng Việt, phù hợp để bắt đầu trong khi lộ trình tiếng Việt đang được bổ sung.
- [LẬP TRÌNH #RUST Part 1: Những khái niệm chung cơ bản của #RUST](https://www.youtube.com/watch?v=k9dM0SiX9iU) — Jayden Dang (tiếng Việt). Trình bày các khái niệm nền tảng bằng tiếng Việt, thích hợp học song song với Chương 1 trước khi chuyển sang tài liệu tiếng Anh.
- [LẬP TRÌNH #RUST: Tổng hợp tài liệu từ Cơ bản tới Nâng cao về #RUST](https://www.youtube.com/watch?v=NufU_fRpStA) — Jayden Dang (tiếng Việt). Tổng hợp hệ thống tài liệu từ cơ bản đến nâng cao, hữu ích khi bạn tiếp tục học Chương 2–8 bằng tiếng Anh.
