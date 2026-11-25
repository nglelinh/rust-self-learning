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

Open your activity monitor right now and look at how much RAM a modern desktop chat app or code editor consumes: often 500 MB to 1.5 GB just to display text, scroll a list, and blink a cursor. The culprit is rarely the application's own logic—it is the fact that shipping an Electron app means bundling an entire Chromium browser engine and Node.js runtime inside every window.

Rust offers a different path: native binaries that start in milliseconds, use a fraction of the memory, and talk directly to the GPU. Yet Rust's GUI landscape can feel bewildering at first because ownership and borrow-checking collide head-on with traditional object-oriented UI trees. This lesson maps the major Rust GUI architectures and explains why GPUI—the engine behind the Zed editor—strikes such a compelling balance.

## Key ideas: How Do Desktop GUI Engines Actually Draw Pixels?

Before comparing frameworks (`egui`, `iced`, `Tauri`, `Slint`, `GPUI`), it helps to understand the four architectural axes that separate every GUI engine on earth:

1. **Immediate Mode vs. Retained Mode**:
   In **immediate mode** (like `egui` or Dear ImGui), your app describes the entire UI from scratch on every frame (`if ui.button("Save").clicked() { ... }`); the framework does not keep a persistent widget object graph. It is wonderfully simple and fits Rust's borrow checker naturally, but can struggle with complex text shaping, accessibility trees, and skipping work when nothing changed. In **retained mode** (like `iced`, HTML DOM, or Qt), the framework maintains a persistent tree of widgets/elements and updates only dirty subtrees—harder to model with Rust ownership, but far better for rich text and OS accessibility.

2. **CPU Rasterization vs. GPU-Accelerated Scene Graphs**:
   Traditional toolkits paint widgets onto CPU memory buffers (`Skia`/Cairo bitmaps) and upload pixels to the screen. Modern **GPU-accelerated** engines (like GPUI, using Metal on macOS, Vulkan/DirectX on Linux/Windows) tessellate quads, shadows, rounded rectangles, and glyph atlases into vertex buffers and draw the entire window in a handful of GPU draw calls at 120 Hz.

3. **Native OS Widgets vs. Custom-Drawn Surfaces**:
   Native bindings (Cocoa, Win32, GTK) automatically inherit OS look-and-feel and screen readers, but behave inconsistently across macOS, Linux, and Windows. Custom-drawn engines render every pixel themselves for identical cross-platform behavior and game-like frame rates, while bridging OS services (IME input, clipboard, menus, accessibility) at the windowing layer.

4. **The Frame Budget & VSync Pipeline**:
   A desktop monitor refreshes at fixed intervals (60 Hz or 120 Hz). Every frame must finish **input handling → state update → layout calculation → paint/command recording → GPU submission** before the next VSync deadline, or the user perceives stutter ("jank").

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

### Explanation: The 120 FPS Frame-Budget Math — Browser DOM vs. GPU-Batched UI

Why does a Rust GPU-accelerated editor like Zed feel noticeably snappier than an Electron app on the exact same laptop? Let us do the exact millisecond arithmetic by hand.

#### Step 1: Computing the Hard Frame Deadline

A modern ProMotion / high-refresh display runs at $f = 120\text{ Hz}$ (standard displays run at $60\text{ Hz}$):
$$T_{60\text{ Hz}} = \frac{1000\text{ ms}}{60} \approx 16.67\text{ ms/frame}, \qquad T_{120\text{ Hz}} = \frac{1000\text{ ms}}{120} \approx 8.33\text{ ms/frame}.$$
If your UI pipeline takes $9.0\text{ ms}$ on a 120 Hz monitor, you **miss** the 8.33 ms VSync window and the OS holds the previous frame until $16.67\text{ ms}$—your effective frame rate instantly drops in half from 120 FPS to 60 FPS!

#### Step 2: Comparing Per-Frame Cost on a 2,000-Item Scrollable List

Imagine scrolling rapidly through a chat history or file tree with 80 visible rows, each containing an avatar, sender name, timestamp, and wrapped message (roughly $80 \times 5 = 400$ visual primitives: backgrounds, borders, and text runs).

| Pipeline Stage | Typical Electron / WebView Frame | Native GPU-Batched Rust (GPUI) Frame |
| :--- | :---: | :---: |
| Event dispatch + JS/GC or Rust state update | $1.5\text{ ms}$ (plus $4$–$12\text{ ms}$ GC pause spikes) | $0.15\text{ ms}$ (zero GC pauses) |
| Style cascade & layout (CSS vs. Taffy Flexbox) | $2.8\text{ ms}$ (DOM style recalc + reflow) | $0.45\text{ ms}$ (contiguous arena, 2-pass flex) |
| Paint / command generation | $2.2\text{ ms}$ (display list + layer tree) | $0.40\text{ ms}$ (emit 400 quad/glyph descriptors) |
| GPU draw calls & compositing | $1.6\text{ ms}$ (IPC to GPU process + tiles) | $0.50\text{ ms}$ (instanced quads + glyph atlas) |
| **Total frame time (typical)** | **$8.1\text{ ms}$** (borderline at 120 Hz; drops frames on GC) | **$1.50\text{ ms}$** ($6.83\text{ ms}$ headroom to spare!) |

#### Step 3: Checking Memory Footprint for the Exact Same Window

- **Electron baseline**: Chromium renderer process ($\approx 140\text{ MB}$) + GPU process ($\approx 70\text{ MB}$) + Node/V8 heap ($\approx 90\text{ MB}$) $\approx \mathbf{300\text{ MB}}$ before your app loads a single message.
- **Native Rust GPU app**: Compiled binary ($\approx 8\text{ MB}$) + glyph atlas texture ($2048 \times 2048 \times 4\text{ B} = 16\text{ MB}$) + app state ($\approx 4\text{ MB}$) $\approx \mathbf{28\text{ MB}}$.

That $10\times$ reduction in memory and $5\times$ reduction in frame time is not magic—it is the direct consequence of eliminating the browser runtime, garbage collector, and multi-process IPC from the render loop.

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

## Video tham khảo

- [Sponsor Lightning Talk (Zed) - Mikayla Maki: “Why Aren't We GUI Yet?”](https://www.youtube.com/watch?v=rpEU9DNbXA4) — Rust Foundation, English. **xem trước.** Watch first: a short Rust Foundation talk on why Rust GUI toolkits lagged, and why Zed cared.
- [Building a simple text editor with iced, a cross-platform GUI library for Rust](https://www.youtube.com/watch?v=gcBJ7cPSALo) — iced-rs, English. iced’s own walkthrough of an Elm-style desktop UI, one cell in this lesson’s matrix.
- [Simple GUI application with Slint and Rust](https://www.youtube.com/watch?v=WBcv4V-whHk) — Slint, English. Slint’s official Rust GUI intro, the declarative option next to iced and GPUI.
