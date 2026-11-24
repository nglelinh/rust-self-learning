---
layout: post
title: "07-37 Modern Applications — Desktop and Cross-Platform Rust UI"
chapter: "07"
order: 37
owner: "OpenCode"
lang: en
categories:
  - chapter07
lesson_type: optional
---

For years, skeptics argued that Rust was great for CLIs, databases, and WebAssembly, but "not ready for real desktop UI." Between 2022 and 2026, a wave of flagship products dismantled that argument in public: **Zed** (editing 100,000-line files at 120 FPS on GPUI), **Rerun** (streaming millions of 3D robotics points per second in `egui`), **Warp** and **WezTerm** (GPU-accelerated terminals), **Cosmic Desktop** (System76's entire Linux desktop environment written in `iced`), and **Tauri** (shrinking web-UI desktop binaries from 150 MB down to under 5 MB).

This lesson looks past framework tutorials to examine the *engineering architectures* that let these production Rust desktop products ship across macOS, Linux, and Windows.

## Background Knowledge Map: What Systems Innovations Power Production Rust Desktop Apps?

Modern Rust desktop products outperform legacy stacks because they rethink four classic bottlenecks:

1. **Glyph Atlases & Subpixel Text Rendering on the GPU**:
   Drawing text letter-by-letter with CPU calls is too slow for code editors and terminals. Production engines rasterize each unique `(font, glyph_id, size, subpixel_offset)` *once* into a shared GPU texture called a **glyph atlas** (packed via a 2D rectangle shelf/guillotine packer). Rendering 10,000 characters on screen then becomes a single instanced GPU draw call reading UV coordinates from that atlas.

2. **Hybrid Architecture (WebView Shell + Rust Core vs. Pure GPU Native)**:
   Not every product needs a custom GPU engine. **Tauri** reuses the OS's already-installed WebView (`WKWebView` on macOS, `WebView2` on Windows, `WebKitGTK` on Linux) for UI chrome while running heavy indexing, SQLite, and cryptography in native Rust over a typed IPC bridge. Pure-Rust GPU apps (`GPUI`, `iced`, `egui`, `Slint`) eliminate the browser engine entirely when frame-time predictability and memory efficiency are core product features.

3. **Data-Oriented Visualization & Zero-Copy GPU Buffers**:
   Tools like **Rerun** and **Bevy-based CAD viewers** bypass widget trees for their main canvas, uploading contiguous Rust slices (`&[Vertex]`, `#[repr(C)]` via `bytemuck`) directly into GPU vertex/storage buffers (`wgpu`) with zero serialization or garbage-collection pauses.

4. **OS Integration Layer (Accessibility, IME, HiDPI, Packaging)**:
   A desktop product is only "done" when it respects the host OS: **AccessKit** exposes native screen-reader accessibility trees across Windows/macOS/Linux; Input Method Editors (IME) allow Vietnamese, Japanese, and Chinese composition; and automated signing/notarization pipelines package `.dmg`, `.msi`, and `.AppImage`/Flatpak bundles.

## 60-minute teaching plan

- 0 to 8 min: Three architectures — GPU retained (GPUI), WebView shell (Tauri), native toolkit (iced/COSMIC, Slint, egui).
- 8 to 25 min: Zed as a shipping editor: why they rewrote GPUI instead of adopting Electron.
- 25 to 40 min: Tauri 2.0 — mobile + desktop, plugins, Rust commands.
- 40 to 52 min: COSMIC / iced and the “Rust desktop environment” bet.
- 52 to 60 min: Choosing a stack without rewriting Chapter 7 theory.

## Objectives

You will be able to place a new desktop idea on a map with three axes: who paints pixels (GPU toolkit vs OS WebView vs immediate-mode), who owns state (one Rust process vs a JS/Rust split), and who ships the binary (tiny Tauri vs full GPU engine). You will cite Zed, Tauri 2.0, and COSMIC as *products*, not crate names. The mental-model shift is from “Rust GUI is experimental” — true in 2019 — to “Rust GUI is a set of shipping trade-offs you can defend in a design review.”

## Prerequisites

Required Chapter 7 landscape, layout, OS concepts, networking, and app architecture. The optional GPUI deep-dive (07-32) is helpful but not required; this lesson treats GPUI as one industry data point among several. You should know what an event loop is and why a desktop app is not a Tokio ping-pong server with a window glued on.

## Introduction

On 3 January 2024 Zed Industries announced that GPUI 2 was in production on their preview channel — a rewrite of the UI framework that paints Zed, after two years of learning. Three weeks later they open-sourced the editor. The pitch was never “we needed a hobby GUI.” It was: a collaborative code editor that cannot drop frames, cannot afford Electron’s RAM tax, and must share an address space between UI and language-intelligence tasks.

On 2 October 2024 Tauri 2.0 went stable. Version 1 had been the “Electron but Rust + system WebView” answer for desktop. Version 2 added iOS and Android, a plugin model that can drop into Swift and Kotlin, and a 1.78 MSRV. That is a different product thesis: reuse the web frontend you already have; put OS and crypto in Rust.

System76 spent the same years building COSMIC, a desktop environment in Rust on iced, aimed at Pop!_OS. Whether or not you run it, the existence of a DE in Rust is the Chapter 7 landscape lecture becoming an operating-system vendor’s roadmap.

This lesson is a comparison shop. We will write only enough code to make the *boundaries* visible.

## Key Concepts

### Who rasterizes?

**GPUI / Zed.** GPU first (Metal / Vulkan / DirectX). The element tree is retained; authoring feels immediate. One process, one language. The optional 07-32 lesson is the API tour; here the point is *why it exists*: editors, IDEs, data-dense tools.

**Tauri.** The OS WebView rasterizes HTML/CSS/JS. Rust is the privileged backend (`#[tauri::command]`). Two runtimes, a serialization boundary, a tiny installer. Tauri 2.0’s mobile support doubles that bet: the same command plugin on Android/iOS via Kotlin/Swift shims.

**iced / COSMIC, Slint, egui.** Toolkit-native. iced is Elm-style messages (a cousin of Chapter 7’s events → state → render). egui is immediate-mode and dominates tools/debug UIs. Slint compiles a declarative UI to Rust (and other languages) for embedded-adjacent desktop.

If you cannot say who paints the pixels, you cannot budget RAM or frame time.

### Who owns the state?

Zed: GPUI entities (`Model`, views) plus Tokio-like async tasks inside one process. A buffer is a Rust type, not a JSON blob sent to Chromium.

Tauri: the UI owns view state in JS/TS; Rust owns files, sockets, secrets. Every button that needs a file path crosses an IPC command. That is Chapter 7 networking and Chapter 8 IPC wearing a WebView.

COSMIC: compositor + applets as Rust processes, iced widgets, Unix desktop conventions (Wayland). State lives where a DE usually puts it — in compositor and session services — but the language is Rust.

### Who pays the binary?

Tauri’s marketing (and a lot of 2024 blog posts) is installer size. Zed’s marketing is frame time. COSMIC’s marketing is a Linux desktop that is not a C++ museum. All three are honest if you pick the matching product.

### Deep-Dive Explanation: Sizing a GPU Glyph Atlas and Instanced Draw Buffer for a 10,000-Character Screen (Read Slowly)

Why can a GPU-accelerated Rust editor or terminal (like Zed or Alacritty) redraw an entire 4K screen full of syntax-highlighted code in under $1\text{ ms}$? Let us compute the exact GPU memory footprint and PCIe/memory bandwidth by hand.

Suppose a developer opens a 4K monitor showing **100 lines $\times$ 100 columns = $10{,}000$ visible characters** of Rust code.

#### Step 1: Computing the Glyph Atlas Texture Footprint

In typical source code, even across bold/italic/regular styles, how many *distinct* glyphs actually appear on screen? Roughly $256$ unique ASCII + punctuation + box-drawing glyphs (or up to $\sim 1{,}000$ with Unicode/CJK).
- Suppose each rasterized glyph cell at $2\times$ Retina resolution fits in a $32 \times 48\text{ px}$ tile.
- At 4 bytes per pixel (RGBA subpixel antialiasing), one glyph occupies:
  $$32 \times 48 \times 4 = 6{,}144\text{ bytes} \approx 6\text{ KB}.$$
- Packing $256$ unique glyphs takes only $256 \times 6\text{ KB} = \mathbf{1.5\text{ MB}}$—easily fitting inside a single $1024 \times 1024$ RGBA GPU texture ($1024 \times 1024 \times 4 = \mathbf{4.0\text{ MB}}$), uploaded **once** and reused across thousands of frames!

#### Step 2: Computing Per-Frame Instanced Quad Data (10,000 Characters)

On every frame, instead of uploading pixels, the CPU uploads one compact `#[repr(C)]` `GlyphInstance` struct per visible character:

```rust
#[repr(C)]
struct GlyphInstance {
    screen_pos: [f32; 2],  // 8 B: (x, y) in window
    size:       [f32; 2],  // 8 B: (w, h) in pixels
    atlas_uv:   [f32; 4],  // 16 B: (u0, v0, u1, v1) inside glyph atlas
    color_rgba: [u8; 4],   // 4 B: syntax highlight color
} // Total = 36 bytes per character
```

For all $10{,}000$ characters on the 4K screen:
$$\text{Per-frame upload} = 10{,}000 \times 36\text{ bytes} = 360{,}000\text{ bytes} \approx \mathbf{352\text{ KB}}.$$

#### Step 3: Comparing Against CPU Full-Framebuffer Pixel Upload

Compare transferring $352\text{ KB}$ of instanced glyph descriptors vs. rasterizing a 4K ($3840 \times 2160$) RGBA framebuffer on the CPU and uploading it every frame:

| Approach | Bytes Uploaded per Frame | Draw Calls | Memory Bandwidth at 120 FPS |
| :--- | :---: | :---: | :---: |
| **CPU Full-Screen Raster** | $3840 \times 2160 \times 4 = \mathbf{33.18\text{ MB}}$ | 1 blit | $\mathbf{3.98\text{ GB/s}}$ (plus heavy CPU raster loop) |
| **GPU Glyph Atlas + Instancing** | $\mathbf{0.35\text{ MB}}$ ($94\times$ smaller!) | **1 instanced call** | $\mathbf{42.2\text{ MB/s}}$ (GPU draws all 10k quads in $< 0.2\text{ ms}$) |

That $94\times$ reduction in per-frame data transfer explains why GPU-native Rust apps stay cool, quiet, and locked at 120 FPS even on battery power.

## Code Walkthroughs

### Naive: “I’ll just spawn a window and block”

```rust
fn main() {
    // Pseudo: open a window, then do network on the UI thread.
    let _window = DummyWindow::open("Notes");
    std::thread::sleep(std::time::Duration::from_secs(5)); // freezes the frame
}
```

Chapter 7 already forbade this. Every stack below still forbids it. The *modern* version of the bug is `await`ing HTTP on the UI executor (GPUI/iced) or doing 200 ms of Rust work inside a Tauri command that the WebView thinks is “one click.”

### Tauri 2 shape: command as the FFI

```rust
#[tauri::command]
fn allow_host(host: String) -> bool {
    !host.ends_with(".invalid")
}

fn main() {
    tauri::Builder::default()
        .invoke_handler(tauri::generate_handler![allow_host])
        .run(tauri::generate_context!())
        .expect("tauri app");
}
```

The frontend calls `invoke('allow_host', { host })`. Serde sits on the boundary (Chapter 7 networking lesson). Ownership is simple: `String` is owned across IPC, then dropped. The new 2.0 work is *where this binary also runs* (phone) and *how plugins* reach camera/notifications through Swift/Kotlin.

A typical compiler error here is a command that is not `Send` + `'static` because it captured a `MutexGuard`. That is Chapter 5 + Chapter 7: the WebView thread is not your lock’s home.

### GPUI / Zed shape: state is an entity, not a JSON message

```rust
// Shape from 07-32 — reminder, not a rewrite of that lesson.
// App::new().run(|cx| { cx.open_window(..., |cx| cx.new_view(|_| Editor::new()) }); }
```

The application lesson: Zed’s 2024 GPUI 2 rewrite existed to make *that* entity graph cheap enough to open-source and to port. If you need a second language in the UI, you are not on this path.

### iced / COSMIC shape: messages

```rust
#[derive(Debug, Clone)]
enum Msg {
    HostChanged(String),
    Check,
}

struct State { host: String, ok: bool }

fn update(state: &mut State, msg: Msg) {
    match msg {
        Msg::HostChanged(h) => state.host = h,
        Msg::Check => state.ok = !state.host.ends_with(".invalid"),
    }
}
```

This is Chapter 7 architecture in iced dialect. COSMIC applets are this loop plus Wayland. You do not need to memorize iced’s widget names; you need to see that *events → state → view* survived contact with a desktop environment vendor.

## Examples

### Example 1 — Zed as a 2024 product

GPUI 2 in production (3 January 2024) and the public `zed-industries/zed` repository (open source 24 January 2024, GPL-3.0-or-later with Apache components) are the primary sources. Zed is a collaborative editor: CRDT-ish buffers, GPU text, LSP via rust-analyzer, async tasks for git and AI features. Every one of those is a Chapter 7 + Chapter 5 system. The lesson is not “fork Zed.” It is “this is what a GPU-native Rust UI looks like when it has users.”

### Example 2 — Tauri 2.0 as a 2024 product

The official *Tauri 2.0 Stable Release* post (2 October 2024) is the citation: mobile, plugin system, CrabNebula’s engineering hours, system WebView. Thousands of intern tools and indie apps picked Tauri because their UI was already React or Svelte. The Rust you write is closer to a privileged microservice that happens to live in-process.

### Example 3 — COSMIC and the rest of the toolkit map

COSMIC (System76) on iced is a desktop environment: panels, launcher, compositor integration. Slint ships industrial/embedded UIs with a designer. egui owns the “debug window in a game or tool” niche (Rerun, many Bevy editors). Dioxus (including 2024–2026 desktop/fullstack work) sits near Tauri’s “web skills, Rust core” story but with a Rust-native VDOM. You should be able to *reject* a toolkit in an interview: “egui is the wrong choice for a design-system marketing site; Tauri is the wrong choice for a 144 Hz shader editor.”

## Applications in Systems Programming

**Editors and IDEs.** Zed, Lapce (earlier Rust editor), and parts of rust-analyzer’s own UI experiments. GPU + entity graph.

**Enterprise wrappers.** Tauri 2 around an existing React console, with Rust doing PKCS#11, USB, or local HTTP.

**Linux desktops.** COSMIC as a vendor-scale Rust GUI; also smaller iced/Relm4 apps.

**Game and science tools.** egui and iced for in-engine overlays; Slint for kiosk/embedded glass.

## Challenges and Extensions

Accessibility and IME still lag the web on several pure-Rust toolkits. Tauri inherits the WebView’s a11y; GPUI/iced have to build it. That is a shipping risk, not a footnote.

Linux fragmentation (X11/Wayland, NVIDIA, window decorations) punished every toolkit in 2022–2025. Zed’s Linux port and COSMIC’s Wayland bet are engineering programs, not crate.toml lines.

Hot-reload and designer tooling are why some teams still pick Slint or a WebView. Rust compile times are part of the GUI story (Chapter 1 tooling, again).

Reflect: if you were building a shared-cache desktop client for a multithreaded backend (Chapter 5), would you put the cache in the UI process (GPUI/iced) or behind Tauri commands? Who owns the `Arc`?

## Exercises

1. **Conceptual.** Fill a 3×2 table: rows GPUI, Tauri 2, iced; columns “who paints” and “who owns file secrets.” No extra research beyond this lesson.
2. **Code fix.** Take the naive `sleep` on the UI thread and move the five-second job to `std::thread` or a Tauri async command, posting a message back. Do not invent a new architecture.
3. **Boundary.** Write the Serde types for a Tauri command that returns `{ ok: bool, reason: Option<String> }`. Why is `reason` not a borrowed `&str`?
4. **Architecture.** Sketch Zed’s path for “user typed a character” from OS event to GPU frame using Chapter 7 vocabulary (event, model, layout, render). One page.
5. **Implementation.** Build a 30-line egui *or* Tauri hello that calls the Chapter 1 `allow` predicate. Record binary size (`ls -lh`) and which process would hold a TLS key.

## References

- [GPUI 2 is now in production](https://zed.dev/blog/gpui-2-on-preview) — Nathan Sobo, 3 January 2024.
- [zed-industries/zed](https://github.com/zed-industries/zed) — open source 24 January 2024.
- [Tauri 2.0 Stable Release](https://v2.tauri.app/blog/tauri-20/) — 2 October 2024.
- [iced](https://iced.rs/) and System76 COSMIC — vendor desktop in Rust.
- [egui](https://www.egui.rs/) and [Slint](https://slint.dev/) — toolkit poles of the landscape lesson.

## Recap

- 2024 made Rust desktop *real*: Zed shipped and open-sourced; Tauri 2.0 added mobile; COSMIC made a DE.
- Pick a rasterizer and an ownership boundary first; crates second.
- This lesson complements 07-32 (GPUI API) and does not replace it.
- The Chapter 7 pipeline (events → state → layout → render) is identical in all three products; only the process split changes.

## Video tham khảo

- [Rendering at 1 million pixels / millisecond with GPUI - Conrad Irwin | EuroRust 2025](https://www.youtube.com/watch?v=sheIOOf-xRo) — EuroRust, English. **xem trước.** Watch first: GPUI as Zed’s GPU toolkit, one corner of this lesson’s product map.
- [Tauri for Beginners: Build a Desktop App in Rust](https://www.youtube.com/watch?v=KhAkFqUi9l0) — Codynn, English. Tauri as the WebView-plus-Rust alternative the lesson contrasts with a full GPU engine.
- [Rust GUI with egui](https://www.youtube.com/watch?v=YNvfqmEgUFQ) — Saad Saleem, English. egui’s immediate mode, the third rasterization choice next to GPUI and Tauri.
