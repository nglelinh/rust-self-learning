---
layout: post
title: "07-36 App Architecture — Events, State, and Rendering"
chapter: "07"
order: 36
owner: "OpenCode"
lang: en
categories:
  - chapter07
lesson_type: required
---

This lesson ties OS concepts, networking, and GPUI into a single coherent architecture. The central insight: every desktop application — no matter how complex — is a pipeline of events flowing into state updates flowing into UI renders. Once you see this pattern clearly, designing a chat application becomes a series of small, manageable decisions rather than a single overwhelming problem.

## 60-minute teaching plan

- 0 to 10 min: The unified architecture — one diagram that explains everything.
- 10 to 25 min: The event system — OS events and network events treated uniformly.
- 25 to 40 min: State management — single source of truth, GPUI Model entities.
- 40 to 50 min: Async with Tokio — spawning tasks, channels, backpressure.
- 50 to 60 min: Practical checklist and mindset shift; recap.

## Learning goals

By the end of this lesson, students can:

- Explain the event → state → render pipeline and trace a message through it.
- Design the state structure for a chat application using GPUI's `Model<T>`.
- Use `tokio::sync::mpsc` channels to pass events from background tasks to the UI.
- Apply the practical OS and networking checklist to their own projects.
- Articulate the mindset shift from browser development to Rust desktop development.

---

## The unified architecture

Every event your application responds to — a keypress, a network message, a timer tick — travels the same path:

```
┌──────────────────┐    ┌──────────────────┐
│   OS Events      │    │  Network Events  │
│                  │    │                  │
│ • Mouse click    │    │ • Message recv'd │
│ • Key press      │    │ • Connected      │
│ • Window resize  │    │ • Disconnected   │
│ • Timer tick     │    │ • Error          │
└────────┬─────────┘    └────────┬─────────┘
         │                       │
         └───────────┬───────────┘
                     │
                     ▼
           ┌─────────────────┐
           │   Event System  │
           │ (GPUI + Tokio)  │
           └────────┬────────┘
                    │
                    ▼
           ┌─────────────────┐
           │   App State     │
           │  (Model<T>)     │
           └────────┬────────┘
                    │
                    ▼
           ┌─────────────────┐
           │       UI        │
           │  (render fn)    │
           └─────────────────┘
```

This diagram is not abstract theory — it maps directly to the code you write:

- **OS Events** → GPUI's input handlers (`on_click`, `on_key_down`)
- **Network Events** → Tokio tasks that read from the WebSocket and send to an mpsc channel
- **Event System** → `cx.update_view` / `cx.notify()` wiring GPUI to Tokio
- **App State** → `Model<ChatState>` holding the message list, connection status, input buffer
- **UI** → `impl Render for ChatView`

Everything flows in one direction. The UI never reaches back to mutate state directly — it dispatches events. State never pushes to the UI directly — it notifies GPUI, which pulls from state by calling `render`.

---

## The event system

### OS events: GPUI handles these

GPUI converts raw OS events (mouse moves, key strokes, system notifications) into typed Rust events and delivers them to your registered handlers. You write handlers; GPUI delivers events.

```rust
div()
    .on_click(cx.listener(|this, event: &ClickEvent, cx| {
        this.handle_send(cx);
    }))
    .on_key_down(cx.listener(|this, event: &KeyDownEvent, cx| {
        if event.keystroke.key == "enter" {
            this.handle_send(cx);
        }
    }))
```

### Network events: you bridge these yourself

GPUI doesn't know about your WebSocket connection. You run the network code in a Tokio task, then bridge events back to GPUI through a channel:

```rust
// The bridge: an mpsc channel from the network task to the UI
#[derive(Debug)]
pub enum AppEvent {
    MessageReceived(ServerMessage),
    Connected,
    Disconnected { reason: String },
    Reconnecting { attempt: u32 },
}
```

The network task sends events; the GPUI view receives them:

