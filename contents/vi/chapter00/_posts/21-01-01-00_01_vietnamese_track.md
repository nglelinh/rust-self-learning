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

Bản tiếng Việt của khóa học **Lập trình Rust: Từ nền tảng đến thành thạo hệ thống** hiện đang trong quá trình hoàn thiện. Mục tiêu của lộ trình này là dần đạt độ bao phủ tương đương bản tiếng Anh, bổ trợ song song chứ không thay thế bộ tài liệu tiếng Anh đã hoàn chỉnh.

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

## Video tham khảo

- [\[Rust Cơ Bản\] Bài 0: Giới Thiệu Ngôn Ngữ Rust – Khóa Học Lập Trình Rust Cho Người Mới](https://www.youtube.com/watch?v=E5pq7X1IFQA) — Vie Lang (tiếng Việt). Nên xem trước: giới thiệu tổng quan về Rust bằng tiếng Việt, phù hợp để bắt đầu trong khi lộ trình tiếng Việt đang được bổ sung.
- [LẬP TRÌNH #RUST Part 1: Những khái niệm chung cơ bản của #RUST](https://www.youtube.com/watch?v=k9dM0SiX9iU) — Jayden Dang (tiếng Việt). Trình bày các khái niệm nền tảng bằng tiếng Việt, thích hợp học song song với Chương 1 trước khi chuyển sang tài liệu tiếng Anh.
- [LẬP TRÌNH #RUST: Tổng hợp tài liệu từ Cơ bản tới Nâng cao về #RUST](https://www.youtube.com/watch?v=NufU_fRpStA) — Jayden Dang (tiếng Việt). Tổng hợp hệ thống tài liệu từ cơ bản đến nâng cao, hữu ích khi bạn tiếp tục học Chương 2–8 bằng tiếng Anh.
