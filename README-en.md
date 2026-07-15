# st-push

A Streamlit real-time push component — mounts a WebSocket route onto Streamlit's Starlette App for true server-to-browser push, with per-connection AES-256-GCM encryption and module routing so multiple components can coexist on one connection.

## Background

Streamlit's native rerun model is not well suited for high-frequency real-time data. st-push mounts a WebSocket route onto Streamlit's Starlette App to establish an encrypted long-lived connection between the server and the browser, enabling *true* push — the frontend handles messages via callbacks directly and does **not** trigger a rerun.

## Features

- **True push**: WebSocket long-lived connection; the frontend dispatches messages via callbacks without triggering reruns
- **Encrypted**: each connection gets its own AES-256-GCM key, with an automatic XOR+HMAC fallback when the `cryptography` package is unavailable
- **Module routing**: multiple components share one connection; messages are dispatched by `moduleId`
- **Multi-device login**: a user maps to many channels (`user → [channel_id]`, 1:N); messages are pushed to all of a user's devices
- **Multi-device send sync**: when sending a message, the message is also echoed to the sender's other channels; the frontend deduplicates
- **PushClient session singleton**: a shared infrastructure created once by the application layer and injected into all components — one session, one WS connection

## Architecture

The project is split into three decoupled layers. A key constraint is that the transport layer knows nothing about users — it only deals with channels. PUSH works even without any user-channel mapping.

| Layer | Module | Responsibility |
|------|------|------|
| Transport | `push_core.py` / `routes.py` / `crypto.py` | `channel → conn` (1:1), encryption, heartbeat, module routing |
| Business | `user_channel_manage.py` | `user → [channel_id]` (1:N) multi-device login mapping |
| Access | `__init__.py` | `PushClient` (session singleton) + `Module` |

## Installation

Not published to PyPI. Build from source:

```bash
pip install st_push-1.1-py3-none-any.whl
```

Requirements: Python 3.8+, Streamlit >= 1.59.1, starlette >= 0.40.0. The `cryptography` package is optional (used for AES-GCM; falls back to XOR+HMAC otherwise).

## Quick start

### 1. Launcher: mount the WebSocket route

st-push does not start a separate port or service. You mount its routes onto Streamlit's Starlette App.

```python
# app.py
from streamlit.web.server.starlette import App
import st_push

app = App("script.py", routes=st_push.create_push_routes())
if __name__ == "__main__":
    app.run()
```

### 2. Script: create a PushClient and use components

```python
import streamlit as st
from st_push import PushClient
from st_push.comp.chat import ChatClient
from st_push.comp.notify import NotificationClient

# Application layer creates a shared PushClient (session singleton)
push_client = PushClient.session_instance(user="alice")

# Inject it into components — they share the same push_client
chat = ChatClient(push_client, user_name="alice")
notifier = NotificationClient(push_client, user_name="alice")

# Render and send
chat.render(peer="bob")
notifier.render()
chat.send("Hello!", to_user="bob")
notifier.send_notification(to_user="bob", title="New task", body="Fix the login bug")
```

### 3. Run

```bash
python app.py
```

## API — Access layer

### PushClient

| Method | Description |
|------|------|
| `PushClient.session_instance(user=None, key=...) -> PushClient` | **Recommended**: get a session-level singleton, creating it if needed. Shared by all components. |
| `PushClient(user=None, key=...)` | Direct constructor (usually replaced by `session_instance`) |
| `bind_user(user)` | Bind/rebind a user (also syncs `query_params`) |
| `get_user() -> str \| None` | Current bound user |
| `connect() -> Any` | Render the invisible WS transport component (called on every rerun by `session_instance`) |
| `register_module(name) -> Module` | Register a module (idempotent, session-cached + globally deduped by PushCore) |
| `send_msg(module, payload, target_user=None) -> bool` | Send a module message to all of a user's channels |
| `broadcast_msg(module, payload) -> bool` | Broadcast to all online channels |
| `is_online(user) -> bool` | Whether the user has any online channel |

### Module

| Method | Description |
|------|------|
| `Module.send_msg(payload, target_user=None) -> bool` | Delegate to PushClient |
| `Module.broadcast_msg(payload) -> bool` | Delegate to PushClient |

### register (convenience)

| Method | Description |
|------|------|
| `register(name) -> Module` | Register a module using the session's default PushClient |

## API — Transport layer (PushCore singleton)

| Method | Description |
|------|------|
| `PushCore.instance()` | Get the singleton |
| `register_channel(conn, channel_id)` | Register a connection (1:1; the old connection is replaced) |
| `unregister(conn)` | Remove a connection |
| `register_module(name) -> ModuleHandle` | Register a module (idempotent, deduped by name) |
| `send_msg(frame) -> bool` | Enqueue a message to `targetChannelId` (fire-and-forget) |
| `broadcast(frame) -> bool` | Broadcast to all online channels |
| `is_online(channel_id) -> bool` | Whether the channel is online |

## API — Business layer (UserChannelManage singleton)

| Method | Description |
|------|------|
| `UserChannelManage.instance()` | Get the singleton |
| `bind(user, channel_id)` | Bind user → channel (multi-device friendly, 1:N) |
| `unbind(channel_id)` | Remove a channel binding |
| `get_channels(user) -> list[str]` | All channels of a user |
| `is_online(user) -> bool` | Whether the user has any online channel |
| `get_users() -> list[str]` | All bound users |
| `cleanup_stale(channel_id)` | Remove a channel during heartbeat cleanup (called internally by PushCore) |

## Core rules

- PUSH = **online-only** push. Offline returns `False`; nothing is buffered. Offline data is fetched by the client via GET.
- **PushClient is shared infrastructure**: created once by the application layer via `session_instance`, injected into components; one session = one WS connection.
- The transport layer is user-agnostic: PushCore only knows channels.
- The business layer serves PushClient only: `UserChannelManage` is not exposed externally.
- Multi-device login: one user may have multiple channels (1:N); one channel belongs to exactly one user.
- Multi-device send sync: `send_msg` does **not** exclude `srcChannelId` — all target channels receive it; the frontend deduplicates by `(fromUser, toUser, content, ts)`.
- Encryption handshake: send a plaintext `ack` (with the key) first, then set `encryption_enabled = True`.
- Module messages do **not** go through `setComponentValue`; the frontend callback handles them directly (avoids needless reruns).
- Thread safety: `loop.call_soon_threadsafe()` for non-blocking cross-thread scheduling (fire-and-forget).
- `declare_component` is lazy-loaded via the `_get_component_func()` pattern to avoid errors when no `ScriptRunContext` is present.
- User identity is persisted via `st.query_params["st_push_user"]` — restored after refresh, independent of `session_id`.
- No npm build step: the frontend `index.html` inlines the Streamlit component protocol.

## Files

| File | Description |
|------|------|
| `__init__.py` | PushClient + Module + register + exports |
| `push_core.py` | PushCore singleton — `channel→conn` (1:1) + per-conn queue + sender task + module registration |
| `user_channel_manage.py` | UserChannelManage singleton — `user→[channel_id]` (1:N) business layer |
| `routes.py` | `/st_push/ws` route + `/st_push/log` log endpoint (type-driven dispatch) |
| `crypto.py` | AES-256-GCM + XOR/HMAC fallback |
| `frontend/build/index.html` | Inlined protocol + WS client + encrypt/decrypt + module dispatch (v3.0 frame format) |
| `comp/chat/` | Optional chat component (ChatClient + chat window UI) |
| `comp/notify/` | Optional notification component (NotificationClient + bell + popup UI) |
## License

MIT