```rust
// Network task — runs on Tokio's thread pool
cx.spawn(|view, mut cx| async move {
    let (tx, mut rx_for_view) = tokio::sync::mpsc::unbounded_channel::<AppEvent>();

    // ... connect and run WebSocket read loop, sending to tx ...

    // On the other side: GPUI view receives events
    while let Some(event) = rx_for_view.recv().await {
        cx.update_view(&view, |this, cx| {
            this.handle_app_event(event, cx);
        }).ok();
    }
}).detach();
```

### The event handler pattern

Keep handlers small. Each handler does three things: validate, update state, notify:

```rust
fn handle_app_event(&mut self, event: AppEvent, cx: &mut ViewContext<Self>) {
    match event {
        AppEvent::MessageReceived(msg) => {
            self.state.update(cx, |state, cx| {
                state.messages.push(msg);
                if state.messages.len() > MAX_HISTORY {
                    state.messages.remove(0);
                }
                cx.notify();
            });
        }
        AppEvent::Connected => {
            self.connection_status = ConnectionStatus::Connected;
            cx.notify();
        }
        AppEvent::Disconnected { reason } => {
            self.connection_status = ConnectionStatus::Disconnected;
            eprintln!("disconnected: {reason}");
            cx.notify();
        }
        AppEvent::Reconnecting { attempt } => {
            self.connection_status = ConnectionStatus::Reconnecting { attempt };
            cx.notify();
        }
    }
}
```

No network calls, no file I/O, no sleeps — just state mutations and a `cx.notify()`. The UI will re-render itself.

---

## State management

### The single source of truth principle

**All state lives in one place.** Your UI derives its appearance from state; it does not maintain its own copy of the data.

Naive: UI widgets store their own copies of messages. When a new message arrives, you must update every widget. Bugs happen when copies diverge.

Idiomatic: one `Model<ChatState>` holds everything. Views subscribe to it. When it changes, subscribed views re-render automatically.

### Designing the state structure

```rust
#[derive(Debug, Clone)]
pub struct Message {
    pub id:        String,
    pub from:      String,
    pub content:   String,
    pub ts:        chrono::DateTime<chrono::Utc>,
    pub status:    MessageStatus,
}

#[derive(Debug, Clone, PartialEq)]
pub enum MessageStatus {
    Pending,    // sent by us, waiting for server confirmation
    Confirmed,  // server echoed it back with an ID
    Failed,     // server returned an error
}

#[derive(Debug, Clone, PartialEq)]
pub enum ConnectionStatus {
    Disconnected,
    Connecting,
    Connected,
    Reconnecting { attempt: u32 },
}

pub struct ChatState {
    pub messages:          Vec<Message>,
    pub connection_status: ConnectionStatus,
    pub current_room:      String,
    pub current_user:      String,
    pub unread_count:      usize,
}
```

### Model entities in GPUI

```rust
// Create the model in app setup
let state: Model<ChatState> = cx.new_model(|_cx| ChatState {
    messages: Vec::new(),
    connection_status: ConnectionStatus::Disconnected,
    current_room: "general".into(),
    current_user: String::new(),
    unread_count: 0,
});

// The view holds a handle and subscribes
struct ChatView {
    state: Model<ChatState>,
}

impl ChatView {
    fn new(state: Model<ChatState>, cx: &mut ViewContext<Self>) -> Self {
        // Re-render this view whenever the state model changes
        cx.observe(&state, |_, _, cx| cx.notify()).detach();
        Self { state }
    }
}
```

### Reading state in render

```rust
impl Render for ChatView {
    fn render(&mut self, cx: &mut ViewContext<Self>) -> impl IntoElement {
        let state = self.state.read(cx);   // immutable borrow

        div()
            .flex()
            .flex_col()
            .size_full()
            .child(self.render_header(&state))
            .child(self.render_message_list(&state))
            .child(self.render_input_bar(&state, cx))
    }
}
```

The `render` method reads state but never writes it. Writing happens only through events.

### What NOT to put in Model state

| Put in Model | Don't put in Model |
|--------------|-------------------|
| Business data (messages, users) | Ephemeral layout state |
| Connection status | Animation frame state |
| Unread counts | Whether a dropdown is open* |
| Configuration | Scroll position* |

*These can live in the view struct itself as UI-local state that doesn't need to survive a re-render.

---

