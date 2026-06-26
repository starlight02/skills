# Configuration Reference

Bridge configuration lives in TOML. Environment variables are reserved for runtime metadata such as log paths and the debugger flag. This document is the source of truth for every TOML key; the desktop apps write the same schema.

## Load Order

The bridge resolves a single `Config` by trying these locations in order. The first existing file wins; the rest are ignored, not merged.

1. `./config.toml` (current directory)
2. `./bridge.toml` (current directory; legacy filename)
3. `~/.config/alma/bridge/config.toml` (used by the macOS menu bar app and Windows tray app)
4. Compiled defaults

When no file is found, the defaults below are used as if they were written verbatim.

## Full Key Reference

### `[bridge]`

| Key | Default | Purpose |
|---|---|---|
| `port` | `8090` | WS + HTTP listen port. 8080 is intentionally avoided because it collides with Docker / nginx-ui on many developer machines. |

### `[alma]`

| Key | Default | Purpose |
|---|---|---|
| `api` | `http://localhost:23001` | Alma REST and WS base URL. Changed via SIGHUP triggers an immediate Alma WS reconnect. |
| `model` | *(none — uses Alma settings)* | Override AI model id, e.g. `anthropic:claude-sonnet-4-20250514`. |
| `timeout` | `120` | Generation **idle** timeout in seconds. The window resets after each emitted stage/tool-call progress event so a long-running tool does not force a premature retry. |
| `max_retries` | `2` | Maximum retry attempts on generation failure. `0` disables retries. |
| `retry_delay_ms` | `3000` | Base delay between retries in ms (exponential backoff). |

### `[database]`

| Key | Default | Purpose |
|---|---|---|
| `path` | `bridge-state.db` | Local Turso DB file. Use a per-process path under `--debugger` to avoid file locks. |

### `[people]`

| Key | Default | Purpose |
|---|---|---|
| `dir` | `~/.config/alma/people` | People Profile root. Each `<qq_id>.md` carries `telegram_id` + `qq_id` + `username`. |

### `[onebot]`

| Key | Default | Purpose |
|---|---|---|
| `api_timeout` | `30` | Timeout in seconds for OneBot WS API calls (echo-correlated). |
| `access_token` | *(none)* | Bearer token required for incoming reverse WS connections and for **non-loopback** HTTP command endpoints. Loopback callers (127.0.0.1 / ::1) bypass token requirements so Alma tools can call the endpoint locally without leaking the token into prompts. |

### `[chat]`

| Key | Default | Purpose |
|---|---|---|
| `group_history_size` | `30` | Recent group messages provided as context. `0` disables the buffer. |
| `thinking_message` | *(none)* | Optional message sent before Alma starts generating, e.g. `"思考中..."`. |
| `show_thinking` | `false` | When true, `<think>...</think>` blocks are emitted as separate QQ messages instead of being stripped. |
| `show_tool_calls` | `false` | When true, the bridge emits `正在调用工具：<tool>\n参数：<compact JSON or 无>` after Alma streams the tool input. Status messages are progress notifications; they never become final-reply prefixes. |
| `segmented_replies` | `false` | When true, each stage/final text is split by paragraphs (`\n\n`) before QQ length chunking. Default keeps replies as a single message unless the 4500-char limit forces chunking. |
| `listen_group_messages` | `true` | When false, all group events are ignored (no logs, no history, no `~/.config/alma/groups` updates). Private chat continues to work. |
| `respond_to_group_messages` | `true` | When false, the bridge still observes groups but does not reply to `@bot` triggers and does not forward Alma GUI replies back to QQ groups. Forced off when `listen_group_messages` is false. |

## SIGHUP Hot Reload (Unix)

Sending `SIGHUP` to the bridge re-reads the config file and updates the live config without restarting. The following keys are picked up immediately:

- `[alma]` `api`, `model`, `timeout`, `max_retries`, `retry_delay_ms`
- `[onebot]` `api_timeout`, `access_token`
- `[people]` `dir`
- `[chat]` *all keys*

Changing `[bridge].port` or `[database].path` requires a full restart because they are bound at startup. If `alma.api` actually changed, the existing Alma WS client is dropped so the next generation request reconnects against the new base URL.

## Environment Variables

| Variable | Purpose |
|---|---|
| `RUST_LOG` | Standard tracing filter. `RUST_LOG=debug` surfaces protocol-level traces. |
| `BRIDGE_LOG_FILE` | Path to write logs. When set, output is routed to this file with rotation (10 MB × 3 backups). The macOS menu bar app and Windows tray app set this so logs land next to the GUI. |
| `--debugger` (CLI flag) | IDE/debugger mode. Switches `database.path` to a per-process temp file and picks the first available port starting at 18090. |

## Example: `config.toml.example`

```toml
[bridge]
port = 8090

[alma]
api = "http://localhost:23001"
# model = "anthropic:claude-sonnet-4-20250514"
timeout = 120
max_retries = 2
retry_delay_ms = 3000

[database]
path = "bridge-state.db"

[people]
# dir = "~/.config/alma/people"

[onebot]
api_timeout = 30
# access_token = ""

[chat]
group_history_size = 30
# thinking_message = ""
# show_thinking = false
# show_tool_calls = false
segmented_replies = false
listen_group_messages = true
respond_to_group_messages = true
```
