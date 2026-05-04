---
layout: post
title: "07-33 Layout Systems (Replacing CSS)"
chapter: "07"
order: 33
owner: "OpenCode"
lang: en
categories:
  - chapter07
lesson_type: required
---

In a browser you reach for flexbox and CSS grid almost automatically. In a Rust GUI framework those tools don't exist — layout is computed in code, not declared in a stylesheet. This lesson builds a ground-up understanding of how layout systems work: box models, constraint trees, positioning algorithms, and spacing/alignment primitives. GPUI is used for examples, but the mental model applies to every framework.

## 60-minute teaching plan

- 0 to 10 min: Why there is no CSS — what a layout system must do without a browser.
- 10 to 25 min: The box model: size, constraints, available space.
- 25 to 40 min: Layout trees: how parent constraints flow down, measured sizes flow up.
- 40 to 55 min: Positioning and alignment in GPUI — flex axis, cross axis, spacing.
- 55 to 60 min: Debugging layout and recap.

## Learning goals

By the end of this lesson, students can:

- Explain what a layout tree is and how constraint propagation works.
- Describe the box model (content box, padding, border, margin) without CSS.
- Use GPUI's flex layout API to build common arrangements.
- Debug layout issues systematically by reasoning about constraints.

## Prerequisites

- Lesson 07-32 (GPUI fundamentals): `div()`, the fluent element API, `render`.

---

## Key concept: why there is no CSS

CSS is a *declarative cascade* — you describe intent, the browser resolves conflicts, computes specificity, and runs a layout engine. None of this infrastructure exists in a Rust GUI framework:

- No DOM, no selector engine, no cascade resolver.
- No browser engine (Gecko, Blink) doing the layout math.
- No font metrics from a browser runtime.

Instead, **you write code that describes constraints directly**. The layout system is a Rust library that:

1. Walks a tree of elements.
2. Passes available space down from parent to child.
3. Receives measured sizes back from children.
4. Places children at computed positions.

This is more explicit, more predictable, and faster — but you need to understand the model to use it effectively.

### What you lose vs CSS

| CSS feature | Rust GUI equivalent |
|-------------|----------------------|
| `display: flex` | `.flex()` — available in most frameworks |
| `display: grid` | Manual: compute column widths, place children at indices |
| Media queries | Observe window size, re-render with different constraints |
| `calc()` | Arithmetic in Rust: `px(base + offset)` |
| `min-width`, `max-width` | Constraint ranges in the layout API |
| Inheritance | Pass values down explicitly via `render`'s local variables |
| `:hover`, `:focus` | State in the view struct, event handlers set flags |
| Animations / transitions | Animate values manually or use a tween library |

The key insight: **CSS solves these problems with runtime string parsing and a cascade. Rust solves them with function calls and types**. More verbose, but the compiler checks it.

---

## The box model

Every layout element occupies a rectangular region. That region is computed from four nested boxes:

```
┌──────────────────────────────────┐
│              margin              │
│  ┌────────────────────────────┐  │
│  │           border           │  │
│  │  ┌──────────────────────┐  │  │
│  │  │        padding       │  │  │
│  │  │  ┌────────────────┐  │  │  │
│  │  │  │  content box   │  │  │  │
│  │  │  └────────────────┘  │  │  │
│  │  └──────────────────────┘  │  │
│  └────────────────────────────┘  │
└──────────────────────────────────┘
```

- **Content box**: where text, images, and children are drawn.
- **Padding**: space between content and border. Part of the element's visible area; background color fills it.
- **Border**: a drawn border line. Occupies space.
- **Margin**: space between this element and its siblings/parent. Transparent.

### The sizing question: what does "width" mean?

This is the most common source of layout confusion. Two models exist:

**Content-box sizing** (CSS default):
```
total_width = content_width + padding_left + padding_right + border_left + border_right
```

**Border-box sizing** (CSS `box-sizing: border-box`):
```
content_width = set_width - padding - border
total_width = set_width
```

GPUI follows the **border-box** model — when you write `.w(px(200.0))`, the element takes 200px including its padding and border. This is almost always what you want.

### In GPUI

```rust
div()
    .w(px(200.0))        // total width = 200px (border-box)
    .h(px(100.0))
    .p(px(16.0))         // padding: 16px all sides
    .border_1()          // 1px border
    .m(px(8.0))          // 8px margin (pushes siblings away)
```