## Async with Tokio: wiring it together

### The bridge pattern

The canonical pattern for connecting Tokio tasks to GPUI views:

```rust
// 1. Create a channel
let (tx, rx) = tokio::sync::mpsc::unbounded_channel::<AppEvent>();

// 2. Spawn the network task — it sends events
let tx_clone = tx.clone();
cx.spawn(|_view, _cx| async move {
    run_network_loop(tx_clone).await;
}).detach();

// 3. Spawn a receiver task — it relays events to the view
cx.spawn(|view, mut cx| async move {
    let mut rx = rx;
    while let Some(event) = rx.recv().await {
        cx.update_view(&view, |this, cx| {
            this.handle_app_event(event, cx);
        }).ok();
    }
}).detach();
```

### The full network loop

```rust
async fn run_network_loop(tx: Sender<AppEvent>) {
    let mut retry_delay = Duration::from_millis(500);

    loop {
        tx.send(AppEvent::Connecting).ok();

        match connect_async("wss://chat.example.com/ws").await {
            Ok((ws_stream, _)) => {
                retry_delay = Duration::from_millis(500); // reset on success
                tx.send(AppEvent::Connected).ok();

                if let Err(e) = run_ws_session(ws_stream, tx.clone()).await {
                    tx.send(AppEvent::Disconnected { reason: e.to_string() }).ok();
                }
            }
            Err(e) => {
                tx.send(AppEvent::Disconnected { reason: e.to_string() }).ok();
            }
        }

        tokio::time::sleep(retry_delay).await;
        retry_delay = (retry_delay * 2).min(Duration::from_secs(30));
    }
}

async fn run_ws_session(
    ws_stream: WsStream,
    tx: Sender<AppEvent>,
) -> anyhow::Result<()> {
    let (mut write, mut read) = ws_stream.split();

    while let Some(msg) = read.next().await {
        match msg? {
            Message::Text(text) => {
                let server_msg: ServerMessage = serde_json::from_str(&text)?;
                tx.send(AppEvent::MessageReceived(server_msg)).ok();
            }
            Message::Close(_) => break,
            _ => {}
        }
    }
    Ok(())
}
```

### Sending outgoing messages

The view needs a way to enqueue outgoing messages. Use a second channel:

```rust
pub struct ChatView {
    state:  Model<ChatState>,
    outbox: tokio::sync::mpsc::UnboundedSender<ClientMessage>,
}

impl ChatView {
    fn send_message(&mut self, content: String, cx: &mut ViewContext<Self>) {
        // Update local state optimistically (pending)
        self.state.update(cx, |s, cx| {
            s.messages.push(Message {
                id:      uuid::Uuid::new_v4().to_string(),
                from:    s.current_user.clone(),
                content: content.clone(),
                ts:      chrono::Utc::now(),
                status:  MessageStatus::Pending,
            });
            cx.notify();
        });

        // Send through the outbox channel to the network task
        let msg = ClientMessage::Send {
            room:    self.state.read(cx).current_room.clone(),
            content,
        };
        self.outbox.send(msg).ok();
    }
}
```

The network task reads from the outbox:

```rust
async fn run_ws_session_with_outbox(
    ws_stream: WsStream,
    tx:      Sender<AppEvent>,
    mut outbox: tokio::sync::mpsc::UnboundedReceiver<ClientMessage>,
) -> anyhow::Result<()> {
    let (mut write, mut read) = ws_stream.split();

    loop {
        tokio::select! {
            Some(msg) = read.next() => {
                match msg? {
                    Message::Text(t) => {
                        let server_msg = serde_json::from_str(&t)?;
                        tx.send(AppEvent::MessageReceived(server_msg)).ok();
                    }
                    Message::Close(_) => break,
                    _ => {}
                }
            }
            Some(client_msg) = outbox.recv() => {
                let text = serde_json::to_string(&client_msg)?;
                write.send(Message::Text(text)).await?;
            }
        }
    }
    Ok(())
}
```

Two channels. One for app events flowing into the view, one for client messages flowing out to the server. Clean separation of concerns.

---

## Practical checklist

Work through this before writing your chat app. If any item is unchecked, you have a gap that will cause problems later.

