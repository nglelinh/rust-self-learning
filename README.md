# Rust Programming: From Foundations to Systems Mastery

Khóa tự học Rust trên Jekyll — từ cú pháp và mô hình sở hữu đến concurrency, async, `unsafe`, FFI và lập trình hệ thống. Site hỗ trợ hai ngôn ngữ (`en` / `vi`) và deploy lên GitHub Pages.

**Trang khóa học:** [https://nglelinh.github.io/rust-self-learning/](https://nglelinh.github.io/rust-self-learning/)  
**Repository:** [https://github.com/nglelinh/rust-self-learning](https://github.com/nglelinh/rust-self-learning)

## Trạng thái nội dung

| Lộ trình | Trạng thái |
|----------|------------|
| **English** | Đầy đủ Chương 1–8 (bài giảng bắt buộc và tùy chọn) |
| **Tiếng Việt** | Đang xây dựng. Chương 1 (Nền tảng và công cụ) đã có bản dịch. Các chương còn lại sẽ được bổ sung dần. |

Chuyển ngôn ngữ bằng nút trên thanh tiêu đề của từng bài. Plugin đa ngôn ngữ khớp bài tương ứng theo `chapter` + `order`. Nếu bản tiếng Việt chưa có, trang sẽ đưa về landing lộ trình tiếng Việt (Chương 00).

## Đối tượng

Khóa học dành cho **người mới ở mức nâng cao đến trung cấp**: đã quen ít nhất một ngôn ngữ (C++, Java, Python, …), biết hàm, vòng lặp và cấu trúc dữ liệu. Không bắt buộc biết pointer; các khái niệm bộ nhớ được giải thích theo cách của Rust.

## Nội dung khóa học

1. **Foundations** — cài đặt, Cargo, chương trình đầu tiên, kiểu dữ liệu, control flow, module/crate  
2. **Ownership and Core Types** — ownership, borrowing, slice/string, struct, enum/`Option`/`Result`  
3. **Traits and Robustness** — xử lý lỗi, collection, iterator, trait  
4. **Advanced Ownership** — lifetime, API thân thiện với ownership, smart pointer, interior mutability  
5. **Concurrency and Async** — thread, message passing, `async`/`await`, Tokio, testing  
6. **Systems and Advanced Topics** — testing nâng cao, macro, `unsafe`, FFI, capstone  
7. **Desktop App Programming** — GUI, GPUI, layout, OS concepts, networking, kiến trúc ứng dụng  
8. **Operating Systems Programming** — process, file I/O, bộ nhớ, IPC, syscall/`nix`, kernel/embedded  

Mỗi bài là tài liệu tự chứa: giải thích khái niệm, ví dụ code tiến dần từ naive đến idiomatic, và bài tập.

## Chạy local

```bash
bundle install

# Dùng `baseurl` trong `_config.yml`; mở:
# http://127.0.0.1:4000/rust-self-learning/
bundle exec jekyll serve

# Chỉ build để kiểm tra
bundle exec jekyll build
```

Cách khác bằng Docker (`jekyll/jekyll:4.2.0`):

```bash
docker-compose up
```

Sau khi sửa `_config.yml`, khởi động lại `jekyll serve` — config chỉ được đọc lúc boot.

## Cấu trúc nội dung

- Trang chương: `contents/{en,vi}/chapterXX/index.html`  
  Front matter bắt buộc: `layout: page`, `lang: en|vi`, `chapter: "XX"`.
- Bài giảng: `contents/{en,vi}/chapterXX/_posts/*.md`
- Thứ tự điều hướng lấy từ `categories: [chapterXX]` và `order: <int>`
- Giữ `chapter` + `order` giống nhau giữa `en` và `vi` cho cùng một bài
- Link nội bộ dùng `{% multilang_post_url ... %}`
- Ảnh đặt trong `img/chapter_img/` và tham chiếu `{{ site.imgurl }}/chapter_img/<file>`
- Công thức dùng `$$ ... $$` (không dùng `$ ... $`)

Front matter bài giảng:

```yaml
---
layout: post
title: "Lesson Title"
chapter: "01"
order: 3
lang: en
categories:
  - chapter01
lesson_type: required
owner: "Author Name"
---
```

## Deploy

GitHub Actions (`.github/workflows/jekyll.yml`) chạy `bundle exec jekyll build`. Trong repo, **Settings > Pages > Source** phải là **GitHub Actions**.

## Đóng góp

Bản tiếng Anh đã đủ để học. Bản tiếng Việt cần dịch tiếp từ Chương 2 trở đi, giữ nguyên `chapter` + `order` so với bài tiếng Anh tương ứng. Xem thêm [AGENTS.md](./AGENTS.md) về convention của repo.

## License và credit

Site dùng theme [Lanyon](https://github.com/poole/lanyon) (Mark Otto) trên Jekyll, phục vụ mục đích giáo dục.