The content area available to children:
$$\text{content\_width} = 200 - 16 - 16 - 1 - 1 = 166\ \text{px}$$

This is the "available space" that children receive.

---

## Layout trees and constraint propagation

### The algorithm (simplified)

A layout pass is a two-phase tree walk:

**Phase 1 — top-down: distribute available space**

```
root receives: available = window_size
root passes to each child: available minus root's padding/border
each child repeats recursively
```

**Phase 2 — bottom-up: measure actual sizes**

```
leaf elements measure themselves (text length, image size, or fixed size)
each parent collects children's sizes and computes its own size
sizes bubble up to the root
```

After both phases, every element has a **position** and a **size** — enough to draw.

### Rust pseudocode for the core idea

```rust
struct Constraints {
    available_width:  Option<f32>,   // None = unconstrained
    available_height: Option<f32>,
}

struct Measured {
    width:  f32,
    height: f32,
}

trait Element {
    // Phase 1: receive constraints from parent
    // Phase 2: return measured size to parent
    fn layout(&self, constraints: Constraints) -> Measured;
    
    fn position(&self) -> Point<f32>;  // set after phase 2
}
```

### Fixed sizes vs intrinsic sizes

| Approach | What it means |
|----------|---------------|
| `.w(px(200.0))` | Fixed: ignore available space, always 200px |
| `.w_full()` | Fill: take all available width from parent |
| `.w_auto()` | Intrinsic: size to fit content (text, images) |
| `.min_w(px(100.0))` | Constraint: at least 100px |
| `.max_w(px(400.0))` | Constraint: at most 400px |

**Mixing fixed and fill** is where layout gets interesting — and where common bugs live:

```rust
// Parent: 500px wide, flex row
div()
    .flex()
    .w(px(500.0))
    .child(
        div().w(px(100.0)).child("fixed")      // takes 100px
    )
    .child(
        div().flex_1().child("fills rest")     // takes remaining 400px
    )
```

`flex_1()` means: "take a flex-grow of 1, claiming available space after fixed-size siblings are placed."

---

## Flex layout in depth

Flex layout organizes children along one axis (the **main axis**) and aligns them on the perpendicular axis (the **cross axis**).

### Main axis: `flex_row` vs `flex_col`

```rust
// Horizontal stack — children left to right
div().flex().flex_row()   // default when .flex() is called

// Vertical stack — children top to bottom
div().flex().flex_col()
```

### Main axis alignment: `justify_*`

Controls how children are distributed along the main axis when there is leftover space:

```
justify_start     [A][B][C]         ←← space
justify_end                  [A][B][C]
justify_center         [A][B][C]
justify_between   [A]    [B]    [C]
justify_around    ·[A]·  ·[B]·  ·[C]·
justify_evenly    ·[A]··[B]··[C]·
```

In code:

```rust
div()
    .flex()
    .justify_between()
    .w_full()
    .child(div().child("left"))
    .child(div().child("right"))
```

### Cross axis alignment: `items_*`

Controls how children align perpendicular to the main axis:

```
items_start    ┌──┐ ┌──────┐ ┌────┐
               └──┘ └──────┘ └────┘

items_center   ┌──┐             
               │  │ ┌──────┐
               │  │ │      │ ┌────┐
               │  │ └──────┘ └────┘
               └──┘

items_end       ┌──┐ ┌──────┐ ┌────┐
                └──┘ └──────┘ └────┘
```

In GPUI:

```rust
div()
    .flex()
    .items_center()  // vertically center children in a row
```

### Individual child alignment: `self_*`

A child can override the parent's `items_*` for itself:

```rust
div()
    .flex()
    .items_start()
    .child(div().self_center().child("I'm centered"))   // overrides items_start
    .child(div().child("I'm at the start"))
```

---

## Sizing strategies in practice

### The common patterns

**1. Fill the container**

```rust
div().size_full()        // width: 100%, height: 100%
div().w_full()           // width: 100%, height: auto
div().h_full()           // height: 100%, width: auto
```

Requires the parent to have a concrete size, or to be in a flex container with a defined cross-axis size.

**2. Fixed size**

```rust
div().w(px(240.0)).h(px(48.0))
```

Use for: icons, buttons, sidebars with a known width, toolbars.

