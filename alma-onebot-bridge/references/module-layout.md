# Module Layout and Cargo.toml Reference

Complete source tree and dependency manifest for `alma-onebot-bridge`. Read this when adding a new module, validating that a refactor matches the current shape, or upgrading dependencies.

## Cargo.toml

The bridge targets Rust edition 2024 (Rust 1.85+) and AGPL-3.0-only.

```toml
[package]
name = "alma-onebot-bridge"
version = "0.2.1"
edition = "2024"
license = "AGPL-3.0-only"

[dependencies]
async-broadcast = "0.7.2"
async-tungstenite = { version = "0.34.1", default-features = false, features = ["smol-runtime"] }
base64 = "0.22.1"
dirs = "6.0.0"
futures-lite = "2.6.1"
futures-util = "0.3.32"
serde = { version = "1.0.228", features = ["derive"] }
serde_json = "1.0.150"
signal-hook = "0.4.4"
smol = "2.0.2"
trillium = "1.3.0"
trillium-client = { version = "0.9.7", features = ["serde_json"] }
trillium-router = "0.5.2"
trillium-rustls = { version = "0.11.3", default-features = false, features = ["client", "platform-verifier", "aws-lc-rs", "tls12"] }
trillium-smol = "0.7.0"
trillium-websockets = "0.8.2"
tracing = "0.1.44"
tracing-appender = "0.2.5"
tracing-subscriber = { version = "0.3.23", features = ["env-filter"] }
uuid = { version = "1.23.3", features = ["v4"] }
turso = { version = "0.7.0-pre.10", default-features = false }
time = { version = "0.3", features = ["formatting", "local-offset", "parsing"] }
toml = "0.8"
url = "2.5.8"

[dev-dependencies]
macro_rules_attribute = "0.2.2"
smol-macros = "0.1.1"
```

Locked-out crates: `tokio`, `warp`, `reqwest`, `tokio-tungstenite`, `tokio-stream`, `async-h1`, `http-types`. The bridge ships a self-contained smol + Trillium stack; reintroducing any Tokio-rooted networking crate would split the runtime and corrupt the existing scheduling assumptions.

`async-tungstenite` is used **only** for the outbound Alma WebSocket client; the OneBot-facing server uses `trillium-websockets`. Do not mix the two `Message` types.

## Source Tree

```
src/
├── main.rs              entry point; smol::block_on(async_main())
├── lib.rs               bridge bootstrap, tracing init, debugger mode,
│                         PID file management, SIGHUP hot-reload (Unix),
│                         log rotation, shutdown signals
├── config.rs            TOML config file loading, env overrides, defaults
├── state.rs             SharedState with Turso persistence, alma_event_tx
│                         broadcast channel, GroupDirectoryEntry/GroupMember
├── auth.rs              token + loopback authorization for WS and HTTP
│                         command endpoints (Bearer / Token / bare token)
├── server.rs            Trillium router + HTTP endpoints + OneBot WS upgrade;
│                         this is the route composition layer only
├── handlers/
│   ├── mod.rs           re-exports http + ws
│   ├── http.rs          GET /health, GET /qq/groups,
│                         POST /qq/group/<id>/send, POST /qq/private/<id>/send
│   └── ws.rs            reverse WS connection lifecycle, event dispatch,
│                         WebSocket writer task that owns WebSocketConn
├── onebot/
│   ├── mod.rs           re-exports api + event; send_text_message,
│                         send_reply_message conveniences
│   ├── event.rs         OneBot v11 serde types, text extraction,
│                         @bot detection, reply segment extraction,
│                         media tagging, forward message parsing
│   └── api.rs           echo-correlated WS API calls; PendingCalls map;
│                         call_api with timeout; ApiResponse handling
├── face_map.rs          QQ face expression ID ↔ Chinese name lookup
│                         (~180 IDs covering common "super expressions")
├── group_log.rs         ~/.config/alma/groups/<chat_id>_<date>.log writer
│                         (matches Alma's built-in group log format);
│                         README.md alma-onebot-bridge marked section
├── alma.rs              Alma REST API: create_thread, fetch_default_model
├── alma_ws.rs           Alma WS client: generate_response protocol,
│                         message_delta accumulation, tool-call boundary
│                         handling, per-thread generation guards
├── pipeline.rs          end-to-end message processing, message formatting
│                         ([From: ...] / [msg:N] / reply context),
│                         bidirectional forwarding, send dedup, length chunking
└── people.rs            ~/.config/alma/people/<qq_id>.md auto-creation;
                          frontmatter writes telegram_id + qq_id + username;
                          per-group card 群名片 list
```

## Runtime Bootstrap Highlights

These behaviours live in `lib.rs` and are easy to overlook:

- **Preflight port bind**. `TcpListener::bind` is called before `SharedState` initialization so port-in-use returns a clean error.
- **Turso state init**. `SharedState::new()` opens the local SQLite-compatible DB; if another bridge holds it, the message points users at `database.path`.
- **PID file**. `~/.config/alma/bridge/bridge.pid` exists for the desktop GUI to discover a running headless bridge. The panic hook removes it on crash.
- **Tracing + rotation**. `BRIDGE_LOG_FILE` env (set by the desktop apps) routes logs to a file with 10 MB rotation × 3 backups. Without it, tracing prints to stderr.
- **SIGHUP hot-reload** (Unix only). Re-reads the TOML config and updates the shared `Config` in place. If `alma.api` changed, the Alma WS client is dropped so the next generation reconnects.
- **Debugger mode** (`--debugger`). Uses a per-process temporary `bridge-state.db` and picks the first available port starting from 18090, so an IDE-launched debug build does not fight a running headless bridge.

## Shared State Surface (`state.rs`)

`SharedState` is `Clone` (Arc internally). It exposes:

- `config: Arc<RwLock<Config>>`
- `alma_event_tx: async_broadcast::Sender<AlmaEvent>` — fan-out for Alma WS events
- `set_alma_ws / get_alma_ws / clear_alma_ws` — lifecycle of the outbound client
- `get_onebot_api_handle / has_onebot_api_handle` — only set while a reverse WS is connected
- `register_sent_reply / has_recently_sent` — bidirectional dedup buffer keyed by thread
- `get_thread_id / set_thread_id` — `private:{user_id}` / `group:{group_id}` ↔ Alma thread mapping
- `group_directory_snapshot / observe_group_member / set_group_title` — feeds `~/.config/alma/groups/README.md` and the `/qq/groups` HTTP endpoint
- `set_default_model / default_model` — caches `chat.defaultModel` from Alma settings
