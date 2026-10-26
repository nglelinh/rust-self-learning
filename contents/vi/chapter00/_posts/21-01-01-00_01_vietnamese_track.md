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

Bản tiếng Việt của khóa **Lập trình Rust: Từ nền tảng đến thành thạo hệ thống** đang được xây dựng. Mục tiêu là dần đạt ngang hàng với lộ trình tiếng Anh, chứ không thay thế nội dung tiếng Anh đã hoàn chỉnh.

## Đã có gì

- **Chương 1 — Nền tảng và công cụ** đã được dịch: cài đặt toolchain, chương trình Rust đầu tiên, kiểu dữ liệu và toán tử, luồng điều khiển, module và crate.
- Trang chủ và cấu hình site nêu rõ lộ trình tiếng Anh đã đầy đủ; tiếng Việt còn đang mở rộng.

## Chưa có gì

Chương 2–8 (ownership, trait, lifetime, concurrency/async, `unsafe`/FFI, GUI, lập trình hệ thống) vẫn chỉ có bản tiếng Anh. Các bài toán/mẫu còn sót từ template khóa học cũ đã được gỡ khỏi lộ trình tiếng Việt.

## Cách học lúc này

1. Học **Chương 1 bằng tiếng Việt** nếu muốn bắt đầu bằng ngôn ngữ mẹ đẻ.
2. Từ Chương 2 trở đi, chuyển sang **bản tiếng Anh** (nút chuyển ngôn ngữ trên thanh tiêu đề).
3. Nếu muốn đóng góp bản dịch, giữ nguyên `chapter` và `order` so với bài tiếng Anh tương ứng — plugin đa ngôn ngữ dựa vào hai trường này để khớp bài.

## Gợi ý đóng góp

Ưu tiên dịch lần lượt từng chương, bắt đầu từ Chương 2, thay vì dịch rải rác. Mỗi bài cần:

- Front matter đủ: `layout`, `title`, `chapter`, `order`, `lang: vi`, `categories`, `lesson_type`, `owner`
- Cùng `chapter` + `order` với bản tiếng Anh
- Giữ khối code Rust nguyên văn; chỉ dịch phần giải thích, mục tiêu học, bài tập
- Công thức (nếu có) dùng `$$ ... $$`

Cảm ơn bạn đã kiên nhẫn trong giai đoạn song ngữ còn đang hoàn thiện.

## Video tham khảo

- [\[Rust Cơ Bản\] Bài 0: Giới Thiệu Ngôn Ngữ Rust – Khóa Học Lập Trình Rust Cho Người Mới](https://www.youtube.com/watch?v=E5pq7X1IFQA) — Vie Lang, tiếng Việt. Xem trước: video tiếng Việt giới thiệu Rust là gì, đúng chỗ để bắt đầu khi lộ trình tiếng Việt còn đang mở.
- [LẬP TRÌNH #RUST Part 1: Những khái niệm chung cơ bản của #RUST](https://www.youtube.com/watch?v=k9dM0SiX9iU) — Jayden Dang, tiếng Việt. Phần khái niệm chung bằng tiếng Việt, đủ để học song song với Chương 1 trước khi chuyển sang bài tiếng Anh.
- [LẬP TRÌNH #RUST: Tổng hợp tài liệu từ Cơ bản tới Nâng cao về #RUST](https://www.youtube.com/watch?v=NufU_fRpStA) — Jayden Dang, tiếng Việt. Bản đồ tài liệu từ cơ bản tới nâng cao, hữu ích vì Chương 2–8 hiện chỉ có bản tiếng Anh.
