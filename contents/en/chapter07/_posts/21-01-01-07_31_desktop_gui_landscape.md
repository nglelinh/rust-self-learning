---
layout: post
title: "07-31 Desktop GUI in Rust — Landscape and Why GPUI"
chapter: "07"
order: 31
owner: "OpenCode"
lang: en
categories:
  - chapter07
lesson_type: required
---

Rust's GUI ecosystem offers several frameworks with very different rendering models and trade-offs. This lesson maps the landscape, explains why this course focuses on GPUI, and sets up the context you need before diving into it in lesson 32.

## 60-minute teaching plan

- 0 to 15 min: Why Rust for desktop? Trade-offs vs Electron/Qt/Swift.
- 15 to 35 min: Framework survey — egui, iced, Tauri, gtk4-rs, Slint, GPUI.
- 35 to 50 min: Rendering models compared — immediate, retained, hybrid GPU.
- 50 to 60 min: Why this course uses GPUI; set up the crate and open a window.

## Learning goals

By the end of this lesson, students can:

- Explain the difference between immediate-mode, retained-mode, and hybrid GPU-accelerated GUI.
- Name the main Rust GUI frameworks and their primary trade-offs.
- Explain why GPUI is the framework this course focuses on.
- Open a minimal GPUI window on their machine.

## Why Rust for desktop?

Rust desktop apps offer a different trade-off from the mainstream alternatives:

- **vs Electron**: single executable, no Node.js runtime, significantly smaller RAM footprint and faster startup.
- **vs Qt/C++**: memory safety without manual smart pointer discipline; a growing ecosystem of pure-Rust alternatives.
- **vs Swift/WinUI**: cross-platform by default — one codebase targeting macOS, Linux, and Windows without large conditional compilation walls.

The sweet spot today: developer tools, data-heavy utilities, editors, and high-performance applications where GPU rendering matters.

## Framework survey

### egui — immediate-mode, pure Rust

Each frame, you call widget functions that both declare the UI and check for events. No widget objects live between frames. Simple mental model; good for tools and quick prototypes. Non-native appearance; CPU usage when idle unless configured for on-demand rendering.

### iced — Elm architecture, retained-mode

Widget tree is built each render pass and diffed. Architecture mirrors Elm: `Message` enum drives state transitions through `update`; `view` is a pure function. More boilerplate than egui; still pre-1.0 API.

### Tauri — web frontend + Rust backend

UI is HTML/CSS/JS running in the OS webview. Rust handles business logic and system calls. Right choice for teams with frontend skills or apps that already have a web UI. Webview rendering differs per OS; JS bridge overhead.

### gtk4-rs — GTK4 bindings

Full-featured, native-looking widgets on Linux; acceptable on macOS/Windows with GTK installed. Best for Linux-first or GNOME-ecosystem apps. GTK must be present on the target machine.

### Slint — declarative markup compiled to Rust

UI defined in `.slint` files, compiled to Rust. Good for embedded displays and kiosk-style apps. Custom markup to learn; commercial license for some use cases.

### GPUI — hybrid immediate + retained, GPU-accelerated

Built by [Zed Industries](https://zed.dev) to power the Zed code editor. Every frame is rendered on the GPU (Metal on macOS, Vulkan on Linux, DirectX on Windows). You write a `render` function that returns an element tree (immediate-mode ergonomics), but the framework diffs the tree and batches GPU draw calls (retained-mode efficiency). State lives in `Model<T>` entities; views subscribe to model changes explicitly.

This is the framework this course uses. Lessons 32 onward build up GPUI systematically.

## Rendering models compared

### Immediate-mode (egui)

```rust
// Called every frame — no widget object persists
if ui.button("Click me").clicked() {
    self.count += 1;
}
ui.label(format!("Count: {}", self.count));
```

The UI is a function of your state, recomputed 60 times per second. Simple to reason about; CPU spins even when nothing changes.

### Retained-mode (iced, gtk4-rs)

```rust
// iced view — returns a widget tree; framework diffs it
fn view(&self) -> Element<Message> {
    column![
        button("Click me").on_press(Message::Increment),
        text(self.count.to_string()),
    ].into()
}
```

The framework owns the widget tree and redraws only what changed. More explicit state propagation; efficient at scale.

### Hybrid GPU (GPUI)

```rust
// Render called when the view is invalidated — not every frame
impl Render for CounterView {
    fn render(&mut self, cx: &mut ViewContext<Self>) -> impl IntoElement {
        let count = self.counter.read(cx).value;
        div()
            .flex().items_center().justify_center().size_full()
            .bg(rgb(0x1e1e2e))
            .child(format!("Count: {count}"))
    }
}
```

You write immediate-mode style, but the runtime diffs the element tree and submits only the delta to the GPU. State changes are explicit (`Model<T>` + `cx.notify()`), so renders are precise rather than speculative.

## Decision matrix

| Criterion | egui | iced | GPUI | Tauri | gtk4-rs |
|---|---|---|---|---|---|
| Learning curve | Low | Medium | High | Medium | High |
| GPU rendering | No | No | Yes (native) | No | No |
| Rendering model | Immediate | Retained | Hybrid | Webview | Retained |
| Cross-platform | Yes | Yes | macOS first | Yes | Partial |
| API stability | Stable | Near-stable | Maturing | Stable | Stable |
| Best fit | tools | polished apps | editor-grade apps | web-team | Linux apps |

**Why GPUI for this course**: it is the most architecturally interesting framework in the Rust ecosystem right now — GPU rendering, an explicit entity/subscription state model, and a real production application (Zed) as a reference. Working through it teaches concepts — GPU pipelines, reactive entity graphs, fluent element APIs — that transfer to any framework.

> **Maturity note**: The standalone `gpui` crate is still being stabilized. APIs may change between versions. macOS is the most polished platform; Linux and Windows are available but less mature.

## Quick setup: open a GPUI window

```bash
cargo new --bin desktop-demo
cd desktop-demo
```

`Cargo.toml`:

```toml
[dependencies]
gpui = "0.1"
```

On Linux, install Vulkan and input headers first:

```bash
sudo apt install libvulkan-dev libxkbcommon-dev libwayland-dev
```

`src/main.rs`:

```rust
use gpui::*;

struct HelloView;

impl Render for HelloView {
    fn render(&mut self, _cx: &mut ViewContext<Self>) -> impl IntoElement {
        div()
            .flex()
            .items_center()
            .justify_center()
            .size_full()
            .bg(rgb(0x1e1e2e))
            .child(
                div()
                    .text_color(rgb(0xcdd6f4))
                    .text_xl()
                    .child("Hello from GPUI!")
            )
    }
}

fn main() {
    App::new().run(|cx: &mut AppContext| {
        cx.open_window(WindowOptions::default(), |cx| {
            cx.new_view(|_cx| HelloView)
        }).unwrap();
    });
}
```

```bash
cargo run
```

A GPU-rendered window should open. This is the foundation for lesson 32.

## In-class exercises

1. Confirm the window opens on your machine. Note the compile time — GPUI is a large crate.
2. Change the background color and text. Find additional color values in the GPUI docs.
3. Read the egui and iced README files. Write down one thing each does better than GPUI.

## Recap

- Immediate-mode (egui): UI as a function of state every frame — simple, CPU-hungry.
- Retained-mode (iced): widget tree diffed by the framework — efficient, more plumbing.
- Hybrid GPU (GPUI): immediate-mode authoring, GPU-diffed rendering, explicit entity state — highest performance, steepest curve.
- This course uses GPUI exclusively from lesson 32 onward.
