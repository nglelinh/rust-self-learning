# AGENTS.md

## What This Repo Is

Jekyll course site with custom plugins for multilingual content (`en`/`vi`). Deploys to GitHub Pages via `.github/workflows/jekyll.yml` (not the default Pages Jekyll build).

## Dev Commands

```bash
bundle install

# Uses `_config.yml` for `baseurl`; browse at:
# http://127.0.0.1:4000/<baseurl>/
bundle exec jekyll serve

# Fast verification that the site builds
bundle exec jekyll build
```

Docker alternative (uses `jekyll/jekyll:4.2.0` and installs an old Bundler):

```bash
docker-compose up
```

## Content Wiring (High-Signal)

- Chapter landing pages live at `contents/{en,vi}/chapterXX/index.html` and must have front matter: `layout: page`, `lang: en|vi`, `chapter: "XX"`.
- Lecture posts live at `contents/{en,vi}/chapterXX/_posts/*.md`.
- Chapter navigation and ordering comes from:
  - `categories: [chapterXX]` (used by `_layouts/page.html` and `_layouts/post.html` via `site.categories["chapterXX"]`)
  - `order: <int>` (used to sort and to compute prev/next)
- Language switching is implemented in `_plugins/multilang.rb`:
  - First tries URL replacement between `/contents/en/` and `/contents/vi/`
  - If the target page/post is missing, it falls back to matching posts by the same `chapter` + `order`
  - Practical rule: keep `chapter` + `order` aligned between `en` and `vi` for corresponding lessons.

Required (repo-assumed) post front matter:

```yaml
---
layout: post
title: "Lesson Title"
chapter: "XX"          # two-digit string
order: 3               # integer
lang: en               # or vi
categories:
  - chapterXX          # must match the chapter dir/name
lesson_type: required  # or optional (drives badges in layouts)
owner: "Name"
---
```

## Internal Links + Assets

- For links to other posts, use `{% multilang_post_url ... %}` (implemented in `_plugins/multilang_post_url.rb`).
- Images: put in `img/chapter_img/` and reference with `{{ site.imgurl }}/chapter_img/<file>`.

## Deployment Notes

- GitHub Pages deploy is via Actions: `.github/workflows/jekyll.yml` runs `bundle exec jekyll build --baseurl "${{ steps.pages.outputs.base_path }}"`.
- Pages setting must be `Settings > Pages > Source: GitHub Actions`.
- If you change `_config.yml`, restart `jekyll serve` (config is read at boot).

## Fork/Template Placeholders To Fix

- `_layouts/default.html`: hardcoded GitHub repo link (`https://github.com/nglelinh/your-repo-name`).
- `_layouts/post.html`: Utterances comments `repo="convex-optimization-for-all/convex-optimization-for-all.github.io"`.

## Stale Automation Gotcha

`.github/workflows/closed_issue.yml` references `src/change_issue_title.py` and secret `CONVEX_ADMIN_TOKEN`, but `src/change_issue_title.py` is not present in this repo.

## Repo-Level Writing Rules

`.cursor/rules/*` are marked `alwaysApply` and will strongly steer generated lecture content and math formatting (notably: use `$$ ... $$`, not `$ ... $`).