### OS side

- [ ] **Background tasks don't block the UI**: all network and file I/O runs inside `cx.spawn` async tasks.
- [ ] **File paths are platform-correct**: using `dirs::config_dir()` / `dirs::data_local_dir()`, not hardcoded paths.
- [ ] **File writes are atomic**: temp file + rename pattern.
- [ ] **Timer granularity is acceptable**: heartbeats are 15–60 seconds, not milliseconds.

### Networking side

- [ ] **WebSocket connection with retry**: exponential backoff, capped at 30–60 seconds.
- [ ] **Heartbeat implemented**: ping/pong every 15–30 seconds; disconnect if no pong within 3× the interval.
- [ ] **Incoming messages are parsed defensively**: `serde_json::from_str` errors are logged, not panicked.
- [ ] **Outgoing messages are queued**: outbox channel is unbounded (or bounded with backpressure), never dropped.
- [ ] **Disconnection is visible in the UI**: connection status in `ChatState` drives a status indicator.
- [ ] **Graceful shutdown**: send Close frame on quit, drain for 3 seconds.

### State side

- [ ] **Single source of truth**: all business data in `Model<ChatState>`.
- [ ] **Optimistic UI for sends**: show message as Pending immediately, update to Confirmed when server echoes.
- [ ] **Bounded message history**: don't grow `Vec<Message>` unboundedly; cap at 500–1000 and trim.

---

## The mindset shift

### From browser development

In a browser, the platform manages almost everything for you:

- The browser's event loop handles OS input.
- The browser's network stack handles HTTP and WebSocket.
- The DOM handles rendering.
- The JavaScript runtime handles concurrency (single-threaded with a task queue).

You write event handlers. The browser wires everything else.

### To Rust + GPUI

In a Rust desktop app, you control all of this explicitly:

| Concern | Browser | Rust + GPUI |
|---------|---------|-------------|
| Event loop | Browser provides | GPUI provides; you hook in |
| Network | Browser manages | You manage with Tokio |
| Rendering | DOM + browser engine | GPUI renders to GPU; you define the tree |
| Concurrency | JS single-thread + microtasks | Rust threads + Tokio async tasks |
| Memory | Garbage collector | Rust ownership + GPUI entity system |

This is more work upfront. The reward: your app starts in milliseconds (not seconds), uses 10–50MB of RAM (not 200–500MB), and has no garbage collector pauses during animations or typing.

### The three mental models to internalize

**1. Everything is an event.**

A user typing, a message arriving, a timer firing — these are all events. Your job is to define how each event transitions state. If you can describe every event and its state transition, you can implement it.

**2. State lives in one place.**

Don't scatter state across views and async tasks. Put everything in `Model<ChatState>`. Views read from it and re-render when it changes. Background tasks update it through channels and `cx.update_view`.

**3. The UI thread is sacred.**

Any blocking on the UI thread — a lock, a file read, a network call — freezes the entire application. The rule is absolute. When in doubt, `cx.spawn`.

---

## Putting it all together: the minimal chat app skeleton