**3. Grow to fill remaining space**

```rust
// In a flex row: sidebar is fixed, content fills the rest
div()
    .flex()
    .h_full()
    .child(div().w(px(240.0)).child(sidebar_content))
    .child(div().flex_1().child(main_content))   // takes 100% - 240px
```

**4. Shrink to content, but don't exceed a max**

```rust
div()
    .max_w(px(600.0))
    .w_auto()       // will size to content, capped at 600px
```

**5. Responsive based on window size**

Since there are no media queries, you check the window size in `render`:

```rust
impl Render for MyView {
    fn render(&mut self, cx: &mut ViewContext<Self>) -> impl IntoElement {
        let window_width = cx.window_bounds().get_bounds().size.width;
        
        if window_width < px(600.0) {
            // compact layout
            div().flex().flex_col()
                .child(self.render_sidebar(cx))
                .child(self.render_main(cx))
        } else {
            // wide layout
            div().flex().flex_row()
                .child(div().w(px(240.0)).child(self.render_sidebar(cx)))
                .child(div().flex_1().child(self.render_main(cx)))
        }
    }
}
```

---

## Spacing and alignment: from first principles

### Spacing between siblings: `gap`

`gap` adds space between children without adding padding/margin to each child. It only applies when the parent is a flex container.

```rust
div()
    .flex()
    .gap(px(8.0))       // 8px between every child
    .gap_x(px(12.0))    // gap on the main axis only
    .gap_y(px(4.0))     // gap on the cross axis only
```

Do not replicate gap with per-child margins — gaps are easier to reason about and don't add unwanted space at the start/end.

### Padding vs margin

| | Padding | Margin |
|---|---------|--------|
| Part of element | Yes | No |
| Background visible | Yes | No (transparent) |
| Click area | Included | Not included |
| Collapses | Never | Yes (in CSS; GPUI: not applicable) |
| Use case | Internal breathing room | Push away from siblings |

In GPUI:

```rust
// Padding: space inside the element
div().p(px(16.0))              // all sides
div().px(px(24.0)).py(px(12.0)) // horizontal / vertical
div().pt(px(8.0)).pb(px(8.0))   // top / bottom independently

// Margin: space outside
div().m(px(8.0))
div().mx(px(auto))   // center horizontally in a block context
div().mt(px(16.0))   // margin-top only
```

### Inset / absolute positioning

For overlays, tooltips, and badges, you need to place elements at a specific position within their parent:

```rust
div()
    .relative()                          // parent: establish a positioning context
    .child(
        div()
            .absolute()                  // child: position relative to parent
            .top(px(8.0))
            .right(px(8.0))
            .child("badge")
    )
```

`.absolute()` removes the element from flex layout — it doesn't affect the size or position of siblings.

---

## Code walkthrough: naive → idiomatic layout

### Naive: margin spam to achieve spacing

```rust
div()
    .flex()
    .child(div().mr(px(8.0)).child("A"))   // manually adding margin to each child
    .child(div().mr(px(8.0)).child("B"))
    .child(div().mr(px(8.0)).child("C"))   // last child has unwanted trailing margin
```

Problems: last child has unnecessary margin; if the order changes you have to update margins; hard to read.

### Idiomatic: `gap` on the container

```rust
div()
    .flex()
    .gap(px(8.0))                          // one declaration, no trailing issue
    .child(div().child("A"))
    .child(div().child("B"))
    .child(div().child("C"))
```

### Naive: manual centering

```rust
div()
    .w(px(500.0)).h(px(300.0))
    .child(
        div()
            .ml(px(225.0)).mt(px(140.0))    // hardcoded offsets — breaks on resize
            .child("centered?")
    )
```

### Idiomatic: flex centering

```rust
div()
    .flex()
    .items_center()
    .justify_center()
    .w(px(500.0)).h(px(300.0))
    .child(div().child("actually centered"))
```

### Naive: sidebar layout with magic numbers

```rust
div()
    .w(px(1000.0))
    .child(
        div()
            .absolute().left(px(0.0))
            .w(px(240.0)).h_full()
            .child(sidebar())
    )
    .child(
        div()
            .absolute().left(px(240.0)).right(px(0.0))
            .child(main_content())
    )
```

Problems: absolute positioning takes elements out of flow; sidebar and main don't interact naturally; resizing breaks it.

