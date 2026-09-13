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
