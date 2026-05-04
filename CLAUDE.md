# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Repo Is

A Jekyll-based course site for "Rust Programming: From Foundations to Systems Mastery", with custom plugins for bilingual content (`en`/`vi`). Deploys to GitHub Pages via GitHub Actions (not the default Pages Jekyll build — `Settings > Pages > Source` must be set to **GitHub Actions**).

## Dev Commands

```bash
bundle install

# Local dev server — browse at http://127.0.0.1:4000/rust-self-learning/
bundle exec jekyll serve

# Verify build without serving
bundle exec jekyll build

# Docker alternative (jekyll/jekyll:4.2.0)
docker-compose up
```

Restart `jekyll serve` after any `_config.yml` change — config is read only at boot.

## Content Architecture

### Lecture Post Front Matter (required fields)

```yaml
---
layout: post
title: "Lesson Title"
chapter: "01"        # two-digit string, must match directory name
order: 3             # integer — drives prev/next navigation and sorting
lang: en             # or vi
categories:
  - chapter01        # must match chapter dir name
lesson_type: required  # or optional — drives badges in layouts
owner: "Author Name"
---
```

### Chapter Landing Pages

`contents/{en,vi}/chapterXX/index.html` — required front matter: `layout: page`, `lang: en|vi`, `chapter: "XX"`.

### Language Switching (`_plugins/multilang.rb`)

- First tries URL substitution between `/contents/en/` and `/contents/vi/`
- Falls back to matching by `chapter` + `order` if the target post is missing
- **Keep `chapter` + `order` identical across `en` and `vi` for corresponding lessons**

### Internal Links

Use `{% multilang_post_url ... %}` (from `_plugins/multilang_post_url.rb`) for cross-post links — never hardcode URLs.

### Images

Place in `img/chapter_img/` and reference as `{{ site.imgurl }}/chapter_img/<file>`.

### Math Formulas

Use `$$ ... $$` (block) or `$$ ... $$` inline — **not** `$ ... $` (not supported).

## Course Structure

| Chapter | Topic area |
|---------|------------|
| 01 | Setup, tooling, first program, types, control flow, modules |
| 02 | Ownership, borrowing, slices, structs, enums/Option/Result |
| 03 | Error handling, collections, traits |
| 04 | Lifetimes, ownership-friendly APIs, smart pointers, interior mutability |
| 05 | Concurrency, async/await, testing |
| 06 | Advanced testing, macros, unsafe Rust, FFI, capstone |
| 07 | Desktop GUI — landscape survey + GPUI (App, View, Model, element API) |
| 08 | OS programming — processes, files, memory, IPC, syscalls/nix, kernel/embedded |

## Lecture Note Style (from `.cursor/rules/lecture-notes-rule.mdc`)

Each lecture is a self-contained Markdown document with these sections (in order): **Title → Objectives → Prerequisites → Introduction → Key Concepts → Code Walkthroughs → Examples → Applications in Systems Programming → Challenges and Extensions → Exercises → References**.

- Target length: 1500–3000 words
- Code examples should evolve from naive → idiomatic Rust, treating compiler errors as teaching moments
- Use `$$ ... $$` for all math
- Embed reflective prompts (e.g., "Why does Rust forbid multiple mutable references…?")

## Known Placeholder Issues to Fix

- `_layouts/default.html` line 24: hardcoded GitHub link (`nglelinh/your-repo-name`) — replace with the actual repo.
- `_layouts/post.html`: Utterances comments widget points to `convex-optimization-for-all/...` — update to this repo.
- `.github/workflows/closed_issue.yml` references `src/change_issue_title.py` and secret `CONVEX_ADMIN_TOKEN` which don't exist in this repo.