### Idiomatic: flex sidebar

```rust
div()
    .flex()
    .h_full()
    .child(
        div()
            .w(px(240.0))
            .flex_shrink_0()       // don't shrink below 240px
            .child(sidebar())
    )
    .child(
        div()
            .flex_1()              // take all remaining width
            .min_w_0()             // allow shrinking below content size (important!)
            .child(main_content())
    )
```

Teaching point: `.min_w_0()` is a common "gotcha" — by default, flex items have `min-width: auto`, which prevents them from shrinking below their content size. Adding `.min_w_0()` allows them to shrink into the available space.

---

## Building common layouts

### App shell (sidebar + main + statusbar)

```rust
fn app_shell(sidebar: impl IntoElement, main: impl IntoElement, status: impl IntoElement) -> impl IntoElement {
    div()
        .flex()
        .flex_col()
        .size_full()
        .child(
            // Top: toolbar (if any) — omitted for brevity
            // Middle: sidebar + main
            div()
                .flex()
                .flex_1()
                .min_h_0()   // allow flex child to shrink in the column
                .child(
                    div().w(px(220.0)).flex_shrink_0().child(sidebar)
                )
                .child(
                    div().flex_1().min_w_0().child(main)
                )
        )
        .child(
            // Bottom: status bar
            div().h(px(24.0)).flex_shrink_0().child(status)
        )
}
```

### Card grid

No CSS grid — compute a fixed number of columns manually:

```rust
fn card_grid(items: &[Item], columns: usize) -> impl IntoElement {
    let rows: Vec<_> = items.chunks(columns).map(|row| {
        div()
            .flex()
            .gap(px(12.0))
            .children(row.iter().map(|item| {
                div()
                    .flex_1()           // equal-width columns
                    .min_w_0()
                    .p(px(16.0))
                    .rounded(px(8.0))
                    .bg(rgb(0x313244))
                    .child(item.title.clone())
            }))
    }).collect();

    div().flex().flex_col().gap(px(12.0)).children(rows)
}
```

### Modal overlay

```rust
fn modal(content: impl IntoElement) -> impl IntoElement {
    div()
        .absolute()
        .inset_0()              // top:0 right:0 bottom:0 left:0 — fills parent
        .flex()
        .items_center()
        .justify_center()
        .bg(black().opacity(0.5))  // semi-transparent backdrop
        .child(
            div()
                .w(px(480.0))
                .p(px(24.0))
                .bg(rgb(0x313244))
                .rounded(px(12.0))
                .shadow_lg()
                .child(content)
        )
}
```

---

## Applications in systems programming

### Editor layout (how Zed thinks about it)

A code editor UI is nested layout trees:

```
Window
└── App shell (flex col)
    ├── Titlebar (fixed height)
    ├── Workspace (flex row, flex-1)
    │   ├── File tree panel (fixed width, resizable)
    │   ├── Editor group (flex-1)
    │   │   ├── Tab bar (fixed height)
    │   │   └── Editor pane (flex-1, scroll container)
    │   └── Outline panel (optional, fixed width)
    └── Status bar (fixed height)
```

Each level is a Rust struct implementing `Render`. The layout of the whole window is the composition of these structs' element trees.

### Dashboard panels

Data dashboards use a grid-like layout — achieved in GPUI by computing column widths as fractions:

```rust
fn two_column_layout(left: impl IntoElement, right: impl IntoElement) -> impl IntoElement {
    div().flex().gap(px(16.0)).h_full()
        .child(div().flex_1().min_w_0().child(left))
        .child(div().flex_1().min_w_0().child(right))
}

fn three_column_layout(a: impl IntoElement, b: impl IntoElement, c: impl IntoElement) -> impl IntoElement {
    div().flex().gap(px(16.0)).h_full()
        .child(div().flex_1().min_w_0().child(a))
        .child(div().flex_1().min_w_0().child(b))
        .child(div().flex_1().min_w_0().child(c))
}
```

`flex_1()` on each column gives equal width. For unequal columns: `.grow(2.0)` and `.grow(1.0)` give a 2:1 ratio.

---

## Debugging layout

When layout doesn't look right, work through this checklist:

### 1. Check if the parent has a concrete size

A `.size_full()` child requires its parent to have a definite size. If the parent is also `.size_full()`, trace up the tree to find the first element with a concrete px or percent size.