```rust
use gpui::*;
use tokio::sync::mpsc;

// ── State ──────────────────────────────────────────────────────────────────

pub struct ChatState {
    pub messages:   Vec<ChatMessage>,
    pub connected:  bool,
    pub input:      String,
}

pub struct ChatMessage {
    pub from:    String,
    pub content: String,
}

// ── Events ─────────────────────────────────────────────────────────────────

pub enum AppEvent {
    MessageReceived { from: String, content: String },
    Connected,
    Disconnected,
}

// ── View ───────────────────────────────────────────────────────────────────

pub struct ChatView {
    state:  Model<ChatState>,
    outbox: mpsc::UnboundedSender<String>,
}

impl ChatView {
    fn new(
        state: Model<ChatState>,
        outbox: mpsc::UnboundedSender<String>,
        cx: &mut ViewContext<Self>,
    ) -> Self {
        cx.observe(&state, |_, _, cx| cx.notify()).detach();
        Self { state, outbox }
    }

    fn send(&mut self, cx: &mut ViewContext<Self>) {
        let content = {
            let mut s = self.state.update(cx, |s, cx| {
                let content = s.input.clone();
                s.input.clear();
                cx.notify();
                content
            });
            s
        };
        if !content.is_empty() {
            self.outbox.send(content).ok();
        }
    }
}

impl Render for ChatView {
    fn render(&mut self, cx: &mut ViewContext<Self>) -> impl IntoElement {
        let state = self.state.read(cx);

        div()
            .flex()
            .flex_col()
            .size_full()
            .bg(rgb(0x1e1e2e))
            .child(
                // Status bar
                div()
                    .px(px(12.0))
                    .py(px(6.0))
                    .text_color(if state.connected {
                        rgb(0xa6e3a1)
                    } else {
                        rgb(0xf38ba8)
                    })
                    .child(if state.connected { "● Connected" } else { "○ Disconnected" })
            )
            .child(
                // Message list
                div()
                    .flex_1()
                    .overflow_hidden()
                    .p(px(12.0))
                    .children(state.messages.iter().map(|msg| {
                        div()
                            .text_color(rgb(0xcdd6f4))
                            .child(format!("{}: {}", msg.from, msg.content))
                    }))
            )
    }
}

// ── App entry point ────────────────────────────────────────────────────────

fn main() {
    App::new().run(|cx: &mut AppContext| {
        let (event_tx, event_rx) = mpsc::unbounded_channel::<AppEvent>();
        let (outbox_tx, outbox_rx) = mpsc::unbounded_channel::<String>();

        let state = cx.new_model(|_| ChatState {
            messages:  Vec::new(),
            connected: false,
            input:     String::new(),
        });

        cx.open_window(WindowOptions::default(), |cx| {
            let view = cx.new_view(|cx| ChatView::new(state.clone(), outbox_tx, cx));

            // Event relay task: network events → view
            let state_for_task = state.clone();
            cx.spawn(|mut cx| async move {
                let mut rx = event_rx;
                while let Some(event) = rx.recv().await {
                    cx.update_model(&state_for_task, |s, cx| {
                        match event {
                            AppEvent::MessageReceived { from, content } => {
                                s.messages.push(ChatMessage { from, content });
                            }
                            AppEvent::Connected    => s.connected = true,
                            AppEvent::Disconnected => s.connected = false,
                        }
                        cx.notify();
                    }).ok();
                }
            }).detach();

            // Network task — replace with real WebSocket logic
            cx.spawn(|_cx| async move {
                let _ = (event_tx, outbox_rx); // wire up real networking here
            }).detach();

            view
        }).unwrap();
    });
}
```

This skeleton compiles and runs. Replace the stub network task with the real WebSocket loop from lesson 35, and add render details from lesson 32–33. The architecture is in place.

---

## In-class exercises

1. Run the skeleton above. Confirm the window opens with the status bar showing "Disconnected".

2. Add a `tokio::time::interval` that fires every 3 seconds and pushes a fake `AppEvent::MessageReceived { from: "bot".into(), content: format!("tick {n}") }`. Watch the message list grow.

3. After exercise 2, add a bound: if `messages.len() > 10`, remove the oldest. Observe the list stays at 10.

4. Wire the status bar: after 5 seconds, transition to `AppEvent::Connected`. After 15 seconds, transition back to `AppEvent::Disconnected`. Confirm the color changes.

5. Add an input bar: a `div` at the bottom with `on_key_down`. When Enter is pressed, call `this.send(cx)` and echo the message into the list immediately as a Pending message.

---

## Recap

- Every desktop app is a pipeline: events → state → render. Make this explicit in your architecture.
- OS events come through GPUI handlers. Network events come through `tokio::sync::mpsc` channels bridged to GPUI via `cx.update_model` / `cx.update_view`.
- All business state lives in one `Model<ChatState>`. Views subscribe and re-render. Background tasks update state through channels.
- Use two channels: one for inbound events (network → view) and one for outbound messages (view → network).
- The UI thread is sacred: never block it. Everything slow goes in `cx.spawn`.
- The mindset shift from browser: you control threads, events, state, and rendering explicitly. This is more work and far more power.