```rust
// Add a temporary colored border to visualize the box
div()
    .border_1().border_color(rgb(0xff0000))   // red outline
    .size_full()
    .child(/* ... */)
```

### 2. Watch out for `min_w_0` / `min_h_0`

A flex item's default minimum size is its content size. If a flex item refuses to shrink, add `.min_w_0()` or `.min_h_0()`.

### 3. `absolute` elements don't participate in flex

Absolute-positioned children are invisible to their flex parent's size computation. A parent with only absolute children will have zero size (unless given a fixed size).

### 4. Check the overflow behavior

By default in GPUI, content that overflows is clipped. If children seem to disappear at the edge, the parent may need `.overflow_visible()` or the child needs to scroll.

### 5. Add `cargo expand` equivalent: print computed sizes

GPUI provides `cx.window_bounds()` and similar APIs to inspect runtime values. Temporarily print sizes in `render` to verify.

---

## Challenges and extensions

1. **Resizable panels**: build a two-panel layout where the divider between panels can be dragged. Track the split ratio in the view state. Update it on mouse drag events.

2. **Overflow + scroll**: wrap a list of 100 items in a scrollable container. Read about `.overflow_y_scroll()` in the GPUI docs.

3. **Layout tree traversal**: conceptually, how does a layout library decide the order in which to measure elements? Why does the bottom-up phase require a full top-down pass first?

4. **Custom layout algorithm**: implement a simple "waterfall" layout (newspaper columns) purely in Rust: given `N` items and `k` columns, distribute items such that column heights are as equal as possible. Render each column as a `flex_col` div.

5. **CSS grid by hand**: implement a 3×3 grid of cells. Each cell has a fixed size. Use nested `.flex().flex_col()` (rows) containing `.flex().flex_row()` (cells in a row). Compare the verbosity to CSS grid.

---

## Common mistakes

1. **`size_full()` without a concrete parent size** — the element collapses to zero.
2. **Forgetting `min_w_0()` or `min_h_0()`** — flex items refuse to shrink below content size.
3. **Using `absolute()` when flex would work** — absolute removes the element from flow, breaking sibling layout.
4. **Margin instead of gap for inter-sibling spacing** — trailing margin adds unwanted space; `gap` doesn't.
5. **Hardcoding pixel offsets for centering** — use `items_center` + `justify_center` instead.
6. **Applying `h_full()` without matching parent height** — if no parent in the chain has a definite height, `h_full()` has no effect.

---

## Exercises

1. Build a two-panel layout: a 200px sidebar on the left, the main area filling the rest. Both panels should fill the window height. Add a 1px divider between them.
2. Build a card grid: 3 equal-width cards per row, with 16px gaps. Each card has a title and a subtitle.
3. Center a `400×300` dialog box in the middle of a `800×600` window. Add a semi-transparent overlay behind it.
4. (**harder**) Build an app shell: titlebar (fixed 40px), sidebar (fixed 220px, scrollable content), main area (fills remaining space), status bar (fixed 24px). The sidebar and main should fill the space between titlebar and status bar.

---

## References

- [GPUI layout primitives — `div()` docs](https://docs.rs/gpui/latest/gpui/struct.Div.html)
- [Taffy layout engine](https://github.com/DioxusLabs/taffy) — the flexbox/grid layout library used by several Rust GUI frameworks (including GPUI internally)
- [CSS box model reference](https://developer.mozilla.org/en-US/docs/Learn/CSS/Building_blocks/The_box_model) — the mental model carries over; CSS syntax doesn't
- [Yoga layout engine](https://www.yogalayout.dev/) — Facebook's flexbox engine; similar algorithms, good conceptual docs

## Recap

- Layout in Rust GUI is constraint propagation on a tree: available space flows down, measured sizes flow up.
- The box model (content, padding, border, margin) exists; GPUI uses border-box sizing.
- Flex layout covers most arrangements: `flex_row`/`flex_col`, `justify_*`, `items_*`, `gap`, `flex_1`.
- There is no CSS grid — achieve multi-column layouts with nested flex containers or computed column widths.
- `min_w_0()` / `min_h_0()` are the most common fixes for "why won't this shrink?" bugs.
- Responsive behavior is a Rust `if` expression inside `render`, not a media query.
